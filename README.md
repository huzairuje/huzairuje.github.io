# huzairuje blog

Personal engineering blog by **Muhammad Huzair** — a backend engineer writing about Go, distributed systems, databases, and production war stories. Built with [Jekyll](https://jekyllrb.com/) and the [Minima](https://github.com/jekyll/minima) theme, hosted on GitHub Pages.

Live site: <https://huzairuje.github.io>

---

## What's in here

```
.
├── _config.yml          # Site identity, theme, plugins, build flags
├── _posts/              # Blog posts (YYYY-MM-DD-slug.markdown)
├── _layouts/
│   └── home.html        # Custom home with tag/date/search filters
├── _includes/           # head / header / footer partials
├── assets/
│   └── main.scss        # Minima skin overrides
├── about.markdown       # /about page
├── index.markdown       # Landing page (uses home layout)
├── search.json          # Lazy-loaded search index for the home filter
├── 404.html
├── robots.txt
└── Gemfile              # Ruby deps (Jekyll 4.4, Minima 2.5, plugins)
```

Plugins enabled: `jekyll-feed`, `jekyll-seo-tag`, `jekyll-sitemap`.

## Run it locally

Requires Ruby 3.x and Bundler.

```bash
bundle install
bundle exec jekyll serve --livereload
```

Then open <http://localhost:4000>.

To preview future-dated drafts locally:

```bash
bundle exec jekyll serve --future
```

## Build

```bash
bundle exec jekyll build
```

Output lands in `_site/`. GitHub Pages does this automatically on push to the default branch.

## Writing a post

Create a file under `_posts/` named `YYYY-MM-DD-slug.markdown` with front matter:

```markdown
---
layout: post
title: "Your Title"
date: 2026-05-21 09:00:00 +0700
categories: go backend
tags: [go, postgres, concurrency]
description: "Short SEO description, also used by search.json."
---

Body in markdown. Use `<!--more-->` to mark the excerpt cutoff.
```

The home page reads `tags`, `date`, `title`, and `description` to build its filter UI and search index.

---

## Why a future-dated post won't show up until you commit again

Short version: Jekyll filters out future posts at **build time**, and GitHub Pages only builds on **push**. If neither happens after the post's date, the post stays invisible.

### The mechanics

1. **`_config.yml` sets `future: false`.** Jekyll compares each post's `date:` against the **moment the build runs**. Posts with a date in the future are dropped from `site.posts` and never rendered into `_site/`.
2. **GitHub Pages builds on push, not on a schedule.** A normal user/project Pages site rebuilds when the default branch receives a commit. There is no cron that wakes up at midnight to re-evaluate your posts. The HTML deployed to the CDN is whatever was generated on the last successful build.
3. **Result:** if you write a post dated `2026-06-01` and push it on `2026-05-21`, the build on `2026-05-21` sees a future post, drops it, and ships a `_site/` without it. On `2026-06-01` nothing happens — no push, no rebuild — so the post still isn't on the site. It only appears when the next push (any push) triggers a fresh build *after* the post's date has passed.

### The three ways to fix it

| Option | What you change | Trade-off |
|---|---|---|
| **A. Flip the flag** | `future: true` in `_config.yml` | Future posts publish immediately on build. You lose the "schedule by date" behavior — anything in `_posts/` is live the moment it's pushed. |
| **B. Trigger a rebuild on the date** | A scheduled GitHub Actions workflow that runs `cron` daily and pushes an empty commit (or calls the Pages build API) | Keeps `future: false` semantics, posts go live near their declared date. Adds a small workflow to maintain. |
| **C. Manual commit on the day** | Push any commit (even a whitespace change) on or after the post's date | Zero config, zero automation, but you have to remember. |

### Implemented: Option B, smart scheduled rebuild

This repo ships [`.github/workflows/scheduled-rebuild.yml`](.github/workflows/scheduled-rebuild.yml). It runs daily at **00:10 Asia/Jakarta** (17:10 UTC the previous day) and:

1. Lists files in `_posts/` and looks for any whose `YYYY-MM-DD-` prefix matches today's date in Asia/Jakarta.
2. If a match exists, pushes an empty commit so GitHub Pages rebuilds and the post goes live.
3. If nothing matches, exits silently — no empty commits on days you weren't publishing.

It also exposes a `workflow_dispatch` trigger with a `force` input, so you can manually kick a rebuild from the Actions tab without waiting for the cron.

**Requirements for it to work:**
- The workflow needs `contents: write` permission. That's already declared in the YAML, but make sure your repo's *Settings → Actions → General → Workflow permissions* is set to **Read and write permissions** (or at least allows the default `GITHUB_TOKEN` to push).
- Post filenames must match the convention `YYYY-MM-DD-slug.markdown` (or `.md`). The `YYYY-MM-DD` prefix is what the workflow uses to detect "goes live today."
- The front-matter `date:` should agree with the filename date. Jekyll uses the front-matter date for `future` filtering, so if the filename says `2026-06-01-` but the front matter says `2026-07-01`, the workflow will commit on June 1 but Jekyll will still drop the post.

**Tuning the schedule:** edit the `cron:` line. Cron in GitHub Actions is UTC. `10 17 * * *` is 00:10 the next day in Asia/Jakarta (UTC+7). If you want it to run earlier or later, adjust both the cron and your mental model of when posts go live.

### Why GitHub Pages doesn't "just know"

The deployed site is **static HTML** sitting on a CDN. There is no Jekyll process running on a server somewhere checking the clock. The filtering happens during `jekyll build`, and the CDN only knows what was last uploaded. Any "publish on date" behavior has to come from re-running the build.

---

## License

Content (posts, images) © Muhammad Huzair. Code and config are MIT unless noted otherwise.
