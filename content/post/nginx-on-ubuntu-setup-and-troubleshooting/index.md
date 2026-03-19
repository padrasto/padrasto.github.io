---
title: "nginx on Ubuntu — Complete Setup and Troubleshooting Guide"
date: 2026-03-18
tags: [linux, nginx, ubuntu, sysadmin, server]
image: "https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&auto=format&fit=crop"
---

If you have ever received a call at 3 AM because a client's site went down, you know there is no time to search through documentation. This is the guide I use in my day-to-day work — installation, configuration and the most common nginx problems on Ubuntu.

## Installation

```bash
sudo apt update
sudo apt install nginx -y
```

Confirm it is running:

```bash
sudo systemctl status nginx
```

Enable automatic start on boot — a lot of people forget this:

```bash
sudo systemctl enable nginx
sudo systemctl is-enabled nginx   # should show "enabled"
```

## File Structure

After installation, nginx organises itself like this:

```
/etc/nginx/
├── nginx.conf                  # main configuration
├── sites-available/            # available configs
│   └── default
└── sites-enabled/              # active configs (symlinks)
    └── default -> ../sites-available/default

/var/www/html/                  # default web root
/var/log/nginx/
├── access.log                  # all HTTP requests
└── error.log                   # internal nginx errors
```

The separation between `sites-available` and `sites-enabled` matters. You can have multiple configs ready in `sites-available` and enable or disable them with a symlink without deleting anything.

Enable a site:

```bash
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
```

Disable a site:

```bash
sudo rm /etc/nginx/sites-enabled/mysite
```

## Basic Site Configuration

Create a file in `sites-available`:

```bash
sudo nano /etc/nginx/sites-available/mysite
```

Minimal config to serve a static site:

```nginx
server {
    listen 80;
    server_name mysite.com www.mysite.com;

    root /var/www/mysite;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

After any change — always test before applying:

```bash
sudo nginx -t
```

If it shows `syntax is ok` and `test is successful`, reload without interrupting the service:

```bash
sudo systemctl reload nginx
```

## Correct File Permissions

nginx runs as the `www-data` user. Site files need to be readable by that user:

```bash
sudo chown -R www-data:www-data /var/www/mysite
sudo find /var/www/mysite -type f -exec chmod 644 {} \;
sudo find /var/www/mysite -type d -exec chmod 755 {} \;
```

Simple rule:
- Files: `644` — nginx reads, nobody writes without permission
- Directories: `755` — nginx enters, nobody writes without permission

## The Two Logs You Need to Know

They are different and serve different purposes:

**access.log** — records all HTTP requests:
```
192.168.1.1 - - [18/Mar/2026:20:27:32] "GET / HTTP/1.1" 200 615
```
This is where you see response codes: 200, 301, 403, 404, 500.

**error.log** — records internal nginx errors:
```
2026/03/18 20:48:45 [error] 1651#0: open() "/var/www/html/favicon.ico" failed
```

Monitor in real time:

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

## Troubleshooting — Most Common Errors

### Site not loading — where to start

```bash
# 1. Is nginx running?
sudo systemctl status nginx

# 2. Does it respond locally?
curl -I localhost

# 3. Check both logs
tail -50 /var/log/nginx/access.log
tail -50 /var/log/nginx/error.log
```

### 403 Forbidden

403 can have several causes — check in this order:

```bash
# File and directory permissions
ls -la /var/www/html/
ls -la /var/www/

# Active site configuration
cat /etc/nginx/sites-enabled/default | grep -E "root|index"

# Is the symlink there?
ls -la /etc/nginx/sites-enabled/
```

Important: if the client reports a 403 but the `access.log` shows nothing, the problem is not on the server — it may be browser cache (`Ctrl+Shift+R`) or the client hitting the wrong URL.

### 502 Bad Gateway

Happens when nginx cannot communicate with the backend (PHP-FPM, Node.js, etc):

```bash
# Check if the backend is running
sudo systemctl status php8.1-fpm

# nginx error log
tail -50 /var/log/nginx/error.log
```

### nginx won't start — port already in use

```bash
# See what is on port 80
sudo ss -tlnp | grep :80

# Check configuration
sudo nginx -t
```

### Config change has no effect

Common mistake: editing the file in `sites-available` but forgetting to reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## Quick Reference

```bash
sudo systemctl start nginx      # start
sudo systemctl stop nginx       # stop
sudo systemctl restart nginx    # restart (drops active connections)
sudo systemctl reload nginx     # reload config (no interruption)
sudo systemctl status nginx     # current status
sudo nginx -t                   # test configuration
```

---

I have been running nginx in production on Debian and Ubuntu servers for several years. If you have questions or ran into a different error, leave a comment below.
