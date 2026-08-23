# Runbook: WAN Link Flapping

Write-up of a real outage. The root cause turned out to be mundane, but two
diagnostic traps made it take far longer than it should have.

## Symptom

Internet dead. The WAN interface alternated between `active` and `no carrier`,
and the kernel log filled with link transitions:

```
ue0: link state changed to DOWN
ue0: link state changed to UP
ue0: link state changed to DOWN
ue0: 3 link states coalesced
```

Roughly **23 transitions per minute**. Load average climbed as DHCP and routing
reconfigured on every bounce.

## Trap 1: ping is useless on this link

The ISP filters ICMP through Advanced DMZ. `ping 1.1.1.1` reports **100% packet
loss even when the connection is perfectly healthy**.

This sent the initial diagnosis in the wrong direction entirely. Test with DNS
and a real TCP transfer instead:

```sh
drill -Q opnsense.org

curl -s --max-time 30 -o /dev/null \
  -w "http=%{http_code} speed=%{speed_download} B/s\n" \
  http://cachefly.cachefly.net/100mb.test
```

## Trap 2: an intermittent link looks fine at the wrong moment

Traffic slips through between flaps, so a page that loads proves nothing. Count
kernel events over a fixed window and get a rate instead:

```sh
ssh -o BatchMode=yes root@192.168.1.1 sh <<'SCRIPT'
B=`dmesg | grep -cE 'ue0: link state'`
sleep 60
A=`dmesg | grep -cE 'ue0: link state'`
echo "flaps in 60s: `expr $A - $B`"
SCRIPT
```

`dmesg` is a **rotating ring buffer**, so the absolute count can go *down*
between samples. Only the delta over a window means anything.

## Narrowing it down

The USB device stayed enumerated the entire time — no detach or re-attach events.
Only the *ethernet* PHY was bouncing. That ruled out USB power and enumeration
problems and pointed at the RJ45 side: cable, switch port, or the adapter's PHY.

A useful cable test: gigabit needs all four twisted pairs, 100 Mbit needs only
two. If forcing 100 Mbit stabilises the link, a pair is damaged. This is
transient and reverts on reboot:

```sh
ifconfig ue0 media 100baseTX mediaopt full-duplex
ifconfig ue0 media autoselect              # revert
```

## Root cause

**The ethernet cable.**

Cable and USB port were changed at the same time, which initially left the cause
ambiguous — and the USB 3.0 port got the blame, because the adapter drew 72 mA
there versus 350 mA in the USB 2.0 port. That power difference looked meaningful
and was a red herring: it is simply normal USB 3.0 versus 2.0 negotiation.

Isolating the variable afterwards settled it. Back in a USB 3.0 port with the
**new** cable: zero flaps at SUPER speed. The port was never at fault.

| | Old cable, USB 3.0 | New cable, USB 2.0 | New cable, USB 3.0 |
|---|---|---|---|
| USB speed | SUPER 5 Gbps | HIGH 480 Mbps | SUPER 5 Gbps |
| Flaps/min | ~23 | 0 | **0** |
| Throughput | unusable | ~310 Mbps | **908 Mbps** |

## If it happens again

1. **Replace the cable first.** It is the proven cause.
2. Confirm the adapter is on USB 3.0 — `usbconfig list | grep -i rtl8153`
   should report `SUPER (5.0Gbps)`, not `HIGH (480Mbps)`.
3. Check `netstat -i -I ue0` for errors. Zero errors with poor throughput means
   a bandwidth ceiling, not a faulty link.
4. Change one variable at a time and measure the flap rate after each. Changing
   two at once is what made this take an extra hour.
