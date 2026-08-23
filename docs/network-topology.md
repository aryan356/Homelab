# Network Topology

## Segments

| Segment | Range | Gateway | Filtered by OPNsense? |
|---|---|---|---|
| ISP LAN | `192.168.2.0/24` | `192.168.2.1` | **No** |
| Firewall LAN | `192.168.1.0/24` | `192.168.1.1` | Yes |
| WAN | public | ISP | n/a |

## Addressing

| Device | Address | Segment |
|---|---|---|
| ISP Home Hub | `192.168.2.1` | ISP LAN |
| TC1 (Proxmox) | `192.168.2.93` | ISP LAN |
| TC2 (Proxmox) | `192.168.2.47` | ISP LAN |
| OPNsense WAN (`ue0`) | `<WAN_IP>` | WAN |
| OPNsense LAN (`re0`) | `192.168.1.1` | Firewall LAN |
| Workstation | `192.168.1.128` | Firewall LAN |

## Traffic paths

Two distinct paths exist, and only one is filtered.

**Filtered** — anything on the firewall LAN:

```
client (192.168.1.x) -> switch -> OPNsense re0 -> NAT -> ue0 -> Home Hub -> internet
```

**Unfiltered** — Proxmox nodes and wireless clients:

```
client (192.168.2.x) -> Home Hub -> internet
```

The second path never touches OPNsense. Firewall rules, logging and DNS
filtering simply do not apply to those hosts.

## Why the cluster ended up on the wrong side

The Proxmox nodes were built before the firewall existed and were addressed
from the ISP router's DHCP pool. Standing up OPNsense created a second, properly
filtered segment but did not move anything onto it.

Migrating them means re-addressing both nodes onto `192.168.1.0/24` and
repointing the cluster configuration, which is why it has not been done
casually — Proxmox cluster membership is tied to node addressing and a careless
change breaks quorum.

## Physical layout

The firewall's WAN port plugs into a LAN port on the ISP Home Hub. It shares
that physical segment with the Proxmox nodes, but receives the **public** IP
via Advanced DMZ rather than a private lease. See
[bell-advanced-dmz.md](bell-advanced-dmz.md).

Downstream, the firewall's LAN port feeds a 10-port unmanaged switch. Because
the switch is unmanaged there is no VLAN capability, so all firewall-side hosts
share one flat broadcast domain.
