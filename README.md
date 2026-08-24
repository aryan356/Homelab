# Homelab

Hands-on lab for learning infrastructure, networking and virtualisation.
Started July 2026 and documented as I go — including the things that went wrong,
which are usually the parts worth writing down.

---

## Current state

| Area | Status |
|---|---|
| Proxmox cluster | 2 nodes running, 3rd node staged |
| Edge firewall | OPNsense — single NAT, full gigabit WAN |
| Wireless | Access point on the firewall LAN — clients migrated and filtered |
| ISP radios | Retired by rename — nothing rejoins them |
| Segmentation | **Partial** — cluster still sits upstream of the firewall |

---

## Hardware

| Host | Role | CPU | RAM |
|---|---|---|---|
| TC1 | Proxmox node | i7 (10th gen) | 24 GB |
| TC2 | Proxmox node | Ryzen 5 | 16 GB |
| TC3 | OPNsense firewall | Ryzen PRO | — |
| AP | D-Link DIR-822 wireless access point | — | — |
| NAS | Backups + ISOs (16 TB) | — | — |
| Switch | 10-port, unmanaged | — | — |

All three compute hosts are Lenovo ThinkCentre small-form-factor machines.

---

## Network overview

```
                 Internet  (Bell Aliant fiber)
                     |
        +------------------------------+
        |  ISP Home Hub   192.168.2.1  |
        |  Advanced DMZ -> firewall    |
        +------------------------------+
                     |
     192.168.2.0/24  |  (ISP LAN - NOT firewalled)
       |             |
       |-- TC1  192.168.2.93   Proxmox
       |-- TC2  192.168.2.47   Proxmox
       |
       +-- OPNsense WAN (ue0) = <WAN_IP>   <- public IP via Advanced DMZ
                     |
        +------------------------------+
        |          OPNsense            |
        +------------------------------+
                     |
        OPNsense LAN (re0)  192.168.1.1/24
                     |
             10-port unmanaged switch
                 |               |
                 |               +-- DIR-822 AP  192.168.1.2
                 |                        )))  wireless clients
                 |
                 +-- Workstation  192.168.1.128
```

### The important caveat

The Proxmox nodes still live on `192.168.2.0/24`, which is **upstream of
OPNsense**. Their traffic reaches the internet through the ISP router and never
traverses the firewall.

Wireless used to have the same problem and no longer does. The access point
bridges onto the filtered segment, and the ISP radios were retired so nothing
drifts back — see [wireless-ap.md](docs/wireless-ap.md#retiring-the-isp-wireless).

Moving the cluster is the last remaining piece; see [Roadmap](#roadmap).

---

## Documentation

| Document | What it covers |
|---|---|
| [docs/network-topology.md](docs/network-topology.md) | Addressing, segments, traffic paths |
| [docs/opnsense-firewall.md](docs/opnsense-firewall.md) | Firewall build, hardware, interfaces |
| [docs/bell-advanced-dmz.md](docs/bell-advanced-dmz.md) | Defeating ISP double NAT |
| [docs/wireless-ap.md](docs/wireless-ap.md) | Access point setup and its throughput ceiling |
| [docs/proxmox-cluster.md](docs/proxmox-cluster.md) | Cluster, VMs, quorum lesson |
| [runbooks/opnsense-access.md](runbooks/opnsense-access.md) | Getting a shell, and the csh trap |
| [runbooks/wan-link-flapping.md](runbooks/wan-link-flapping.md) | Diagnosing an unstable WAN link |

---

## Conventions

Sensitive values are redacted with angle-bracket placeholders so these notes
can stay public:

| Placeholder | Meaning |
|---|---|
| `<WAN_IP>` | Public IP assigned by the ISP |
| `<WAN_MAC>` | MAC address of the firewall WAN interface |
| `<SSID>` | Wireless network name |

Private RFC1918 addresses are shown as-is — they are meaningless outside the LAN
and the docs are unreadable without them.

**Never commit** `config.xml` exports or key material; both are covered by
`.gitignore`.

---

## Roadmap

- [x] Add a dedicated access point on the firewall LAN
- [x] Migrate wireless clients off the ISP radios
- [ ] Disable the ISP radios outright — optional now that nothing joins them
- [ ] Move the Proxmox nodes behind OPNsense so cluster traffic is filtered
- [ ] Disable IPv6 router advertisements on the AP (see wireless-ap.md)
- [ ] Replace the unmanaged switch with a managed one and introduce VLANs
- [ ] Replace the AP with a gigabit model to lift the ~95 Mbps wireless cap
- [ ] Bring TC3 into the cluster, or repurpose it now that it runs OPNsense
- [ ] Add a third vote to Proxmox for real quorum (see cluster notes)
- [ ] Complete the OPNsense setup wizard — hostname, timezone and DNS are default
