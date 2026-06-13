# March Site with Sigil — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. March logic tasks follow TDD (forge test); artifact tasks (CSS, workflow, content) are create-then-build-verify.

**Goal:** Build and deploy the March language's GitHub Pages site using the Sigil static-site generator.

**Architecture:** Two concerns. **(A)** Small, tested enhancements to the Sigil library itself — subpath-safe internal links and nested output paths — so it can serve a docs-shaped site under a GitHub *project* URL. **(B)** A new `march-www` project that depends on Sigil as a local-path library, supplies a custom `main.march` (landing + doc + default layouts capturing `Site.Info`), a theme, client-side March syntax highlighting, the content tree, and a GitHub Actions deploy workflow.

**Tech Stack:** March + forge (build/test), Sigil SSG (`lib/*.march`), Prism.js (client-side highlighting), GitHub Actions + GitHub Pages.

---

## Scope & Decomposition

This plan has three phases. **Phase A is separable** (it's library work in *this* repo and ships value on its own — it's covered by `forge test`). **Phases B and C** build and ship the actual site in a *new* `march-www` project and depend on Phase A being merged & installed.

| Phase | Where | Produces |
|-------|-------|----------|
| **A. Sigil library** | this `sigil` repo | Subpath-safe links + nested output, green test suite |
| **B. march-www site** | new `march-www/` project | Themed, highlighted, navigable site that builds locally |
| **C. Deployment** | `march-www/` + GitHub | Live site at `https://<org>.github.io/march/` via Actions |

If you only want the *smallest cut to ship*, do **A1 → B1–B5 → C** and skip A3/B-nested (keep content flat, no duplicate basenames). A3 (nested output) is the one change that pokes the native-codegen RC paths, so it carries the most risk — it is isolated into its own phase deliberately.

**Key facts about the existing code (verify before editing):**
- `String.slice_bytes(s, start, length)` — second arg is a **length**, not an end index (confirmed against `lib/siteconfig.march:89` and `lib/page.march:171`).
- Internal hrefs are currently **root-absolute**: `href="/<slug>.html"` in `lib/build.march` `index_items` (~line 609) and `tag_page_items` (~line 474). These break under a `/march/` subpath.
- Output is **flat**: `lib/build.march` `slug_of`/`Page.slug_of` key by basename, so `content/a/x.md` and `content/b/x.md` collide (`lib/build.march:42-46`).
- The custom-layout hooks already exist: `Build.run_with_selector(cfg, Selector)`, `Build.LayoutFn`, and `Site.Info` (`lib/site.march`) — the march site uses these; no new hook is required for theming.
- Tests live in `test/sigil_test.march`, use `describe`/`test`/`assert`, and write to `/tmp/sigil_*_${System.monotonic_time()}` temp dirs.

---

## File Structure

**Phase A — modified in this repo:**
- Modify: `lib/build.march` — add `link_prefix`, `rel_slug`, `strip_trailing_slash`, `drop_md_ext`; thread a link prefix into collection writers; thread `content_root` into the render/index passes.
- Modify: `test/sigil_test.march` — new `describe` blocks for the helpers + an integration test for nested output and prefixed links.

**Phase B — new `march-www/` project (sibling of the `march` compiler repo, or a subdir of it):**
- Create: `march-www/forge.toml` — package + local-path dep on sigil
- Create: `march-www/sigil.toml` — title + `base_url`
- Create: `march-www/main.march` — `mod Main`: `Site.Info`, layouts, `Build.run_with_selector`
- Create: `march-www/content/index.md` — landing page
- Create: `march-www/content/docs/*.md`, `march-www/content/reference/*.md`, `march-www/content/blog/*.md`
- Create: `march-www/static/style.css` — theme
- Create: `march-www/static/prism.css`, `march-www/static/prism.js`, `march-www/static/prism-march.js` — highlighting + March grammar
- Create: `march-www/static/.nojekyll`

**Phase C — deployment:**
- Create: `march-www/.github/workflows/deploy.yml`
- Create: `march-www/README.md`

---

## Phase A — Sigil library enhancements

### Task A1: `link_prefix` — derive the root-relative URL prefix from `base_url`

**Files:**
- Modify: `lib/build.march` (add helpers near the other `pfn` string helpers, e.g. after `tag_slug` ~line 83)
- Test: `test/sigil_test.march`

- [ ] **Step 1: Write the failing tests**

Add to `test/sigil_test.march`:

```
  describe "Build.link_prefix" do
    test "empty base_url has no prefix" do
      assert (Build.link_prefix("") == "")
    end
    test "domain root has no prefix" do
      assert (Build.link_prefix("https://example.com") == "")
    end
    test "trailing slash on domain root has no prefix" do
      assert (Build.link_prefix("https://example.com/") == "")
    end
    test "project subpath yields a leading-slash prefix" do
      assert (Build.link_prefix("https://org.github.io/march") == "/march")
    end
    test "subpath trailing slash is stripped" do
      assert (Build.link_prefix("https://org.github.io/march/") == "/march")
    end
    test "nested subpath is preserved" do
      assert (Build.link_prefix("https://org.github.io/a/b") == "/a/b")
    end
  end
```

