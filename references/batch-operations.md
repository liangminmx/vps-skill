# Batch VPS Operations

When you need to scan, check, or run a command across all managed VPS, use
`execute_code` with an SSH-alias loop instead of individual `terminal()` calls.
This is faster and keeps the conversation clean.

## Pattern

```python
from hermes_tools import terminal

vps_aliases = ["tanzhen", "jgw", "vmiss", "hk", "dedirock", "61", "zouter-jp"]

for alias in vps_aliases:
    cmd = f"ssh -F /root/.ssh/vps.config {alias} '...remote command...'"
    r = terminal(cmd, timeout=12)
    out = r.get("output", "")
    ec = r.get("exit_code", -1)
    # process output per alias
```

## Key rules

1. **Use `-F /root/.ssh/vps.config`** — the alias config file has the correct ports,
   keys (`~/.ssh/vps`), and users. Don't hardcode IPs/ports/keys.
2. **Keep remote commands simple** — no heredocs, no multi-level quoting. If you
   need complex remote logic, write it as a small script first.
3. **Handle connection failures gracefully** — some VPS may be down. Check for
   "CONNECTED" marker or exit code 255.
4. **Detection filtering** — `grep` on command names will match the grep command
   itself AND `ls` error messages containing the search term. Filter carefully:
   check for actual processes, files, or systemd units, not just substring match.

## Cleanup pattern (systemd services)

When removing a systemd-based service across VPS:

```bash
sudo systemctl stop <service>.service <service>.timer 2>&1
sudo systemctl disable <service>.service <service>.timer 2>&1
sudo pkill -f <process_pattern> 2>/dev/null
sudo rm -rf /opt/<service_dir> /etc/systemd/system/<service>*
sudo systemctl daemon-reload
```

**Pitfall:** `pkill` can kill the SSH connection itself if the pattern matches
sshd or the current session. Run `pkill` last among stop commands, and verify
cleanup in a separate SSH call afterward.

## Example: IP-Sentinel removal (2026-07-16)

Found on JGW only. Services: `ip-sentinel-runner`, `ip-sentinel-agent-daemon`,
`ip-sentinel-report`, `ip-sentinel-updater` + matching timers. Files at
`/opt/ip_sentinel/`. Agent process: `python3 /opt/ip_sentinel/core/webhook.py`.

First `pkill -f ip_sentinel` killed the SSH connection (exit 255), so file
cleanup ran separately in a second call. Use two-step cleanup: stop+kill in
first call, rm+verify in second.