---
title: "Setting Up a WireGuard VPN Server on Ubuntu"
date: 2023-02-27
categories: ["Ubuntu"]
tags: ["wireguard", "vpn", "ubuntu", "networking", "sysadmin"]
---

WireGuard is a modern VPN protocol built directly into the Linux kernel since version 5.6. Compared to OpenVPN or IPsec, it's significantly simpler to configure, faster, and has a much smaller attack surface — the entire codebase is around 4,000 lines, versus hundreds of thousands for OpenVPN.

WireGuard uses a peer-to-peer model with public/private key pairs — similar to SSH keys. Each peer generates its own keypair, shares its public key with the other side, and WireGuard handles authentication and encryption automatically. No certificates, no certificate authorities, no complex handshake configuration.

This article covers a full manual setup: server configuration, firewall rules, and client configuration — no scripts.

## Install WireGuard

On the server:

```bash
sudo apt update && sudo apt install wireguard
```

## Generate the Server Keys

Generate the private key first, then derive the public key from it:

```bash
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
```

Restrict permissions on the private key:

```bash
sudo chmod 600 /etc/wireguard/server_private.key
```

Note both values — you'll need them in the configuration file:

```bash
sudo cat /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_public.key
```

## Configure the Server

Create the server configuration file:

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = <server_private_key>
Address = 10.66.66.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

What each field does:
- `PrivateKey` — the server's private key generated above
- `Address` — the server's IP inside the VPN tunnel. `10.66.66.1/24` defines the tunnel network as `10.66.66.0/24`
- `ListenPort` — the UDP port WireGuard listens on. `51820` is the standard port
- `PostUp` — iptables rules that run when the interface comes up. The `FORWARD` rule allows traffic to pass through the server; `MASQUERADE` does NAT so client traffic appears to come from the server's IP on `eth0`
- `PostDown` — removes those rules when the interface goes down

Restrict permissions on the config file:

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

## Enable IP Forwarding

By default, Linux doesn't forward packets between interfaces. Enable it so the server can route VPN traffic to the internet:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment or add:

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Apply without rebooting:

```bash
sudo sysctl -p
```

## Start the WireGuard Interface

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

Confirm it's running:

```bash
sudo wg show
```

You should see the `wg0` interface with the listen port and no peers yet.

## Generate the Client Keys

On the client machine:

```bash
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key
sudo chmod 600 /etc/wireguard/client_private.key
```

Note both values — the private key goes into the client config, the public key gets added as a peer on the server.

## Add the Client as a Peer on the Server

On the server, add a `[Peer]` block to `/etc/wireguard/wg0.conf`:

```ini
[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.66.66.2/32
```

- `PublicKey` — the client's public key
- `AllowedIPs` — the IP assigned to this client inside the tunnel. `/32` means only that exact IP routes to this peer

Restart the interface to apply:

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

## Configure the Client

Install WireGuard:

```bash
sudo apt install wireguard
```

Create the client config:

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = <client_private_key>
Address = 10.66.66.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public_key>
Endpoint = <server_public_ip>:51820
AllowedIPs = 0.0.0.0/0,::/0
PersistentKeepalive = 25
```

- `Address` — the client's tunnel IP, matching what you set in `AllowedIPs` on the server
- `DNS` — DNS server to use while connected
- `Endpoint` — the server's public IP and WireGuard port
- `AllowedIPs = 0.0.0.0/0,::/0` — routes all traffic through the VPN. To route only the VPN subnet, use `10.66.66.0/24` instead
- `PersistentKeepalive = 25` — sends a keepalive every 25 seconds, useful when the client is behind NAT

Bring the tunnel up:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

## Verify the Connection

On the client:

```bash
sudo wg show
```

You should see the server listed as a peer with a recent handshake and traffic counters. Confirm your public IP changed:

```bash
curl ifconfig.me
```

It should return the server's IP.

## Adding More Clients

Each additional client follows the same process: generate a keypair, add a `[Peer]` block on the server with a unique `AllowedIPs` (`10.66.66.3/32`, `10.66.66.4/32`, etc.), and create a matching config on the client.
