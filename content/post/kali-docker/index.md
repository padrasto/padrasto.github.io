---
title: "Kali Linux in Docker with Xfce and RDP Access"
date: 2023-02-28
categories: ["Kali-Linux", "Docker"]
tags: ["kali-linux", "docker", "xfce", "rdp", "sysadmin"]
---

Running Kali Linux inside Docker might seem like an odd choice at first, but it has real advantages. The main one is speed — Docker containers start in seconds, compared to the boot time of a full virtual machine. There's also the storage benefit: the base Kali image is minimal, and you only install what you actually need.

A typical use case: you need to run a specific tool like `nmap` or `nikto`, you don't want to keep a full VM around just for that, and you want to be up and running in under a minute.

## Running Kali from the Base Image

Pull the official Kali rolling image:

```bash
docker pull kalilinux/kali-rolling
```

The base image comes with no tools installed by default. That's intentional — you build on top of it with only what you need.

List your local images to confirm it's there:

```bash
docker images
```

Start an interactive container:

```bash
docker run -it kalilinux/kali-rolling
```

The `-it` flags attach your terminal to the container (`-i` keeps stdin open, `-t` allocates a pseudo-TTY). You'll land directly in a Kali shell.

From here, install whatever tools you need. For example:

```bash
apt update && apt install nmap
```

Or the full top 10 Kali tools metapackage:

```bash
apt install kali-tools-top10
```

## Saving Your Work with docker commit

The problem with `docker run` is that when you exit the container, any changes you made are gone. The container is destroyed and you're back to the base image.

To preserve your installed tools, commit the container as a new image. While the container is running, open a second terminal and find its ID:

```bash
docker ps
```

Then commit it:

```bash
docker commit <container_id> <new_image_name>
```

Example:

```bash
docker commit 0c448678d82d kali-custom
```

Next time, start from your saved image instead of the base:

```bash
docker run -it kali-custom
```

Your tools will be there, no reinstallation needed.

## Adding a GUI with Xfce and RDP

For a full graphical environment, you can install Xfce inside the container and access it remotely via RDP. This is useful when you need a desktop interface for tools that don't work well on the command line.

Start a new container with port 3389 forwarded (standard RDP port):

```bash
docker run -it -p 3389:3389 --name kali-desktop kalilinux/kali-rolling
```

Flag breakdown:
- `-it` — interactive terminal
- `-p 3389:3389` — maps port 3389 on the host to port 3389 in the container
- `--name kali-desktop` — gives the container a readable name instead of a random hash

Install the Xfce desktop and the xrdp server:

```bash
apt install kali-desktop-xfce xrdp
```

Set a password for the root user:

```bash
passwd root
```

Start the xrdp service:

```bash
service xrdp start
```

Now open any RDP client — Remmina works well on Linux — and connect to:

```
Host:     localhost:3389
Username: root
Password: (the one you set above)
```

You'll get a full Xfce desktop running inside the container.

When you're done, commit this container as well so your desktop setup is preserved:

```bash
docker commit kali-desktop kali-desktop-saved
```

## Notes

- The base image has no tools, which keeps it small. Install only what you need for each use case.
- For persistent work across sessions, `docker commit` is the quick approach. For something more structured, a `Dockerfile` is cleaner — but that's a separate topic.
- Running as root inside the container is fine for a local lab setup, but keep in mind that the container shares the host kernel.
