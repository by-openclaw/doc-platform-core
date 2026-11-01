# infra/0015 — Media Network and Multicast Address Plan

**Status:** Draft (proposal — the owner reviews and merges this one)
**Date:** 2026-11-01
**Scope:** How the broadcast media flows (SMPTE ST 2110, MPEG-TS and SRT contribution, NDI, Dante and AES67 audio) and their clock (PTP) share the platform's switching and the new hypervisor cluster, and the multicast address plan recorded in the IPAM next to the unicast one. Builds on `infra/0004` (VLAN registry), `infra/0009` (node class, media hosts), `infra/0014` (routing domains, IPAM of record). Not in scope: NMOS control, audio-over-IP device configuration, the broadcast equipment itself.
**Related:** `infra/0004-network-architecture`, `infra/0009-hypervisor-cluster`, `infra/0014-routing-domains-vips-ipam-sdn`, `services/0003-netbox-cmdb`

---

## Context

The rack already holds the media gear (two EVS Neuron NFR2000 gateways, an Ateme decoder, an Axon control panel) and an Arista fabric for the broadcast plane (7060/7020, point-to-point VLANs 110–121 and the broadcast control VLANs 600–999 reserved in `infra/0004` §2). The owner's direction of 2026-11-01: the new hypervisor cluster must host the DevOps platform **and** media workloads — ST 2110, NDI, Dante, AES67, transport streams and SRT — with one or two nodes carrying NVIDIA ConnectX cards for PTP and 2×100 GbE ST 2110, and the IPAM must cover multicast for 2110 and TS as it covers servers, guests and services.

Today no multicast plan exists anywhere; the media VLAN of `infra/0004` is one flat segment; PTP has no documented domain or grandmaster.

## Decision

### 1. Media segments (rows added to `infra/0004` §3)

| VLAN | Segment | IPv4 | Purpose | IGMP |
|---|---|---|---|---|
| 1310 | MEDIA-RED | 10.1.31.0/24 | ST 2110 path A (ST 2022-7), PTP | snooping + querier, IGMPv3 (SSM) |
| 1320 | MEDIA-BLUE | 10.1.33.0/24 | ST 2110 path B (ST 2022-7), PTP | snooping + querier, IGMPv3 (SSM) |
| 1330 | AUDIO | 10.1.34.0/24 | Dante, AES67, PTP | snooping + querier, IGMPv2/v3 |
| 1340 | NDI | 10.1.35.0/24 | NDI (unicast by default; mDNS discovery) | snooping + querier |
| 1350 | TS | 10.1.36.0/24 | MPEG-TS over UDP/RTP and SRT contribution/distribution | snooping + querier, IGMPv3 |
| 1300 | MEDIA | 10.1.30.0/24 | as today: TV, IPTV, OTT, gaming consumers | unchanged |

The two 2110 paths are physically separate (red and blue leaves, two NIC ports, two VLANs); a flow is sent on both and the receiver reconstructs (ST 2022-7). Test tier mirror: 23x0 / 10.11.3x.

### 2. Multicast address plan (NetBox prefixes with role `multicast`, one per family)

| Prefix | Family | Convention |
|---|---|---|
| 239.1.0.0/16 | ST 2110 path A (red) | third octet = essence: 1 video, 2 audio, 3 ancillary; fourth = flow number; one UDP port per flow (ST 2110-10) |
| 239.2.0.0/16 | ST 2110 path B (blue) | same fourth octet as its red twin |
| 239.10.0.0/16 | TS/SRT contribution (inbound) | per source |
| 239.11.0.0/16 | TS distribution (outbound) | per service |
| 239.69.0.0/16 | AES67 | the AES67 default range, kept |
| 239.255.0.0/16 | Dante | fixed by Dante, cannot move: the AUDIO VLAN is where it lives, so nothing else uses 239.255/16 |
| 239.100.0.0/16 | NDI multicast mode, if ever enabled | NDI defaults to unicast; its optional multicast range would collide with Dante, hence this block |
| 224.0.1.129, 224.0.0.107 | PTP (IEEE 1588 event/general, peer delay) | not allocated by us; recorded as reserved |
| 224.0.0.251 | mDNS (NDI, Dante discovery) | reserved |
| 232.0.0.0/8 | SSM, when a flow is source-specific | recorded as reserved |

