# Alexandria: nightly backups silently stopped, from three small bugs stacked together

| | |
|---|---|
| **Scenario** | [Alexandria](https://sadservers.com/scenario/alexandria) |
| **Difficulty** | DIFFICULTY_ALEXANDRIA |
| **Skills** | cron, bash, lock files, logging |
| **Date solved** | November 2026 |

## Symptom

An automated backup job (`/opt/backup/backup.sh`) was supposed to drop a timestamped archive into `/var/backups/daily/`. No new backups were appearing, and there were no errors anywhere. The task was to fix *all* the issues.

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `crontab -l`, `sudo crontab -l`, `cat /etc/crontab`, `ls /etc/cron.d/`, `systemctl list-timers --all` | Checked every place a scheduled job can live. Found the job in a crontab. |
| 2 | (the crontab line) | It called **`/opt/backup/old_backup.sh`**, the wrong script. |
| 3 | (same line) | Output went to `> /dev/null 2>&1`, and `MAILTO="broken@nonexistent.local"`. Every error was being thrown away. |
| 4 | `ls -la /opt/backup/backup.sh` | Permissions were fine (`-rwxr-xr-x`). |
| 5 | `cat /opt/backup/backup.sh` | `LOCK_FILE="/opt/backup/backup.lock"`, a guard that exits if the lock exists, and `rm -f` of the lock only on the last line. |
| 6 | `ls -la /opt/backup/` | A stale `backup.lock` was sitting there. |

**Dead ends**

- **"The script looks fine."** The syntax and permissions were fine, so at first I thought the script wasn't the problem. The bug was in the script's logic, not its syntax. The comment `# FIX: Moved lock file to a persistent location` was the clue.
- **Time pressure.** I had about 3 minutes left by the time I found the lock file. Checking every scheduler location first (step 1) is what saved time overall.

## Root cause

Three faults, each enough to stop backups on its own:

1. Cron pointed at an old script name.
2. All output and error mail went nowhere, so nobody could see it failing.
3. The lock file had been moved to a persistent path and was only removed on a clean exit. One crash or reboot mid-backup left a stale lock, and every later run exited immediately with "Backup already running".

## Fix

```bash
# 1. point cron at the right script and keep its output
sudo crontab -e
#   MAILTO=root
#   */5 * * * * /opt/backup/backup.sh >> /var/log/backup.log 2>&1

# 2. clear the stale lock
sudo rm -f /opt/backup/backup.lock

# 3. run it once by hand and confirm an archive appears
sudo /opt/backup/backup.sh
ls -la /var/backups/daily/
```

## Prevention

- Use `flock` instead of a hand-made lock file: `*/5 * * * * flock -n /run/backup.lock /opt/backup/backup.sh`. The kernel releases the lock when the process dies.
- If a lock file is unavoidable, put it in `/run` (cleared on reboot) and remove it with `trap 'rm -f "$LOCK_FILE"' EXIT`.
- Never send cron output to `/dev/null`. Log it, and alert when the newest backup is older than expected ("dead man's switch" monitoring).

## What I learned

A backup that fails silently is worse than one that fails loudly. Always check where cron output and errors actually go.
