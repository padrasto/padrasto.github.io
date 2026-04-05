---
title: "Installing Proxmox on a Laptop — Full Disk, Lid Closed, Screen Off"
date: 2023-07-19
categories: ["Proxmox"]
tags: ["proxmox", "virtualisation", "homelab", "sysadmin", "linux"]
---

Proxmox VE is an open-source virtualisation platform that combines KVM-based virtual machines and LXC containers under a single web interface. It's typically deployed on dedicated server hardware, but a laptop makes a perfectly capable homelab node — especially if it's sitting unused. It has a built-in UPS (the battery), a screen for local access, and low idle power consumption compared to a tower.

Installing on a laptop introduces two problems the standard Proxmox documentation doesn't address well: the default partitioning wastes most of the disk, and closing the lid suspends the machine. This article covers both, along with the full installation process.

## What Proxmox Installs

Proxmox runs on Debian and uses LVM (Logical Volume Manager) to manage storage. The default installer creates three logical volumes on a volume group called `pve`:

- `pve/root` — the root filesystem for the Proxmox OS (around 96GB by default regardless of disk size)
- `pve/swap` — swap space
- `pve/data` — a thin-provisioned LVM pool intended for VM disk images

The problem is that `pve/data` is not directly usable as a standard filesystem — it's a thin pool, designed for LVM thin provisioning. The Proxmox web UI uses it through the `local-lvm` storage definition. On a laptop with a single disk, this setup leaves the `local` storage (which maps to `/var/lib/vz` on `pve/root`) severely limited, and the split between `local` and `local-lvm` adds unnecessary complexity for a single-node homelab.

The solution is to remove `pve/data`, reclaim that space into `pve/root`, and use only the `local` storage for everything — ISOs, VM disks, backups, and container templates.

## Installation

Download the Proxmox VE ISO from [proxmox.com/downloads](https://www.proxmox.com/en/downloads) and write it to a USB drive:

```bash
sudo dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress
```

Replace `/dev/sdX` with your USB device. Boot from the USB and follow the graphical installer. During installation:

- Set a static IP address for the management interface — you'll need it to access the web UI
- Set the hostname to something meaningful (`proxmox.local` works fine for a homelab)
- The disk partitioning is automatic — don't worry about it now, you'll fix it after boot

After installation, access the web UI from another machine at `https://<proxmox_ip>:8006`. Ignore the SSL certificate warning — Proxmox uses a self-signed certificate by default.

## Reclaiming Disk Space

After the first login, open the Proxmox shell (Node → Shell in the web UI, or SSH into the machine) and run the following three commands.

**Remove the thin-pool volume:**

```bash
lvremove /dev/pve/data
```

Type `y` to confirm. This removes the LVM thin pool. No data is lost because the machine is freshly installed.

**Expand the root volume to use all freed space:**

```bash
lvresize -l +100%FREE /dev/pve/root
```

This grows the `pve/root` logical volume to consume all the space that was occupied by `pve/data`.

**Resize the filesystem to match the new volume size:**

```bash
resize2fs /dev/mapper/pve-root
```

LVM operates at the block level — resizing the logical volume doesn't automatically resize the filesystem inside it. `resize2fs` extends the ext4 filesystem to fill the newly available space.

Verify the result:

```bash
df -h /
```

The root filesystem should now reflect the full available disk space.

## Remove local-lvm from the Web UI

The `local-lvm` storage definition in the Proxmox UI still points to the volume you just removed. Go to **Datacenter → Storage**, select `local-lvm`, and click **Remove**.

Then edit the `local` storage: select it, click **Edit**, and make sure **Content** includes all types — Disk image, Container, ISO image, Container template, Snippets, Backup. This allows `local` to handle everything that `local-lvm` previously handled.

## Disable Lid Suspend

By default, closing the laptop lid triggers a suspend event managed by `systemd-logind`. On a server, this is the last thing you want.

Edit the logind configuration:

```bash
nano /etc/systemd/logind.conf
```

Find and set (or uncomment) these two lines:

```
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```

`HandleLidSwitch` controls what happens when the lid is closed on battery or AC power. `HandleLidSwitchDocked` controls the behaviour when connected to a docking station. Setting both to `ignore` ensures the machine stays running regardless.

Restart the service for the change to take effect:

```bash
systemctl restart systemd-logind.service
```

Test it by closing the lid — the machine should remain accessible over the network.

## Turn Off the Screen Automatically

With the lid closed and suspend disabled, the screen stays on indefinitely, wasting power. The `consoleblank` kernel parameter blanks the console after a set number of seconds of inactivity.

Edit the GRUB configuration:

```bash
nano /etc/default/grub
```

Find this line:

```
GRUB_CMDLINE_LINUX=""
```

Change it to:

```
GRUB_CMDLINE_LINUX="consoleblank=300"
```

`300` is seconds — the screen will blank after 5 minutes of inactivity. Adjust to your preference.

Update GRUB to apply the change:

```bash
update-grub
```

Reboot for the kernel parameter to take effect:

```bash
reboot
```

After rebooting, the screen will blank after 5 minutes. The machine remains fully operational — SSH, web UI, and all running VMs continue unaffected.

## Verify Everything

Check disk usage:

```bash
df -h
lsblk
lvdisplay
```

Check that `systemd-logind` has the correct configuration loaded:

```bash
systemctl show systemd-logind | grep -i lid
```

Check the active kernel parameters including `consoleblank`:

```bash
cat /proc/cmdline
```

Check Proxmox node status in the web UI — the storage panel should now show `local` with the full available capacity.

## Network Access

The Proxmox web UI is always at `https://<ip>:8006`. For SSH:

```bash
ssh root@<proxmox_ip>
```

Proxmox does not create a non-root user by default. For daily management, the web UI is the primary interface — SSH is for configuration and recovery tasks.

## Optional — Static IP After Installation

If you need to change or set the IP after installation, edit the network interface file:

```bash
nano /etc/network/interfaces
```

A typical static configuration:

```
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports eth0
    bridge-stp off
    bridge-fd 0
```

`vmbr0` is the default Linux bridge Proxmox creates for VM networking. VMs attach to this bridge and share the physical network interface.

Restart networking:

```bash
systemctl restart networking
```
