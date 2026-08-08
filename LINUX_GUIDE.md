# Aerostat's Guide To Linux (standalone)

Portable summary of the project’s Ubuntu hardening guide for readers who prefer plain Markdown outside the Jekyll site.

**Canonical web version:** [`index.md`](./index.md) · https://binarylawyer.github.io/Linux_Guide/

**Version 1.1** — Ubuntu 22.04 LTS focus (also applicable to current LTS releases)

---

## Goals

- Stand up a fresh Ubuntu LTS server safely
- Apply a practical hardening baseline (SSH, firewall, updates, users, logging)
- Know when PaaS (Vercel and similar) is no longer enough
- Leave checklists you can reuse and improve

---

## Quick start checklist

1. Install Ubuntu LTS (22.04 or 24.04) and fully update packages.
2. Set hostname, timezone, and enable NTP (`systemd-timesyncd`).
3. Create a non-root sudo user; install your SSH public key.
4. Disable root SSH login and password authentication.
5. Enable UFW; allow only required ports (start with OpenSSH).
6. Install and enable fail2ban.
7. Turn on unattended security updates.
8. Confirm backups and a tested restore path.

---

## Core commands

```bash
# Updates
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y

# Admin user
sudo adduser youradmin
sudo usermod -aG sudo youradmin

# Firewall
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable

# Intrusion prevention
sudo apt install fail2ban
sudo systemctl enable --now fail2ban

# Automatic security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## SSH baseline (`/etc/ssh/sshd_config`)

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers youradmin
X11Forwarding no
```

```bash
sudo sshd -t && sudo systemctl reload ssh
```

Prefer Ed25519 keys. Consider 2FA (`libpam-google-authenticator`) on high-value hosts.

---

## Encrypt traffic

Use `ssh`, `scp`, `sftp`, and `rsync` over SSH. Prefer HTTPS/TLS (Let’s Encrypt) for services. Avoid telnet and plain FTP on public networks.

---

## Extra hardening (see full guide)

The full [`index.md`](./index.md) also covers:

- Kernel `sysctl` hardening and boot/`/boot` locks
- Password policy, sudo, and inactive account locks
- TLS testing, AIDE, auditd, Lynis, rkhunter
- Log management (rsyslog, journald, logrotate)
- Container (Docker) security basics
- **When to move from Vercel / PaaS to a self-managed server** (cost, control, stateful workloads, migration checklist)

---

## PaaS → self-managed (rule of thumb)

Once a PaaS bill is consistently about **$50–$100+/month**, or you need custom networking, long-running processes, or portable infra, evaluate a VPS and follow the migration checklist in the full guide.

---

## Further reading

- [Ubuntu Server docs](https://ubuntu.com/server/docs)
- [CIS Ubuntu benchmarks](https://www.cisecurity.org/benchmark/ubuntu_linux)
- [Mozilla OpenSSH guidelines](https://infosec.mozilla.org/guidelines/openssh)
- Full narrative guide: [`index.md`](./index.md)

---

## Contributing

Improvements welcome via pull request. Earlier mixed template/content is archived in [`index-backup-1.md`](./index-backup-1.md).
