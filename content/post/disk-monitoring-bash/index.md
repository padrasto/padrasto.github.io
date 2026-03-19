---
title: "Disk Monitoring on Linux — Bash Script and Automation"
date: 2026-03-19
tags: [linux, bash, monitoring, sysadmin, automation]
categories: [Disk Monitoring, bash]

---

As a support engineer, disk space issues are one of the most common — and most avoidable — problems you will deal with. A full disk can bring down a web server, corrupt a database, or fill up logs until nothing works. The fix is simple: monitor before it happens.
![nginx server](https://images.unsplash.com/photo-1602469011629-37c6414c006a?w=1200&auto=format&fit=crop)

In this article I will show you how to build a disk monitoring script from scratch, explain every line, and automate it with cron so it runs without you thinking about it.

## Why df -P?

The starting point is `df`, the standard tool for checking disk usage. But raw `df` output can vary between systems — column widths shift, line breaks appear differently. The `-P` flag forces POSIX output format, which is consistent and safe to parse in scripts.

```bash
df -P
```

Output example:

```
Filesystem     1024-blocks    Used Available Capacity Mounted on
/dev/sda1         20511312 8342816  11118412      43% /
tmpfs              4096        0     4096       0% /dev/shm
```

## The Script

```bash
#!/bin/bash
LIMIT=80
alert=0

while read -r filesystem size used avail percent mountpoint
do
    usage=${percent%\%}
    if [ "$usage" -ge "$LIMIT" ]; then
        echo "WARNING: $filesystem is at ${usage}% (limit: ${LIMIT}%)"
        alert=1
    fi
done < <(df -P | tail -n +2)

if [ "$alert" -eq 0 ]; then
    echo "All OK. No partition above ${LIMIT}%."
fi
```

Save it as `/usr/local/bin/check-disk.sh` and make it executable:

```bash
sudo chmod +x /usr/local/bin/check-disk.sh
```

## Breaking It Down

**`LIMIT=80`** — The threshold percentage. Any filesystem at or above this value triggers a warning. Adjust to your needs — production servers often use 85 or 90.

**`while read -r filesystem size used avail percent mountpoint`** — This reads the `df -P` output line by line, assigning each column to a named variable. The `-r` flag prevents backslash interpretation, which keeps the data clean.

**`done < <(df -P | tail -n +2)`** — The `< <(...)` is process substitution. It feeds the output of `df -P | tail -n +2` into the while loop as if it were a file. The `tail -n +2` skips the header line so we only process actual filesystem data.

**`usage=${percent%\%}`** — This is parameter expansion. The `%\%` strips the trailing `%` character from the value (for example, `43%` becomes `43`). This is necessary because bash cannot compare strings like `43%` numerically.

**`[ "$usage" -ge "$LIMIT" ]`** — Numeric comparison in bash. `-ge` means "greater than or equal to". Always quote variables in conditionals to avoid errors when values are empty.

## Testing It

Run it manually first:

```bash
bash /usr/local/bin/check-disk.sh
```

Expected output when everything is fine:

```
All OK. No partition above 80%.
```

To test the alert without filling up your disk, temporarily lower the threshold:

```bash
LIMIT=5 bash /usr/local/bin/check-disk.sh
```

You should see warnings for most partitions.

## Automating with Cron

Once the script works, automate it. Open the crontab:

```bash
crontab -e
```

Run every day at 8am and log the output:

```bash
0 8 * * * /usr/local/bin/check-disk.sh >> /var/log/disk-monitor.log 2>&1
```

Or run every 6 hours for more frequent checks:

```bash
0 */6 * * * /usr/local/bin/check-disk.sh >> /var/log/disk-monitor.log 2>&1
```

## Going Further

This script covers the basics well. From here you can extend it to:

- **Send email alerts** using `mail` or `sendmail` when a threshold is crossed
- **Integrate with monitoring tools** like Nagios, Zabbix, or Prometheus node exporter
- **Log with timestamps** by replacing `echo` with `echo "$(date '+%Y-%m-%d %H:%M:%S') - ..."` for easier debugging

The core pattern — parse `df -P`, strip the `%`, compare numerically — stays the same regardless of how complex you make the alerting.

## Quick Reference

| Command | Purpose |
|---|---|
| `df -P` | Disk usage in POSIX format |
| `tail -n +2` | Skip header line |
| `${var%\%}` | Strip trailing `%` character |
| `while read -r` | Read output line by line |
| `[ -ge ]` | Numeric comparison (greater or equal) |
| `< <(...)` | Process substitution |
| `crontab -e` | Edit cron jobs for current user |
