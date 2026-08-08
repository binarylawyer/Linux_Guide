---
layout: page
title: Binary Lawyer
---

# Aerostat's Guide To Linux
### (Ubuntu Edition)

[**Web View**](https://binarylawyer.github.io/Linux_Guide/)
[**GitHub View**](https://github.com/binarylawyer/Linux_Guide/blob/master/index.md)

### A Simpler Guide To Linux.
#### An Aerostat & Co. pub.

**Version 1.0**

---

## Introduction

This is a guide to setting up and hardening a Linux server. It covers Ubuntu 22.04 LTS and includes links to additional resources, but it will never be complete — that's where you come in.

You can submit links, corrections, or design suggestions via pull request. This guide is published on GitHub so that Aerostat and the broader community can contribute to something that is a public good.

Feel free to use this guide and fork it as much as you like. If it helps you learn Linux and Git a little bit faster, then this whole project is worth it.

---

## Table of Contents

1. [Initial Server Setup](#initial-server-setup)
2. [Locking Down the OS](#locking-down-the-os)
3. [User Account Hardening](#user-account-hardening)
4. [SSH Hardening](#ssh-hardening)
5. [Firewall Configuration](#firewall-configuration)
6. [Automatic Updates](#automatic-updates)
7. [Encrypt Data Communication](#encrypt-data-communication)
8. [Filesystem Security](#filesystem-security)
9. [Monitoring and Auditing](#monitoring-and-auditing)
10. [Useful Commands](#useful-commands)
11. [Resources](#resources)

---

## Initial Server Setup

After first booting your Ubuntu server, run the following to update all packages:

```bash
sudo apt update && sudo apt upgrade -y
```

Set your hostname:

```bash
sudo hostnamectl set-hostname your-server-name
```

Set your timezone:

```bash
sudo timedatectl set-timezone America/New_York
```

---

## Locking Down the OS

### Lock the Boot Directory

The `/boot` directory contains important files related to the Linux kernel. Lock it to read-only by adding the following line to `/etc/fstab`:

```
LABEL=/boot   /boot   ext2   defaults,ro   1 2
```

> **Note:** Remember to set it back to read-write temporarily when upgrading the kernel.

### Disable Unused Filesystems

Prevent loading of uncommon filesystem types by creating a blacklist file:

```bash
sudo nano /etc/modprobe.d/uncommon-fs.conf
```

Add the following:

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

---

## User Account Hardening

### Disable Root Login

Edit `/etc/passwd` and change the root shell to `/sbin/nologin`:

```
root:x:0:0:root:/root:/sbin/nologin
```

### Set Strong Password Policies

Install `libpam-pwquality` and configure password strength:

```bash
sudo apt install libpam-pwquality -y
```

Edit `/etc/security/pwquality.conf`:

```
minlen = 14
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
```

### Lock Inactive Accounts

```bash
sudo useradd -D -f 30
```

---

## SSH Hardening

Edit `/etc/ssh/sshd_config` to apply the following settings:

```
Port 2222                        # Change the default port
PermitRootLogin no               # Disable root login over SSH
PasswordAuthentication no        # Use key-based authentication only
X11Forwarding no
MaxAuthTries 3
AllowUsers your_username
```

Restart SSH after making changes:

```bash
sudo systemctl restart sshd
```

---

## Firewall Configuration

### UFW (Uncomplicated Firewall)

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp       # Your custom SSH port
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

### Fail2Ban

Install Fail2Ban to block repeated failed login attempts:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Create a local jail config at `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 3600
```

---

## Automatic Updates

Enable unattended security upgrades:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Encrypt Data Communication

Use `scp`, `ssh`, `rsync`, or `sftp` for all file transfers. You can also mount a remote server filesystem using `sshfs`:

```bash
sudo apt install sshfs -y
sshfs user@remote_host:/remote/path /local/mountpoint
```

### Generate SSH Keys

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id -p 2222 user@remote_host
```

---

## Filesystem Security

### Set Correct Permissions on Key Files

```bash
chmod 700 /root
chmod 600 /etc/ssh/sshd_config
chmod 644 /etc/passwd
chmod 640 /etc/shadow
```

### Enable Auditing with `auditd`

```bash
sudo apt install auditd -y
sudo systemctl enable auditd
sudo systemctl start auditd
```

Watch critical files for changes:

```bash
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
sudo auditctl -w /etc/shadow -p wa -k shadow_changes
sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes
```

---

## Monitoring and Auditing

### Check Listening Ports

```bash
ss -tulpn
```

### Check Logged-in Users

```bash
who
w
last
```

### Check Running Services

```bash
sudo systemctl list-units --type=service --state=running
```

### Rootkit Detection

```bash
sudo apt install rkhunter chkrootkit -y
sudo rkhunter --update
sudo rkhunter --check
sudo chkrootkit
```

---

## Useful Commands

| Command | Description |
|---|---|
| `uname -a` | Show kernel version and system info |
| `df -h` | Show disk usage |
| `free -h` | Show memory usage |
| `top` / `htop` | Display running processes |
| `journalctl -xe` | View system logs |
| `netstat -tulpn` | Show open network ports |
| `lsof -i` | List open files and connections |
| `ps aux` | List all running processes |
| `chmod`, `chown` | Change file permissions/ownership |
| `sudo visudo` | Safely edit sudoers file |

---

## Resources

1. [Book of Zeus – Harden Ubuntu](http://bookofzeus.com/harden-ubuntu/)
2. [Pluralsight – Linux Server Hardening Checklist](https://www.pluralsight.com/blog/it-ops/linux-hardening-secure-server-checklist)
3. [Ubuntu Security Documentation](https://ubuntu.com/security)
4. [CIS Ubuntu Benchmark](https://www.cisecurity.org/benchmark/ubuntu_linux)
5. [Lynis – Security Auditing Tool](https://cisofy.com/lynis/)
6. [OpenSSH Security Best Practices](https://infosec.mozilla.org/guidelines/openssh)

---

## How to Contribute

This guide is open source. If you'd like to contribute:

- Fork this repository on GitHub
- Create a branch for your changes
- Submit a pull request with your additions or corrections

All contributions are welcome and appreciated.

---

*Published by Aerostat & Co. — Use freely, fork widely.*
