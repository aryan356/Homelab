# Defeating ISP Double NAT (Bell Advanced DMZ)

## The problem

With the ISP Home Hub in its default configuration, the firewall received a
**private** WAN lease from the hub's own DHCP pool:

```
WAN interface : 192.168.2.43   <- private
Public IP     : <WAN_IP>       <- held by the hub
```

Two layers of NAT. Port forwards, inbound VPN and anything else needing a
reachable address would have had to be configured twice — once on the hub and
once on OPNsense — and some protocols simply do not survive it.

## Why not bridge mode

Bell Home Hubs have **no true bridge mode**. The substitute is *Advanced DMZ*,
which hands the real public IP to one designated device over DHCP while the hub
stays in place.

That last part matters: keeping the hub means IPTV and any other ISP services
that depend on it keep working, which a full PPPoE bypass would break.

## Configuration

On the hub at `http://192.168.2.1` → **My Network → Advanced Tools & Settings → DMZ**:

1. DMZ toggle **ON**
2. Select the firewall by its WAN MAC (`<WAN_MAC>`)
3. Tick **Advanced DMZ**
4. Save — the hub reboots

> **Advanced DMZ, not plain DMZ.** Plain DMZ only forwards ports to a device
> that still holds a private address. It does not remove the second NAT layer.

## Renew the lease afterwards

The hub reboots but the firewall keeps its existing private lease, which may not
expire for days. Force a release and renew:

```sh
configctl interface reconfigure wan
```

Without this the change looks like it silently failed.

## Verification

The WAN interface address must equal the address the internet sees. If they
differ, there is still a NAT layer in between.

```sh
WAN=`ifconfig ue0 | grep 'inet ' | awk '{print $2}'`
PUB=`curl -s --max-time 15 https://ifconfig.me`
echo "WAN=$WAN PUBLIC=$PUB"
```

A traceroute still shows the hub as hop 1 — it continues to forward traffic.
That is expected. It is no longer *translating*, which is the part that matters.

## Fragility worth knowing

The DMZ binding is tied to the **MAC address** of the WAN interface.

Replace that NIC and the binding breaks **silently**. There is no error anywhere;
the firewall simply picks up a private lease again and double NAT returns. If a
NIC is ever swapped, re-point the DMZ entry at the new MAC.
