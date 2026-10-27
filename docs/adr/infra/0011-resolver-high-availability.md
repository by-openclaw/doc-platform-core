# infra/0011 — Platform Resolver: Two Instances Behind One Address

**Status:** Draft
**Date:** 2026-10-27 (proposed)
**Scope:** How the platform's guests resolve names when the resolver they are handed fails: the number of resolver instances, the address the guests are given, the failover mechanism and its acceptance test. Not in scope: the resolver chain itself (AdGuard Home → the firewall's Unbound → dnscrypt-proxy, owner-stated 2026-05-25), filtering policy, the firewall's DHCP and router-advertisement catalog entries that carry the address.
**Related:** `infra/0004-network-architecture` (OPNsense is the only router, DHCP and DNS authority), `infra/0007-monitoring`, `infra/0010-provisioning-ansible-only`, `security/0003-hardening`

---

## Context

On 2026-10-24 the platform was down for three days. The first cause was one full disk on the single resolver VM (`vm-adguard-01`): Docker does not start on a full disk, so after the first reboot of the day nothing answered on the resolver's address. The platform was rebooted seven times that day; none of the reboots could help.

Every guest already had a fallback. The resolver role writes three nameservers: AdGuard's IPv4 address, AdGuard's IPv6 address, then the firewall's Unbound, which answers every internal name on its own (verified 2026-10-27). The fallback was never reached in time:

| Client | Behaviour with AdGuard dead (measured 2026-10-27) |
|---|---|
| glibc, defaults (`timeout:5 attempts:2`, fallback third) | 20 s per lookup |
| Blackbox probes | 9.5 s = their own timeout, then failure |
| glibc after `ansible-platform` #1040 (fallback second, `timeout:1 attempts:2`) | 1 s, then ≤ 5 ms |
| Docker's embedded DNS, container started before #1040 | 10 s per lookup |
| Docker's embedded DNS, container restarted after #1040 | 5 s per lookup |

Two facts follow from the vendors' documentation, not from the platform:

- `resolv.conf(5)`: the resolver asks the listed servers in order and moves to the next one only after `timeout` seconds, `attempts` times; at most three servers are read.
- Docker's documentation on container networking: name resolution in a container is handled by the embedded DNS server at `127.0.0.11`, which forwards what it cannot answer to the external servers configured for the container, taken from the host at container creation. The per-server wait of that forwarder is not a documented, configurable setting. A platform that depends on it extrapolates.

The services that coordinate a cluster do so by name, as their vendors document it (Patroni's `etcd3.hosts`, etcd's advertised URLs, Redis `replica-announce-ip`). With the resolver's address dead, the PostgreSQL endpoint had no leader from 02:00 to 11:00 UTC and Redis Sentinel named no primary for three days. The cluster did not fail because a member failed: it lost the one address every member needs.

The owner's rule (2026-10-27): no workarounds. Every service is set the way its vendor documents it, aligned to ISO 27001:2022, with no mismatch between what a document says and what runs. The owner approved a second AdGuard instance the same day.

AdGuard Home documents no high-availability mechanism: an instance is a single server with one configuration file. Its configuration on this platform is rendered by `roles/adguard` from the inventory, so a second instance is the same template rendered twice — no synchronisation tool between the two, no copy of one instance's state into the other.

## Decision

### 1. Two resolver instances, one address

The resolver tier is two AdGuard Home instances (`vm-adguard-01`, `vm-adguard-02`), each provisioned by `infra-terraform-proxmox` and configured by `roles/adguard` from the same inventory values. They share one virtual address per family in the services VLAN (IPv4 and IPv6), owned at any moment by exactly one instance through VRRP (RFC 5798) as implemented by keepalived, with a `vrrp_script` that lowers the instance's priority when its own AdGuard stops answering (keepalived.conf(5): `vrrp_script`, `track_script`, `weight`).

The guests are handed the virtual address. A guest never waits on a dead server because the address it uses does not die: when an instance stops, the other owns the address within the VRRP advertisement interval (one second by default).

### 2. What hands the address out

| Channel | Value | Where it is declared |
|---|---|---|
| DHCPv4 option `domain-name-servers` | the IPv4 virtual address | the firewall catalog (`infra/0004`: OPNsense is the DHCP authority) |
| Router advertisement RDNSS | the IPv6 virtual address | the firewall catalog |
| `/etc/resolv.conf` and `systemd-resolved` on managed guests | the virtual address first, the firewall's Unbound second, AdGuard's IPv6 virtual address third | `roles/resolver` |
| Docker's embedded DNS | inherits the host's list at container creation (Docker documentation) | nothing to declare |

The firewall's Unbound stays the second entry: it covers the loss of both instances, and the `timeout:1 attempts:2` options of #1040 stay so that even that case costs about two seconds, as `resolv.conf(5)` describes.

### 3. Placement

The two instances run on two hypervisors as soon as two are available (`br-pmox-03` returns with its replacement network card). Until then both run on `srv-proxmox-poc-01`: that still covers an instance's own failure (full disk, crash, maintenance of one VM), which is what took the platform down, and not the loss of the hypervisor, which a single-node platform cannot cover anyway.

### 4. Acceptance test (the same test every time the tier changes)

With both instances up, stop AdGuard on the instance that owns the address. From a guest host and from a container on that guest, lookups of an internal name and a public name must keep answering with no lookup slower than 100 ms, and the address must move within 3 s. Then stop keepalived on the owner (not only AdGuard) and repeat. Then start everything again and confirm the address returns to the preferred instance. The result is recorded in the pull request that changes the tier.

### 5. What stays as it is

- The chain AdGuard Home → Unbound → dnscrypt-proxy, and the filtering it does.
- Cluster members keep coordinating by name, exactly as their vendors document. With the resolver address highly available there is no reason to depart from the documented configuration.
- `roles/resolver` keeps writing the firewall's Unbound as the fallback: a second address, not a second behaviour.

## Consequences

- One more VM in the services VLAN, provisioned and configured like the first (`infra/0010`: Ansible only; no sync tool, no manual step on the second instance).
- A new `roles/keepalived` (the package and its configuration file, keepalived.conf(5)), used by the resolver tier first; any later service that needs one address for two instances uses the same role.
- The firewall catalog changes its DHCPv4 option and its RDNSS value from an instance address to the virtual address, through the catalog and lib-opnsense, never in the UI (`no manual firewall drift`).
- The monitoring (`infra/0007`) probes both instances by their own addresses and the virtual address; `ResolverDown` fires on the virtual address, `ResolverInstanceDown` (new, warning) on an instance.
- ISO 27001:2022 controls served: A.8.14 (redundancy of information processing facilities), A.8.9 (configuration management: the address every guest uses is declared in one place), A.8.16 (monitoring activities), A.8.6 (capacity management, together with the disk alert tier of `ansible-platform` #1041).
- What this does not cover: both instances failing together (the firewall's Unbound covers resolution; filtering is lost until an instance is back), and the loss of the only hypervisor until a second one carries the second instance.
