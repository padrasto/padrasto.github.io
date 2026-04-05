---
title: "SSH Hardening on Ubuntu — Key Authentication and Secure Configuration"
date: 2023-07-18
categories: ["Ssh", "Ubuntu"]
tags: ["ssh", "ubuntu", "security", "sysadmin", "hardening"]
---

SSH is the primary way to manage a Linux server remotely. The default configuration gets you connected, but it leaves several attack vectors open — password authentication being the most obvious. A server exposed to the internet with password auth enabled will be under constant brute-force attack within minutes of going online.

This article covers a complete initial hardening setup: creating a non-root admin user, configuring key-based authentication, disabling passwords, and locking down `sshd_config` properly.

## Why Disable Password Authentication

Password authentication is vulnerable to brute-force attacks. Any server with port 22 open receives thousands of login attempts per day from automated scanners. Even with a strong password, you're relying entirely on its complexity.

SSH key authentication works differently: the client holds a private key, the server holds the corresponding public key. Authentication succeeds only if the client can prove it holds the private key — without ever sending it over the network. A private key protected with a passphrase adds a second factor.

There is no realistic brute-force attack against a properly generated SSH key pair.

## Step 1 — Create a Non-Root Admin User

Log into the server as root:

```bash
ssh root@<server_ip>
```

Running everything as root is dangerous — a single mistake can cause irreversible damage. Create a regular user and grant it sudo privileges instead:

```bash
adduser user
usermod -aG sudo user
```

Verify the user was added to sudo:

```bash
groups user
```

The output should include `sudo`.

## Step 2 — Generate an SSH Key Pair on the Client

Do this on your **local machine**, not the server.

Generate an Ed25519 key pair — Ed25519 is faster and more secure than the default RSA:

```bash
ssh-keygen -t ed25519 -C "your_label"
```

If you need RSA for compatibility reasons, use at least 4096 bits:

```bash
ssh-keygen -t rsa -b 4096
```

The command will ask where to save the key (default `~/.ssh/id_ed25519`) and optionally for a passphrase. Use a passphrase — it encrypts the private key on disk so it can't be used if the file is stolen.

This creates two files:
- `~/.ssh/id_ed25519` — the private key. Never share this, never copy it to a server
- `~/.ssh/id_ed25519.pub` — the public key. This is what you put on the server

## Step 3 — Copy the Public Key to the Server

The easiest method is `ssh-copy-id`, which handles the file creation and permissions automatically:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@<server_ip>
```

You'll be prompted for the user's password (this is the last time you'll need it). The command copies the public key to `~/.ssh/authorized_keys` on the server with the correct permissions.

If `ssh-copy-id` is not available, do it manually. On the server, switch to the new user:

```bash
su - user
```

Create the `.ssh` directory with the correct permissions:

```bash
mkdir ~/.ssh
chmod 700 ~/.ssh
```

The `700` permission means only the owner can read, write, or enter the directory. SSH will refuse to use `authorized_keys` if the directory permissions are too open.

Create the `authorized_keys` file and paste your public key into it:

```bash
nano ~/.ssh/authorized_keys
```

Set the correct permissions on the file:

```bash
chmod 600 ~/.ssh/authorized_keys
```

`600` means only the owner can read and write the file. Again, SSH enforces this — wrong permissions and key auth silently fails.

Return to root:

```bash
exit
```

## Step 4 — Test Key Authentication Before Disabling Passwords

**Do not disable password authentication before confirming key auth works.** If you lock yourself out, you'll need console access to recover.

Open a new terminal (keep the current session open as a fallback) and test:

```bash
ssh -i ~/.ssh/id_ed25519 user@<server_ip>
```

If you connect without being asked for a password (or only asked for your key passphrase), key auth is working. Only proceed to the next step after confirming this.

## Step 5 — Harden sshd_config

Edit the SSH daemon configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply the following settings:

**Disable password authentication:**

```
PasswordAuthentication no
```

**Ensure public key authentication is enabled:**

```
PubkeyAuthentication yes
```

**Disable root login:**

```
PermitRootLogin no
```

Root login over SSH should always be disabled. Use your sudo user instead. This prevents attackers from targeting the one account that exists on every Linux system.

**Disable challenge-response authentication:**

```
ChallengeResponseAuthentication no
```

This disables PAM-based authentication methods like one-time passwords and keyboard-interactive prompts that could still allow password-based access through the back door.

**Disable empty passwords:**

```
PermitEmptyPasswords no
```

**Change the default port (optional but reduces noise):**

```
Port 2222
```

Changing from port 22 eliminates the majority of automated scanners, which almost always target the default port. It's not a security measure by itself — anyone doing a full port scan will find it — but it dramatically reduces log noise. If you change this, update your firewall rules and remember to specify the port when connecting:

```bash
ssh -p 2222 user@<server_ip>
```

**Limit authentication attempts:**

```
MaxAuthTries 3
```

After 3 failed attempts, the connection is dropped.

**Set a login grace time:**

```
LoginGraceTime 30
```

The server disconnects unauthenticated connections after 30 seconds.

**Disable X11 forwarding if you don't need it:**

```
X11Forwarding no
```

**Restrict which users can log in (optional but recommended):**

```
AllowUsers user
```

Only the listed users can authenticate. Any other account — including ones created by software — cannot log in via SSH even if they have valid credentials.

The full set of changes in context:

```
Port 22
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
X11Forwarding no
AllowUsers user
```

## Step 6 — Reload sshd

Apply the changes:

```bash
sudo systemctl reload sshd
```

Test the connection again from a new terminal before closing the existing session:

```bash
ssh user@<server_ip>
```

If it connects, you're done. If something went wrong, the existing session keeps you from being locked out.

## Step 7 — Firewall

If UFW is not already configured, set it up:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

If you changed the SSH port:

```bash
sudo ufw allow 2222/tcp
sudo ufw deny 22/tcp
```

## Managing Multiple Keys

If you manage multiple servers, use `~/.ssh/config` on your local machine to avoid typing long commands:

```
Host myserver
    HostName 192.168.1.10
    User user
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

With this in place, you connect with just:

```bash
ssh myserver
```

## Verifying the Configuration

Check which authentication methods are active:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication|permitrootlogin|port'
```

`sshd -T` prints the full effective configuration, including defaults. This is more reliable than reading `sshd_config` directly because it shows what sshd actually has loaded.

Check active SSH sessions:

```bash
who
```

Check recent login attempts (successful and failed):

```bash
sudo journalctl -u ssh --no-pager | tail -50
```

Or check the auth log directly:

```bash
sudo tail -f /var/log/auth.log
```

Watching this file after hardening shows the brute-force attempts hitting the server and failing — a good confirmation that the configuration is working.
