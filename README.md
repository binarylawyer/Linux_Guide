# Aerostat's Guide To Linux

A practical Ubuntu LTS setup and hardening guide, published as a simple [GitHub Pages](https://pages.github.com/) / [Jekyll](https://jekyllrb.com/) site.

**Live site:** https://binarylawyer.github.io/Linux_Guide/

**Version:** 1.1 (see [`index.md`](./index.md))

## Documents

| File | Purpose |
|------|---------|
| [`index.md`](./index.md) | Full guide (site homepage) |
| [`LINUX_GUIDE.md`](./LINUX_GUIDE.md) | Standalone Markdown summary |
| [`index-backup-1.md`](./index-backup-1.md) | Archive of the previous homepage draft |

## Topics covered

- Initial Ubuntu LTS server setup
- OS lockdown (`/boot`, unused filesystems, core dumps)
- Kernel hardening with sysctl
- User accounts, password policy, and sudo
- SSH hardening and optional 2FA
- UFW firewall and fail2ban
- Automatic security updates
- Encrypted transfers and TLS/SSL practices
- Filesystem security, AIDE, auditd
- Logging, monitoring, and audits (Lynis, rkhunter)
- When to move from Vercel / PaaS to a self-managed server
- Container (Docker) security basics

## Local preview

This site uses the Bootstrap 4 + Jekyll template under the hood.

```bash
bundle install
bundle exec jekyll serve
```

Open http://localhost:4000

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). Fixes, clearer steps, and tested hardening tips are especially welcome.

## License

See [LICENSE.md](./LICENSE.md).

## Template credit

Site chrome is based on [bootstrap-4-github-pages](https://github.com/nicolas-van/bootstrap-4-github-pages).
