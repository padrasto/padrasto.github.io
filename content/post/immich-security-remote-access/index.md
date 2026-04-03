---
title: "Immich, Firewall, and Private Remote Access"
description: "Migrating from Google Photos to Immich, locking down the server with UFW and Fail2ban, and accessing it remotely without opening ports — or selling your privacy."
date: 2026-04-03
categories:
  - Self-Hosting
  - Linux
tags:
  - Immich
  - Debian
  - Homelab
  - Tailscale
  - UFW
  - Fail2ban
  - Privacy
  - CasaOS
---

*This is Part 2 of the ThinkPad T480 home server series. [Part 1](/post/thinkpad-home-server/) covers CasaOS installation, static IP, battery protection, and screen blanking.*

---

With the server running, the next step was getting my photos off Google and into Immich, then making sure the server was properly locked down and reachable from outside the house — without opening ports on the router or compromising privacy.

---

## Step 01 — Importing Google Takeout into Immich

Google exports your photos in a specific structure: each image comes with a separate `.json` file containing its metadata (original date, GPS location, etc.).

```
Takeout/
  Google Photos/
    Photos from 2018/
      photo.jpg
      photo.jpg.json   ← metadata: date, location, etc.
    Photos from 2019/
      ...
```

> **Warning:** If you import photos directly without handling the `.json` files, you lose original dates and locations. The Immich CLI handles this automatically.

### Install the Immich CLI

The photos live on your local PC, not the server. Install the CLI locally — Node.js is required:

```bash
# verify Node.js is installed
node --version
npm --version

# install the CLI
sudo npm install -g @immich/cli
```

### Authenticate with an API key

The CLI uses an API key, not email/password. To get one:

**Immich → avatar (top right) → Account Settings → API Keys → New API Key**

```bash
immich login http://192.168.1.x:2283 <YOUR-API-KEY>
```

Replace `192.168.1.x` with your server's static IP.

### Upload

```bash
immich upload --recursive \
  ~/Pictures/Takeout/Google\ Photos
```

The CLI will crawl all subfolders, hash every file, check for duplicates, and upload what's new. On a 13 GB library it found 4848 files and 0 duplicates.

During the import the server will run hot — all CPU cores at 95–100% while Immich generates thumbnails and runs its machine learning models for face recognition and scene classification. This is normal and temporary.

> **Tip:** Monitor background processing progress at **Administration → Jobs** in the Immich web panel. Face detection and AI classification can take several hours depending on library size.

---

## Step 02 — Firewall with UFW

By default, Debian has no active firewall. Before the server does anything on the network, it's worth defining exactly what's allowed in — and blocking everything else.

The rule is simple: Immich, Jellyfin, and CasaOS should only be reachable from the local network. Same for SSH. Nothing should be directly exposed to the internet.

First, find your network interface name:

```bash
ip link show
```

You'll see something like `eth0`, `enp0s25`, or `enp3s0`. Replace `<interface>` in the commands below with yours.

```bash
# default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH — local network only
sudo ufw allow in on <interface> from 192.168.1.0/24 to any port 22

# Jellyfin — local network only
sudo ufw allow in on <interface> from 192.168.1.0/24 to any port 8097

# Immich — local network only
sudo ufw allow in on <interface> from 192.168.1.0/24 to any port 2283

# CasaOS panel — local network only
sudo ufw allow in on <interface> from 192.168.1.0/24 to any port 80

# enable
sudo ufw enable
sudo ufw status verbose
```

The `on <interface>` flag scopes each rule to your ethernet interface specifically — more precise than relying solely on the subnet filter.

---

## Step 03 — Remote Access Without Opening Ports

I want to reach Immich from outside the house. My router doesn't support port forwarding. The two most common solutions are **Cloudflare Tunnel** and **Tailscale** — but they have very different privacy implications.

### The privacy comparison

| Service | Sees content? | Jurisdiction | Free? |
|---|---|---|---|
| Cloudflare Tunnel | ⚠️ Yes (SSL termination) | 🇺🇸 CLOUD Act | ✅ |
| Tailscale | ✅ No (E2E encrypted) | 🇨🇦 Canada | ✅ |
| Pangolin + Hetzner VPS | ✅ No | 🇪🇺 GDPR | ❌ ~€4/month |

Cloudflare performs SSL termination — your traffic is decrypted on their servers before being re-encrypted. That means they can technically read your photos. For a personal family archive, that's a non-starter. Their free plan also prohibits using tunnels for media streaming, so Jellyfin would violate the terms of service.

The most private European option would be self-hosting [Pangolin](https://github.com/fosrl/pangolin) on a Hetzner VPS (~€4/month), which gives you full control under GDPR. For now, **Tailscale** is a reasonable middle ground: free, end-to-end encrypted with WireGuard, and your content never touches their servers.

What Tailscale *can* see: connection metadata — when you connected, from where, how much traffic. What they *cannot* see: the content itself.

### Install Tailscale on the server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

A link will appear — open it in a browser to authenticate with your Tailscale account. Then check the assigned IP:

```bash
tailscale ip
```

### Install the Tailscale app on your devices

Download the Tailscale app on your phone or laptop, sign in with the same account, and you can reach Immich from anywhere via the server's Tailscale IP:

```
http://100.x.x.x:2283
```

### UFW rules for Tailscale

```bash
# allow all traffic from the Tailscale interface
sudo ufw allow in on tailscale0

# allow the Tailscale UDP port
sudo ufw allow 41641/udp
```

---

## Step 04 — Fail2ban

UFW blocks ports, but it doesn't protect against someone repeatedly guessing your SSH password. Fail2ban watches the logs and automatically bans IPs that fail too many times.

```bash
sudo apt install fail2ban -y
```

Create the local configuration file:

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled  = true
port     = 22
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# verify
sudo fail2ban-client status sshd
```

With this configuration, any IP that fails SSH login 3 times in 10 minutes gets banned for 1 hour.

---

## Summary

| Configuration | Status |
|---|---|
| CasaOS + Jellyfin + Immich | ✅ Running |
| Static IP | ✅ Configured |
| Battery protection 20–80% (TLP) | ✅ Active |
| Lid closed without suspend | ✅ logind.conf |
| Screen blanks at 5 min | ✅ consoleblank=300 |
| Photos imported from Google Takeout | ✅ Immich CLI |
| Firewall (UFW) | ✅ Local network only |
| Private remote access | ✅ Tailscale E2E |
| SSH brute-force protection | ✅ Fail2ban |

**Optional next steps:** SSH key authentication instead of password (`ssh-keygen -t ed25519`) and automatic security updates with `unattended-upgrades`.
