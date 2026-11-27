# Jakarta: "it's always DNS", but not the DNS I was checking

| | |
|---|---|
| **Scenario** | [Jakarta: "It's always DNS"](https://sadservers.com/scenario/jakarta) |
| **Difficulty** | Hard |
| **Skills** | Name resolution, `/etc/nsswitch.conf`, `getent`, systemd-resolved |
| **Date solved** | Nov 2026 |

## Symptom

`ping google.com` failed with "ping: google.com: Name or service not known". The task was to fix the DNS configuration so the name resolves. This was a free account, which cannot ping outside the VM, so SadServers has no automated check for this scenario. "Check My Solution" says it is not implemented here.

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `nslookup` | "not found". The tool is not installed on this minimal Debian, and `dig` is missing too. So the usual DNS tools were not an option. |
| 2 | `getent hosts google.com` | A resolver test that needs no extra package. It goes through the same lookup path as `ping`, which is more useful here than `nslookup`. |
| 3 | `cat /etc/resolv.conf` | The nameserver settings looked fine. |
| 4 | `systemctl status systemd-resolved` | The service was healthy. |
| 5 | `grep '^hosts:' /etc/nsswitch.conf` | `hosts: files`. There is no `dns` in the list. |

**Dead ends**

- **`nslookup` and `dig`.** Neither is installed, and they query DNS directly anyway. They would not have shown what `ping` was doing.
- **`/etc/resolv.conf` and systemd-resolved.** Both looked healthy. They were red herrings: a working resolver is no use if the system is never told to ask it.
- **Pinging `8.8.8.8` to test connectivity.** Free accounts cannot reach outside the VM, so this told me nothing.

## Root cause

`/etc/nsswitch.conf` had `hosts: files` and no `dns`. That line tells the C library which sources to use for host lookups, and in what order. With only `files`, it checked `/etc/hosts` and stopped. It never asked a DNS server at all, which is why a correct `resolv.conf` made no difference.

## Fix

```bash
sudo sed -i 's/^hosts:.*/hosts: files dns/' /etc/nsswitch.conf
getent hosts google.com ; ping -c 1 google.com
```

How I verified: I got the IP address back in the terminal. Because there is no automated check on a free account, I then opened the official clue and compared. It matched: edit `/etc/nsswitch.conf` and add `dns` to the `hosts:` line after `files`.

## Prevention

- Keep `/etc/nsswitch.conf` under configuration management and alert on drift, since a one-word change silently breaks every hostname lookup.
- Test name resolution the way applications do it, with `getent hosts <name>`, not only with `nslookup` or `dig`. Those query DNS directly and can pass while the system's own lookup path is broken.
- Bake `dnsutils` (or `bind-utils`) into debug images so the tools are there when you need them.

## What I learned

When `resolv.conf` is right and lookups still fail, check `nsswitch.conf`: it decides which sources get asked at all.
