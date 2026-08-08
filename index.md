---
layout: page
title: Linux Guide
---

# Aerostat's Guide To Linux
### Ubuntu LTS edition

[**Web View**](https://binarylawyer.github.io/Linux_Guide/) ·
[**GitHub View**](https://github.com/binarylawyer/Linux_Guide/blob/master/index.md)

### A simpler guide to Linux
#### an Aerostat & Co. publication

**Version 1.0**

This guide covers setting up and hardening an Ubuntu LTS server. It will never be complete — contributions are welcome.

Fork it, improve it, and share what helps you learn Linux and Git faster. That is the point of the project.

---

## Table of contents

1. [Who this is for](#who-this-is-for)
2. [Choose an Ubuntu LTS release](#choose-an-ubuntu-lts-release)
3. [Initial server setup](#initial-server-setup)
4. [Lock down SSH](#lock-down-ssh)
5. [Firewall with UFW](#firewall-with-ufw)
6. [Keep the system updated](#keep-the-system-updated)
7. [Create a non-root admin user](#create-a-non-root-admin-user)
8. [Fail2ban](#fail2ban)
9. [Encrypt data in transit](#encrypt-data-in-transit)
10. [Protect the boot configuration](#protect-the-boot-configuration)
11. [Basic filesystem and permission hygiene](#basic-filesystem-and-permission-hygiene)
12. [Logging and monitoring basics](#logging-and-monitoring-basics)
13. [Optional hardening checklist](#optional-hardening-checklist)
14. [Further reading](#further-reading)
15. [Contributing](#contributing)

---

## Who this is for

- People standing up a fresh Ubuntu VPS or cloud instance
- Developers who want a practical hardening baseline
- Contributors who want a living checklist rather than a textbook

This is not a substitute for your cloud provider’s shared-responsibility model, compliance requirements, or a full security audit.

---

## Choose an Ubuntu LTS release

Prefer a current **Long Term Support (LTS)** release:

| Release | Codename | Notes |
|---------|----------|--------|
| Ubuntu 24.04 LTS | Noble Numbat | Current LTS (recommended for new servers) |
| Ubuntu 22.04 LTS | Jammy Jellyfish | Still widely supported |
| Ubuntu 20.04 LTS | Focal Fossa | Approaching end of standard support — plan upgrades |

Avoid non-LTS releases on production servers unless you have a reason to track the interim cycle.

---

## Initial server setup

After the provider finishes installing Ubuntu:

1. Log in with the credentials they gave you (often `root` or a default sudo user over SSH).
2. Confirm the release:

```bash
lsb_release -a
uname -r
```

3. Set the hostname and timezone:

```bash
sudo hostnamectl set-hostname your-server-name
sudo timedatectl set-timezone UTC
```

4. Sync the package index and apply updates (see [Keep the system updated](#keep-the-system-updated)).

---

## Lock down SSH

SSH is the most common remote entry point. Harden it before you open the box to the internet for real work.

### Prefer key-based authentication

On your **local** machine:

```bash
ssh-keygen -t ed25519 -C "you@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@your-server
```

Confirm you can log in with the key **before** disabling password authentication.

### Edit the SSH daemon config

```bash
sudo nano /etc/ssh/sshd_config
```

Recommended baseline (adjust to your environment):

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
AllowUsers youradmin
ClientAliveInterval 300
ClientAliveCountMax 2
```

Optional but useful:

- Change `Port 22` only if you understand the operational trade-offs (security through obscurity is weak; combine with firewall + keys + fail2ban).
- Use `AllowUsers` / `AllowGroups` to limit who may connect.

Validate and reload:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Keep an existing session open until a **new** SSH login succeeds.

---

## Firewall with UFW

[UFW](https://help.ubuntu.com/community/UFW) is a simple front end to iptables/nftables.

```bash
sudo apt update
sudo apt install ufw
```

Allow SSH **before** enabling the firewall:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
# If you changed the SSH port: sudo ufw allow 2222/tcp
sudo ufw enable
sudo ufw status verbose
```

Only open ports you actually need (`80/tcp`, `443/tcp`, etc.).

---

## Keep the system updated

```bash
sudo apt update
sudo apt full-upgrade
sudo apt autoremove --purge
```

Enable unattended security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Reboot when the kernel or critical libraries require it:

```bash
sudo reboot
```

---

## Create a non-root admin user

Do not day-to-day as `root`.

```bash
sudo adduser youradmin
sudo usermod -aG sudo youradmin
```

Copy your SSH public key into that user’s `~/.ssh/authorized_keys`, then log in as `youradmin` and confirm `sudo` works:

```bash
sudo whoami
```

After that, disable root SSH login as shown above.

---

## Fail2ban

Fail2ban watches logs and bans repeated offenders at the firewall.

```bash
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
```

Create a local jail override so package upgrades do not wipe your settings:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Ensure the SSH jail is enabled, then:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

---

## Encrypt data in transit

Never use clear-text tools for remote administration or file transfer on untrusted networks.

| Prefer | Avoid (on the public internet) |
|--------|--------------------------------|
| `ssh`, `scp`, `sftp`, `rsync` over SSH | `telnet`, unencrypted `ftp`, `rsh` |
| HTTPS / TLS for web and APIs | plain HTTP for sensitive traffic |
| VPN or WireGuard for private nets | ad-hoc open management ports |

Mount a remote directory over SSH when needed:

```bash
sudo apt install sshfs
sshfs user@remote:/path ./local-mount
```

---

## Protect the boot configuration

The boot path holds kernel and bootloader files. Treat it carefully.

### Make `/boot` read-only when appropriate

If `/boot` is a separate partition, you can mount it read-only in `/etc/fstab` after updates, then remount read-write only when installing kernels:

```bash
sudo nano /etc/fstab
```

Example option set (only if you understand your partition layout):

```text
UUID=your-boot-uuid  /boot  ext4  defaults,ro  0  2
```

Before kernel upgrades:

```bash
sudo mount -o remount,rw /boot
```

Afterward:

```bash
sudo mount -o remount,ro /boot
```

**Warning:** misconfigured `fstab` can make a system unbootable. Take a snapshot or console access plan first.

### Firmware and secure boot

On bare metal, enable UEFI Secure Boot when your stack supports it. Keep firmware updated through the vendor or `fwupd` where available:

```bash
sudo apt install fwupd
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

---

## Basic filesystem and permission hygiene

- Keep home directories `750` or tighter; keep private keys at `600`.
- Avoid world-writable directories (`find / -xdev -type d -perm -0002` as a periodic check).
- Put application data on dedicated volumes when possible; separate OS and data for easier backups and recovery.
- Use full-disk encryption (LUKS) on laptops and sensitive hosts at install time.

Example permission check for SSH files:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## Logging and monitoring basics

```bash
# Recent auth failures / logins
sudo journalctl -u ssh --since "1 day ago"
last -a | head
sudo grep "Failed password" /var/log/auth.log | tail
```

Forward logs off-box if the host is important (cloud logging agent, rsyslog/journal remote, or a SIEM). Local-only logs disappear if the disk is wiped.

Consider:

- Automatic snapshots from your cloud provider
- Off-site backups tested with an actual restore
- Disk and load alerts (simple cron + email is better than nothing)

---

## Optional hardening checklist

Use this as a post-install punch list:

- [ ] Ubuntu LTS installed and fully updated
- [ ] Timezone and hostname set
- [ ] Non-root sudo user created; root SSH disabled
- [ ] SSH key auth only; passwords disabled for SSH
- [ ] UFW enabled with least-privilege ports
- [ ] fail2ban running for SSH
- [ ] unattended-upgrades enabled
- [ ] Unnecessary packages and services removed
- [ ] Application secrets not stored in the repo or world-readable files
- [ ] Backups configured and restore-tested
- [ ] Provider firewall / security groups aligned with UFW
- [ ] Documentation of what changed (for your future self)

Advanced topics (out of scope for v1.0, good follow-ups):

- AppArmor profiles for exposed daemons
- auditd rules for privileged actions
- Disk quotas and separate partitions (`/var`, `/tmp`, `/home`)
- 2FA for SSH (`libpam-google-authenticator` or hardware keys)
- Configuration management (Ansible, cloud-init)

---

## Further reading

1. [Ubuntu Server Guide](https://ubuntu.com/server/docs)
2. [CIS Ubuntu Linux Benchmarks](https://www.cisecurity.org/benchmark/ubuntu_linux)
3. [Book of Zeus — Harden Ubuntu](http://bookofzeus.com/harden-ubuntu/)
4. [Linux server hardening checklist (Pluralsight)](https://www.pluralsight.com/blog/it-ops/linux-hardening-secure-server-checklist)
5. [OpenSSH manual pages](https://man.openbsd.org/sshd_config)
6. [UFW community help](https://help.ubuntu.com/community/UFW)
7. [fail2ban](https://github.com/fail2ban/fail2ban)

---

## Contributing

This guide is intentionally public-domain-friendly learning material for Aerostat, LOCM, and anyone else who wants a clearer path into Linux ops.

- Open a pull request with fixes, clearer wording, or new sections
- File issues for broken links or dangerous advice
- Keep suggestions practical and tested on Ubuntu LTS when possible

Previous homepage content (Bootstrap template notes mixed with an earlier draft) is preserved in [`index-backup-1.md`](./index-backup-1.md).

If the guide helped you and you want to support the publication, reach out for donation wallet details (ETH, NEAR, and similar). Forks and improvements are the best contribution.
