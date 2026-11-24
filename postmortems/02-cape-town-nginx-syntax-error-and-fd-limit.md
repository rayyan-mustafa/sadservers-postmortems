# Cape Town: Nginx would not start, then returned 500s because of a file-descriptor limit

| | |
|---|---|
| **Scenario** | [Cape Town: "Borked Nginx"](https://sadservers.com/scenario/capetown) |
| **Difficulty** | Medium |
| **Skills** | Nginx, systemd unit files, resource limits |
| **Date solved** | November 2026 |

## Symptom

Nginx was down. The goal was for `curl -Is 127.0.0.1:80 | head -1` to return `HTTP/1.1 200 OK`.

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `sudo nginx -t` | `unexpected ";"` on line 1 of `/etc/nginx/sites-enabled/default`. |
| 2 | `sudo nano /etc/nginx/sites-enabled/default` | I couldn't see the bad line. Dead end, see below. |
| 3 | `head -n 5 /etc/nginx/sites-enabled/default \| cat -n` | Showed the stray `;` on line 1 with the line number next to it. |
| 4 | `sudo nano +1 ...`, remove the `;`, then `sudo nginx -t` | `syntax is ok` and `test is successful`. |
| 5 | `sudo systemctl start nginx` and `curl -Is 127.0.0.1:80` | Nginx was running now, but it returned **HTTP 500**. |
| 6 | `sudo tail -n 10 /var/log/nginx/error.log` | `(24: Too many open files)`. |
| 7 | `cat /etc/systemd/system/nginx.service` | A custom unit file overriding the packaged one, with a tiny `LimitNOFILE`. |

**Dead ends**

- **Reading the file in nano.** nano opened scrolled past line 1, so the line with the error was off screen. Printing the lines with `cat -n` made it obvious.
- **Thinking the job was done once Nginx started.** The 500 was a second, unrelated bug. The config fix only got me halfway.

## Root cause

Two faults:

1. A stray `;` on line 1 of the site config stopped Nginx from starting.
2. A custom `/etc/systemd/system/nginx.service` set `LimitNOFILE` so low that Nginx ran out of file descriptors as soon as it served a request. Every connection, log file and static file uses one, so it failed with a 500.

## Fix

```bash
# 1. remove the stray ';' on line 1
sudo nano +1 /etc/nginx/sites-enabled/default
sudo nginx -t

# 2. raise the limit in the unit file (LimitNOFILE=65536) or remove the override
sudo nano /etc/systemd/system/nginx.service
sudo systemctl daemon-reload
sudo systemctl restart nginx

# verify
curl -Is 127.0.0.1:80 | head -1   # HTTP/1.1 200 OK
```

## Prevention

- Run `nginx -t` in CI or in a pre-deploy hook, and only ever `nginx -t && systemctl reload nginx`.
- Don't copy whole unit files into `/etc/systemd/system/`. Use `systemctl edit nginx` drop-ins, so the change is small, visible and reviewable.
- Alert on 5xx rate, not just on "is the process up". Here the process was up and every request still failed.

## What I learned

"Running" is not "working". After the service starts, check the error log before calling it fixed.
