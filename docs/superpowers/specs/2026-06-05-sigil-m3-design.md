# Sigil M3 — Templating and Layouts

## Goal

Replace the hardcoded `Html.layout` call with a user-controlled layout system. Site authors write `~H` layout templates in March, get type checking and LSP support, and dispatch to different layouts by front-matter field.

## Architecture

Sigil gains two new library modules (`Site`, `Layout`) and a new entry point (`Build.run_with_layout`). The existing `Build.run` is preserved unchanged for the bare `sigil build` CLI. User sites are forge `--app` projects that depend on sigil as a library, wire site info and a layout function in `main.march`, and compile everything together.

## New modules in `lib/`

### `lib/site.march` — `mod Site`

Site-wide metadata passed to every layout call.

```march
type Info = {
  title       : String,
  base_url    : String,
  author      : String,
  description : String,
}

fn default() : Info do
  { title = "My Site", base_url = "", author = "", description = "" }
end
```

**Note: no `lib/layout.march` module.** `LayoutFn` lives in `mod Build` (see below), not a separate module, to avoid a naming conflict with the user's own layout module.

## Updated `lib/build.march`

### New types and helpers added to `mod Build`

```march
-- Wrapper for a user-supplied layout function.
-- Avoids March's curried function-type annotation ambiguity.
-- Users wrap their render function: Build.LayoutFn(MyTemplates.render)
type LayoutFn = LayoutFn(Site.Info -> Page.Meta -> IOList -> IOList)

-- Apply a layout function using explicit let-bindings so Perceus can
-- track liveness of each intermediate closure.
pfn apply_layout(lf : LayoutFn, site : Site.Info, page : Page.Meta, content : IOList) : IOList do
  match lf do
  LayoutFn(f) ->
    let f1 = f(site)
    let f2 = f1(page)
    f2(content)
  end
end

-- Default layout (used by Build.run): wraps Html.layout.
pfn default_layout_fn() : LayoutFn do
  LayoutFn(fn site -> fn page -> fn content ->
    Html.layout(page.title, IOList.empty(), content)
  )
end
```

### New public function

```march
fn run_with_layout(cfg : Config.Site, site : Site.Info, lf : LayoutFn) : Result(Unit, String)
```

Identical pipeline to `Build.run` but calls `apply_layout(lf, site, page_meta, md_html)` instead of `Html.layout(title, ...)`. Passes `Site.Info` and `LayoutFn` through `build_all` as direct parameters (no string extraction needed — verified safe with the existing Perceus fix).

### Unchanged

```march
fn run(cfg : Config.Site) : Result(Unit, String)
```

Becomes a one-line wrapper:
```march
fn run(cfg : Config.Site) : Result(Unit, String) do
  run_with_layout(cfg, Site.default(), default_layout_fn())
end
```

All existing CLI behaviour (`sigil build`) is preserved with zero change.

## User site structure

A site using custom layouts is a forge `--app` project:

```
mysite/
  forge.toml          ← [deps] sigil = { path = "/path/to/sigil" }
  lib/
    layout.march      ← mod Layout — ~H templates + dispatch on page.layout
    main.march        ← mod Main — site info + wires Build.run_with_layout
  content/
    index.md          ← layout: page  (or omit for default)
    posts/
      hello.md        ← layout: post
  static/
    style.css
```

### `lib/layout.march` (user-written)

```march
mod Layout do

  fn render(site : Site.Info, page : Page.Meta, content : IOList) : IOList do
    match page.layout do
    "post" -> post(site, page, content)
    _      -> base(site, page, content)
    end
  end

  pfn base(site : Site.Info, page : Page.Meta, content : IOList) : IOList do
    ~H"""
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>#{page.title} | #{site.title}</title>
      <link rel="stylesheet" href="/style.css">
    </head>
    <body>
      <header><a href="/">#{site.title}</a></header>
      <main>#{content}</main>
      <footer>#{site.author}</footer>
    </body>
    </html>
    """
  end

  pfn post(site : Site.Info, page : Page.Meta, content : IOList) : IOList do
    ~H"""
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>#{page.title} | #{site.title}</title>
      <link rel="stylesheet" href="/style.css">
    </head>
    <body>
      <header><a href="/">← #{site.title}</a></header>
      <article>
        <h1>#{page.title}</h1>
        <time>#{page.date}</time>
        #{content}
      </article>
    </body>
    </html>
    """
  end

end
```

### `lib/main.march` (user-written)

```march
mod Main do

  fn main() do
    let site_info = {
      title       = "My Blog",
      base_url    = "https://myblog.com",
      author      = "Ada Lovelace",
      description = "Writing about March and compilers.",
    }
    match Ssg.parse_args(System.argv()) do
    Err(msg) -> println(msg)
    Ok(build_cfg) ->
      match Build.run_with_layout(build_cfg, site_info, Build.LayoutFn(Layout.render)) do
      Err(msg) -> println("error: " ++ msg)
      Ok(_)    -> println("done.")
      end
    end
  end

end
```

### Building the site

```bash
# Interpreted (fast dev iteration)
MARCH_LIB_PATH=/path/to/sigil/lib:lib \
  march lib/main.march

# Compiled (production)
MARCH_LIB_PATH=/path/to/sigil/lib:lib \
  march --compile -o mysite lib/main.march && ./mysite
```

Or via forge tasks (defined in the site's `forge.toml`).

## Test coverage

Tests live in `test/sigil_test.march` (the sigil library's own tests):

1. `Site.default()` returns expected defaults
2. `Layout.default_fn()` produces `<!DOCTYPE html>` output
3. `Layout.apply` calls the wrapped function with correct args
4. `Build.run_with_layout` produces HTML using the provided layout
5. `Build.run` still works unchanged (regression)
6. Front-matter `layout: post` is passed through to the layout function

The test blog (`sigil_blog`) is updated to a forge project with a custom layout to verify end-to-end.

## RC safety

Verified by reproducer: passing `Site.Info` (record) and `Layout.LayoutFn` (wrapped closure) through a recursive `build_all` loop compiles and runs correctly under the existing Perceus fix (`0b52510`). No string-extraction workaround needed.

## Out of scope for M3

- `sigil new` scaffold command (deferred to M4)
- `Serve.run_with_layout` — dev server layout support (can be added once `task_spawn` bug is fixed)
- CSS/asset pipeline (M3 uses `Assets.copy` as-is from M2)
- Collection pages, pagination, feeds (M4)
