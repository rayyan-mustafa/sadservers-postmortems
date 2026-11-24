# Manhattan: Postgres refused writes because its data disk was 100% full

| | |
|---|---|
| **Scenario** | [Manhattan: "Can't write data into database"](https://sadservers.com/scenario/manhattan) |
| **Difficulty** | Medium |
| **Skills** | PostgreSQL, disk space, mounts, logs |
| **Date solved** | November 2026 |

## Symptom

An insert into a Postgres database failed:

```bash
sudo -u postgres psql -d dt -c "insert into persons(name) values ('jane smith');"
```

The scenario hinted that the problem was not Postgres-specific and pointed at `data_directory`.

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `systemctl status postgresql` | `active (exited)`. That looked fine but was misleading, see below. |
| 2 | `tail -n 20 /var/log/postgresql/postgresql-14-main.log` | `FATAL: could not create lock file "postmaster.pid": No space left on device`. The actual Postgres server had never started. |
| 3 | `grep data_directory /etc/postgresql/14/main/postgresql.conf` | Data lives in `/opt/pgdata`. |
| 4 | `df -h` | `/dev/nvme0n1` mounted on `/opt/pgdata` was **100% full** (28K left). The root disk `/` still had 6.1G free. |
| 5 | `df -i` | Inodes were fine, so it was a space problem, not a file-count problem. |
| 6 | `du -sh /opt/pgdata/* \| sort -h` | `file1.bk` 7.0G, `file2.bk` 923M, `file3.bk` 488K. The real database (`main`) was only 50M. |

**Dead ends**

- **Trusting `systemctl status postgresql`.** On Debian, `postgresql.service` is a wrapper that only launches the per-cluster unit (`postgresql@14-main`). It shows `active (exited)` even when the real database is down. The log file was the real source of truth.
- **Looking at `/`.** The root disk had plenty of space. The full disk was a separate mount, which is why `df -h` with every mount listed mattered.

## Root cause

Old backup files (`*.bk`, about 8 GB) had been left inside the Postgres data mount. They filled it completely, so Postgres could not even create its `postmaster.pid` lock file and never started.

## Fix

```bash
# remove the junk, leave the real data ("main") alone
rm -f /opt/pgdata/file*.bk /opt/pgdata/deleteme
df -h /opt/pgdata            # back down to about 1% used

sudo systemctl restart postgresql
sudo -u postgres psql -d dt -c "insert into persons(name) values ('jane smith');"
# INSERT 0 1
```

If `du` had added up to much less than `df`, the next step would have been `lsof +L1`. It finds deleted files that a process still holds open, which keep using space until that process restarts.

## Prevention

- Alert on disk usage **per mount** (for example at 80% and 90%), not only on `/`.
- Never store backups on the same volume as the live database. Send them to separate storage with a retention policy.
- Monitor that Postgres actually accepts connections (`pg_isready`), not just that the systemd wrapper is "active".

## What I learned

Read the service's own log before trusting `systemctl status`, and always check `df -h` for every mount.
