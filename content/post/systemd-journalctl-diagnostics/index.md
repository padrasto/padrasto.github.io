---
title: "Diagnosing Failing Services with systemd and journalctl"
date: 2026-03-30
draft: false
tags: ["linux", "systemd", "journalctl", "troubleshooting", "sysadmin"]
categories: ["Linux , journalctl"]
description: "How to read what systemd and journalctl are telling you, so you can fix the root cause instead of the symptom."
---

When a service fails on a Linux system, the instinct is often to restart it and move on.
That works until it doesn't. This article covers how to actually read what systemd and
journalctl are telling you, so you can fix the root cause instead of the symptom.

## Start with systemctl status

The first command after any service failure:

```bash
systemctl status servicename
```

This gives you three things immediately: the current state (active, failed, activating),
the last few log lines, and the PID. Most of the time, the error is already visible here.

Pay attention to the state codes:

- `failed (Result: exit-code)` — the process started and crashed
- `failed (Result: timeout)` — the process never became ready in time
- `activating` stuck indefinitely — usually a missing dependency or a blocked socket

## Read the logs with journalctl

For the full picture:

```bash
journalctl -u servicename
```

By default this shows all logs since boot. More useful in practice:

```bash
journalctl -u servicename -n 50 --no-pager
```

This shows the last 50 lines without piping through less. Fast and readable.

If the service keeps crashing in a loop:

```bash
journalctl -u servicename -f
```

This follows the log in real time — equivalent to `tail -f` but for systemd units.

## Check what happened at a specific boot

If the service was working yesterday and broke overnight:

```bash
journalctl --list-boots
```

Pick the boot you want and pass it with `-b`:

```bash
journalctl -b -1 -u servicename
```

`-b -1` means the previous boot. `-b -2` is two boots ago. Useful after unexpected reboots.

## Filter by priority

Logs contain a lot of noise. To see only errors and critical messages:

```bash
journalctl -u servicename -p err
```

Priority levels: `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug`.
In practice, `err` and `warning` are where you spend most of your time.

## Check dependencies and ordering

A service can fail not because of its own configuration, but because something it depends
on is not ready. Check the unit file:

```bash
systemctl cat servicename
```

Look at `After=`, `Requires=`, and `Wants=`. If a dependency is missing or failed,
your service will fail silently or with a misleading error.

To see the full dependency tree:

```bash
systemctl list-dependencies servicename
```

## The pattern in practice

A real diagnostic session usually looks like this:

1. `systemctl status servicename` — get the immediate error
2. `journalctl -u servicename -n 100 --no-pager` — read the full recent log
3. `systemctl cat servicename` — check the unit file for misconfigurations
4. `journalctl -b -1 -u servicename` — if the issue started after a reboot
5. Fix, then `systemctl daemon-reload && systemctl restart servicename`

The `daemon-reload` step is often skipped and causes confusion — if you edit a unit file,
systemd does not pick up the changes until you reload.

## Common mistakes

**Restarting without reading logs.** If you restart a failed service without understanding
why it failed, you are just resetting a countdown to the next failure.

**Forgetting daemon-reload.** Edit a unit file, skip the reload, restart the service,
see the old behaviour, conclude the edit did nothing. Always reload after editing unit files.

**Ignoring the state code.** `exit-code` and `timeout` point to completely different
problems. Reading the state before diving into logs saves time.

---

systemd and journalctl give you enough information to diagnose almost any service failure
without guessing. The skill is knowing where to look and in what order.
