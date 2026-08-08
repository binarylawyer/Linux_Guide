---
layout: page
title: Binary Lawyer
---

# Aerostat's Guide To Linux
### (Ubuntu Edition)

[**Web View**](https://binarylawyer.github.io/Linux_Guide/) | [**GitHub View**](https://github.com/binarylawyer/Linux_Guide/blob/master/index.md)

### A Simpler Guide To Linux
#### An Aerostat & Co. pub.

**Version 1.0**

---

## Introduction

This is a guide to setting up and hardening a Linux server running **Ubuntu 22.04 LTS**.

This guide links to useful resources but will never be complete — that's where you come in.

You can send in your links to this guide or simple design suggestions. It's all welcome and appreciated.

This guide was published on GitHub so that Aerostat and LOCM folks can contribute to something that is a public good.

Feel free to use this guide and fork it as much as you like. If it helps you learn Linux and Git a little bit faster, then this whole project is worth it.

> This version is focused on **securing and hardening servers**.

---

## Table of Contents

1. [Initial Server Setup](#initial-server-setup)
2. [Locking Down the OS](#locking-down-the-os)
3. [User & SSH Hardening](#user--ssh-hardening)
4. [Firewall Setup](#firewall-setup)
5. [Encrypting Data Communication](#encrypting-data-communication)
6. [System Updates & Patch Management](#system-updates--patch-management)
7. [Monitoring & Logging](#monitoring--logging)
8. [Useful Commands Reference](#useful-commands-reference)
9. [Resources & Links](#resources--links)

---

## Initial Server Setup

After first login as root, create a new sudo user:

```bash
adduser newuser
usermod -aG sudo newuser
```

Switch to the new user:

```bash
su - newuser
```

---

## Locking Down the OS

### Lock the Boot Directory

The `/boot` directory contains critical Linux kernel files. Lock it to read-only by editing `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add or update the entry for `/boot`:

```
LABEL=/boot   /boot   ext2   defaults,ro   1 2
```

After making changes, remount:

```bash
sudo mount -o remount /boot
```

### Disable Unused Filesystems

Prevent loading of uncommon filesystem types by creating a config file:

```bash
sudo nano /etc/modprobe.d/disable-filesystems.conf
```

Add:

```
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
install vfat /bin/true
```

### Secure Shared Memory

Edit `/etc/fstab` to add:

```
tmpfs   /run/shm   tmpfs   defaults,noexec,nosuid   0 0
```

---

## User & SSH Hardening

### Disable Root SSH Login

Edit the SSH configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```
PermitRootLogin no
PasswordAuthentication no
AllowUsers newuser
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

### Use SSH Key Authentication

Generate a key pair on your local machine:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Copy the public key to the server:

```bash
ssh-copy-id newuser@your_server_ip
```

### Set a Strong Password Policy

Install and configure `libpam-pwquality`:

```bash
sudo apt install libpam-pwquality
sudo nano /etc/security/pwquality.conf
```

Example settings:

```
minlen = 14
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
```

---

## Firewall Setup

### Enable UFW (Uncomplicated Firewall)

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

---

## Encrypting Data Communication

Use encrypted protocols for all file transfers:

| Tool | Use Case |
|------|----------|
| `scp` | Secure copy over SSH |
| `sftp` | Interactive secure FTP |
| `rsync` | Efficient sync over SSH |
| `sshfs` | Mount remote filesystem locally |

Example using `rsync`:

```bash
rsync -avz -e ssh /local/path/ user@server:/remote/path/
```

---

## System Updates & Patch Management

Keep the system up to date:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

Enable automatic security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Monitoring & Logging

### Check Failed Login Attempts

```bash
sudo journalctl -u ssh --since "1 hour ago"
sudo grep "Failed password" /var/log/auth.log
```

### Install Fail2Ban

Fail2Ban automatically bans IPs with repeated failed login attempts:

```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check status:

```bash
sudo fail2ban-client status sshd
```

### Monitor System Resources

```bash
top          # Real-time process monitor
htop         # Enhanced process viewer (install with: sudo apt install htop)
df -h        # Disk usage
free -h      # Memory usage
netstat -tuln  # Open network ports
```

---

## Useful Commands Reference

| Command | Description |
|---------|-------------|
| `ls -la` | List files with details and hidden files |
| `chmod 700 file` | Set file permissions |
| `chown user:group file` | Change file ownership |
| `grep -r "pattern" /dir` | Search recursively |
| `find / -name filename` | Find a file by name |
| `tar -czf archive.tar.gz dir/` | Create compressed archive |
| `tar -xzf archive.tar.gz` | Extract compressed archive |
| `sudo systemctl status service` | Check service status |
| `sudo journalctl -xe` | View system logs |
| `uname -r` | Show kernel version |
| `lsb_release -a` | Show OS version |

---

## Resources & Links

1. [Book of Zeus — Harden Ubuntu](http://bookofzeus.com/harden-ubuntu/)
2. [Pluralsight — Linux Server Hardening Checklist](https://www.pluralsight.com/blog/it-ops/linux-hardening-secure-server-checklist)
3. [Ubuntu Security Guide](https://ubuntu.com/security/certifications/docs/usg)
4. [DigitalOcean — Initial Server Setup with Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)
5. [Fail2Ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)
6. [UFW Essentials](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)
7. [SSH Hardening Guide](https://www.ssh.com/academy/ssh/sshd_config)
8. [Linux Command Line Cheat Sheet](https://cheatography.com/davechild/cheat-sheets/linux-command-line/)

---

*This guide is a living document. Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).*
