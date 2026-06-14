# Sigil

A small static-site generator written in [March](https://github.com/march-language/march). Point it at a tree of Markdown — or [Scroll](https://github.com/march-language/scroll) `.scrollmd` notebooks — with optional YAML-ish front-matter, and it renders a static HTML site: custom layouts, blog-style collections (tag indexes, RSS feed, sitemap), and a live-reload dev server.

It ships its own Markdown parser and a built-in dev server, with no dependencies beyond the March toolchain.

## Quick start

```sh
forge build                            # compile → .march/build/debug/sigil
alias sigil=.march/build/debug/sigil   # or add that directory to your PATH

sigil new mysite                       # scaffold content/, static/, sigil.toml
cd mysite
sigil serve                            # build + live-reload on http://localhost:4000
```

Edit files under `content/` and the browser reloads as you save. When you're ready to publish:

```sh
sigil build                            # render content/ → out/, copy static/ across
```

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
base_url = "https://example.com"   # or "https://org.github.io/repo" for a GitHub project page

[build]
content_dir = "content"
output_dir  = "out"
static_dir  = "static"
```

`base_url` drives absolute URLs in the sitemap/feed and the root-relative prefix for internal links (so a site served under `/repo/` links correctly).

## Content

Each content file may begin with front-matter:

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

The output tree mirrors the content tree (`content/docs/intro.md` → `out/docs/intro.html`). Drafts are skipped. A `layout:` value is dispatched to a layout function (see `Build.Selector` / `Build.LayoutFn`).

### Markdown support

Sigil ships its own Markdown parser — a pragmatic subset, not full CommonMark:

- **Headings** — ATX (`#`…`######`) and setext (`===` / `---` underlines), with auto-generated `id` anchors
- **Inline** — `**bold**`, `*italic*` / `_italic_` (word-boundary aware), `` `code` ``, and backslash escapes
- **Links** — inline `[text](url "title")`; reference `[text][label]`, `[label][]`, and `[label]` resolved against `[label]: url "title"` definitions; autolinks `<https://…>` / `<mailto:…>`
- **Images** — `![alt](src "title")`
- **Lists** — bullet (`-`, `*`) with indentation-based nesting, and ordered (`1.`)
- **Blockquotes** — `> …` (single-line)
- **Tables** — pipe tables with a header separator row
- **Code** — fenced blocks ```` ```lang ```` → `<pre><code class="language-lang">`
- **Thematic breaks** — `---` / `***`

URLs are sanitized against an allowlist (http/https/mailto/tel/ftp), so `javascript:` and other script-bearing schemes are neutralized.

### Scroll notebooks (`.scrollmd`)

Sigil also builds [Scroll](https://github.com/march-language/scroll) notebooks — plain Markdown with fenced `march` code cells — as ordinary pages. `.scrollmd` files are discovered alongside `.md`, and both `march` and `march:server` cells render as static, syntax-highlightable `language-march` code blocks (no execution).

## Build modes

- `Build.run` / `run_with_layout` / `run_with_selector` — full build **with** blog collections (chronological home index, tag pages, `feed.xml`, `sitemap.xml`).
- `Build.run_pages` / `run_pages_with_layout` / `run_pages_with_selector` — **docs mode**: same render pass, no collections (so a hand-authored `content/index.md` isn't clobbered by the auto archive).

## Build & test

Sigil is a `forge` project. From the repo root:

```sh
forge build      # compile the `sigil` binary (.march/build/debug/sigil)
forge test       # run the test suite (compiled natively)
forge lint       # style / dead-code checks
```

The entry point is `mod Ssg` in [`lib/sigil.march`](lib/sigil.march).

## Repository structure

| Path | Contents |
|------|----------|
| `lib/sigil.march` | entry point + CLI argument parsing (`mod Ssg`) |
| `lib/build.march` | two-pass build pipeline, layouts, collections |
| `lib/markdown/` | Markdown parser — `block.march`, `inline.march`, `render.march`, `ast.march` |
| `lib/config.march`, `lib/siteconfig.march` | site config + `sigil.toml` loading |
| `lib/page.march`, `lib/frontmatter.march` | per-page metadata + front-matter splitting |
| `lib/siteindex.march` | the site index used to build collections |
| `lib/assets.march`, `lib/pathutil.march` | static-asset copying + path helpers |
| `lib/serve.march`, `lib/watcher.march` | dev server + live-reload file watcher |
| `lib/scaffold.march` | `sigil new` starter-site generator |
| `test/` | the test suite |
| `specs/`, `docs/` | design notes and plans |