- [ ] **Step 2: Run to verify failure**

Run: `forge test`
Expected: FAIL — `Unknown function Build.link_prefix` (or unresolved name).

- [ ] **Step 3: Implement the helpers**

Add to `mod Build` in `lib/build.march` (these are pure; mark `fn` not `pfn` so tests can call them):

```
  -- Remove a single trailing "/" if present.
  pfn strip_trailing_slash(s : String) : String do
    if String.ends_with(s, "/") do
      String.slice_bytes(s, 0, String.byte_size(s) - 1)
    else
      s
    end
  end

  -- Derive the root-relative URL prefix from a site base_url.
  --   ""                              -> ""        (relative / domain root)
  --   "https://example.com"           -> ""        (domain root)
  --   "https://org.github.io/march"   -> "/march"  (GitHub *project* page)
  --   "https://org.github.io/a/b/"    -> "/a/b"
  -- Used to prefix every internal href so links resolve under a subpath.
  fn link_prefix(base_url : String) : String do
    match String.split_first(base_url, "://") do
    None -> ""
    Some(pair) ->
      match pair do
      (_, host_and_path) ->
        match String.split_first(host_and_path, "/") do
        None -> ""
        Some(hp) ->
          match hp do
          (_, path) ->
            let trimmed = strip_trailing_slash(path)
            if String.is_empty(trimmed) do "" else "/" ++ trimmed end
          end
        end
      end
    end
  end
```

- [ ] **Step 4: Run to verify pass**

Run: `forge test`
Expected: PASS for the `Build.link_prefix` block.

- [ ] **Step 5: Commit**

```bash
git add lib/build.march test/sigil_test.march
git commit -m "feat(build): add link_prefix to derive subpath URL prefix from base_url"
```

---

### Task A2: Thread the link prefix into generated internal links

`write_index` and the tag pages currently emit `href="/<slug>.html"`. Prefix them so they work under `/march/`. `base_url` is already available inside `build_collections`; compute the prefix once and pass it down.

**Files:**
- Modify: `lib/build.march` — `build_collections`, `write_index`, `index_items`, `build_tag_pages`, `write_tag_page`, `tag_page_items`
- Test: `test/sigil_test.march`

- [ ] **Step 1: Write the failing integration test**

Add to `test/sigil_test.march`:

```
  describe "Build subpath links" do
    test "home index links are base_url-prefixed" do
      let tmp = "/tmp/sigil_prefix_" ++ to_string(System.monotonic_time())
      let content_dir = Path.join(tmp, "content")
      let output_dir  = Path.join(tmp, "out")
      let _ = Dir.mkdir_p(content_dir)
      let _ = File.write(Path.join(content_dir, "hello.md"),
                "---\ntitle: Hello\ndate: 2026-01-01\ntags: [x]\n---\nbody")
      let site = { content_dir = content_dir, output_dir = output_dir,
                   static_dir = "/tmp/no_static", title = "March",
                   base_url = "https://org.github.io/march" }
      let _ = Build.run(site)
      let idx = match File.read(Path.join(output_dir, "index.html")) do
                Ok(s) -> s | Err(_) -> "" end
      assert (String.contains(idx, "href=\"/march/hello.html\""))
      let _ = Dir.rm_rf(tmp)
    end
  end
```

- [ ] **Step 2: Run to verify failure**

Run: `forge test`
Expected: FAIL — index links are `/hello.html`, not `/march/hello.html`.

- [ ] **Step 3: Implement — thread `prefix` through the collection writers**

In `build_collections` (`lib/build.march:386`), compute the prefix once (note: `base_url` is `:borrow` here, so read it before consuming uses; `link_prefix` borrows it) and pass it to `build_tag_pages` and `write_index`:

```
    let prefix = link_prefix(base_url)
```

Change the call sites inside `build_collections` to forward `prefix`:
- `build_tag_pages(output_dir, prefix, tags, triples)`
- `write_index(output_dir, prefix, site_title, sorted)`

Update `write_index` (`~line 581`) and `index_items` (`~line 599`) signatures to accept `prefix : String` and emit it. The href line in `index_items` becomes:

```
      let item =
        "<li><a href=\"" ++ prefix ++ "/" ++ slug ++ ".html\">" ++ Html.escape(title) ++ "</a>" ++ date_html ++ "</li>"
```

> **RC note:** `prefix` is read once per item and once per recursion. Follow the file's existing discipline: `prefix` is `:borrow` in `index_items` (it is only concatenated, never stored), so pass it straight through the tail recursion exactly like `base_url` is threaded in `sitemap_items`/`feed_items`. No `IncRC` gymnastics needed for a borrowed String that is only concatenated.

Update `build_tag_pages` (`~line 432`), `write_tag_page` (`~line 451`), and `tag_page_items` (`~line 468`) the same way — add a `prefix : String` parameter and change the href in `tag_page_items` to:

