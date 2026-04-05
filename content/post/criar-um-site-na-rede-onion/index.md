---
title: "Hosting an Onion Site on Debian — A Complete Guide"
date: 2023-05-03
categories: ["Debian", "Tor"]
tags: ["tor", "onion", "debian", "apache", "privacy", "self-hosting"]
---

The Tor network allows you to host a website that is accessible only through the Tor Browser, with no domain registrar, no DNS, no public IP address, and no personally identifiable information attached to it. The server's real location remains hidden from visitors, and visitors remain hidden from the server.

This is useful for legitimate reasons: publishing content anonymously, running a service in a country with internet censorship, providing a privacy-respecting mirror of an existing site, or simply learning how onion services work.

This guide covers a complete setup on Debian — Apache as the web server, Tor as the hidden service layer, and the security hardening steps that most tutorials skip.

## How Onion Services Work

When you configure a Tor hidden service, Tor generates a public/private key pair for your server. The public key becomes your `.onion` address — a 56-character string derived from the key itself. There are no registrars and no central authority: the address *is* the key.

Your server connects outward to the Tor network through a set of relay nodes called introduction points, and registers itself there. When a client wants to connect, Tor negotiates a circuit through a rendezvous point — a third relay that neither side controls. Traffic between client and server passes through this circuit, encrypted end-to-end. Neither party learns the other's IP address.

This means your server never needs to accept inbound connections from the public internet, and your `.onion` address works even if your server is behind NAT.

## Install Apache

Install the Apache web server:

```bash
sudo apt update && sudo apt install apache2
```

Start and enable it:

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```

Verify it's running:

```bash
sudo systemctl status apache2
```

## Harden Apache for an Onion Service

Before configuring Tor, harden Apache so it doesn't leak information about the server.

Disable the default server signature and token in `/etc/apache2/conf-available/security.conf`:

```bash
sudo nano /etc/apache2/conf-available/security.conf
```

Set these values:

```apache
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```

`ServerTokens Prod` makes Apache return only `Apache` in the `Server` header instead of the full version string. `ServerSignature Off` removes the server version from error pages. `TraceEnable Off` disables the HTTP TRACE method, which can be used for cross-site tracing attacks.

Disable directory listing, which exposes file names if there's no index file:

```bash
sudo nano /etc/apache2/apache2.conf
```

Find the `<Directory /var/www/>` block and ensure `Options` does not include `Indexes`:

```apache
<Directory /var/www/>
    Options FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

Apply the changes:

```bash
sudo a2enconf security
sudo systemctl reload apache2
```

## Configure the Virtual Host

Create a dedicated virtual host for the onion site rather than using the default:

```bash
sudo nano /etc/apache2/sites-available/onion.conf
```

```apache
<VirtualHost 127.0.0.1:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/onion

    <Directory /var/www/onion>
        Options -Indexes -FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/onion_error.log
    CustomLog ${APACHE_LOG_DIR}/onion_access.log combined
</VirtualHost>
```

Binding to `127.0.0.1:80` instead of `0.0.0.0:80` means Apache only listens on the loopback interface — it won't accept connections from the public internet, only from Tor running on the same machine.

Create the document root:

```bash
sudo mkdir -p /var/www/onion
sudo chown -R www-data:www-data /var/www/onion
```

Enable the new site and disable the default:

```bash
sudo a2ensite onion
sudo a2dissite 000-default
sudo systemctl reload apache2
```

## Create the Site Content

Create a basic HTML page:

```bash
sudo nano /var/www/onion/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Onion Site</title>
</head>
<body>
    <h1>Welcome</h1>
    <p>This site is hosted on the Tor network.</p>
</body>
</html>
```

## Install Tor

Install Tor from the Debian repositories:

```bash
sudo apt install tor
```

For a more up-to-date version, you can add the Tor Project's official repository. Add their signing key and source:

```bash
sudo apt install apt-transport-https gpg
curl https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | gpg --dearmor | sudo tee /usr/share/keyrings/tor-archive-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] https://deb.torproject.org/torproject.org $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/tor.list
sudo apt update && sudo apt install tor deb.torproject.org-keyring
```

## Configure the Hidden Service

Edit the Tor configuration file:

```bash
sudo nano /etc/tor/torrc
```

Add these lines at the end:

```
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:80
```

What each line does:
- `HiddenServiceDir` — the directory where Tor stores the hidden service keys and hostname. Tor creates this directory automatically with the correct permissions
- `HiddenServicePort` — maps a virtual port on the onion address (port 80) to the actual service on the local machine (`127.0.0.1:80`). The virtual port and local port don't have to match — you could expose port 80 externally while running Apache on port 8080 locally

Restart Tor to generate the keys and activate the service:

```bash
sudo systemctl restart tor
```

## Get Your .onion Address

Tor generates the address automatically from the key pair. Retrieve it:

```bash
cat /var/lib/tor/hidden_service/hostname
```

You'll see a 56-character `.onion` address like:

```
duskgytldkxiuqc6.onion
```

Open this address in the Tor Browser. Your site should load.

## Understanding the Key Files

The hidden service directory contains three files:

```bash
sudo ls -la /var/lib/tor/hidden_service/
```

- `hostname` — your `.onion` address
- `hs_ed25519_secret_key` — the private key. This is your identity. If you lose it, the address is gone. If someone else gets it, they can impersonate your site
- `hs_ed25519_public_key` — the corresponding public key

**Back up the private key securely.** This is the only way to restore your `.onion` address if the server is lost or rebuilt.

```bash
sudo cp /var/lib/tor/hidden_service/hs_ed25519_secret_key /path/to/secure/backup/
```

Permissions on this directory must remain restricted — Tor will refuse to start if they're too open:

```bash
sudo chmod 700 /var/lib/tor/hidden_service/
```

## v3 Onion Addresses

Tor has been using v3 onion addresses since 2020. v3 addresses are 56 characters long and use Ed25519 keys, which are stronger than the RSA-1024 keys used by the old v2 format (16 characters). v2 services were deprecated and disabled in Tor 0.4.6. If you see a 16-character `.onion` address anywhere, it's no longer functional.

The address you get from a current Tor installation will always be v3.

## Verify Tor Is Working

Check that the hidden service is active:

```bash
sudo systemctl status tor
```

Check the Tor logs for any errors:

```bash
sudo journalctl -u tor --no-pager | tail -30
```

A successful startup will include a line like:

```
[notice] Tor has successfully opened a circuit. Looks like client functionality is working.
[notice] Self-testing indicates your ORPort is reachable from the outside.
```

## Keep the Clearnet Site Separate

If your server also hosts a regular website accessible on the public internet, make sure the onion virtual host is completely separate from it. The onion site should be bound only to `127.0.0.1`, and the public site should be on a different document root. This prevents accidental information leakage between the two.

## Firewall

If you have a firewall configured with UFW, there's nothing to open for the onion service — Tor makes outbound connections, not inbound. No ports need to be exposed to the internet. In fact, you can block all inbound traffic and the onion service will still work:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

This is one of the key security properties of onion services: the server's real IP never needs to be reachable from the public internet.
