# Esbuild Integration for Sigil SSG

**Date:** 2026-06-21  
**Status:** Draft

## Goal

Add a first-class `Esbuild` module to sigil that lets site authors bundle CSS and JS as part of the sigil build pipeline, without writing their own shell commands. The API is shaped as `entry → output` so a future March-to-JS compiler can slot in with an identical call signature.

---

## Background

`Assets.march` currently copies `static/` verbatim — no build pipeline exists. Site authors wire in esbuild manually today (`by_chase` calls `npx esbuild ... >/dev/null 2>&1` via `Process.run("sh", ...)`). The new module replaces that boilerplate with a typed, composable API.

---

## Module: `lib/esbuild.march`

### Public API

```march
mod Esbuild do

  -- Bundle an entry point to an output file.
  -- Equivalent to: npx esbuild <entry> --bundle --outfile=<output>
  -- Works for both CSS and JS entry points.
  fn bundle(entry : String, output : String) : Result(Unit, String)

  -- Bundle with minification for production.
  -- Adds: --minify --legal-comments=none
  fn bundle_minify(entry : String, output : String) : Result(Unit, String)

  -- Hook wrappers for use in Hook.Set.before.
  fn hook(entry : String, output : String) : Hook.Fn
  fn minify_hook(entry : String, output : String) : Hook.Fn

end
```

### Invocation

Use `npx esbuild` — no global install required, only Node.js. Always pass `--bundle`. Output directory is created by esbuild automatically.

Progress is printed: `[esbuild] src/css/site.css → static/css/site.css`

### Error handling

| Failure | Error message |
|---------|---------------|
| `npx` not found (Node.js not installed) | `esbuild requires Node.js — install from nodejs.org` |
| Non-zero exit (syntax error, bad import) | `esbuild failed: <stderr>` |

---

## Serve mode

esbuild runs **once at serve startup**, before the first content build. CSS/JS source changes require a server restart to pick up — this avoids the esbuild/watcher race condition encountered in by_chase (esbuild writing `static/css/site.css` after the watcher snapshots mtimes, triggering a spurious content rebuild).

Watch-mode integration — re-running esbuild automatically when `src/` files change — is deferred. The watcher would need to be extended with a configurable source-watch directory and a hook list, which is a larger change.

---

## Extension point: March-to-JS

The `bundle` / `hook` signatures are entry-point-oriented: `(entry : String, output : String)`. A future `MarchJs` module will expose:

```march
MarchJs.hook(entry : String, output : String) : Hook.Fn
```

with the same signature. Site authors add it to `Hook.Set.before` alongside `Esbuild.hook` calls — no call-site changes needed for the rest of the pipeline.

---

## Usage in a site

### Build (production)

```march
let before = Cons(Esbuild.minify_hook("src/css/site.css", "static/css/site.css"),
             Cons(Esbuild.minify_hook("src/js/app.js",    "static/js/app.js"), Nil))
let hooks = { before: before, after: Nil }
Build.run_with_selector_hooks_and_index(cfg, selector(), hooks, homepage_index_fn())
```

### Serve (development)

```march
-- Run esbuild once before starting the watcher.
match Esbuild.bundle("src/css/site.css", "static/css/site.css") do
Err(msg) -> println("error: " ++ msg)
Ok(_)    -> Serve.run_with_selector_and_index(cfg, selector(), "lib", homepage_index_fn(), port)
end
```

This replaces the hand-rolled `build_css()` in by_chase.

---

## What is NOT in scope

- `--watch` / `--serve` integration (deferred — requires watcher extension)
- TypeScript / JSX / React support (esbuild handles these automatically; no sigil config needed)
- Multiple output formats (ESM, CJS, IIFE) — `--format` flag is not exposed; site authors can use `Hook.shell` for advanced cases
- Source maps — not exposed in this release

---

## Files changed

| File | Change |
|------|--------|
| `lib/esbuild.march` | New module |

No changes to `Build`, `Serve`, `Assets`, `Hook`, or `Watcher` — esbuild composes into the existing hook system.
