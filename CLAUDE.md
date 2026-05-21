# CLAUDE.md

Guidance for Claude / AI assistants working in this repository.

## What this project is

A personal Jekyll blog (`huzairuje.github.io`) using the Minima theme, deployed via GitHub Pages. Content focus: Go, distributed systems, databases, backend engineering war stories.

- **Static site generator:** Jekyll 4.4
- **Theme:** Minima 2.5 (dark skin)
- **Hosting:** GitHub Pages (builds on push to default branch)
- **Plugins:** `jekyll-feed`, `jekyll-seo-tag`, `jekyll-sitemap`

## Repository layout

| Path | Purpose |
|---|---|
| `_config.yml` | Site config — title, author, theme, plugins, `future` flag |
| `_posts/` | Blog posts, named `YYYY-MM-DD-slug.markdown` |
| `_layouts/home.html` | Custom home layout with tag/date/search filter UI |
| `_includes/` | `head.html`, `header.html`, `footer.html` partials |
| `assets/main.scss` | Minima skin overrides |
| `about.markdown`, `index.markdown` | Top-level pages |
| `search.json` | Lazy-loaded JSON search index built by Liquid |
| `Gemfile` / `Gemfile.lock` | Ruby dependencies |

Build output goes to `_site/` (gitignored).

## Common commands

```bash
bundle install                          # install gems
bundle exec jekyll serve --livereload   # local dev at :4000
bundle exec jekyll serve --future       # also render future-dated posts
bundle exec jekyll build                # one-shot build into _site/
```

## Authoring rules

### Front matter contract

Every post must have:

```yaml
---
layout: post           # default also applied via _config.yml, but be explicit
title: "..."
date: YYYY-MM-DD HH:MM:SS +0700   # Asia/Jakarta offset
categories: ...
tags: [..., ...]       # array form, lowercase, hyphenated
description: "..."     # used by SEO + search.json
---
```

- `tags` must be an **array**. The home page filter splits on comma and reads `data-tags`.
- `description` is what `search.json` exposes to the client-side search. Keep it accurate, not just SEO bait.
- Use `<!--more-->` for the excerpt cutoff (configured via `excerpt_separator`).

### Filenames

`_posts/YYYY-MM-DD-kebab-case-slug.markdown`. Date in the filename **must** match the front-matter `date:`.

### Style for blog posts

Match the existing voice in `_posts/`:
- First-person, war-story framing — "I hit this, here's what fixed it."
- Concrete code blocks over abstract description.
- Headings build a narrative, not a textbook TOC.
- No marketing fluff, no "in today's fast-paced world."

## Critical config gotcha — `future: false`

`_config.yml` sets `future: false`. This drops any post with a `date:` later than build time. Combined with GitHub Pages only building on push, **future-dated posts never appear on the live site until a new commit lands after the date passes**. See the README for the full explanation and three fixes (flag flip, scheduled rebuild workflow, manual commit).

When asked to "schedule a post," confirm which behavior the user wants before changing the flag:
- `future: true` → publish immediately on push
- Keep `future: false` + add scheduled rebuild → publish on date

Don't silently toggle `future`.

## Things to be careful about

- **Don't edit `_site/`, `.jekyll-cache/`, `.bundle/`, or `vendor/`.** All generated or vendored.
- **Don't bump the Minima major version** without checking the custom `_layouts/home.html` and `_includes/` still render — they override theme defaults.
- **Don't rename `search.json`** — `_layouts/home.html` fetches it by relative URL.
- **Preserve the `+0700` timezone** in post dates. Site `timezone: Asia/Jakarta`; mismatched offsets cause posts to appear on the wrong day.
- **Image assets** go under `assets/`. Default OG image is `/assets/og-default.png` (referenced in `_config.yml` defaults).

## Verifying changes

Before declaring a change done:

```bash
bundle exec jekyll build 2>&1 | tail -20
```

A clean build prints `done in N seconds` with no warnings about deprecated front matter, missing layouts, or Liquid errors. If you touched `home.html` or `search.json`, also run `jekyll serve` and load `/` to confirm the filters and search still populate.

## Deployment

`git push` to the default branch. GitHub Pages handles the rest. There's no separate deploy step, no CI workflow checked in (see future-post note above for one optional workflow).

## Out of scope

- Comments system (none currently; don't add one without asking).
- Analytics (none currently; don't add tracking without asking).
- Switching themes or static generators.

When in doubt, prefer small, focused changes and match the existing patterns in `_posts/` and `_layouts/`.