Every multicast group in use is a NetBox IP address in its prefix, with the sender, the VLAN and the essence as fields: the same record a unicast service has (`services/0003` §2). Group allocation is a pull request, like a VLAN.

### 3. Clock

- Two PTP profiles coexist: ST 2059-2 for 2110 (domain 127 by convention) and the AES67/Dante media profile (domain 0). One grandmaster serves both when it supports several domains; otherwise one per profile. The grandmaster is a dedicated device (GNSS-disciplined), never a server.
- The Arista leaves run as PTP boundary clocks on the media VLANs; every 2110 host NIC has a hardware clock disciplined by `ptp4l`/`phc2sys` (NVIDIA Rivermax PTP guidelines).
- The platform's own time (`chrony` on every guest) stays NTP; PTP never reaches the platform domain.

### 4. Hosts for media on the cluster

- One or two cluster nodes are **media nodes**: NVIDIA ConnectX-6 Dx or ConnectX-7, two 100 GbE ports (one per 2110 path), hardware timestamping; ST 2022-7 hardware support from ConnectX-6 Dx on (NVIDIA Rivermax). Rivermax is the ST 2110 stack on those cards.
- Media VMs take the NIC through PCI passthrough or SR-IOV virtual functions pinned to the media node; they do not migrate (`infra/0009` §6: real-time media never shares a node's scheduling budget with the platform; pinned cores, huge pages). The platform's guests live on every node, the media guests on the media nodes only.
- The media NIC ports connect to the Arista 7060 (red and blue); the platform fabric (Ceph, migration, tenants) stays on its own 10/25/100 GbE ports and switch.

### 5. Routing and policy

- Media VLANs are routed by nobody for data: flows stay layer 2 within their VLAN; IGMP snooping with a querier per VLAN on the switch; inter-VLAN multicast only if a design asks for it (PIM-SSM on the Arista, never on the firewall).
- Control (NMOS, device web UIs, SRT handshakes) reaches the media VLANs through the firewall from SVC/MGMT only; the media VLANs reach nothing of the platform (`infra/0004` trust table: Media = untrusted).
- TS/SRT to the internet terminates on the DMZ (SRT gateway), never from the media VLAN directly.

## Consequences

- `infra/0004` §3 gains five segments; the trust rows for them are "untrusted, no inter-VLAN except control from SVC/MGMT".
- The BOM of `infra/0009` gains: the media node NICs (ConnectX-6 Dx or -7, 2×100 GbE each, two nodes), the PTP grandmaster, the 100 GbE optics for the 7060 leaves.
- NetBox IPAM gains the multicast prefixes and the group records; the CMDB playbook loads them with the unicast plan.
- The Neuron gateways, decoder and future SDI/2110 edges are NetBox devices with their multicast groups as IP addresses.

## Alternatives considered

- **One flat media VLAN with everything in it.** Rejected: Dante's fixed range collides with NDI's optional one, PTP domains differ, and 2022-7 needs two paths.
- **Routing multicast through the firewall.** Rejected: the firewall is not a multicast router and must not be in the media path.
- **Media on the same nodes as the platform without pinning.** Rejected per `infra/0009` §6.

## CISO mapping

| Framework | Controls covered |
|---|---|
| ISO 27001:2022 | A.8.20 (networks security), A.8.22 (segregation of networks — media isolated from the platform), A.8.9 (configuration management — every group a record), A.8.17 (clock synchronisation — PTP domains documented) |
| NIS2 | Art. 21(2)(e) |

## Revision triggers

- The grandmaster and the media NICs are chosen (BOM).
- The first 2110 flow goes live on the cluster.
- NMOS control is introduced (its own ADR).
