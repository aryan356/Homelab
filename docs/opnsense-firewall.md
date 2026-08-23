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
| WAN | `ue0` | Realtek RTL8153, USB gigabit | **Must sit in a USB 3.0 port** |
| LAN | `re0` | Realtek RTL8111/8168, PCIe gigabit | Onboard |

### The USB NIC caveat

The WAN interface is a USB dongle rather than a real NIC. It works and currently
sustains ~908 Mbps, but it must be plugged into a **USB 3.0** port.

In a USB 2.0 port the ethernet side still negotiates gigabit while the USB bus
caps at 480 Mbps signalling, which yields roughly 310 Mbps of real throughput.
The interface reports a healthy gigabit link and zero errors, so the ceiling is
invisible unless you check the bus:

```sh
usbconfig list | grep -i rtl8153
# SUPER (5.0Gbps) = USB 3.0, good
# HIGH  (480Mbps) = USB 2.0, throttled
```

A low-profile Intel i210/i350 PCIe card would retire the dongle entirely. Not
urgent while USB 3.0 delivers line rate, but it removes a fragile component from
the one device everything else depends on.

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
