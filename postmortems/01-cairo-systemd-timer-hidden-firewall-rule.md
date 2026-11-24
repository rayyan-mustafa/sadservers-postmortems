# Cairo: a health check that never logged OK, because of a dormant timer and a hidden firewall rule

| | |
|---|---|
| **Scenario** | [Cairo: "Time for a Timer"](https://sadservers.com/scenario/cairo) |
| **Difficulty** | DIFFICULTY_CAIRO |
| **Skills** | systemd timers, Nginx, curl, iptables / ip6tables |
| **Date solved** | November 2026 |

## Symptom

A health check script, `/opt/scripts/health.sh`, was supposed to run every 10 seconds through a systemd timer. It curls the local Nginx, looks for "Welcome to nginx", and writes `STATUS: OK` or `STATUS: FAILED` to `/var/log/health.log`. Nothing was logging `STATUS: OK`.

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `crontab -l` | Nothing useful. Dead end, see below. |
| 2 | `cat /opt/scripts/health.sh` | The script logic was fine: `curl -s --max-time 2 http://localhost \| grep -q "Welcome to nginx"`. |
| 3 | `systemctl list-timers --all` | `health.timer` was not in the list at all, so it had never been started. |
| 4 | `cat /etc/systemd/system/health.{timer,service}` | Both unit files were valid: `OnBootSec=10`, `OnUnitActiveSec=10`, `Type=oneshot`, `ExecStart=/opt/scripts/health.sh`. |
| 5 | `systemctl status nginx` | Nginx was `active (running)`. |
| 6 | `curl http://localhost` | Printed nothing and hung. |
| 7 | `curl -v http://localhost` | Stuck on `Trying [::1]:80...` and then `Trying 127.0.0.1:80...`. No "Connection refused", no response. |
| 8 | `sudo iptables -L -n -v --line-numbers` | Rule 1 in the OUTPUT chain: `DROP tcp ... 127.0.0.1 tcp dpt:80`, with the comment `The hidden problem (IPv4)`. |
| 9 | `sudo ip6tables -L -n -v --line-numbers` | A matching DROP rule for `::1` port 80. |

**Dead ends**

- **Looking in crontab.** This job was a systemd timer. Cron also cannot schedule anything every 10 seconds, so cron was never going to be the answer.
- **Plain `curl` and `curl -s`.** They showed nothing at all, which told me nothing. Only `curl -v` showed *where* it was stuck.
- **A typo while verifying.** After the fix I ran `grep "Welcome to ngnix"` and got no output, even though curl had downloaded the full 615-byte page. The fix was working; my grep was misspelled.
- **Running out of time.** The first VM hit the 15-minute limit mid-investigation, so I had to start a fresh one and replay the steps quickly.

## Root cause

Two separate faults stacked on top of each other:

1. `health.timer` existed but had never been enabled or started, so the check never ran.
2. iptables and ip6tables had OUTPUT rules that silently dropped traffic to localhost port 80. Even if the timer had run, `curl` would time out after 2 seconds and the script would log `STATUS: FAILED`.

## Fix

```bash
# remove the IPv4 and IPv6 DROP rules (rule 1 in each OUTPUT chain)
sudo iptables -D OUTPUT 1
sudo ip6tables -D OUTPUT 1

# confirm Nginx answers again
curl -s http://localhost | grep "Welcome to nginx"

# start the timer and kick off a first run
sudo chmod +x /opt/scripts/health.sh
sudo systemctl daemon-reload
sudo systemctl enable --now health.timer
sudo systemctl start health.service

# after ~15 seconds there should be fresh STATUS: OK lines
cat /var/log/health.log
```

## Prevention

- Ship timers with `enable --now` in the provisioning script, and alert when an expected timer is missing from `systemctl list-timers`.
- Keep firewall rules in version control (for example an `nftables.conf` in a config repo) so an unexpected DROP rule shows up in review.
- Make health checks log the curl exit code, not just OK/FAILED. Exit code 28 (timeout) points straight at a network block.

## What I learned

If a service is running but `curl -v` hangs on "Trying..." instead of getting "Connection refused", suspect a firewall DROP rule first.
