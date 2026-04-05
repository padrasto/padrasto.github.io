---
title: "Running Wireshark Without Root on Linux"
date: 2023-02-26
categories: ["Wireshark", "Linux"]
tags: ["wireshark", "linux", "permissions", "sysadmin"]
---

Most tutorials tell you to run `sudo wireshark` and move on. That works, but it's the wrong approach — you're running a large, complex GUI application as root, which is an unnecessary security risk.

The correct solution is to run Wireshark as a normal user while giving only the packet capture component the privileges it actually needs.

## Why Wireshark Needs Special Permissions

Capturing raw network traffic requires access to network interfaces at a low level. On Linux, this is controlled by two kernel capabilities:

- **`cap_net_raw`** — allows opening raw sockets and capturing packets
- **`cap_net_admin`** — allows configuring network interfaces

By default, only root has these. The goal is to grant them precisely — without touching the rest of the system.

## The Role of dumpcap

Wireshark uses a separation of privileges model. The GUI itself doesn't capture packets — that job belongs to a small helper binary called `dumpcap`. Only `dumpcap` needs elevated capabilities. Wireshark reads the data `dumpcap` provides and runs as a normal user.

This means you can give `dumpcap` the minimum permissions it needs, and keep Wireshark itself completely unprivileged.

## Setup

Install Wireshark:

```bash
sudo apt install wireshark
```

During installation, it will ask: *"Should non-superusers be able to capture packets?"* — select **Yes**. If you already have it installed and skipped this step, reconfigure it:

```bash
sudo dpkg-reconfigure wireshark-common
```

Create the `wireshark` group (if it doesn't exist yet):

```bash
sudo groupadd wireshark
```

Add your user to the group. Replace `user` with your actual username:

```bash
sudo adduser user wireshark
```

Set `dumpcap` group ownership to `wireshark`:

```bash
sudo chgrp wireshark /usr/bin/dumpcap
```

Restrict execution to root and the `wireshark` group:

```bash
sudo chmod 750 /usr/bin/dumpcap
```

Grant `dumpcap` the two capabilities it needs:

```bash
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap
```

Reboot for the group membership to take effect:

```bash
sudo reboot
```

## What setcap Actually Does

The `setcap` command assigns Linux capabilities directly to a binary, without making it setuid root. The flag `=eip` means the capabilities are set as **Effective**, **Inheritable**, and **Permitted** — which is what `dumpcap` needs to open raw sockets.

This is preferable to `setuid root` because the privileges are scoped to exactly what the binary needs, and don't escalate the entire process to root.

## Verify It Works

After rebooting, open Wireshark as your normal user. You should see your network interfaces listed and be able to start a capture without any permission errors.

You can also confirm `dumpcap` has the capabilities set:

```bash
getcap /usr/bin/dumpcap
```

Expected output:

```
/usr/bin/dumpcap cap_net_admin,cap_net_raw=eip
```

If it's empty, repeat the `setcap` command.
