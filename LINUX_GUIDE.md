# Aerostat's Guide To Linux (standalone)

This file is a portable copy of the project’s Linux hardening guide for readers who prefer a plain Markdown document outside the Jekyll site.

The canonical web version lives in [`index.md`](./index.md) and is published at:

**https://binarylawyer.github.io/Linux_Guide/**

---

## Ubuntu LTS server setup and hardening

**Version 1.0** — Ubuntu LTS edition

### Goals

- Stand up a fresh Ubuntu LTS server safely
- Apply a practical hardening baseline (SSH, firewall, updates, users)
- Leave a checklist you can reuse and improve

### Quick start checklist

1. Install a current Ubuntu LTS release (24.04 or 22.04).
2. Create a non-root sudo user and install your SSH public key.
3. Disable root SSH login and password authentication.
4. Enable UFW; allow only required ports (start with OpenSSH).
5. Install and enable fail2ban.
6. Turn on unattended security updates.
7. Confirm backups and a restore path exist.

### Core commands

```bash
# Identity and updates
lsb_release -a
sudo apt update && sudo apt full-upgrade

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

### SSH baseline (`/etc/ssh/sshd_config`)

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

### Encrypt traffic

Use `ssh`, `scp`, `sftp`, and `rsync` over SSH. Prefer HTTPS/TLS for services. Avoid telnet and plain FTP on public networks.

### Boot and disk notes

- Treat `/boot` carefully; read-only mounts are optional and easy to get wrong.
- Prefer full-disk encryption (LUKS) on sensitive hosts at install time.
- Keep firmware current with vendor tools or `fwupd` where supported.

### Further reading

- [Ubuntu Server docs](https://ubuntu.com/server/docs)
- [CIS Ubuntu benchmarks](https://www.cisecurity.org/benchmark/ubuntu_linux)
- [sshd_config manual](https://man.openbsd.org/sshd_config)
- Full narrative guide: [`index.md`](./index.md)

### Contributing

Improvements welcome via pull request. Earlier mixed template/content is archived in [`index-backup-1.md`](./index-backup-1.md).
