---
title: "Tor Hidden Service and Server Hardening on Debian"
description: "How to set up a Tor onion mirror of a static site, configure nginx securely, and harden a Debian server from scratch."
date: 2026-04-09
categories:
  - Networking
  - Security
tags:
  - Tor
  - nginx
  - Debian
  - AppArmor
  - fail2ban
  - ufw
  - Hardening
---

Today's lab session covered two things that go well together: setting up a Tor hidden service to mirror a static blog, and hardening the Debian server it runs on. Everything was done on a local Debian 13 VM for learning purposes.

## What We Built

```
Internet → padrasto.github.io (original blog)
                    ↓ wget mirror
         Debian VM (local)
              nginx (127.0.0.1:80)
                    ↓
              Tor daemon
                    ↓
         xxxxxxxx.onion (hidden service)
```

The blog content is fetched with `wget`, served by nginx listening only on loopback, and exposed exclusively through Tor. No port forwarding, no public IP required.

---

## Step 1 — Install Dependencies

```bash
sudo apt update
sudo apt install -y nginx tor wget
```

---

## Step 2 — Mirror the Blog

```bash
sudo mkdir -p /var/www/blog-mirror

sudo wget \
  --mirror \
  --convert-links \
  --adjust-extension \
  --page-requisites \
  --no-parent \
  --directory-prefix=/var/www/blog-mirror \
  https://padrasto.github.io/
```

- `--mirror` — recursive, with timestamps
- `--convert-links` — rewrites links to work locally
- `--adjust-extension` — adds `.html` where needed
- `--page-requisites` — fetches CSS, images, JS

Fix permissions so nginx can read the files:

```bash
sudo chown -R www-data:www-data /var/www/blog-mirror/
sudo chmod -R 755 /var/www/blog-mirror/
```

---

## Step 3 — Configure nginx

`/etc/nginx/sites-available/blog-mirror`:

```nginx
server {
    listen 127.0.0.1:80;
    server_name localhost;

    root /var/www/blog-mirror/padrasto.github.io;
    index index.html;

    etag off;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Content-Security-Policy "default-src 'self'" always;

    if ($request_method !~ ^(GET|HEAD)$) {
        return 405;
    }

    location / {
        try_files $uri $uri/ $uri.html =404;
    }
}
```

Key decisions:

- `listen 127.0.0.1:80` — loopback only, never directly reachable
- `etag off` — ETag headers can leak inode numbers
- `Referrer-Policy: no-referrer` — prevents the `.onion` address from leaking in outbound requests
- Only GET and HEAD allowed — no POST, PUT, DELETE on a static site

Enable the site and reload:

```bash
sudo ln -s /etc/nginx/sites-available/blog-mirror /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Also confirm `server_tokens off` is set in `/etc/nginx/nginx.conf` — this hides the nginx version from response headers.

---

## Step 4 — Configure the Tor Hidden Service

Add to `/etc/tor/torrc`:

```
HiddenServiceDir /var/lib/tor/blog-mirror/
HiddenServicePort 80 127.0.0.1:80
```

```bash
sudo systemctl restart tor
# Wait ~30 seconds
sudo cat /var/lib/tor/blog-mirror/hostname
```

The file will contain your v3 onion address — a 56-character string ending in `.onion`. Open it in Tor Browser and the blog should appear.

Tor generates an Ed25519 keypair stored in `/var/lib/tor/blog-mirror/`. Back up the `hs_ed25519_secret_key` file if you want to keep the address permanently.

---

## Step 5 — Auto-Sync with Cron

Create a sync script:

```bash
sudo nano /usr/local/bin/update-blog-mirror.sh
```

```bash
#!/bin/bash
wget --mirror \
     --convert-links \
     --adjust-extension \
     --page-requisites \
     --no-parent \
     --directory-prefix=/var/www/blog-mirror \
     https://padrasto.github.io/ -q

chown -R www-data:www-data /var/www/blog-mirror/
systemctl reload nginx
```

```bash
sudo chmod +x /usr/local/bin/update-blog-mirror.sh
sudo crontab -e
```

Add:

```
*/10 * * * * /usr/local/bin/update-blog-mirror.sh
```

Test it manually first:

```bash
sudo /usr/local/bin/update-blog-mirror.sh && echo "OK"
```

---

## Step 6 — Server Hardening

### SSH

`/etc/ssh/sshd_config` settings confirmed:

```
PasswordAuthentication no
PermitRootLogin no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowAgentForwarding no
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

Key-only authentication eliminates the brute force attack surface. The cipher restrictions enforce modern cryptography only.

### Firewall — ufw

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw enable
```

Tor does not need an inbound rule — it only makes outbound connections to the Tor network and receives traffic through those circuits.

### fail2ban

```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

In `jail.local`, configure the SSH jail:

```ini
[sshd]
enabled = true
port    = 22
maxretry = 3
bantime  = 1h
findtime = 10m
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### sysctl Hardening

`/etc/sysctl.d/99-hardening.conf`:

```ini
net.ipv4.ip_forward = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
```

```bash
sudo sysctl --system
```

### AppArmor

Tor ships with an AppArmor profile that is enabled by default on Debian:

```bash
sudo aa-status | grep tor
# Output: /usr/bin/tor (enforce mode)
```

nginx does not have an official profile in Debian 13's package repositories, so we create a minimal one:

```bash
sudo apt install -y apparmor-utils
sudo nano /etc/apparmor.d/usr.sbin.nginx
```

```
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,
  capability setuid,
  capability setgid,
  capability dac_override,

  /usr/sbin/nginx mr,
  /etc/nginx/** r,
  /var/log/nginx/** w,
  /var/www/blog-mirror/** r,
  /run/nginx.pid rw,
  /var/lib/nginx/** rw,

  unix (send, receive) type=stream,
}
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
sudo systemctl restart nginx
```

Verify both processes are confined:

```bash
sudo aa-status | grep -E "nginx|tor"
```

### Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
sudo systemctl is-enabled unattended-upgrades
```

---

## Final State

| Component | Status |
|---|---|
| SSH — key-only, no root, no password | ✅ |
| SSH — modern ciphers only | ✅ |
| ufw — deny incoming, SSH allowed | ✅ |
| fail2ban — sshd active | ✅ |
| sysctl — network hardening | ✅ |
| unattended-upgrades | ✅ |
| nginx — loopback only, server_tokens off | ✅ |
| nginx — security headers | ✅ |
| nginx — GET/HEAD only, no ETag | ✅ |
| AppArmor — nginx enforce | ✅ |
| AppArmor — Tor enforce | ✅ |
| Tor hidden service v3 | ✅ |
| Auto-sync cron every 10 minutes | ✅ |

The attack surface here is deliberately minimal: no public-facing ports except SSH (key-only), nginx never reachable directly, and both nginx and Tor confined by AppArmor. For a learning lab this is a solid baseline that maps directly to real production practices.
