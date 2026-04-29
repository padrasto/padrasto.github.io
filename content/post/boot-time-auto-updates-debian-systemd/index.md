---
title: "Boot-time Auto-Updates on Debian 13: apt + Flatpak with systemd Timers"
description: "Step-by-step tutorial for running apt and user-level Flatpak updates automatically, 30 seconds after every boot."
date: 2026-04-29
categories:
  - Linux
  - Debian
  - Sysadmin
tags:
  - systemd
  - debian
  - bash
  - automation
  - flatpak
slug: boot-time-auto-updates-debian-systemd
---

Goal: refresh apt packages and user Flatpaks automatically, 30 seconds after every boot, no interaction. apt runs as root via a system service; Flatpak runs as my user via a user service. Both wired to systemd timers.

Tested on Debian 13 (trixie), kernel 6.12, GNOME 48.

## 1. Create the apt update script

```bash
sudo nano /usr/local/sbin/auto-update.sh
```

Paste:

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG="/var/log/auto-update.log"
exec >>"$LOG" 2>&1

echo "===== $(date -Iseconds) start ====="

# Wait for network up to ~60s
for i in {1..12}; do
    if getent hosts deb.debian.org >/dev/null 2>&1; then
        echo "[net] DNS resolved after $((i*5))s"
        break
    fi
    sleep 5
done

export DEBIAN_FRONTEND=noninteractive

echo "[apt] update"
apt-get update

echo "[apt] upgrade"
apt-get -y \
    -o Dpkg::Options::="--force-confdef" \
    -o Dpkg::Options::="--force-confold" \
    upgrade

echo "[apt] autoremove"
apt-get -y autoremove --purge
echo "[apt] autoclean"
apt-get -y autoclean

HELD=$(apt-mark showhold)
if [[ -n "$HELD" ]]; then
    echo "[warn] packages held back:"
    echo "$HELD"
fi

if [[ -f /var/run/reboot-required ]]; then
    echo "[warn] reboot required"
    [[ -f /var/run/reboot-required.pkgs ]] && cat /var/run/reboot-required.pkgs
fi

echo "===== $(date -Iseconds) end ====="
echo
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

## 2. Set permissions and create the log file

```bash
sudo chown root:root /usr/local/sbin/auto-update.sh
sudo chmod 750 /usr/local/sbin/auto-update.sh
sudo touch /var/log/auto-update.log
sudo chown root:adm /var/log/auto-update.log
sudo chmod 640 /var/log/auto-update.log
```

## 3. Test the script manually

```bash
sudo /usr/local/sbin/auto-update.sh
sudo cat /var/log/auto-update.log
```

You should see a `===== start =====` ... `===== end =====` block.

## 4. Create the apt systemd service

```bash
sudo nano /etc/systemd/system/auto-update.service
```

Paste:

```ini
[Unit]
Description=Auto update Debian (apt)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/auto-update.sh
Nice=10
IOSchedulingClass=idle
TimeoutStartSec=30min
```

Save and exit.

## 5. Create the apt systemd timer

```bash
sudo nano /etc/systemd/system/auto-update.timer
```

Paste:

```ini
[Unit]
Description=Run auto-update 30s after boot

[Timer]
OnBootSec=30s
Unit=auto-update.service
Persistent=false

[Install]
WantedBy=timers.target
```

Save and exit.

## 6. Activate the apt timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now auto-update.timer
```

Verify:

```bash
systemctl list-timers auto-update.timer
systemctl status auto-update.timer
```

Should report `active (waiting)`.

## 7. Create the Flatpak user service

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/flatpak-user-update.service
```

Paste:

```ini
[Unit]
Description=Auto update user Flatpaks
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/flatpak update -y --noninteractive --user
ExecStartPost=/usr/bin/flatpak uninstall --unused -y --noninteractive --user
```

Save and exit.

## 8. Create the Flatpak user timer

```bash
nano ~/.config/systemd/user/flatpak-user-update.timer
```

Paste:

```ini
[Unit]
Description=Run user Flatpak update 30s after boot

[Timer]
OnBootSec=30s
Unit=flatpak-user-update.service
Persistent=false

[Install]
WantedBy=timers.target
```

Save and exit.

## 9. Enable linger and activate the user timer

`enable-linger` lets the user systemd manager start at boot, even without an interactive login. Without it, a user timer with `OnBootSec=30s` would silently miss the window.

```bash
systemctl --user daemon-reload
systemctl --user enable --now flatpak-user-update.timer
sudo loginctl enable-linger "$USER"
```

Verify:

```bash
loginctl show-user "$USER" | grep Linger
# Linger=yes

systemctl --user list-timers flatpak-user-update.timer
systemctl --user status flatpak-user-update.service
```

## 10. Reboot and confirm

```bash
sudo reboot
```

After login, wait ~1 minute, then:

```bash
sudo tail -20 /var/log/auto-update.log
journalctl --user -u flatpak-user-update.service -n 20 --no-pager
```

Both should show entries timestamped after the boot. Done — the system updates itself from now on.

## Useful commands afterwards

```bash
# All system timers, with next/last run
systemctl list-timers --all

# All user timers, with next/last run
systemctl --user list-timers --all

# Force a manual run without rebooting
sudo systemctl start auto-update.service
systemctl --user start flatpak-user-update.service

# Watch the apt log live
sudo tail -f /var/log/auto-update.log
```
