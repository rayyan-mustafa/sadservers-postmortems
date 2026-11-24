# Bergen: freeing a port without breaking the site that was already using it

| | |
|---|---|
| **Scenario** | [Bergen: "Port already in use"](https://sadservers.com/scenario/bergen) |
| **Difficulty** | Easy |
| **Skills** | ss / lsof, systemd, Nginx reverse proxy |
| **Date solved** | November 2026 |

## Symptom

A compiled binary, `/home/admin/standalone`, had to run and print `OK`, but it failed with `bind: address already in use`. The binary could not be modified. At the same time, `curl http://localhost:80` had to keep returning `hello SadServers`.

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `/home/admin/standalone` | It needs port **8000** and the port was taken. |
| 2 | `sudo ss -tulpn \| grep :80` | Only showed Nginx on port 80. Wrong port, see below. |
| 3 | `sudo lsof -i :8000` | `python3` was holding port 8000. |
| 4 | `systemctl status <PID>` | That PID belonged to `django.service` (`/etc/systemd/system/django.service`), running `manage.py runserver 0.0.0.0:8000`. |
| 5 | `sudo grep -rn 8000 /etc/nginx/` | `/etc/nginx/sites-available/bergen` had `proxy_pass http://127.0.0.1:8000;`. So the site on port 80 depended on Django being on 8000. |

**Dead ends**

- **Checking port 80 first.** Port 80 was what the test curled, but the conflict was on 8000. The binary's own error message told me the right port.
- **Killing the python3 PID.** It was tempting, but systemd would restart it, and even if it stayed dead, the site on port 80 would break because Nginx proxies to it.
- **Reading the binary.** `cat` on a compiled Go binary prints garbage. `file` and `strings` can help, but they weren't needed here.

## Root cause

Two programs wanted the same port. `django.service` was listening on `0.0.0.0:8000`, and Nginx proxied port 80 to it. `standalone` was hard-coded to 8000 and could not change. So the fix had to move Django and update Nginx together.

## Fix

```bash
# 1. move Django to 127.0.0.1:8001 (loopback only: Nginx is the only client)
sudo sed -i 's/0.0.0.0:8000/127.0.0.1:8001/' /etc/systemd/system/django.service
sudo systemctl daemon-reload
sudo systemctl restart django.service
sudo ss -tulpn | grep -E '8000|8001'     # python3 on 8001, nothing on 8000

# 2. point Nginx at the new port, test, then reload gracefully
sudo sed -i 's/8000/8001/' /etc/nginx/sites-available/bergen
sudo nginx -t && sudo systemctl reload nginx

# 3. both checks
/home/admin/standalone                   # OK
curl http://localhost:80                 # hello SadServers
```

In a real system I'd use `sudo systemctl edit django.service` (a drop-in override) rather than `sed` on the main unit file, so a package update or redeploy doesn't overwrite the change.

## Prevention

- Bind backends to `127.0.0.1` (or a Unix socket) when only the local reverse proxy talks to them. That stops anyone bypassing Nginx, and it reduces clashes with other services.
- Keep a simple port registry in the repo (which service owns which port).
- In production, run Django behind Gunicorn on a Unix socket, not `manage.py runserver` on a TCP port.

## What I learned

Map who depends on a port (`ss` → `systemctl status <PID>` → `grep` the proxy config) before you kill anything.
