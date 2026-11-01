# Restore drills

One file per drill, written by `ansible-platform` `playbooks/restore-drill.yml` (infra/0008 §Restore
testing, decision 10: monthly; ISO 27001:2022 A.8.13, NIS2 Art. 21(2)(c)). A drill restores one guest
from the PBS datastore to a spare VMID with no network device, waits for a running system, times it and
destroys the guest. The file is the evidence; a FAIL is an incident, never a retry.

| Date | Guest | Result | Restore | To a running system |
|---|---|---|---|---|
| 2026-11-01 | lxc-pgadmin-01 (container, 5.06 GB archive, off-site datastore) | PASS | 905 s | 31 s |

Reading the numbers: the restore time is the read of the archive from the off-site provider (about
5.6 MB/s for this drill); a larger guest scales with its size, which is why the NAS copy stays the
fast restore path on site and the off-site datastore is the copy that survives the site.
