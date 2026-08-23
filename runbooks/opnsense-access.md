# Runbook: OPNsense Shell Access

## Connecting

SSH to the firewall at `192.168.1.1` as `root`, using key authentication.

- Password authentication is **disabled** — key only
- SSH listens on **LAN and loopback only**, never WAN
- Verify the host key fingerprint on first connect against the console

```sh
ssh root@192.168.1.1
```

## The csh trap

Read this before scripting anything against the firewall.

root's login shell is FreeBSD **`csh`**, not `sh`. SSH hands your command string
to that login shell, and csh parses it *before* anything else runs. It chokes on
`$(...)` and on bare `$`.

Both of these fail:

```sh
ssh root@192.168.1.1 'echo $(whoami)'
# -> Illegal variable name.

ssh root@192.168.1.1 'sh -c "echo $(whoami)"'
# -> Variable name must contain alphanumeric characters.
```

The second is the surprising one. Wrapping in `sh -c` does not help, because csh
has already mangled the string before `sh` ever sees it.

### The pattern that works

Pipe the script over **stdin** so `sh` receives it untouched:

```sh
ssh -o BatchMode=yes root@192.168.1.1 sh <<'SCRIPT'
echo "user is `whoami`"
ifconfig ue0 | grep status
SCRIPT
```

Inside the heredoc, use backticks rather than `$(...)`.

## Useful commands

```sh
# Version and uptime
opnsense-version
uptime

# Interfaces
ifconfig -a
netstat -rn -f inet

# Force a WAN DHCP release and renew
configctl interface reconfigure wan

# Running services
service -e

# Config backups
ls /conf/backup
```

> `config.xml` contains credentials. Never copy it into this repo — it is
> covered by `.gitignore`.
