---
title: "Cronjob Not Running? Here's How to Actually Debug It"
date: 2026-03-29
tags: ["linux", "cron", "sysadmin", "troubleshooting"]
categories: ["Linux", "Troubleshooting"]
description: "A practical guide to diagnosing and fixing broken cronjobs on Linux — with real commands and the most common causes."
---

Cronjobs are one of those things that work silently when healthy and fail just as silently when broken. No alerts, no obvious errors — the job simply doesn't run. This article walks through a structured approach to diagnosing broken cronjobs, covering the most common causes and the exact commands to find them.

## The Basics: How Cron Works

Cron is a daemon (`cron` or `crond` depending on the distro) that reads crontab files and executes scheduled commands. Each user can have their own crontab, and there is also a system-wide crontab at `/etc/crontab` and drop-in directories under `/etc/cron.d/`, `/etc/cron.daily/`, etc.

A crontab entry follows this format:

```
# ┌───────── minute (0–59)
# │ ┌───────── hour (0–23)
# │ │ ┌───────── day of month (1–31)
# │ │ │ ┌───────── month (1–12)
# │ │ │ │ ┌───────── day of week (0–7, 0 and 7 = Sunday)
# │ │ │ │ │
  * * * * *  /path/to/command
```

## Step 1: Is the Cron Daemon Running?

Before anything else, confirm the service is active:

```bash
systemctl status cron
```

On Red Hat-based systems, the service may be called `crond`:

```bash
systemctl status crond
```

If it is not running:

```bash
sudo systemctl start cron
sudo systemctl enable cron
```

## Step 2: Check the Logs

This is the most important step. Cron logs its activity, and the logs tell you whether a job was triggered at all — and whether it produced any errors.

On modern systemd-based systems:

```bash
journalctl -u cron --since "1 hour ago"
```

Or search across all logs for cron-related entries:

```bash
journalctl | grep -i cron
```

On older systems or those using syslog:

```bash
grep CRON /var/log/syslog
```

What to look for:

- `(username) CMD (/path/to/script)` — the job was triggered
- `(CRON) error` — something went wrong at the cron level
- No entry at all for the expected time — cron never ran the job

If the job was triggered but still failed, the error is in the command itself, not in cron.

## Step 3: Run the Command Manually

If the job appears in logs but produces no output or wrong results, run the exact command from the crontab manually as the same user:

```bash
sudo -u username /path/to/script.sh
```

This isolates whether the problem is in the script itself or in how cron is calling it.

## Step 4: The Most Common Causes

### 1. Wrong or Relative Paths

Cron runs with a minimal `PATH`. A command that works in your shell may not work in cron because the binary is not in cron's default PATH.

**Bad:**
```
* * * * * backup.sh
```

**Good:**
```
* * * * * /usr/local/bin/backup.sh
```

You can also set PATH explicitly at the top of the crontab:

```
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
```

### 2. Missing Environment Variables

Scripts that rely on environment variables set in `.bashrc` or `.profile` will not have them in cron. Cron does not source those files.

Fix: set the variables directly in the crontab or at the top of the script:

```bash
#!/bin/bash
export MYVAR="value"
# rest of script
```

### 3. Script Not Executable

If the script lacks execute permission, cron will fail silently or log a permission error:

```bash
chmod +x /path/to/script.sh
```

Verify:

```bash
ls -l /path/to/script.sh
```

### 4. No Newline at the End of the Crontab

This is a subtle but real issue. Cron requires the crontab file to end with a newline character. If the last line has no trailing newline, the last job may be silently ignored.

Always edit crontabs with:

```bash
crontab -e
```

And make sure there is an empty line after the last entry.

### 5. Output Is Being Discarded

By default, cron sends job output to the local mail spool. On many systems this goes nowhere visible. To capture output for debugging, redirect it explicitly:

```
* * * * * /path/to/script.sh >> /var/log/myjob.log 2>&1
```

The `2>&1` redirects stderr along with stdout, so you catch errors too.

### 6. Wrong User Permissions

A crontab entry runs as the user who owns it. If the script needs to write to a directory owned by root, it will fail unless the crontab is in root's crontab or the script is run with appropriate permissions.

Check which user's crontab you are editing:

```bash
crontab -l           # your own
sudo crontab -l      # root's
sudo crontab -l -u username   # another user's
```

### 7. Syntax Errors in the Crontab

A single malformed line can prevent the entire crontab from loading. Check for:

- Incorrect number of time fields
- Spaces inside time expressions where commas are expected
- Missing path separators

Use an online crontab validator or test against a known-good entry to isolate the problem.

## Step 5: Test With a Short Interval

When you suspect cron is not running a job at all, add a test entry that fires every minute and writes a timestamp to a file:

```
* * * * * echo "cron alive: $(date)" >> /tmp/cron_test.log
```

Wait two minutes, then check:

```bash
cat /tmp/cron_test.log
```

If entries appear, cron is working and the problem is in your specific job. Remove the test entry when done.

## Quick Reference: Diagnostic Checklist

| Check | Command |
|---|---|
| Cron daemon running? | `systemctl status cron` |
| Job appearing in logs? | `journalctl -u cron --since "1 hour ago"` |
| Script executable? | `ls -l /path/to/script.sh` |
| Script runs manually? | `sudo -u user /path/to/script.sh` |
| PATH issue? | Use absolute paths in crontab |
| Output captured? | Append `>> /var/log/job.log 2>&1` |

## Summary

Most broken cronjobs fail for one of a small set of reasons: the daemon is not running, the path is wrong, the script lacks execute permission, or output is being silently discarded. Start with the logs — `journalctl -u cron` tells you immediately whether the job was triggered at all, which cuts the problem space in half.

Once you have that information, the fix is usually straightforward.
