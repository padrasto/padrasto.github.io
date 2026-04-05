---
title: "Installing and Configuring QEMU-KVM and libvirt on Ubuntu"
date: 2023-03-09
categories: ["Kvm", "Ubuntu"]
tags: ["kvm", "qemu", "libvirt", "virtualisation", "ubuntu", "sysadmin"]
---

KVM, QEMU, and libvirt are three separate pieces that work together to provide virtualisation on Linux. Understanding what each one does makes the setup easier to reason about.

**KVM** (Kernel-based Virtual Machine) is a Linux kernel module that turns the kernel itself into a hypervisor. It uses the CPU's hardware virtualisation extensions (Intel VT-x or AMD-V) to run virtual machines at near-native speed. KVM alone is just a kernel interface — it doesn't handle disk images, networking, or device emulation.

**QEMU** fills that gap. It emulates hardware — CPU, RAM, disk controllers, network cards — and when paired with KVM, delegates the CPU-intensive work to the kernel module. The result is fast, hardware-accelerated virtualisation with full device emulation.

**libvirt** sits on top of both. It's a management API and daemon (`libvirtd`) that provides a consistent interface for creating, starting, stopping, and configuring VMs — regardless of the underlying hypervisor. Tools like `virt-manager` (GUI) and `virsh` (CLI) talk to libvirt, not directly to QEMU or KVM.

## Check Hardware Support

Before installing, confirm your CPU supports hardware virtualisation:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

A result of `1` or more means you're good. `vmx` is Intel VT-x, `svm` is AMD-V. If the result is `0`, KVM won't work — you'd need to enable virtualisation in the BIOS/UEFI.

## Install the Required Packages

```bash
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils libguestfs-tools genisoimage virtinst libosinfo-bin virt-manager dnsmasq
```

What each package does:
- `qemu-kvm` — the QEMU emulator with KVM support
- `libvirt-daemon-system` — the libvirt daemon and system integration
- `libvirt-clients` — command-line tools including `virsh`
- `bridge-utils` — tools for managing network bridges (needed for VM networking)
- `libguestfs-tools` — utilities for accessing and modifying VM disk images
- `genisoimage` — creates ISO images (useful for cloud-init and VM provisioning)
- `virtinst` — the `virt-install` command for creating VMs from the CLI
- `libosinfo-bin` — OS information database, used by virt-manager to auto-configure VMs
- `virt-manager` — graphical interface for managing VMs
- `dnsmasq` — provides DHCP and DNS for the default NAT network

## Add Your User to the Required Groups

Add yourself to the `libvirt` group to manage VMs without root:

```bash
sudo adduser $USER libvirt
```

```bash
sudo adduser $USER libvirt-qemu
```

The `libvirt-qemu` group gives your user permission to access QEMU processes managed by libvirt. Without it, you may get permission denied errors when starting VMs.

Add yourself to the `kvm` group for direct KVM device access:

```bash
sudo addgroup "$(whoami)" libvirt
sudo addgroup "$(whoami)" kvm
```

The `/dev/kvm` device is what allows processes to use hardware virtualisation. Members of the `kvm` group can access it without root.

## Apply the Changes

Reboot for the group memberships to take effect:

```bash
sudo reboot
```

## Verify the Installation

After rebooting, confirm libvirt is running:

```bash
sudo systemctl status libvirtd
```

Check that your user can interact with libvirt without sudo:

```bash
virsh list --all
```

If it returns an empty list with no errors, the setup is correct. Open `virt-manager` from your application menu to create and manage VMs graphically.
