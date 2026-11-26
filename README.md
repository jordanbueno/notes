# notes

Personal knowledge base: guides, reference lists, and documentation, mostly endpoint engineering (Intune, Autopilot, ConfigMgr, Jamf, PowerShell) and homelab.

## Index

### Guides
- [SSH keys in PowerShell 7](docs/guides/ssh-keys-pwsh.md): create, import, agent, servers, config, troubleshooting

### Reference
- [RSS feeds](docs/reference/rss-feeds.md): master feed list for NetNewsWire

## Layout

```
docs/
├── guides/      step-by-step how-tos
└── reference/   lists and lookups
```

**Conventions**
- Markdown only, lowercase-hyphenated file names (`like-this.md`)
- Organize by type first (guides vs. reference); add topic subfolders only once a folder gets crowded
- Add every new page to the index above
- Nothing employer- or client-specific: genericize names, tenants, domains, and credentials before committing

## Future state: MkDocs Material

The goal is to publish this repo as a searchable docs site with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) on GitHub Pages. The `docs/` layout is already what MkDocs expects, so the migration is additive:

1. Add `mkdocs.yml` at the repo root (`site_name`, `theme: name: material`, `nav`)
2. Optionally add `docs/index.md` as the site home page (this README's index can move there)
3. Add a GitHub Actions workflow that runs `mkdocs gh-deploy --force` on push to `main`
4. Enable GitHub Pages from the `gh-pages` branch

Until then, everything reads fine directly on GitHub.