```
        IOList.from_string("<li><a href=\"" ++ prefix ++ "/" ++ slug ++ ".html\">" ++ Html.escape(title) ++ "</a></li>")
```

- [ ] **Step 4: Run to verify pass**

Run: `forge test`
Expected: PASS, including the new test and all pre-existing tests (the empty-`base_url` case yields `prefix == ""`, so existing `href="/hello.html"` assertions still hold).

- [ ] **Step 5: Commit**

```bash
git add lib/build.march test/sigil_test.march
git commit -m "feat(build): prefix generated internal links with base_url subpath"
```

---

### Task A3: Nested output — mirror the content directory tree (higher-risk)

> **Risk:** This threads a new `content_root` String through `collect_md`, `collect_index`, and the render pass, and changes the output path computation. The existing comment at `lib/build.march:42-46` notes this was deferred because the native RC path "handles poorly." Implement and **verify natively** (`forge build` of the march-www site, not just interpreter tests). If native build regresses, keep the flat behaviour and require unique basenames instead — Phase B can ship without this.

**Files:**
- Modify: `lib/build.march` — add `rel_slug`/`drop_md_ext`; thread `content_root` into `collect_index`, `collect_one`, `build_all_layout`, `build_page_layout`; create parent dirs before writing.
- Test: `test/sigil_test.march`

- [ ] **Step 1: Write the failing unit tests for `rel_slug`**

Add to `test/sigil_test.march`:

```
  describe "Build.rel_slug" do
    test "top-level file keeps its slug" do
      assert (Build.rel_slug("content", "content/hello.md") == "hello")
    end
    test "nested file mirrors its subdirectory" do
      assert (Build.rel_slug("content", "content/docs/intro.md") == "docs/intro")
    end
    test "trailing slash on root is tolerated" do
      assert (Build.rel_slug("content/", "content/docs/intro.md") == "docs/intro")
    end
    test "path outside root falls back to basename" do
      assert (Build.rel_slug("content", "/elsewhere/x.md") == "x")
    end
  end
```

- [ ] **Step 2: Run to verify failure**

Run: `forge test`
Expected: FAIL — `Build.rel_slug` unresolved.

- [ ] **Step 3: Implement `drop_md_ext` and `rel_slug`**

Add to `mod Build`:

```
  -- Strip a trailing ".md" (3 bytes) if present.
  pfn drop_md_ext(s : String) : String do
    if String.ends_with(s, ".md") do
      String.slice_bytes(s, 0, String.byte_size(s) - 3)
    else
      s
    end
  end

  -- Compute a root-relative slug that mirrors the source tree, with ".md" removed.
  --   rel_slug("content", "content/hello.md")     == "hello"
  --   rel_slug("content", "content/docs/intro.md") == "docs/intro"
  -- A path that does not live under content_root falls back to its basename
  -- (preserving the old flat behaviour for stray inputs).
  fn rel_slug(content_root : String, src : String) : String do
    let root     = strip_trailing_slash(content_root)
    let with_sep = root ++ "/"
    let plen     = String.byte_size(with_sep)
    let rel =
      if String.starts_with(src, with_sep) do
        String.slice_bytes(src, plen, String.byte_size(src) - plen)
      else
        Path.basename(src)
      end
    drop_md_ext(rel)
  end
```

- [ ] **Step 4: Run unit tests to verify pass**

Run: `forge test`
Expected: PASS for the `Build.rel_slug` block.

- [ ] **Step 5: Thread `content_root` into the render + index passes**

In `run_with_selector` (`lib/build.march:185`), derive an owned copy of the content dir for slug computation and pass it down. After the existing `content_dir_r2`/`md_paths_p2` lines, add:

```
          let content_root_p2 = Config.site_content_dir(cfg)
```

Change the render-pass call to forward it:

```
          match build_all_layout(output_dir_p2a, content_root_p2, sel, md_paths_p2) do
```

And the index pass — pass the root so the index keys match the output paths:

```
          let content_root_p1 = Config.site_content_dir(cfg)
          let idx = collect_index(content_root_p1, md_paths_p1, SiteIndex.empty())
```

Update `build_all_layout` (`~line 290`) and `build_page_layout` (`~line 305`) to take `content_root : String` (borrowed — only read by `rel_slug`) and replace the `slug` computation:

```
    -- OLD:  let slug = slug_of(src)
    let slug = rel_slug(content_root, src)
```

In `build_page_layout`, before writing, create the parent directory so nested slugs land in real folders. Replace the write block:

```
          let out     = Path.join(output_dir, slug ++ ".html")
          let out_dir = Path.dirname(out)
          match Dir.mkdir_p(out_dir) do
          Err(e) -> Err("cannot create " ++ out_dir ++ ": " ++ fmt_err(e))
          Ok(_) ->
            match File.write(out, IOList.to_string(html)) do
            Err(e) -> Err("cannot write " ++ out ++ ": " ++ fmt_err(e))
            Ok(_)  -> Ok(())
            end
          end
```

