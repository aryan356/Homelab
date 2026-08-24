# Wireless Access Point (D-Link DIR-822)

Wireless for the firewall LAN. Runs as a **pure access point** — no routing,
no DHCP, no NAT.

## Why a separate device

The firewall has an Intel Wireless-AC 9260 onboard, but it cannot serve as an
access point — FreeBSD's `iwm` driver is station-only. See
[opnsense-firewall.md](opnsense-firewall.md#wireless-why-this-box-cannot-be-an-access-point).

So wireless needs dedicated hardware on the LAN.

## Hardware

| | |
|---|---|
| Model | D-Link DIR-822 |
| Hardware rev | V2 |
| Firmware | 1.00 |
| Radios | AC1200 — 802.11ac dual band |
| **Ethernet** | **4x 10/100 + 1x 10/100 WAN** |
| Address | `192.168.1.2` (static, outside the DHCP pool) |

### The throughput ceiling

"AC1200" describes the radios, not the ports. This model ships with **Fast
Ethernet** only, so every wireless client is capped at roughly **95 Mbps**
regardless of what the 867 Mbps 5 GHz radio negotiates.

The WAN link runs at 908 Mbps, so wireless sees about a tenth of the available
bandwidth. Acceptable for phones, tablets and laptops. Anything that needs real
throughput should be wired.

A gigabit AP (DIR-842, DIR-825, or a UniFi/Omada unit) would remove this cap.

## Configuration

This firmware has **no "Access Point Mode" checkbox**, so it is configured
manually. Settings → Network:

| Setting | Value |
|---|---|
| LAN IP Address | `192.168.1.2` |
| Subnet Mask | `255.255.255.0` |
| DHCP Server | **Disabled** |

Wireless: SSID `<SSID>`, WPA2/WPA3, same name on both bands so clients roam
between them.

No gateway or DNS field exists on that page, and none is needed. With DHCP
disabled the device only bridges frames — OPNsense supplies gateway and DNS to
clients directly.

### Cabling — the part that matters

```
D-Link  LAN port  ---->  10-port switch  ---->  OPNsense re0
D-Link  WAN port  ---->  (empty, permanently)
```

Using the **WAN port** would make it route and NAT, adding a third translation
layer behind the firewall and undoing the single-NAT work. Only LAN ports.

## Expected quirks

**"Internet Disconnected" with a red X on the home page is normal.** That status
reflects the WAN port, which is deliberately empty. The device is bridging
through a LAN port and works fine.

Do **not** click "Click to repair" — it tries to reconfigure the WAN.

**Saving the LAN IP change drops your connection.** Expected. The device moves to
`192.168.1.2`, which is on a different network than the one you configured it
from. It is reachable again once cabled to the switch. Do not factory reset.

## Verification

The real test is not "does WiFi work" but "does wireless traffic cross the
firewall". Two checks:

From the firewall — the AP answers, and wireless clients hold **OPNsense** leases:

```sh
ping -c3 192.168.1.2
arp -an | grep 192.168.1
cat /var/db/dnsmasq.leases
```

From a wireless client — the default route points at the firewall, not the AP:

```
route print -4
# 0.0.0.0  0.0.0.0  192.168.1.1  <wifi address>
```

If a wireless client shows an address from a different subnet, or a gateway that
is not `192.168.1.1`, the AP is still routing.

## Retiring the ISP wireless

Standing up the access point does not by itself remove the unfiltered wireless
path — it only adds a filtered one. Until the ISP router's radios stop being
used, clients can sit on either side of the firewall.

### The SSID collision hazard

The ISP router and the access point were initially given near-identical network
names. That is normally a good idea — clients treat one SSID as one network and
roam to the strongest signal.

Here it is actively dangerous. The two networks sit on **opposite sides of the
firewall**. Identical SSIDs mean devices silently drift between filtered and
unfiltered depending on signal strength, with nothing in the UI to indicate
which side they are on. The lab would look correctly segmented while half the
traffic bypassed it.

**Keep the two names clearly distinct**, or remove one entirely.

### The Giga Hub has no radio switch where you expect it

On the ISP router, **Manage my Wi-Fi** exposes only SSID, password, and a
**Whole Home Wi-Fi** toggle. That toggle controls mesh and band-steering
optimisation — it is **not** a radio on/off switch. Turning it off leaves the
radios broadcasting.

Per-band radio control lives under **Advanced tools and settings → Wi-Fi**, and
some firmware revisions only expose it through the ISP's mobile app.

### What was done instead

The ISP SSID was **renamed**. This turned out to be simpler and just as
effective:

- Every wireless client immediately dropped, because the network they had saved
  no longer exists
- Reconnecting means choosing the access point's SSID, so devices migrate
  without being configured individually
- Nothing can accidentally rejoin the unfiltered network later

Radios still broadcast, but with a name nothing has saved and a password nothing
knows. Disabling them outright is tidier and still worth doing, but it is no
longer load-bearing.

### Do not power down the ISP router

It must keep running. The firewall's WAN plugs into it and receives the public
IP through it via Advanced DMZ — see
[bell-advanced-dmz.md](bell-advanced-dmz.md). Only the radios are being retired,
never the device.

### Verifying the migration

Count clients holding leases from the firewall, not from the ISP router:

```sh
cat /var/db/dnsmasq.leases
arp -an | grep 192.168.1
```

Cross-check against the device list in the ISP router's UI. Anything still
listed there as *wireless* has not migrated yet. Expect a gap — wired hosts and
sleeping devices both show up there and neither is a problem.

## Known issue: IPv6 router advertisements

The DIR-822 continues sending IPv6 Router Advertisements, claiming to be an IPv6
router while having no IPv6 upstream — its WAN port is empty.

IPv4 is unaffected. But clients may attempt IPv6 through that dead end and wait
for a timeout, which presents as sites being slow to start loading.

Fix: Settings → Network → IPv6, set to *Link-Local Only* or disable IPv6, so
OPNsense handles IPv6 for the whole LAN as it already does for wired hosts.
