# OPNsense Firewall

Edge firewall and router for the lab.

## Platform

| | |
|---|---|
| Hardware | Lenovo ThinkCentre SFF, AMD Ryzen PRO |
| OS | OPNsense 26.7 (amd64) |
| Base | FreeBSD 15.1-RELEASE-p1 |
| Plugins | none |
| DNS / DHCP | `dnsmasq` (the 26.7 default) |

## Interfaces

| Name | Device | Role | Notes |
|---|---|---|---|
| WAN | `ue0` | Realtek RTL8153, USB gigabit | **FAILED — being replaced** |
| LAN | `re0` | Realtek RTL8111/8168, PCIe gigabit | Onboard |

### The USB NIC failed

**This adapter died and is being replaced with real NICs.** Keeping the notes
here because the failure mode is worth recognising early.

It degraded over a single evening: intermittent link flapping at gigabit, then
flapping even when forced to 100 Mbit, then no link at all. Along the way it
took down DNS, tore out the default route repeatedly, and produced a series of
symptoms that each looked like a separate problem. Full write-up in
[../runbooks/wan-link-flapping.md](../runbooks/wan-link-flapping.md).

The cable was blamed twice before the adapter was properly isolated. The test
that settled it: put the same cable on a desktop PCIe NIC — it held a stable
1 Gbps for 179 MB under load with zero errors, which a damaged cable cannot do.

**Do not use a USB ethernet adapter as the WAN interface on a router.** It is
fine on a laptop dock. On the one device the whole network depends on, running
unattended, it is the weakest component in the build.

### Bus speed gotcha (while a USB adapter is still in use)

A USB adapter must sit in a **USB 3.0** port. In USB 2.0 the ethernet side still
negotiates gigabit while the bus caps at 480 Mbps signalling, giving roughly
310 Mbps of real throughput. The interface reports a healthy gigabit link and
zero errors, so the ceiling is invisible unless the bus is checked:

```sh
usbconfig list | grep -i rtl8153
# SUPER (5.0Gbps) = USB 3.0, good
# HIGH  (480Mbps) = USB 2.0, throttled to ~310 Mbps
```

The 72 mA versus 350 mA power difference between USB 3.0 and 2.0 is normal
negotiation, **not** a fault signal. It was misread as one during diagnosis.

## Firewall posture

- Default inbound deny on WAN
- Block private networks and bogons enabled
- SSH bound to LAN and loopback only, key auth only, passwords disabled

Because the firewall now holds a **public** address directly rather than sitting
behind ISP NAT, every WAN rule added is genuinely internet-facing. That is a
meaningful change in blast radius from the old double-NAT arrangement.

## Wireless: why this box cannot be an access point

The machine has an Intel Wireless-AC 9260 (`iwm0`). It cannot serve as an AP.

FreeBSD's `iwm` driver is station-only for Intel chips — AP mode was never
implemented. The newer `if_iwlwifi` is the same on FreeBSD 15.

```
drivercaps=<STA,PMGT,SHSLOT,SHPREAMBLE,WPA1,WPA2,WME>
#           ^^^ STA present, HOSTAP absent

ifconfig wlan0 create wlandev iwm0 wlanmode hostap
# -> SIOCIFCREATE2: Operation not supported
```

More broadly, FreeBSD AP support is limited to older Atheros `ath` parts and
some Ralink USB chipsets, and tops out at 802.11n — no ac, no ax. Building an
access point out of OPNsense is a dead end regardless of card. The answer is a
dedicated AP on the LAN in bridge mode.