> Verify `Path.dirname` exists in the installed stdlib (`forge-search` skill, or grep the stdlib). If it is among the buggy `Some(0)`-pattern path helpers like `Path.extension`/`Path.strip_extension` (see `lib/build.march:23-35`), compute the parent inline: find the last `/` in `out` with `String.last_index_of(out, "/")` and `String.slice_bytes(out, 0, i)`.

Update `collect_index`/`collect_one` (`~line 242`) to take `content_root` and key the index by `rel_slug(content_root, src)` instead of `slug_of(src)`, so home/tag/feed links match the nested output paths.

> **RC note:** `content_root` is `:borrow` everywhere here (only `rel_slug` reads it, and `rel_slug` borrows). Thread it as a plain parameter alongside `output_dir`, which is already borrowed the same way through `build_all_layout`. Capture the recursive continuation *before* any consuming call, exactly as the existing `recurse`/`sel` discipline does (`lib/build.march:294-302`).

- [ ] **Step 6: Write the nested-output integration test**

```
  describe "Build nested output" do
    test "subdirectory structure is mirrored into output" do
      let tmp = "/tmp/sigil_nested_" ++ to_string(System.monotonic_time())
      let content_dir = Path.join(tmp, "content")
      let docs_dir    = Path.join(content_dir, "docs")
      let output_dir  = Path.join(tmp, "out")
      let _ = Dir.mkdir_p(docs_dir)
      let _ = File.write(Path.join(docs_dir, "intro.md"), "# Intro\nbody")
      let site = { content_dir = content_dir, output_dir = output_dir,
                   static_dir = "/tmp/no_static", title = "M", base_url = "" }
      let _ = Build.run(site)
      let html = match File.read(Path.join(Path.join(output_dir, "docs"), "intro.html")) do
                 Ok(s) -> s | Err(_) -> "" end
      assert (String.contains(html, "Intro"))
      let _ = Dir.rm_rf(tmp)
    end
  end
```

- [ ] **Step 7: Run interpreter tests**

Run: `forge test`
Expected: PASS — `out/docs/intro.html` exists and contains the heading. All prior tests still green (top-level files keep flat slugs).

- [ ] **Step 8: Verify NATIVELY**

Run: `forge build && forge test` (native path). If the toolchain has an interpreter env toggle (e.g. `MARCH_TEST_INTERPRETER=1`), run **both** modes and confirm green. This is the RC-sensitive change — a native pass is the real acceptance gate.
Expected: native build + test green. If native regresses with an RC underflow/SIGSEGV, capture a minimal repro and either fix per the `compiler-rc` skill or **revert this task** and document "flat output; unique basenames required" as the shipped behaviour.

- [ ] **Step 9: Commit**

```bash
git add lib/build.march test/sigil_test.march
git commit -m "feat(build): mirror content subdirectories into nested output paths"
```

---

### Task A4: Install the updated Sigil

- [ ] **Step 1: Build & lint**

Run: `forge build && forge lint`
Expected: clean.

- [ ] **Step 2: Install the binary/library** so `march-www` can depend on it (use whatever install path the repo uses; if `march-www` uses a **local-path** forge dep, no install is needed — it compiles sigil from source).

---

## Phase B — The `march-www` site project

> Create `march-www/` as a sibling of the March compiler repo (or `march/www/`). It is a separate forge package that consumes Sigil as a library and supplies a custom `main.march`. Pick the actual location with the user; the steps below use `march-www/` as the root.

### Task B1: Scaffold the project and wire the Sigil dependency

**Files:** Create `march-www/forge.toml`, `march-www/sigil.toml`, `march-www/static/.nojekyll`

- [ ] **Step 1: Create `march-www/forge.toml`**

```toml
[package]
name = "march-www"
version = "0.1.0"
type = "tool"
description = "The March language website (built with Sigil)"
author = "March"

[deps]
sigil = { path = "../sigil" }
```

> Adjust the `path` to the real relative location of the sigil repo. If sigil is published to a registry instead, use the `forge-add` skill to add it as a registry/git dep.

- [ ] **Step 2: Create `march-www/sigil.toml`**

```toml
[site]
title = "March"
base_url = "https://YOUR_ORG.github.io/march"

[build]
content_dir = "content"
output_dir = "out"
static_dir = "static"
```

> Replace `YOUR_ORG`. If March gets a **custom domain** (e.g. `march-lang.org`), set `base_url` to that and the `link_prefix` from A1 becomes `""` automatically (domain root) — no other change needed.

- [ ] **Step 3: Create `march-www/static/.nojekyll`** (empty file) so GitHub Pages serves the output verbatim instead of running Jekyll.

```bash
mkdir -p march-www/static march-www/content
touch march-www/static/.nojekyll
```

- [ ] **Step 4: Commit**

```bash
cd march-www && git init && git add -A && git commit -m "chore: scaffold march-www sigil project"
```

---

### Task B2: `main.march` — custom layouts with a real `<head>`

