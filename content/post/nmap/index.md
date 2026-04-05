---
title: "nmap — A Practical Guide for Sysadmins"
date: 2023-03-13
categories: ["Nmap"]
tags: ["nmap", "networking", "security", "sysadmin", "linux"]
---

nmap (Network Mapper) is an open-source tool for network discovery and security auditing. It's one of the most used tools in a sysadmin's toolkit — not just for penetration testing, but for everyday tasks like checking which ports are open on a server, discovering hosts on a network, or verifying firewall rules are working as expected.

This article covers installation and the most useful scan types, with an explanation of what each one actually does under the hood.

## Installation

```bash
sudo apt install nmap
```

Verify the installed version:

```bash
nmap --version
```

## Understanding Scan Types

Before running scans, it's worth understanding the two main methods nmap uses to probe ports.

**TCP Connect scan (`-sT`)** — completes a full TCP three-way handshake (SYN → SYN-ACK → ACK). It's reliable but noisy: the connection is fully logged by the target.

**SYN scan (`-sS`)** — sends a SYN packet and waits for a response, but never completes the handshake. If the port is open, the target replies with SYN-ACK; nmap then sends RST to tear it down without establishing a connection. This is the default when run as root, and it's faster and less likely to appear in application logs.

**UDP scan (`-sU`)** — scans UDP ports. Slower than TCP scans because UDP has no handshake — nmap has to wait for ICMP "port unreachable" responses or timeouts to determine port state.

## Basic Host Scan

Scan a single host for open ports:

```bash
nmap 192.168.1.10
```

By default, nmap scans the 1,000 most commonly used TCP ports. Open ports, their state, and the service name are shown in the output.

Scan a specific port or range:

```bash
nmap -p 22 192.168.1.10
nmap -p 22,80,443 192.168.1.10
nmap -p 1-1024 192.168.1.10
```

Scan all 65,535 ports:

```bash
nmap -p- 192.168.1.10
```

## Host Discovery on a Network

Discover which hosts are up on a subnet without scanning their ports:

```bash
nmap -sn 192.168.1.0/24
```

`-sn` (previously `-sP`) sends ICMP echo requests and ARP probes to each address in the range and reports which ones respond. Useful for mapping a network quickly.

To scan all discovered hosts and their ports in one command:

```bash
nmap 192.168.1.0/24
```

## SYN Scan

The default scan type when run as root. Faster and stealthier than a full TCP connect scan:

```bash
sudo nmap -sS 192.168.1.10
```

## Service and Version Detection

Identify the software and version running on each open port:

```bash
nmap -sV 192.168.1.10
```

nmap sends probes to each open port and compares responses against its service fingerprint database. The output will show something like `OpenSSH 8.9p1` or `nginx 1.18.0` next to the port number.

Combine with `-p-` to check all ports:

```bash
nmap -sV -p- 192.168.1.10
```

## OS Detection

Attempt to identify the operating system of the target based on TCP/IP stack behaviour:

```bash
sudo nmap -O 192.168.1.10
```

nmap compares packet characteristics (TTL values, TCP window size, IP flags) against a fingerprint database. It works best when at least one open and one closed port are found. Results include a confidence percentage — treat them as educated guesses rather than certainties.

## Combining Options

The most common all-in-one scan for a full picture of a host:

```bash
sudo nmap -sS -sV -O 192.168.1.10
```

Or use the aggressive scan flag, which enables OS detection, version detection, script scanning, and traceroute:

```bash
sudo nmap -A 192.168.1.10
```

`-A` is convenient but noisy — don't use it on networks you don't own or have explicit permission to test.

## Script Engine (NSE)

nmap includes a scripting engine (NSE) with hundreds of scripts for tasks ranging from service enumeration to vulnerability detection. Scripts are located in `/usr/share/nmap/scripts/`.

Run scripts against a target:

```bash
nmap --script <script_name> 192.168.1.10
```

Run all scripts in the `vuln` category to check for known vulnerabilities:

```bash
nmap -sV --script vuln 192.168.1.10
```

Check the default scripts (safe, non-intrusive):

```bash
nmap -sC 192.168.1.10
```

List available scripts by category:

```bash
ls /usr/share/nmap/scripts/ | grep smb
```

## Output Formats

Save scan results to a file. nmap supports several formats:

Normal text output:

```bash
nmap -oN scan.txt 192.168.1.10
```

XML output (useful for parsing with other tools):

```bash
nmap -oX scan.xml 192.168.1.10
```

Grepable format:

```bash
nmap -oG scan.gnmap 192.168.1.10
```

Save all three formats at once:

```bash
nmap -oA scan 192.168.1.10
```

This creates `scan.txt`, `scan.xml`, and `scan.gnmap`.

## Timing and Performance

nmap has six timing templates (`-T0` to `-T5`) that control scan speed and aggressiveness:

- `-T0` / `-T1` — very slow, designed to evade IDS detection
- `-T2` — slow
- `-T3` — default
- `-T4` — faster, recommended for local networks
- `-T5` — fastest, may miss results on congested networks

```bash
nmap -T4 192.168.1.0/24
```

## A Note on Permission

nmap is a powerful tool. Always use it only on networks and systems you own or have explicit written permission to test. Running intrusive scans against systems you don't control is illegal in most jurisdictions, regardless of intent.

For lab practice, set up a local VM — something like Metasploitable2 or a basic Ubuntu Server — and scan that instead.
