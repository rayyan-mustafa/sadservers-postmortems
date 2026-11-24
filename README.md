# SadServers post-mortems

Write-ups of broken Linux servers I diagnosed and fixed on [SadServers](https://sadservers.com), a site that gives you a real, deliberately broken VM and a timer.

Each entry is a short incident post-mortem: what was broken, the commands I ran and what each one told me (dead ends included), the root cause, the fix, and what would prevent it in production. The point is the diagnosis, not the answer. SadServers publishes official solutions, so these notes focus on how I got there.

## Incidents

| # | Scenario | Difficulty | Skills | Write-up |
|---|---|---|---|---|
| 1 | Cairo: "Time for a Timer" | DIFFICULTY_CAIRO | systemd timers, Nginx, iptables | [Dormant timer + hidden firewall DROP rule](postmortems/01-cairo-systemd-timer-hidden-firewall-rule.md) |
| 2 | Cape Town: "Borked Nginx" | Medium | Nginx, systemd limits | [Syntax error, then 500s from a file-descriptor limit](postmortems/02-cape-town-nginx-syntax-error-and-fd-limit.md) |
| 3 | Manhattan: "Can't write data into database" | Medium | PostgreSQL, disk, mounts | [Postgres down because its data disk was full](postmortems/03-manhattan-postgres-full-data-disk.md) |
| 4 | Bergen: "Port already in use" | DIFFICULTY_BERGEN | ss / lsof, systemd, Nginx proxy | [Freeing a port without breaking the site using it](postmortems/04-bergen-port-conflict-behind-nginx.md) |
| 5 | Alexandria | DIFFICULTY_ALEXANDRIA | cron, bash, lock files | [Silent backup failure: wrong script, no logs, stale lock](postmortems/05-alexandria-silent-cron-backup-stale-lock.md) |

## Patterns so far

- **Running is not working.** In Cape Town and Manhattan, the service looked "up" while every request failed. The service's own log told the truth.
- **Two bugs at once is normal.** Cairo, Cape Town and Alexandria each had more than one fault. Fixing the first one is only halfway.
- **Silence is the enemy.** A DROP rule (no "connection refused") and cron output sent to `/dev/null` both hid the problem. Good systems fail loudly.

## My first five commands on a broken server

```bash
systemctl --failed; systemctl status <service>   # what is down, and what does systemd say
journalctl -u <service> -n 50 --no-pager         # the service's own story
df -h; df -i                                     # disk and inodes, every mount
sudo ss -tulpn                                   # who is listening on which port
curl -v http://localhost                         # what a client actually sees
```

## How I work

I solve each scenario live, against the clock, using an AI assistant the way I'd use a senior engineer sitting next to me: I run the commands and read the output, and ask for a hint when I'm stuck. These write-ups are my own reconstruction of the reasoning, including the parts I got wrong.

New entries follow [TEMPLATE.md](TEMPLATE.md).
