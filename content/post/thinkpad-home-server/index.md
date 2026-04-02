---
title: "ThinkPad T480 as a Home Server with CasaOS"
description: "Jellyfin, Immich, battery protection, lid closed — done in an afternoon."
date: 2026-04-02
categories:
    - Self-hosting
    - Linux
tags:
    - thinkpad
    - debian
    - casaos
    - homelab
    - jellyfin
    - immich
    - tlp
---

I had a ThinkPad T480 collecting dust. Instead of leaving it that way, I turned it into a home server running **CasaOS** — a clean web dashboard for managing self-hosted apps in containers. The goal: **Jellyfin** for media streaming and **Immich** for photo backup, fully self-hosted and under my control.

Along the way I also dealt with three problems that come up whenever you use a laptop as a server: protecting the battery, keeping the machine running with the lid closed, and blanking the screen automatically so it doesn't wear out.

Here's the full guide, step by step.

---

## Step 01 — Install CasaOS

CasaOS is an open-source home cloud system that runs on top of Debian (or Ubuntu) and provides a clean web panel for installing and managing containerised apps. Installation is a single command:

```bash
curl -fsSL https://get.casaos.io | sudo bash
```

After installation, the dashboard is available in your browser at the server's local IP:

```
http://<laptop-ip>
```

From there, install **Jellyfin** and **Immich** directly from CasaOS's built-in App Store — no manual Docker work needed.

---

## Step 02 — Static IP

For the server's address to never change (so you don't lose access to the panel), you need a static IP. First, find your network interface name:

```bash
ip link show
```

You'll see something like `eth0`, `enp0s25`, or `eno1`. Then edit the interfaces file:

```bash
sudo nano /etc/network/interfaces
```

```
# replace enp0s25 with your actual interface name
auto enp0s25
iface enp0s25 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 1.1.1.1 8.8.8.8
```

```bash
sudo systemctl restart networking
```

**Alternative — via NetworkManager:**

```bash
# find the connection name
nmcli connection show

# apply the static IP
sudo nmcli connection modify "Connection-Name" \
    ipv4.method manual \
    ipv4.addresses 192.168.1.100/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns "1.1.1.1 8.8.8.8"

sudo nmcli connection up "Connection-Name"
```

> **Tip:** Before choosing an IP, check your router's DHCP range and pick an address *outside* that range to avoid conflicts. Alternatively, reserve the IP in the router itself using the ThinkPad's MAC address.

---

## Step 03 — Battery Protection (20–80%)

The T480 has two batteries (BAT0 internal, BAT1 external). Charging constantly to 100% degrades the cells over time. **TLP** lets you set charge thresholds directly in the Lenovo hardware:

```bash
sudo apt install tlp tlp-rdw -y
```

Edit `/etc/tlp.conf`:

```ini
# Internal battery
START_CHARGE_THRESH_BAT0=20
STOP_CHARGE_THRESH_BAT0=80

# External battery
START_CHARGE_THRESH_BAT1=20
STOP_CHARGE_THRESH_BAT1=80
```

```bash
sudo systemctl enable tlp
sudo systemctl start tlp
sudo tlp start

# verify it applied correctly
sudo tlp-stat -b
```

---

## Step 04 — Lid Closed Without Suspending

By default, Debian suspends or hibernates when the lid is closed. On a server, that's the last thing you want. Configure it in `logind.conf`:

```bash
sudo nano /etc/systemd/logind.conf
```

```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

```bash
sudo systemctl restart systemd-logind
```

---

## Step 05 — Auto Screen Blank

Leaving the screen on 24/7 wears out the panel. On a headless server (no desktop environment), the simplest approach is setting `consoleblank` directly in GRUB:

```bash
sudo nano /etc/default/grub
```

```ini
# 180 = 3 minutes | 300 = 5 minutes
GRUB_CMDLINE_LINUX_DEFAULT="quiet consoleblank=300"
```

```bash
sudo update-grub
```

If you have a desktop environment installed, you can use DPMS instead: `xset dpms 300 300 300` — add it to `~/.bashrc` to make it permanent.

---

## Summary

| Goal | Solution |
|---|---|
| CasaOS + Jellyfin + Immich | Official script from casaos.io |
| Static local IP | `/etc/network/interfaces` or `nmcli` |
| Battery protected (20–80%) | TLP with charge thresholds |
| Lid closed without suspend | `logind.conf` → `HandleLidSwitch=ignore` |
| Screen blanks at 5 min | GRUB → `consoleblank=300` |

> **Before you finish:** Reboot after all changes — especially the GRUB and logind ones — to make sure everything is active. CasaOS starts automatically on boot.
