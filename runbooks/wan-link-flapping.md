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

## Root cause: FreeBSD/USB specific — not the cable, not the adapter

Both the cable and the USB adapter were **proven good in isolation**. The fault
appears only when that adapter runs on this machine under FreeBSD.

### The evidence

Three tests, same cable and same ISP router port throughout. Only the adapter
and the host operating system changed:

| Test | Link drops / 60s | Speed | Data moved | Errors |
|---|---|---|---|---|
| Cable → desktop PCIe NIC (Windows) | **0** | 1 Gbps stable | 179 MB | **0** |
| Cable → RTL8153 dongle → desktop (Windows) | **0** | 1 Gbps stable | 119 MB | **0** |
| Cable → RTL8153 dongle → firewall (FreeBSD) | Constant | Flapped; needed 100 Mbit forcing | unusable | — |

The identical dongle that was unusable on the firewall moved 119 MB at gigabit
on Windows without a single drop or error, minutes later.

### What that leaves

- FreeBSD's driver for the RTL8153 (`ure` / `cdce`)
- USB power management or autosuspend behaviour on that host under FreeBSD
- Some interaction between the two

Not a dead adapter, and **not the cabling** — which is what two rounds of cable
swapping wrongly suggested.

### Two wrong conclusions, recorded so they are not repeated

**First: "the cable is at fault."** This came from changing two variables at once
— a new cable *and* a different USB port — after which the flapping stopped. It
looked solved; it was coincidence.

**Second: "the adapter has failed."** Reached after the cable tested clean on a
desktop NIC. Correct that the cable was innocent, wrong about the adapter, because
the adapter itself was never tested in isolation until later.

A power-draw difference between USB ports (72 mA on USB 3.0 versus 350 mA on
USB 2.0) was also read as meaningful. It is not — that is normal USB 3.0 versus
2.0 negotiation, and it was a red herring.

The general lesson: **test each component on a known-good host before concluding
anything.** Swapping consumables proves nothing.

## Resolution

**Move the firewall onto hardware with real onboard NICs.** Onboard interfaces
use entirely different drivers and no USB stack, so this failure mode disappears
regardless of which layer is actually responsible.

No configuration fixes it. Forcing 100 Mbit reduced the flapping but never
stopped it, and reseating cables bought minutes at a time.

Broader point: a USB ethernet adapter is fine on a laptop dock. As the
internet-facing interface on a router expected to run unattended, it adds a
driver and a bus that the rest of the build does not need — and when it misbehaves
it takes DNS, routing and every client down with it.

## Lessons

1. **Check the WAN link before anything else.** DNS failures, missing default
   routes and dead clients are all downstream symptoms.
2. **Never count dmesg lines.** Sample `ifconfig` status over a window.
3. **Change one variable at a time.** Changing two together produced a wrong root
   cause that stood for hours.
4. **Move the suspect component, not the consumables.** Swapping cables repeatedly
   proved nothing. Testing the cable *and* the adapter separately on a known-good
   host settled it in two minutes each.
5. **A component that fails in one host is not necessarily broken.** Prove it on
   different hardware before declaring it dead — the driver and the host bus are
   part of the system under test.
