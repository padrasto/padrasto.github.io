---
title: "Linux User and Permissions Management: A Practical Guide"
date: 2026-03-29
tags: ["linux", "users", "permissions", "sysadmin", "troubleshooting"]
categories: ["Linux", "Troubleshooting"]
description: "A hands-on guide to managing users, groups, and file permissions on Linux — with real commands and common pitfalls."
---

User and permissions management is one of the most fundamental skills in Linux system administration. Get it wrong and you either lock legitimate users out or open doors that should stay closed. This article covers the essential commands and concepts you will use on a daily basis as a sysadmin.

## Users and Groups: The Basics

Every file and process on Linux is owned by a user and a group. Permissions are then defined for three categories:

- **owner** — the user who owns the file
- **group** — users belonging to the file's group
- **others** — everyone else

You can see this with `ls -l`:

```bash
ls -l /etc/nginx/nginx.conf
-rw-r--r-- 1 root root 1447 Mar 10 12:00 /etc/nginx/nginx.conf
```

Reading left to right: file type and permissions (`-rw-r--r--`), link count, owner (`root`), group (`root`), size, date, filename.

The permission string breaks down as:

```
- rw- r-- r--
│ │   │   └── others: read only
│ │   └────── group: read only
│ └────────── owner: read and write
└──────────── file type (- = file, d = directory, l = symlink)
```

## Managing Users

### Create a user

```bash
sudo useradd -m -s /bin/bash username
```

- `-m` creates the home directory
- `-s /bin/bash` sets the default shell

Set the password immediately after:

```bash
sudo passwd username
```

### Delete a user

```bash
sudo userdel username
```

To also remove the home directory:

```bash
sudo userdel -r username
```

### Modify a user

Change the default shell:

```bash
sudo usermod -s /bin/bash username
```

Lock an account (disables login without deleting):

```bash
sudo usermod -L username
```

Unlock it:

```bash
sudo usermod -U username
```

### View user information

```bash
id username
```

Output:

```
uid=1001(username) gid=1001(username) groups=1001(username),27(sudo)
```

This shows the user's UID, primary group GID, and all supplementary groups.

## Managing Groups

### Create a group

```bash
sudo groupadd groupname
```

### Add a user to a group

```bash
sudo usermod -aG groupname username
```

The `-a` flag is critical — without it, `usermod -G` **replaces** the user's group list instead of appending. This is a common mistake that can strip sudo access.

The change takes effect on the user's next login. To apply it in the current session:

```bash
newgrp groupname
```

### Remove a user from a group

```bash
sudo gpasswd -d username groupname
```

### List all groups a user belongs to

```bash
groups username
```

## File Permissions with chmod

### Symbolic mode

```bash
chmod u+x script.sh      # add execute for owner
chmod g-w file.txt       # remove write for group
chmod o+r file.txt       # add read for others
chmod a+x script.sh      # add execute for all (owner, group, others)
```

Letters: `u` = owner, `g` = group, `o` = others, `a` = all.
Operators: `+` = add, `-` = remove, `=` = set exactly.

### Numeric (octal) mode

Each permission has a numeric value: read = 4, write = 2, execute = 1. Add them together for each category.

| Octal | Permissions |
|---|---|
| 7 | rwx |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |
| 0 | --- |

Common examples:

```bash
chmod 644 file.txt      # owner rw, group r, others r  (typical for files)
chmod 755 script.sh     # owner rwx, group rx, others rx (typical for scripts)
chmod 700 ~/.ssh        # owner rwx only (required for SSH)
chmod 600 ~/.ssh/id_rsa # owner rw only (required for SSH private keys)
```

### Recursive chmod

Apply permissions to a directory and everything inside it:

```bash
chmod -R 755 /var/www/html
```

Be careful with `-R` — applying execute to all files in a directory is rarely what you want. A common pattern is to set directories and files separately:

```bash
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;
```

## Changing Ownership with chown

### Change the owner

```bash
sudo chown username file.txt
```

### Change the owner and group

```bash
sudo chown username:groupname file.txt
```

### Change only the group

```bash
sudo chown :groupname file.txt
# or
sudo chgrp groupname file.txt
```

### Recursive chown

```bash
sudo chown -R username:groupname /var/www/mysite
```

## sudo: Granting Elevated Privileges

### Add a user to the sudo group

On Debian/Ubuntu:

```bash
sudo usermod -aG sudo username
```

On Red Hat/CentOS:

```bash
sudo usermod -aG wheel username
```

### Edit the sudoers file safely

Never edit `/etc/sudoers` directly. Always use:

```bash
sudo visudo
```

This validates the syntax before saving, preventing a broken sudoers file that could lock you out entirely.

To allow a user to run a specific command without a password:

```
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

## Special Permissions

### setuid (SUID)

When set on an executable, it runs as the file's owner instead of the user who launched it. Classic example: `/usr/bin/passwd` runs as root so unprivileged users can change their own password.

```bash
chmod u+s /path/to/binary
# or
chmod 4755 /path/to/binary
```

Visible as `s` in the owner execute position: `-rwsr-xr-x`

### setgid (SGID)

On a directory, new files created inside inherit the directory's group rather than the creator's primary group. Useful for shared project directories.

```bash
chmod g+s /shared/project
# or
chmod 2755 /shared/project
```

### Sticky bit

On a directory, only the file's owner (or root) can delete it, even if others have write permission. Standard on `/tmp`.

```bash
chmod +t /shared/uploads
# or
chmod 1777 /shared/uploads
```

Visible as `t` in the others execute position: `drwxrwxrwt`

## Diagnosing Permission Problems

When a command fails with `Permission denied`, work through this checklist:

**1. Check what user you are running as:**

```bash
whoami
id
```

**2. Check the file permissions:**

```bash
ls -l /path/to/file
ls -ld /path/to/directory
```

Note: to access a file, you need execute permission on every directory in the path, not just the file itself.

**3. Check group membership:**

```bash
groups
```

If you just added yourself to a group, log out and back in (or use `newgrp`).

**4. Test with sudo to isolate the issue:**

```bash
sudo command
```

If it works as root but not as the user, the problem is permissions. If it fails as root too, the problem is elsewhere.

## Quick Reference

| Task | Command |
|---|---|
| Create user | `sudo useradd -m -s /bin/bash user` |
| Set password | `sudo passwd user` |
| Add to group | `sudo usermod -aG group user` |
| Check user info | `id user` |
| Check file permissions | `ls -l file` |
| Change permissions | `chmod 644 file` |
| Change owner | `sudo chown user:group file` |
| Edit sudoers | `sudo visudo` |

## Summary

User and permissions management comes down to three questions: who owns this file, what are the permission bits, and what groups is the user in? When something fails with a permission error, `ls -l`, `id`, and `groups` will answer all three in seconds. From there, `chmod`, `chown`, and `usermod` fix the vast majority of problems you will encounter.
