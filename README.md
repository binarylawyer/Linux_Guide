# Aerostat's Guide To Linux

A practical Ubuntu LTS setup and hardening guide, published as a simple [GitHub Pages](https://pages.github.com/) / [Jekyll](https://jekyllrb.com/) site.

**Live site:** https://binarylawyer.github.io/Linux_Guide/

## Documents

| File | Purpose |
|------|---------|
| [`index.md`](./index.md) | Full guide (site homepage) |
| [`LINUX_GUIDE.md`](./LINUX_GUIDE.md) | Standalone Markdown summary |
| [`index-backup-1.md`](./index-backup-1.md) | Archive of the previous homepage draft |

## Topics covered

- Choosing an Ubuntu LTS release
- Initial server setup
- SSH hardening (keys, disable root/password login)
- UFW firewall baseline
- Unattended security updates
- Non-root admin users
- fail2ban
- Encrypted remote access and transfers
- Boot/`/boot` care and basic permission hygiene
- Logging, backups, and a post-install checklist

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