The default Sigil layout (`Html.layout`) emits an empty `<head>`. The site needs a stylesheet link, meta description, canonical URL, and (later) Prism. Build the document HTML directly in a custom `LayoutFn` that captures `Site.Info`.

**Files:** Create `march-www/main.march`

- [ ] **Step 1: Write `main.march`**

```
-- The March website generator: a thin Sigil front-end that supplies custom
-- layouts (landing / doc / default) and site-wide <head> metadata.
mod Main do

  -- Site-wide metadata captured into every layout closure.
  pfn site_info() : Site.Info do
    { title       = "March",
      base_url    = "https://YOUR_ORG.github.io/march",
      author      = "The March Team",
      description = "March is a statically-typed functional language with Perceus reference counting." }
  end

  -- Root-relative asset/link prefix (e.g. "/march"), derived once from base_url.
  pfn prefix() : String do
    Build.link_prefix(Site.info_base_url(site_info()))
  end

  -- Render the shared <head>. `page_title` is the per-page title.
  pfn head(page_title : String) : String do
    let p    = prefix()
    let desc = Site.info_description(site_info())
    "<!DOCTYPE html>\n<html lang=\"en\"><head>"
    ++ "<meta charset=\"utf-8\">"
    ++ "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">"
    ++ "<title>" ++ Html.escape(page_title) ++ " — March</title>"
    ++ "<meta name=\"description\" content=\"" ++ Html.escape_attr(desc) ++ "\">"
    ++ "<link rel=\"stylesheet\" href=\"" ++ p ++ "/style.css\">"
    ++ "<link rel=\"stylesheet\" href=\"" ++ p ++ "/prism.css\">"
    ++ "</head><body>"
  end

  -- Shared site chrome: top nav + closing scripts.
  pfn top_nav() : String do
    let p = prefix()
    "<nav class=\"top\"><a class=\"brand\" href=\"" ++ p ++ "/index.html\">March</a>"
    ++ "<a href=\"" ++ p ++ "/docs/intro.html\">Docs</a>"
    ++ "<a href=\"" ++ p ++ "/reference/index.html\">Reference</a>"
    ++ "<a href=\"" ++ p ++ "/blog/index.html\">Blog</a>"
    ++ "<a href=\"https://github.com/YOUR_ORG/march\">GitHub</a></nav>"
  end

  pfn footer() : String do
    let p = prefix()
    "<script src=\"" ++ p ++ "/prism.js\"></script>"
    ++ "<script src=\"" ++ p ++ "/prism-march.js\"></script>"
    ++ "</body></html>"
  end

  -- The "default" / "doc" layout: head + nav + main content + footer.
  pfn doc_layout_fn() : Build.LayoutFn do
    Build.LayoutFn(fn page -> fn content -> do
      let title = Page.meta_title(page)
      IOList.from_string(head(title) ++ top_nav() ++ "<main class=\"doc\">")
      |> IOList.append(content)
      |> IOList.append(IOList.from_string("</main>" ++ footer()))
    end)
  end

  -- The "landing" layout: full-width hero, no <main> padding.
  pfn landing_layout_fn() : Build.LayoutFn do
    Build.LayoutFn(fn page -> fn content -> do
      let title = Page.meta_title(page)
      IOList.from_string(head(title) ++ top_nav() ++ "<div class=\"landing\">")
      |> IOList.append(content)
      |> IOList.append(IOList.from_string("</div>" ++ footer()))
    end)
  end

  -- Dispatch on the page's front-matter `layout:` field.
  pfn selector() : Build.Selector do
    Build.Selector(fn name ->
      match name do
      "landing" -> landing_layout_fn()
      _         -> doc_layout_fn()
      end)
  end

  fn main() do
    let d = Config.default()
    let d_content = d.content_dir
    let d_output  = d.output_dir
    let d_static  = d.static_dir
    let d_title   = d.title
    let d_base    = d.base_url
    match SiteConfig.load(SiteConfig.default_path(), d_content, d_output, d_static, d_title, d_base) do
    Err(msg) -> println("error: " ++ msg)
    Ok(cfg) ->
      match Build.run_with_selector(cfg, selector()) do
      Err(msg) -> println("error: " ++ msg)
      Ok(_)    -> println("built march-www")
      end
    end
  end

end
```

