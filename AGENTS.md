# AGENTS.md

## Project

Jekyll static blog hosted on GitHub Pages at https://luyiisme.github.io/. Based on the Hux Blog theme.

## Build & Preview

- `jekyll serve` — local preview at http://localhost:4000
- `npm run watch` — Grunt watch (LESS/JS) + Jekyll serve combined
- Push to `master` — GitHub Pages auto-deploys

## Content

- Posts live in `_posts/` as Markdown files named `YYYY-MM-DD-slug.markdown`
- Config in `_config.yml` (kramdown GFM, rouge highlighting, paginate 10)
- Static pages: `about.html`, `tags.html`, `404.html`

## Editing

- New posts: create a `.markdown` file in `_posts/`
- Style: edit `less/` files, Grunt compiles to `css/`
- JS: edit `js/hux-blog.js`, Grunt minifies to `js/hux-blog.min.js`

## Compliance & Security Rules

This is a personal blog authored by an employee of a large tech company. Every post, image, and code snippet MUST pass the checks below before being committed or published. When in doubt, err on the side of omission.

### Content Boundaries

- **Topic must be open-source, industry-standard, or public-domain**: only generic technology solutions, protocol discussions, open-source project usage/analysis, or community practices. No content derived from internal systems, proprietary architecture, or non-public business logic.
- **No internal project/work references**: strip all internal project names, product codenames, team names, org structures, roadmap details, or internal metrics (performance data, traffic numbers, user counts, revenue).
- **No customer or partner data**: never mention client names, partner companies, or any customer-specific scenarios — even anonymized. Composite or hypothetical examples must be clearly labeled as such.
- **No internal URLs, hostnames, or infrastructure references**: remove internal Git URLs, CI/CD endpoints, artifact repository paths, internal domain names, and references to internal service names.
- **Code snippets must be self-written or from OSS**: never copy-paste code from internal repositories. Snippets should be minimal, illustrative, and generic — not production logic. License compliance required for OSS snippets.

### Text Sanitization

- **No company/org abbreviations** in post body, filenames, or metadata (e.g., no "ali", "ant", "taobao" — use generic alternatives).
- **No internal IPs or private network segments**: 10.x, 192.168.x, 172.16-31.x, and any internal service discovery addresses must be replaced with RFC 5737 example addresses (`203.0.113.0/24`, `198.51.100.0/24`) or `<EXAMPLE_IP>`.
- **Genericize tool names**: if an internal platform has a unique name, describe it by function (e.g., "internal monitoring system" not "AntMonitor"). Prefer OSS equivalents.
- **No credentials in code snippets**: never include real AK/SK, tokens, passwords, internal OSS signed URLs, or any form of access credentials — even as placeholder values. Use `<YOUR_TOKEN>`, `<EXAMPLE_KEY>`, or RFC 5737 IPs.
- **Review git history**: old commits may contain sensitive info. Squash or clean history before pushing if needed.

### Image & Asset Sanitization

- **No screenshots of internal dashboards, tools, platforms, or terminals** that show internal URLs, project names, cluster IDs, or proprietary UIs.
- **Blur/redact** any company logos, watermarks, or identifiable branding elements.
- **Alt text and image filenames** must not leak internal naming conventions.
- **No internal IPs, hostnames, or service identifiers** visible in terminal captures or browser address bars.

### Commit & Publish Checklist

Before `git push`, confirm:

1. [ ] Post topic is OSS / industry-standard / public-domain
2. [ ] No internal project names, codenames, or org references in text
3. [ ] No internal IPs, URLs, or hostnames (visible or invisible)
4. [ ] Code snippets are self-written generic illustrations, not copied from internal repos
5. [ ] Images are sanitized: no dashboards, logos, or identifiable branding
6. [ ] Image filenames and alt text are generic
7. [ ] No customer/partner references or business data
8. [ ] No AK/SK, tokens, or credentials in code snippets
9. [ ] Commit messages and file metadata are clean

### Common Pitfalls

| Risk | Example | Fix |
|------|---------|-----|
| Internal project name | "LinkE platform" | "a service discovery platform" |
| Org abbreviation in username/path | `/home/alipay/xxx` | `/home/user/xxx` |
| Internal monitoring screenshot | Grafana dashboard with cluster IDs | OSS Grafana with demo data, or remove |
| Copy-pasted internal code | Snippet with internal SDK import | Rewrite as generic illustration |
| IP leak in terminal | `ping 10.x.x.x` | Replace with `198.51.100.1` or `EXAMPLE_IP` |
| Internal Git URL | `git.code.alipay.com/...` | `github.com/example/...` |
| Leaked AK/SK or token | `AccessKeyId=LTAI...` in code block | `AccessKeyId=<YOUR_KEY>` |
