# Network Topology

## Segments

| Segment | Range | Gateway | Filtered by OPNsense? |
|---|---|---|---|
| ISP LAN | `192.168.2.0/24` | `192.168.2.1` | **No** |
| Firewall LAN | `192.168.1.0/24` | `192.168.1.1` | Yes |
| WAN | public | ISP | n/a |

Wireless clients join the **firewall LAN** — the access point bridges onto
`192.168.1.0/24` rather than creating a segment of its own.

## Addressing

| Device | Address | Segment |
|---|---|---|
| ISP Home Hub | `192.168.2.1` | ISP LAN |
| TC1 (Proxmox) | `192.168.2.93` | ISP LAN |
| TC2 (Proxmox) | `192.168.2.47` | ISP LAN |
| OPNsense WAN (`ue0`) | `<WAN_IP>` | WAN |
| OPNsense LAN (`re0`) | `192.168.1.1` | Firewall LAN |
| DIR-822 access point | `192.168.1.2` | Firewall LAN |
| Workstation | `192.168.1.128` | Firewall LAN |
| DHCP pool | `192.168.1.100` - `.199` | Firewall LAN |

Static assignments live outside the pool — `.2` to `.99` and `.200` to `.254`.

## Traffic paths

Two paths exist, and only one is filtered.

**Filtered** — wired and wireless hosts on the firewall LAN:

```
client (192.168.1.x) -> switch -> OPNsense re0 -> NAT -> ue0 -> Home Hub -> internet
                          ^
                      DIR-822 AP bridges wireless onto this segment
```

**Unfiltered** — the Proxmox nodes:

```
TC1 / TC2 (192.168.2.x) -> Home Hub -> internet
```

The second path never touches OPNsense. Firewall rules, logging and DNS
filtering do not apply to those hosts.

## Why the cluster is still on the wrong side

The Proxmox nodes were built before the firewall existed and were addressed from
the ISP router's DHCP pool. Standing up OPNsense created a properly filtered
segment but did not move anything onto it.

Wireless has since been migrated — see [wireless-ap.md](wireless-ap.md). The
cluster has not, because Proxmox cluster membership is tied to node addressing
and re-addressing carelessly breaks quorum. It needs a planned change, not an
opportunistic one.

## Physical layout

The firewall's WAN port plugs into a LAN port on the ISP Home Hub, sharing that
physical segment with the Proxmox nodes but receiving the **public** IP via
Advanced DMZ rather than a private lease. See
[bell-advanced-dmz.md](bell-advanced-dmz.md).

Downstream, the firewall's LAN port feeds a 10-port unmanaged switch, which
carries the workstation and the access point. Because the switch is unmanaged
there is no VLAN capability, so all firewall-side hosts — wired and wireless —
share one flat broadcast domain.

Segmenting guests or IoT onto their own networks would need a managed switch and
an AP that supports multiple SSIDs mapped to VLANs.
