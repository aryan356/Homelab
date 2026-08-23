# Proxmox Cluster

## Nodes

| Node | Address | CPU | RAM | Status |
|---|---|---|---|---|
| TC1 | `192.168.2.93` | i7 (10th gen) | 24 GB | Active |
| TC2 | `192.168.2.47` | Ryzen 5 | 16 GB | Active |
| TC3 | — | Ryzen PRO | — | Repurposed as the OPNsense firewall |

Proxmox VE 9.2.2, cluster name `Homelab`.

## Virtual machines

| VM | OS | Host | Storage |
|---|---|---|---|
| 100 | Ubuntu Server 24.04 | TC1 | local-lvm |
| 101 | Ubuntu Server 24.04 | TC1 | local-lvm |

A 16 TB NAS holds backups and ISOs. It is deliberately **not** used for VM
disks — running VM disks over the network would make every guest dependent on
NAS availability and the link to it.

## Lesson: two nodes is not redundancy

The cluster was built with two nodes on the assumption that either could
survive the other failing. That is not how Proxmox works.

Proxmox uses quorum, and each node gets one vote. Two nodes means two votes, so
a majority requires both. Lose either one and the survivor **loses quorum** and
stops making cluster decisions — the exact opposite of the intended outcome.

Options for fixing it:

- Add a third voting node
- Add a QDevice — a lightweight external vote, which can run on something small
  rather than a full third host

Originally TC3 was going to be the third node, but it became the OPNsense
firewall instead, so this remains open.

## Open items

- [ ] Restore real quorum with a third vote (node or QDevice)
- [ ] Migrate both nodes onto the firewall LAN — see
      [network-topology.md](network-topology.md). They currently sit upstream of
      the firewall and are unfiltered.
- [ ] Document the backup schedule to the NAS
