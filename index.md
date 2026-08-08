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

**Version 1.1**

---

## Introduction

This is a practical guide to setting up and hardening a Linux server, focused on **Ubuntu 22.04 LTS**. It covers everything from initial provisioning through advanced security controls, with curated links to authoritative external resources throughout.

This guide is published on GitHub so that Aerostat and the broader community can contribute. You can submit links, corrections, or new sections via pull request — all contributions are welcome.

Feel free to use this guide and fork it freely. If it helps you learn Linux security a little bit faster, this whole project is worth it.

---

## Table of Contents

1. [Initial Server Setup](#initial-server-setup)
2. [Locking Down the OS](#locking-down-the-os)
3. [Kernel Hardening with sysctl](#kernel-hardening-with-sysctl)
4. [User Account Hardening](#user-account-hardening)
5. [Sudo Configuration](#sudo-configuration)
6. [SSH Hardening](#ssh-hardening)
7. [Two-Factor Authentication (2FA)](#two-factor-authentication-2fa)
8. [Firewall Configuration](#firewall-configuration)
9. [Automatic Updates](#automatic-updates)
10. [Encrypt Data Communication](#encrypt-data-communication)
11. [TLS/SSL Best Practices](#tlsssl-best-practices)
12. [Filesystem Security](#filesystem-security)
13. [Intrusion Detection with AIDE](#intrusion-detection-with-aide)
14. [Log Management](#log-management)
15. [Monitoring and Auditing](#monitoring-and-auditing)
16. [Container Security (Docker)](#container-security-docker)
17. [Useful Commands](#useful-commands)
18. [Resources](#resources)

---

## Initial Server Setup

After first booting your Ubuntu server, update all packages immediately:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

Set your hostname:

```bash
sudo hostnamectl set-hostname your-server-name
```

Set your timezone:

```bash
sudo timedatectl set-timezone America/New_York
# List available timezones: timedatectl list-timezones
```

Synchronize time with NTP (essential for log accuracy and certificate validation):

```bash
sudo systemctl enable systemd-timesyncd
sudo systemctl start systemd-timesyncd
timedatectl status
```

Remove unnecessary packages to reduce the attack surface:

```bash
sudo apt purge telnet ftp rsh-client rsh-redone-client -y
sudo apt autoremove -y
```

> 📖 **Reference:** [Ubuntu Server Initial Setup – DigitalOcean](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)

---

## Locking Down the OS

### Lock the Boot Directory

The `/boot` directory contains important files related to the Linux kernel. Lock it to read-only by adding the following line to `/etc/fstab`:

```
LABEL=/boot   /boot   ext2   defaults,ro   1 2
```

> **Note:** Remember to set it back to read-write (`rw`) temporarily when upgrading the kernel:
> `sudo mount -o remount,rw /boot`

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

### Disable Core Dumps

Core dumps can expose sensitive data. Disable them in `/etc/security/limits.conf`:

```
* hard core 0
* soft core 0
```

And in `/etc/sysctl.conf`:

```
fs.suid_dumpable = 0
```

### Disable USB Storage (optional, for high-security environments)

```bash
echo "install usb-storage /bin/true" | sudo tee /etc/modprobe.d/disable-usb-storage.conf
```

> 📖 **Reference:** [CIS Ubuntu Linux Benchmark (PDF)](https://www.cisecurity.org/benchmark/ubuntu_linux)

---

## Kernel Hardening with sysctl

The Linux kernel exposes a range of tunable parameters via `sysctl`. Applying secure defaults protects against network attacks, privilege escalation, and information leakage.

Create or edit `/etc/sysctl.d/99-hardening.conf`:

```ini
# --- IP Spoofing Protection ---
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# --- Ignore ICMP Broadcast Requests ---
net.ipv4.icmp_echo_ignore_broadcasts = 1

# --- Ignore Bogus ICMP Error Responses ---
net.ipv4.icmp_ignore_bogus_error_responses = 1

# --- SYN Flood Protection ---
net.ipv4.tcp_syncookies = 1

# --- Disable IP Source Routing ---
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# --- Disable ICMP Redirects ---
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# --- Log Suspicious Packets ---
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# --- Disable IPv6 if not needed ---
# net.ipv6.conf.all.disable_ipv6 = 1

# --- Protect against SUID/SGID privilege escalation ---
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2

# --- Prevent ptrace attacks ---
kernel.yama.ptrace_scope = 1

# --- Disable Magic SysRq Key ---
kernel.sysrq = 0

# --- Randomize virtual address space (ASLR) ---
kernel.randomize_va_space = 2
```

Apply immediately:

```bash
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
```

> 📖 **Reference:** [Kernel Self-Protection Project](https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project)
> 📖 **Reference:** [sysctl Hardening – Arch Wiki](https://wiki.archlinux.org/title/Security#Kernel_parameters)

---

## User Account Hardening

### Create a Non-Root Admin User

Never use `root` for daily tasks. Create a dedicated admin user:

```bash
adduser adminuser
usermod -aG sudo adminuser
```

### Disable the Root Account

```bash
sudo passwd -l root
```

Or set its shell to `/usr/sbin/nologin` in `/etc/passwd`:

```
root:x:0:0:root:/root:/usr/sbin/nologin
```

### Set Strong Password Policies

Install `libpam-pwquality` and configure password strength:

```bash
sudo apt install libpam-pwquality -y
```

Edit `/etc/security/pwquality.conf`:

```
minlen = 14
minclass = 4
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
maxrepeat = 3
```

Set password aging in `/etc/login.defs`:

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14
```

Apply aging to an existing user:

```bash
sudo chage -M 90 -m 7 -W 14 username
```

### Lock Inactive Accounts

Automatically disable accounts that have not been used in 30 days:

```bash
sudo useradd -D -f 30
```

### Restrict `su` to the Wheel Group

Edit `/etc/pam.d/su` and uncomment or add:

```
auth required pam_wheel.so use_uid
```

> 📖 **Reference:** [Linux User Account Security – SANS Institute](https://www.sans.org/reading-room/whitepapers/linux/)

---

## Sudo Configuration

Edit the sudoers file safely with `visudo`:

```bash
sudo visudo
```

Recommended settings:

```
# Require password for sudo every time (no caching)
Defaults timestamp_timeout=0

# Log all sudo commands
Defaults logfile="/var/log/sudo.log"

# Restrict sudo to a specific group
%sudo ALL=(ALL:ALL) ALL

# Limit specific users to specific commands
deployuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

Audit who has sudo access:

```bash
grep -Po '^sudo.+:\K.*$' /etc/group
```

> 📖 **Reference:** [sudoers Manual (sudo.ws)](https://www.sudo.ws/docs/man/sudoers.man/)

---

## SSH Hardening

Edit `/etc/ssh/sshd_config` to apply the following settings:

```
# Change the default port to reduce noise from automated scanners
Port 2222

# Disable root login
PermitRootLogin no

# Require key-based authentication only
PasswordAuthentication no
ChallengeResponseAuthentication no

# Disable unused features
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no

# Restrict authentication attempts
MaxAuthTries 3
MaxSessions 2

# Only allow specific users
AllowUsers your_username

# Set a login grace period
LoginGraceTime 30

# Use modern key exchange and ciphers only
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Disconnect idle sessions
ClientAliveInterval 300
ClientAliveCountMax 2
```

Restart SSH after making changes (keep your current session open until you verify):

```bash
sudo sshd -t          # Test configuration for syntax errors
sudo systemctl restart sshd
```

### Generate Strong SSH Keys

Use Ed25519 (preferred) or RSA-4096:

```bash
# Ed25519 (recommended)
ssh-keygen -t ed25519 -C "your_email@example.com"

# RSA 4096-bit (legacy compatibility)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Copy public key to server
ssh-copy-id -p 2222 user@remote_host
```

> 📖 **Reference:** [Mozilla OpenSSH Guidelines](https://infosec.mozilla.org/guidelines/openssh)
> 📖 **Reference:** [SSH Audit Tool (ssh-audit.com)](https://www.ssh-audit.com/)

---

## Two-Factor Authentication (2FA)

Adding TOTP-based 2FA to SSH provides an extra layer of protection even if a private key is compromised.

Install Google Authenticator PAM module:

```bash
sudo apt install libpam-google-authenticator -y
```

Run the setup as your user:

```bash
google-authenticator
```

Follow the prompts to configure TOTP. Then edit `/etc/pam.d/sshd`:

```
# Add at the top:
auth required pam_google_authenticator.so
```

In `/etc/ssh/sshd_config`, enable challenge-response:

```
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

> 📖 **Reference:** [How To Set Up Multi-Factor Authentication for SSH – DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-multi-factor-authentication-for-ssh-on-ubuntu-20-04)

---

## Firewall Configuration

### UFW (Uncomplicated Firewall)

UFW is the recommended frontend for `iptables` on Ubuntu:

```bash
sudo apt install ufw -y

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential services (use your actual SSH port)
sudo ufw allow 2222/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Rate-limit SSH to prevent brute force
sudo ufw limit 2222/tcp

sudo ufw enable
sudo ufw status verbose
```

### Fail2Ban

Fail2Ban monitors log files and bans IPs that show malicious behavior:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Create a local jail config at `/etc/fail2ban/jail.local` (this overrides defaults and survives upgrades):

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 3
destemail = admin@yourdomain.com
action = %(action_mwl)s

[sshd]
enabled  = true
port     = 2222
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 86400

[nginx-http-auth]
enabled  = true
```

Check banned IPs:

```bash
sudo fail2ban-client status sshd
```

> 📖 **Reference:** [Fail2Ban Documentation](https://www.fail2ban.org/wiki/index.php/MANUAL_0_8)
> 📖 **Reference:** [UFW Essentials – DigitalOcean](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)

---

## Automatic Updates

Enable unattended security upgrades to ensure critical patches are applied automatically:

```bash
sudo apt install unattended-upgrades apt-listchanges -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Edit `/etc/apt/apt.conf.d/50unattended-upgrades` to enable automatic reboots (optional):

```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
Unattended-Upgrade::Mail "admin@yourdomain.com";
```

Verify it runs:

```bash
sudo unattended-upgrade --dry-run --debug
```

> 📖 **Reference:** [Automatic Updates – Ubuntu Documentation](https://help.ubuntu.com/community/AutomaticSecurityUpdates)

---

## Encrypt Data Communication

Use `scp`, `ssh`, `rsync`, or `sftp` for all file transfers — never FTP or Telnet in plaintext.

### Secure Remote File Transfers

```bash
# Copy a file to remote server
scp -P 2222 file.txt user@remote:/path/

# Sync a directory (efficient, only transfers diffs)
rsync -avz -e "ssh -p 2222" ./local/ user@remote:/remote/

# Mount remote filesystem locally
sudo apt install sshfs -y
sshfs -p 2222 user@remote_host:/remote/path /local/mountpoint
```

### Verify Remote Host Fingerprints

Always verify a server's host key on first connection:

```bash
ssh-keyscan -p 2222 remote_host | ssh-keygen -lf -
```

### Use GPG for File Encryption at Rest

```bash
sudo apt install gnupg -y

# Encrypt a file
gpg --symmetric --cipher-algo AES256 sensitive_file.txt

# Decrypt
gpg sensitive_file.txt.gpg
```

> 📖 **Reference:** [Rsync + SSH – man page](https://linux.die.net/man/1/rsync)
> 📖 **Reference:** [GnuPG Documentation](https://www.gnupg.org/documentation/)

---

## TLS/SSL Best Practices

### Obtain Free TLS Certificates with Let's Encrypt

```bash
sudo apt install certbot -y

# For Nginx
sudo apt install python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renewal is handled by a systemd timer — verify:
sudo systemctl status certbot.timer
```

### Test Your TLS Configuration

After configuring HTTPS, test your server's TLS grade:

- [SSL Labs Server Test (ssllabs.com)](https://www.ssllabs.com/ssltest/) — industry-standard TLS grader
- [testssl.sh](https://testssl.sh/) — command-line TLS scanner you can run locally

Recommended Nginx TLS settings (`/etc/nginx/snippets/ssl-params.conf`):

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header Referrer-Policy "no-referrer-when-downgrade";
```

> 📖 **Reference:** [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
> 📖 **Reference:** [Let's Encrypt Documentation](https://letsencrypt.org/docs/)

---

## Filesystem Security

### Set Correct Permissions on Key Files

```bash
chmod 700 /root
chmod 600 /etc/ssh/sshd_config
chmod 644 /etc/passwd
chmod 640 /etc/shadow
chmod 440 /etc/sudoers
chmod 600 /boot/grub/grub.cfg
```

### Secure /tmp and /var/tmp

Mount `/tmp` with `noexec`, `nosuid`, and `nodev` to prevent execution of malicious scripts:

```
tmpfs   /tmp       tmpfs   defaults,noexec,nosuid,nodev   0 0
tmpfs   /var/tmp   tmpfs   defaults,noexec,nosuid,nodev   0 0
```

### Enable Auditing with `auditd`

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable auditd
sudo systemctl start auditd
```

Add rules to `/etc/audit/rules.d/audit.rules`:

```
# Watch authentication files
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/group -p wa -k group_changes
-w /etc/sudoers -p wa -k sudoers_changes

# Monitor SSH configuration
-w /etc/ssh/sshd_config -p wa -k sshd_config

# Detect privilege escalation
-a always,exit -F arch=b64 -S setuid -k priv_esc
-a always,exit -F arch=b64 -S setgid -k priv_esc

# Track all commands run as root
-a exit,always -F arch=b64 -F euid=0 -S execve -k root_commands
```

Apply rules without reboot:

```bash
sudo auditctl -R /etc/audit/rules.d/audit.rules
```

Search audit logs:

```bash
sudo ausearch -k passwd_changes
sudo aureport --auth
```

> 📖 **Reference:** [Linux Audit Documentation (linux-audit.com)](https://linux-audit.com/)

---

## Intrusion Detection with AIDE

AIDE (Advanced Intrusion Detection Environment) monitors filesystem integrity by creating a baseline database and alerting you to unexpected changes.

```bash
sudo apt install aide aide-common -y

# Initialize the baseline database (run after setting up the server)
sudo aideinit
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Run a check (compare current state to baseline)
sudo aide --check
```

Automate daily checks with cron:

```bash
sudo crontab -e
```

Add:

```
0 3 * * * /usr/bin/aide --check | mail -s "AIDE Report: $(hostname)" admin@yourdomain.com
```

> 📖 **Reference:** [AIDE Manual](https://aide.github.io/)

---

## Log Management

### Centralized Logging with rsyslog

Ubuntu uses `rsyslog` by default. To forward logs to a central log server:

Edit `/etc/rsyslog.conf`:

```
*.* @logserver.yourdomain.com:514   # UDP
*.* @@logserver.yourdomain.com:514  # TCP (more reliable)
```

### Log Rotation with logrotate

Check configuration at `/etc/logrotate.conf` and `/etc/logrotate.d/`. Example for a custom app:

```
/var/log/myapp/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
}
```

### Systemd Journal

```bash
# View all logs since last boot
journalctl -b

# Follow logs in real time
journalctl -f

# Filter by service
journalctl -u nginx.service

# View authentication logs
journalctl _COMM=sshd
```

Persist journal logs across reboots (edit `/etc/systemd/journald.conf`):

```
Storage=persistent
```

> 📖 **Reference:** [Linux Logging – Loggly Guide](https://www.loggly.com/ultimate-guide/linux-logging-basics/)
> 📖 **Reference:** [journalctl – man page](https://www.man7.org/linux/man-pages/man1/journalctl.1.html)

---

## Monitoring and Auditing

### Check Listening Ports

```bash
ss -tulpn
# or
sudo netstat -tulpn
```

### Check Logged-in Users

```bash
who          # Currently logged in
w            # Logged in with activity
last         # Login history
lastfail     # Failed login attempts
```

### Check Running Services

```bash
sudo systemctl list-units --type=service --state=running
```

Disable services you don't need:

```bash
sudo systemctl disable --now cups bluetooth avahi-daemon
```

### System Security Audit with Lynis

Lynis is a comprehensive security auditing tool for Linux:

```bash
sudo apt install lynis -y
sudo lynis audit system
```

It produces a scored report with actionable hardening suggestions.

### Rootkit Detection

```bash
sudo apt install rkhunter chkrootkit -y

# rkhunter
sudo rkhunter --update
sudo rkhunter --propupd     # Baseline file properties
sudo rkhunter --check

# chkrootkit
sudo chkrootkit
```

### Vulnerability Scanning with OpenVAS / Greenbone

For periodic vulnerability scanning, consider running [Greenbone Community Edition](https://greenbone.github.io/docs/latest/) or using an external service such as [Tenable Nessus Essentials](https://www.tenable.com/products/nessus/nessus-essentials) (free for up to 16 IPs).

> 📖 **Reference:** [Lynis Documentation (cisofy.com)](https://cisofy.com/documentation/lynis/)
> 📖 **Reference:** [NIST National Vulnerability Database](https://nvd.nist.gov/)

---

## Container Security (Docker)

If you run Docker on your server, apply these additional hardening measures.

### Run as Non-Root

Never run application containers as root. In your `Dockerfile`:

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
```

### Use Read-Only Filesystems

```bash
docker run --read-only --tmpfs /tmp myimage
```

### Limit Container Capabilities

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```

### Scan Images for Vulnerabilities

```bash
# Using Docker Scout (built into Docker Desktop / CLI)
docker scout cves myimage:latest

# Using Trivy (open source)
sudo apt install trivy -y
trivy image myimage:latest
```

### Secure the Docker Daemon

Edit `/etc/docker/daemon.json`:

```json
{
  "icc": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true,
  "userns-remap": "default"
}
```

> 📖 **Reference:** [Docker Security Best Practices](https://docs.docker.com/engine/security/)
> 📖 **Reference:** [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
> 📖 **Reference:** [Trivy – Open Source Vulnerability Scanner](https://aquasecurity.github.io/trivy/)

---

## Useful Commands

| Command | Description |
|---|---|
| `uname -a` | Show kernel version and system info |
| `lsb_release -a` | Show Ubuntu version |
| `df -h` | Show disk usage |
| `du -sh /var/log/*` | Show size of log directories |
| `free -h` | Show memory usage |
| `top` / `htop` | Display running processes |
| `iotop` | Monitor disk I/O by process |
| `journalctl -xe` | View recent system log errors |
| `journalctl -f` | Follow live system log |
| `ss -tulpn` | Show open network ports |
| `lsof -i` | List open network connections |
| `ps aux --sort=-%cpu` | List processes by CPU usage |
| `chmod`, `chown` | Change file permissions/ownership |
| `sudo visudo` | Safely edit sudoers file |
| `sudo lynis audit system` | Full security audit |
| `sudo aide --check` | Check filesystem integrity |
| `sudo rkhunter --check` | Check for rootkits |
| `sudo ufw status verbose` | Show firewall rules |
| `sudo fail2ban-client status` | Show Fail2Ban jail status |
| `sudo auditctl -l` | List active audit rules |
| `sudo ausearch -k key` | Search audit logs by key |
| `openssl s_client -connect host:443` | Test TLS certificate |

---

## Resources

### Official Documentation & Standards

| Source | Link |
|---|---|
| Ubuntu Security Notices | [ubuntu.com/security/notices](https://ubuntu.com/security/notices) |
| Ubuntu Server Guide | [ubuntu.com/server/docs](https://ubuntu.com/server/docs) |
| CIS Ubuntu Linux Benchmark | [cisecurity.org/benchmark/ubuntu_linux](https://www.cisecurity.org/benchmark/ubuntu_linux) |
| NIST SP 800-123 – Server Security | [nvlpubs.nist.gov](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-123.pdf) |
| NSA/CISA Linux Hardening Guide | [media.defense.gov](https://media.defense.gov/2023/Feb/15/2003151804/-1/-1/0/CSI_RHEL_NixOS_Hardening_UOO169780-23.PDF) |

### Hardening Guides & Checklists

| Source | Link |
|---|---|
| Book of Zeus – Harden Ubuntu | [bookofzeus.com/harden-ubuntu](http://bookofzeus.com/harden-ubuntu/) |
| Pluralsight – Server Hardening Checklist | [pluralsight.com](https://www.pluralsight.com/blog/it-ops/linux-hardening-secure-server-checklist) |
| DigitalOcean Security Tutorials | [digitalocean.com/community/tags/security](https://www.digitalocean.com/community/tags/security) |
| Arch Linux Security Wiki | [wiki.archlinux.org/title/Security](https://wiki.archlinux.org/title/Security) |
| Linux Hardening Guide (trimstray) | [github.com/trimstray/the-practical-linux-hardening-guide](https://github.com/trimstray/the-practical-linux-hardening-guide) |

### SSH & Encryption

| Source | Link |
|---|---|
| Mozilla OpenSSH Guidelines | [infosec.mozilla.org/guidelines/openssh](https://infosec.mozilla.org/guidelines/openssh) |
| SSH Audit Tool | [ssh-audit.com](https://www.ssh-audit.com/) |
| Mozilla SSL Configuration Generator | [ssl-config.mozilla.org](https://ssl-config.mozilla.org/) |
| SSL Labs Server Test | [ssllabs.com/ssltest](https://www.ssllabs.com/ssltest/) |
| Let's Encrypt | [letsencrypt.org](https://letsencrypt.org/) |

### Tools

| Tool | Description | Link |
|---|---|---|
| Lynis | Security auditing & compliance | [cisofy.com/lynis](https://cisofy.com/lynis/) |
| AIDE | Filesystem integrity monitoring | [aide.github.io](https://aide.github.io/) |
| Fail2Ban | Brute-force IP banning | [fail2ban.org](https://www.fail2ban.org/) |
| rkhunter | Rootkit detection | [rkhunter.sourceforge.net](https://rkhunter.sourceforge.net/) |
| Trivy | Container vulnerability scanner | [aquasecurity.github.io/trivy](https://aquasecurity.github.io/trivy/) |
| testssl.sh | Command-line TLS scanner | [testssl.sh](https://testssl.sh/) |
| OpenSCAP | SCAP compliance scanner | [open-scap.org](https://www.open-scap.org/) |

### Learning & Community

| Source | Link |
|---|---|
| SANS Reading Room (Linux) | [sans.org/reading-room](https://www.sans.org/reading-room/) |
| Linux Audit Blog | [linux-audit.com](https://linux-audit.com/) |
| Kernel Self-Protection Project | [kernsec.org](https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project) |
| CVE Details (Ubuntu) | [cvedetails.com/vendor/51/Ubuntu.html](https://www.cvedetails.com/vendor/51/Ubuntu.html) |
| NIST NVD | [nvd.nist.gov](https://nvd.nist.gov/) |

---

## How to Contribute

This guide is open source. If you'd like to contribute:

- Fork this repository on GitHub
- Create a branch for your changes
- Submit a pull request with your additions or corrections

All contributions are welcome and appreciated.

---

*Published by Aerostat & Co. — Use freely, fork widely.*
