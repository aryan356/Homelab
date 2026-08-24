# Runbook: WAN Link Flapping

Write-up of a real outage that recurred repeatedly over one evening. The root
cause was a failing USB network adapter. Getting there took far longer than it
should have, because three separate diagnostics lie about this particular fault.

## Symptom

Internet dead or wildly intermittent. The WAN interface alternates between
`active` and `no carrier`:

```
ue0: link state changed to DOWN
ue0: link state changed to UP
ue0: 3 link states coalesced
```

Downstream effects that look like separate problems but are not:

- **Unbound returns SERVFAIL** or stops answering — it cannot reach upstream
  resolvers while the link is down
- **The default route disappears** and does not come back
- **Clients report "no internet"** even though the LAN is healthy
- Load average climbs as DHCP and routing reconfigure on every bounce

Chasing any of these individually is wasted effort. Check the WAN link first.

## Trap 1: ping is useless here

The ISP filters ICMP through Advanced DMZ. `ping 1.1.1.1` reports **100% packet
loss even when the connection is perfectly healthy**.

Test with DNS and a real TCP transfer instead:

```sh
drill -Q opnsense.org @47.55.55.55
curl -s --max-time 30 -o /dev/null -w "http=%{http_code}\n" \
  http://cachefly.cachefly.net/100mb.test
```

## Trap 2: the dmesg counter is worthless

`dmesg` is a **rotating ring buffer**. Counting `link state` lines and comparing
across a window produces **negative deltas** when the buffer rotates, which reads
as "not flapping" when the link is in fact flapping badly.

This trap fired twice in one evening and sent the diagnosis down the wrong path
both times. **Do not count dmesg lines.**

## The diagnostic that actually works

Sample interface status directly. This is the only reliable check:

```sh
ssh -o BatchMode=yes root@192.168.1.1 sh <<'SCRIPT'
i=0; up=0; down=0
while [ $i -lt 20 ]; do
  s=`ifconfig ue0 | grep -o 'status: [a-z ]*'`
  echo "$s" | grep -q active && up=`expr $up + 1` || { down=`expr $down + 1`; echo "t+`expr $i \* 3`s DROP"; }
  i=`expr $i + 1`; sleep 3
done
echo "active: $up/20  drops: $down/20"
SCRIPT
```

A healthy link is 20/20. Even **1 drop in 20** is a failing link — that is enough
to tear down the default route and take everything offline.

## Trap 3: intermittent links look fine at the wrong moment

Traffic slips through between drops, so a page that loads, a successful `curl`,
or a DNS answer proves nothing. A test that passes once and fails a minute later
is not a flaky test — it is an intermittent link. Always sample over a window.

## Isolating the fault: test the cable on another NIC

The decisive test. Unplug the WAN cable from the firewall and plug it straight
into a desktop PC, leaving the cable and switch port unchanged. Then check what
the PC's NIC negotiates under load:

```powershell
Get-NetAdapter -Name 'Ethernet' | Select-Object LinkSpeed,MediaConnectionState
Get-NetAdapterStatistics -Name 'Ethernet' |
  Select-Object ReceivedPacketErrors,ReceivedDiscardedPackets,OutboundPacketErrors
```

Gigabit needs all four twisted pairs. **If the cable negotiates a stable
1 Gbps and moves real data with zero errors, the cable is sound** and the fault
is the adapter. Damaged pairs cannot sustain gigabit — they drop to 100 Mbit or
accumulate errors under load.

## Root cause: the USB adapter

**The Realtek RTL8153 USB gigabit adapter was failing.** Not the cable, not the
USB port, not the switch port.

It degraded progressively over one evening:

1. Intermittent flapping at gigabit
2. Flapping even when forced to 100 Mbit
3. No link at all

### The wrong turn — recorded so it is not repeated

The cable was blamed first, and this document previously said so. That conclusion
came from changing **two variables at once** — a new cable *and* a different USB
port — and the flapping stopped. It looked solved.

It was not. The reseat merely perturbed a dying adapter into working briefly.
Every recurrence afterwards was the adapter degrading further.

A power-draw difference between ports (72 mA on USB 3.0 versus 350 mA on USB 2.0)
was also read as meaningful. It is not — that is normal USB 3.0 versus 2.0
negotiation, and it was a red herring.

### The evidence that settled it

Same cable, same ISP router port, same building — only the adapter differed:

| | On the RTL8153 dongle | On a desktop PCIe NIC |
|---|---|---|
| Link drops | Constant | **0 in 60s** |
| Negotiated speed | Flapped; needed 100 Mbit forcing | **Stable 1 Gbps** |
| Data moved | Link unusable | **179 MB** |
| RX / TX errors | — | **0 / 0** |

## Resolution

**Replace the USB adapter with a real NIC.** No configuration fixes a failing
adapter, and forcing 100 Mbit only reduces the flapping rather than stopping it.

A USB dongle is acceptable on a laptop dock. As the internet-facing interface on
a router expected to run unattended, it is the weakest component in the build and
the single point of failure for the entire network.

## Lessons

1. **Check the WAN link before anything else.** DNS failures, missing default
   routes and dead clients are all downstream symptoms.
2. **Never count dmesg lines.** Sample `ifconfig` status over a window.
3. **Change one variable at a time.** Changing two together produced a wrong root
   cause that stood for hours.
4. **Move the suspect component, not the consumables.** Swapping cables repeatedly
   proved nothing. Putting the same cable on a different NIC settled it in 60
   seconds.