> **RC note:** each layout closure reads `page` once (`Page.meta_title`) and concatenates owned Strings; `site_info()` is reconstructed per call (cheap, avoids threading a cross-module record through the render loop — exactly the discipline `Build.LayoutFn`'s doc comment prescribes in `lib/build.march:5-9`).

- [ ] **Step 2: Add a minimal landing page so the build has input**

Create `march-www/content/index.md`:

```markdown
---
title: March
layout: landing
---

# March

A statically-typed functional language with Perceus reference counting.

[Get started](docs/intro.html) · [Reference](reference/index.html)
```

- [ ] **Step 3: Build and verify the head/layout**

Run: `cd march-www && forge build && ./<built-binary>` (or `forge run` per the project's convention).
Expected: prints `built march-www`; `out/index.html` exists, contains `<!DOCTYPE html>`, `<title>March — March</title>`, `<link rel="stylesheet" href="/march/style.css">`, and the top nav.

Verify with:
```bash
grep -q '/march/style.css' march-www/out/index.html && grep -q 'class="brand"' march-www/out/index.html && echo OK
```

- [ ] **Step 4: Commit**

```bash
cd march-www && git add -A && git commit -m "feat: custom layouts with real <head>, nav, and meta"
```

---

### Task B3: Theme — `static/style.css`

**Files:** Create `march-www/static/style.css`

- [ ] **Step 1: Write the stylesheet** (docs-oriented: readable column, nav bar, code styling that pairs with Prism)

```css
:root {
  --fg: #1a1a1a; --bg: #ffffff; --muted: #666;
  --accent: #2d6cdf; --rule: #e5e7eb; --code-bg: #f6f8fa;
  --max: 46rem;
}
* { box-sizing: border-box; }
body {
  margin: 0; color: var(--fg); background: var(--bg);
  font: 16px/1.65 system-ui, -apple-system, "Segoe UI", sans-serif;
}
nav.top {
  display: flex; gap: 1.25rem; align-items: center;
  padding: 0.85rem 1.25rem; border-bottom: 1px solid var(--rule);
  position: sticky; top: 0; background: var(--bg);
}
nav.top a { color: var(--fg); text-decoration: none; }
nav.top a:hover { color: var(--accent); }
nav.top .brand { font-weight: 700; margin-right: auto; }
main.doc { max-width: var(--max); margin: 0 auto; padding: 2rem 1.25rem 4rem; }
.landing { max-width: 52rem; margin: 0 auto; padding: 4rem 1.25rem; }
.landing h1 { font-size: 3rem; margin: 0 0 0.5rem; }
a { color: var(--accent); }
h1, h2, h3 { line-height: 1.25; }
h2 { margin-top: 2.5rem; border-bottom: 1px solid var(--rule); padding-bottom: 0.3rem; }
code { background: var(--code-bg); padding: 0.12em 0.34em; border-radius: 4px; font-size: 0.92em; }
pre { background: var(--code-bg); padding: 1rem; border-radius: 8px; overflow: auto; }
pre code { background: none; padding: 0; }
table { border-collapse: collapse; width: 100%; }
th, td { border: 1px solid var(--rule); padding: 0.4rem 0.6rem; text-align: left; }
blockquote { border-left: 3px solid var(--accent); margin: 1rem 0; padding: 0.2rem 1rem; color: var(--muted); }
```

- [ ] **Step 2: Build and eyeball**

Run: `cd march-www && forge build && ./<binary>` then open `out/index.html` (or use the `verify`/`run` skill to screenshot). Confirm the nav bar, centered content, and code styling render.

- [ ] **Step 3: Commit**

```bash
cd march-www && git add -A && git commit -m "feat: docs theme stylesheet"
```

---

### Task B4: March syntax highlighting (Prism + custom grammar)

Sigil already emits `<pre><code class="language-march">` (`lib/markdown/render.march:26-33`). Prism just needs the assets and a March grammar (Prism ships none).

**Files:** Create `march-www/static/prism.js`, `march-www/static/prism.css`, `march-www/static/prism-march.js`

- [ ] **Step 1: Vendor Prism core**

Download the Prism core JS and a theme CSS into the static dir (pin a version):

```bash
cd march-www/static
curl -fsSL "https://cdn.jsdelivr.net/npm/prismjs@1.29.0/prism.min.js" -o prism.js
curl -fsSL "https://cdn.jsdelivr.net/npm/prismjs@1.29.0/themes/prism.min.css" -o prism.css
```

> Vendoring (vs. a CDN `<script>`) keeps the site self-contained and avoids a third-party request — appropriate for a language's canonical site.

- [ ] **Step 2: Write the March grammar `prism-march.js`**

```javascript
// Minimal Prism grammar for the March language.
// Loaded after prism.js; registers `march` (and aliases `mr`).
Prism.languages.march = {
  'comment': /--.*/,
  'string': {
    pattern: /"""[\s\S]*?"""|"(?:\\.|[^"\\])*"/,
    greedy: true
  },
  'keyword': /\b(?:mod|do|end|fn|pfn|let|match|if|else|then|type|import|return|with)\b/,
  'boolean': /\b(?:true|false)\b/,
  'builtin': /\b(?:String|List|Option|Result|Bool|Int|Unit|IOList|Dir|File|Path)\b/,
  'constructor': /\b[A-Z][A-Za-z0-9_]*\b/,
  'function': /\b[a-z_][A-Za-z0-9_]*(?=\s*\()/,
  'operator': /\|>|\+\+|->|::|[=<>!]=?|[+\-*\/]|&&|\|\||\|/,
  'punctuation': /[{}()\[\].,;:]/
};
Prism.languages.mr = Prism.languages.march;
```

> This is a pragmatic first grammar — keywords come from the constructs visible across `lib/*.march` (`mod/do/end/fn/pfn/let/match/if/else/then/type`). Refine token lists against the real March grammar as needed.

- [ ] **Step 3: Add a doc page with a March code fence to exercise it**

Create `march-www/content/docs/intro.md`:

````markdown
---
title: Introduction
---

# Introduction

A first taste of March:

```march
mod Hello do
  fn main() do
    println("hello, march")
  end
end
```
````

- [ ] **Step 4: Build and verify highlighting works**

Run: `cd march-www && forge build && ./<binary>`. Confirm `out/docs/intro.html` contains `class="language-march"`. Then load it in a browser (or the `verify` skill) and confirm Prism colorizes the block.

```bash
grep -q 'language-march' march-www/out/docs/intro.html && echo OK
```

> Note `prism.js`/`prism-march.js` are loaded by `footer()` from Task B2. If A3 (nested output) was **not** done, place `intro.md` at `content/docs-intro.md` (flat) and adjust the nav link to `/docs-intro.html`.

- [ ] **Step 5: Commit**

```bash
cd march-www && git add -A && git commit -m "feat: Prism highlighting with a March grammar"
```

---

### Task B5: Sidebar navigation for doc pages

Give `doc_layout_fn` a left sidebar with the docs/reference index so pages are navigable. Keep it a hand-maintained list (simple, predictable) rather than auto-generated, to avoid threading the site index into the layout closure.

**Files:** Modify `march-www/main.march`, `march-www/static/style.css`

- [ ] **Step 1: Add a `sidebar()` helper to `main.march`**

```
  pfn sidebar() : String do
    let p = prefix()
    "<aside class=\"side\"><h4>Docs</h4><ul>"
    ++ "<li><a href=\"" ++ p ++ "/docs/intro.html\">Introduction</a></li>"
    ++ "<li><a href=\"" ++ p ++ "/docs/types.html\">Types</a></li>"
    ++ "<li><a href=\"" ++ p ++ "/docs/perceus.html\">Memory (Perceus)</a></li>"
    ++ "</ul><h4>Reference</h4><ul>"
    ++ "<li><a href=\"" ++ p ++ "/reference/index.html\">Stdlib</a></li>"
    ++ "</ul></aside>"
  end
```

- [ ] **Step 2: Wrap doc content with the sidebar**

In `doc_layout_fn`, change the opening/closing markup to include the sidebar in a flex wrapper:

```
      IOList.from_string(head(title) ++ top_nav() ++ "<div class=\"layout\">" ++ sidebar() ++ "<main class=\"doc\">")
      |> IOList.append(content)
      |> IOList.append(IOList.from_string("</main></div>" ++ footer()))
```

- [ ] **Step 3: Add sidebar CSS to `style.css`**

```css
.layout { display: flex; gap: 2rem; max-width: 64rem; margin: 0 auto; padding: 1.5rem 1.25rem; }
.side { flex: 0 0 13rem; font-size: 0.92rem; }
.side h4 { margin: 1rem 0 0.4rem; text-transform: uppercase; letter-spacing: 0.04em; color: var(--muted); font-size: 0.75rem; }
.side ul { list-style: none; margin: 0; padding: 0; }
.side a { display: block; padding: 0.2rem 0; color: var(--fg); text-decoration: none; }
.side a:hover { color: var(--accent); }
.layout main.doc { padding: 0; max-width: 46rem; }
@media (max-width: 48rem) { .layout { flex-direction: column; } .side { flex: none; } }
```

- [ ] **Step 4: Build & verify the sidebar renders and links resolve**

Run: `cd march-www && forge build && ./<binary>`; open `out/docs/intro.html`.
Expected: sidebar visible, links point at `/march/docs/...`.

- [ ] **Step 5: Commit**

```bash
cd march-www && git add -A && git commit -m "feat: doc sidebar navigation"
```

---

### Task B6: Author the initial content tree

**Files:** Create the markdown pages referenced by the nav/sidebar.

- [ ] **Step 1: Create the doc pages** (real prose can be filled iteratively, but every nav target must exist so links don't 404):
  - `content/docs/intro.md` (done in B4)
  - `content/docs/types.md` — front-matter `title: Types`, a heading, a `march` code fence.
  - `content/docs/perceus.md` — `title: Memory (Perceus)`.
  - `content/reference/index.md` — `title: Standard Library`.
  - `content/blog/index.md` — `title: Blog`.

Each file follows this shape:

```markdown
---
title: Types
---

# Types

(content)
```

- [ ] **Step 2: Build with zero broken internal links**

Run: `cd march-www && forge build && ./<binary>`. Then check every generated href resolves to a file on disk:

```bash
cd march-www/out && grep -rho 'href="/march/[^"]*"' . | sed 's#href="/march/##;s/"$//' | sort -u \
  | while read f; do [ -e "$f" ] || echo "MISSING: $f"; done
```
Expected: no `MISSING:` lines (external `https://` links are not matched by this pattern).

- [ ] **Step 3: Commit**

```bash
cd march-www && git add -A && git commit -m "content: initial docs/reference/blog pages"
```

---

## Phase C — Deployment to GitHub Pages

### Task C1: GitHub Actions build-and-deploy workflow

**Files:** Create `march-www/.github/workflows/deploy.yml`

- [ ] **Step 1: Write the workflow**

```yaml
name: Deploy March site

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive   # if sigil is vendored as a submodule

      # Install the March toolchain (forge + march).
      # Replace this step with the project's real install path:
      #   - a release tarball, an apt package, or `cargo install`/build-from-source.
      - name: Install March toolchain
        run: |
          # e.g. curl -fsSL https://.../install.sh | sh
          # ensure `forge` and `march` are on PATH
          forge --version

      - name: Build site
        working-directory: march-www
        run: |
          forge build
          ./$(forge bin-path 2>/dev/null || echo target/march-www)   # run the generator

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: march-www/out

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

> The **Install March toolchain** and **run the generator** lines are the only project-specific bits — wire them to however March is installed in CI and however a `type = "tool"` package is invoked after `forge build`. If sigil is a local-path dep, vendor it (submodule or a checkout step) so the path in `forge.toml` resolves in CI.

- [ ] **Step 2: Enable Pages**

In the GitHub repo: **Settings → Pages → Build and deployment → Source: GitHub Actions**. (This is a one-time manual step; note it in the README.)

- [ ] **Step 3: Confirm `base_url` matches the Pages URL**

Verify `sigil.toml` `base_url` equals the repo's Pages URL exactly (`https://<org>.github.io/<repo>`), since `Build.link_prefix` (A1) derives the `/repo` link prefix from it. Mismatch = broken links/assets.

- [ ] **Step 4: Commit & push to trigger the first deploy**

```bash
cd march-www && git add -A && git commit -m "ci: GitHub Pages build-and-deploy workflow"
git push origin main
```

- [ ] **Step 5: Verify the live site**

Watch the Actions run; once green, load `https://<org>.github.io/<repo>/`. Confirm: landing renders, CSS + Prism load (no 404s in devtools network), nav and sidebar links work, March code blocks are highlighted. Re-run the link-check from B6 against the live URLs if anything looks off.

---

### Task C2: README and authoring docs

**Files:** Create `march-www/README.md`

- [ ] **Step 1: Document local build + deploy**

```markdown
# march-www

The March language website, built with [Sigil](../sigil).

## Local build
    forge build && ./<binary>   # writes ./out
    # serve locally:
    cd out && python3 -m http.server 8000

## Deploy
Push to `main`; `.github/workflows/deploy.yml` builds and publishes to GitHub Pages.
Pages source must be set to "GitHub Actions" in repo settings (one-time).

## Editing
- `content/` — markdown (front-matter: `title`, optional `layout: landing`, `date`, `tags`, `draft`)
- `static/`  — CSS, Prism, assets (copied verbatim)
- `main.march` — layouts, nav, `<head>`
- `sigil.toml` — title + `base_url` (must equal the Pages URL)
```

- [ ] **Step 2: Commit**

```bash
cd march-www && git add README.md && git commit -m "docs: march-www README"
```

---

## Self-Review Checklist (run before executing)

- **Spec coverage:** custom theme/layout → B2/B3/B5; nested output → A3; subpath links/deploy → A1/A2/C1; sidebar nav → B5; March highlighting → B4. The originally-flagged `Page.Meta` "description" gap is covered site-side via `Site.Info` + `head()` (B2) rather than a new `Page.Meta` field — simpler and sufficient for site-wide meta.
- **Type consistency:** `Build.link_prefix`, `Build.rel_slug`, `Build.strip_trailing_slash`, `Build.drop_md_ext` are defined in A1/A3 and consumed in A2/A3/B2. `Build.LayoutFn`/`Build.Selector`/`Build.run_with_selector` and `Site.info_*`/`Page.meta_title` are pre-existing (`lib/build.march`, `lib/site.march`, `lib/page.march`).
- **Verify-before-trust list (do these during execution, don't assume):**
  1. `String.split_first`, `String.starts_with`, `String.ends_with`, `String.slice_bytes(s, start, length)`, `String.byte_size`, `String.is_empty` exist with these signatures (all used already in `lib/`).
  2. `Path.dirname` exists and is **not** miscompiled (A3 Step 5) — fall back to `String.last_index_of` + `slice_bytes` if it is.
  3. How a `type = "tool"` forge package is invoked after `forge build` (the binary path) — used in B3/B4/C1.
  4. How the March toolchain is installed in CI (C1).
- **Risk gate:** A3 must pass a **native** `forge build`/test (A3 Step 8), not just interpreter tests. If it regresses, ship flat output (unique basenames) and drop the nested-path bits from B (adjust nav links to flat slugs).

---

## Execution order summary

```
A1 → A2 → A3 → A4   (sigil repo; A3 optional/gated)
B1 → B2 → B3 → B4 → B5 → B6   (march-www)
C1 → C2   (deploy)
```
