# Sigil

A small static-site generator written in [March](https://github.com/march-language/march). It turns a tree of Markdown files (with YAML-ish front-matter) into a static HTML site, with custom layouts, blog-style collections (tag indexes, RSS feed, sitemap), and a live-reload dev server.

## Build & test

Sigil is a `forge` project. From the repo root:

```sh
forge build      # compile the `sigil` binary
forge test       # run the test suite (compiles natively)
forge lint       # style/dead-code checks
```

The entry point is `mod Ssg` in [`lib/sigil.march`](lib/sigil.march).

## CLI

```
sigil <new DIR | build | serve> [--content-dir DIR] [--output-dir DIR] [--static-dir DIR] [--title TITLE] [--base-url URL]
```

- `sigil new DIR` — scaffold a starter site (content, static, `sigil.toml`, README) into `DIR`.
- `sigil build` — render `content/` to `out/` and copy `static/` across.
- `sigil serve` — build, then serve `out/` on `http://localhost:4000` with live reload on content changes.

Flags override values from `sigil.toml`. Precedence: built-in defaults < `sigil.toml` < CLI flags.

## Configuration (`sigil.toml`)

```toml
[site]
title    = "My Site"
base_url  = "https://example.com"   # or "https://org.github.io/repo" for a GitHub project page

[build]
content_dir = "content"
output_dir  = "out"
static_dir  = "static"
```

`base_url` drives absolute URLs in the sitemap/feed and the root-relative prefix for internal links (so a site served under `/repo/` links correctly).

## Content

Each `content/**/*.md` file may begin with front-matter:

```markdown
---
title: Hello World
date: 2026-06-12
layout: post           # selects a layout; "default" if omitted
tags: [hello, sigil]
draft: false           # true to skip rendering
nav_order: 3           # ordering hint for data-driven nav
description: A short summary used in <head>.
---

# Hello

Body in **Markdown**.
```

The output tree mirrors the content tree (`content/docs/intro.md` → `out/docs/intro.html`). Drafts are skipped. A `layout:` value is dispatched to a layout function (see `Build.Selector`/`Build.LayoutFn`).

## Build modes

- `Build.run` / `run_with_layout` / `run_with_selector` — full build **with** blog collections (chronological home index, tag pages, `feed.xml`, `sitemap.xml`).
- `Build.run_pages` / `run_pages_with_layout` / `run_pages_with_selector` — **docs mode**: same render pass, but no collections (so a hand-authored `content/index.md` isn't overwritten by the auto archive).

## Layout

- `lib/` — the library: entry/CLI (`sigil.march`), build pipeline (`build.march`), Markdown parser (`lib/markdown/`), config (`config.march`, `siteconfig.march`), page/front-matter (`page.march`, `frontmatter.march`), site index (`siteindex.march`), assets (`assets.march`), dev server + watcher (`serve.march`, `watcher.march`), scaffolder (`scaffold.march`), shared path helpers (`pathutil.march`).
- `test/` — the test suite.
- `specs/`, `docs/` — design notes and plans.
