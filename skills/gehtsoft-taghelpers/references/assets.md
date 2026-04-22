# Local Assets Reference — Gehtsoft.TagHelpers

How to wire project-owned CSS and JavaScript alongside the output of the `<x-*>` tag helpers. This covers the assets *you* own — site stylesheets, per-feature scripts, inline behavior on a widget — not the Kendo UI assets (those live in `setup.md` §5).

## Contents

- [1. Three layers of assets](#1-three-layers-of-assets)
- [2. wwwroot layout for a tag-helper project](#2-wwwroot-layout-for-a-tag-helper-project)
- [3. Site CSS — plain `<link>` vs bundled](#3-site-css--plain-link-vs-bundled)
- [4. Site JS — plain `<script>` vs bundled](#4-site-js--plain-script-vs-bundled)
- [5. BundleFactory — the full API](#5-bundlefactory--the-full-api)
   - [5.1 Creating a bundle](#51-creating-a-bundle)
   - [5.2 Appending content (inline / from string)](#52-appending-content-inline--from-string)
   - [5.3 Appending files](#53-appending-files)
   - [5.4 Appending embedded resources](#54-appending-embedded-resources)
   - [5.5 Emitting the bundle — `<x-bundle>`](#55-emitting-the-bundle--x-bundle)
   - [5.6 Cache busting (content-hash URLs)](#56-cache-busting-content-hash-urls)
- [6. Per-page assets — `@section`](#6-per-page-assets--section)
- [7. Embedded JS via `<x-script>`](#7-embedded-js-via-x-script)
- [8. Ordering rules — what must come before what](#8-ordering-rules--what-must-come-before-what)
- [9. When to bypass `<x-lib-includes>`](#9-when-to-bypass-x-lib-includes)
- [10. Differences between Kendo 2022 and 2026 for local assets](#10-differences-between-kendo-2022-and-2026-for-local-assets)

---

## 1. Three layers of assets

A tag-helper project serves assets from three distinct layers. Keep them mentally separate — each has its own wiring:

| Layer | What it is | How it's included |
|-------|-----------|-------------------|
| **Vendor (Kendo UI)** | jQuery, Kendo scripts, Kendo theme CSS | `<x-lib-includes>` (emits bundled `<link>`/`<script>` pointing into `wwwroot/lib/Kendo.UI/`) |
| **Library internals** | `taghelper.css`, `taghelper-bootstrap.css` / `taghelper-other.css`, `formAjax.js`, and other JS resources embedded inside the `Gehtsoft.TagHelpers` assembly | Automatically stitched into the same bundle as the Kendo scripts/styles by `<x-lib-includes>`; you do not reference these by filename |
| **Local (your project)** | `wwwroot/css/site.css`, `wwwroot/js/site.js`, per-feature JS, inline event handlers, app icons, fonts | **This document.** Either plain `<link>`/`<script>` tags in `_Layout.cshtml`, or `<x-bundle>` backed by a `BundleFactory` bundle, or `<x-script>` for inline JS |

The library never ships local CSS/JS for you. It provides the plumbing (`BundleFactory`, `<x-bundle>`, `<x-script>`) and leaves *what* to include up to the app.

## 2. wwwroot layout for a tag-helper project

A typical project's `wwwroot/` looks like:

```
wwwroot/
├── css/
│   ├── site.css                  ← app-wide stylesheet
│   └── <feature>.css             ← optional per-feature styles
├── js/
│   ├── site.js                   ← app-wide script
│   └── <feature>.js              ← optional per-feature scripts
├── img/
│   └── logo.png
├── fonts/
│   └── ...
├── favicon.ico
└── lib/
    └── Kendo.UI/                 ← vendored per setup.md §5 (gitignored)
```

The `lib/` folder is owned by the Kendo install step; everything else is app source (checked into git). Nothing stops you from adding further top-level folders (`wwwroot/i18n/`, `wwwroot/templates/`, etc.) as long as `UseStaticFiles()` is in the pipeline.

## 3. Site CSS — plain `<link>` vs bundled

### 3.a Plain `<link>` (simplest, fine for most apps)

```cshtml
<head>
    <x-lib-includes theme="bootstrap" minimize="true" />
    <link rel="stylesheet" href="~/css/site.css?v=5" />
</head>
```

- Order matters: `<x-lib-includes>` first so Kendo's theme CSS sits *below* your overrides in the cascade and your rules win.
- `?v=5` is a manual cache-buster; bump it when you change `site.css`.
- You can include as many per-feature stylesheets as you want this way.

### 3.b Bundled (cache-friendly, content-hashed URL)

If you want the ASP.NET Core pipeline to serve your CSS with a fingerprinted URL (so browsers aggressively cache and automatic invalidation on change), register the bundle in `Configure`:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, BundleFactory bundles)
{
    bundles.CreateBundle("site-styles", BundleType.Styles);
    await bundles["site-styles"].AppendFileAsync(
        Path.Combine(env.WebRootPath, "css/site.css"),
        preMinimized: false);
    // Optionally append more files to aggregate them into one request:
    // await bundles["site-styles"].AppendFileAsync(
    //     Path.Combine(env.WebRootPath, "css/print.css"), preMinimized: false);
}
```

Then reference from the layout:

```cshtml
<head>
    <x-lib-includes theme="bootstrap" minimize="true" />
    <x-bundle name="site-styles" minimize="true" />
</head>
```

`<x-bundle>` emits a `<link rel="stylesheet" href="...">` whose URL includes a hash of the current content — when the file changes, the URL changes, and browsers re-fetch automatically.

## 4. Site JS — plain `<script>` vs bundled

### 4.a Plain `<script>`

```cshtml
<head>
    <x-lib-includes theme="bootstrap" minimize="true" />
    <script src="~/js/site.js?v=3"></script>
</head>
```

- Plain `<script>` in `<head>` with no `defer`/`async` runs before the document is parsed — so use it for scripts that only *define* helpers (functions, objects). Do not reach into the DOM from a `<head>` script unless you wrap it in `$(function() { ... })`.
- To run against the DOM without jQuery ready wrapping, put the `<script>` near the end of `<body>`, or prefer `<x-script>` (which wraps in `$(document).ready`).

### 4.b Bundled script (single content-hashed URL, aggregates many sources)

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, BundleFactory bundles)
{
    bundles.CreateBundle("site-scripts", BundleType.Scripts);

    // Inline string (the "site-scripts" convention the library uses internally):
    bundles["site-scripts"].AppendContent("/* shared helpers */", preMinimized: false);

    // A file on disk:
    await bundles["site-scripts"].AppendFileAsync(
        Path.Combine(env.WebRootPath, "js/site.js"),
        preMinimized: false);

    // Something pulled from an embedded resource (see §5.4):
    bundles["site-scripts"].AppendContent(
        GetEmbeddedScript(typeof(MyTypeInAssembly)),
        preMinimized: false);
}
```

```cshtml
<head>
    <x-lib-includes theme="bootstrap" minimize="true" />
    <x-bundle name="site-scripts" minimize="true" />
</head>
```

The name `site-scripts` is just the convention used by the library's own test apps — you can use any name. Multiple bundles are fine (e.g., `"site-scripts"` for always-on code, `"admin-scripts"` for admin-only pages referenced only from an admin layout).

## 5. BundleFactory — the full API

`BundleFactory` is registered by `AddTagHelperServices(env)` (see `setup.md` §2). It's the single point for registering, populating, and retrieving bundles.

### 5.1 Creating a bundle

```csharp
bundles.CreateBundle(string name, BundleType type);
// BundleType.Scripts  → served as application/javascript, URL ends in .js
// BundleType.Styles   → served as text/css, URL ends in .css
```

Creating a bundle is cheap — the factory only tracks the name and an empty content buffer. Creating a bundle that is never referenced from `<x-bundle>` is harmless.

### 5.2 Appending content (inline / from string)

```csharp
bundles["myBundle"].AppendContent(string content, bool preMinimized);
bundles["myBundle"].AppendContent(byte[] content, bool preMinimized, Encoding encoding = null);
```

- `preMinimized: true` — the content is already minimized; the bundle keeps it verbatim.
- `preMinimized: false` — when the bundle is served with `minimize="true"`, the non-pre-minimized portion will be run through `Yahoo.Yui.Compressor` (CssCompressor / JavaScriptCompressor). This happens once per digest; results are cached.

The byte-array overload strips a UTF-8 BOM if present.

### 5.3 Appending files

```csharp
await bundles["myBundle"].AppendFileAsync(
    string fileName,
    bool preMinimized,
    bool setAsBase = false,
    Encoding encoding = null);
```

- `fileName` is a **full file-system path**. Combine with `env.WebRootPath` when pulling from `wwwroot/`.
- `setAsBase: true` adds the file's directory to the bundle's search path for `@import`/`url(...)` resolution — useful for CSS. Set this on the *primary* CSS file; leave it `false` for secondary appends.
- The file's last-write-time is folded into the bundle's freshness check.

### 5.4 Appending embedded resources

A common pattern is to bundle JS that lives as embedded resources inside an assembly (for example, scripts packaged alongside a custom tag helper defined in your own codebase):

```csharp
public static string GetScript(Type type)
{
    var b = new StringBuilder();
    foreach (string s in type.Assembly.GetManifestResourceNames())
    {
        if (s.ToLower().EndsWith(".js"))
        {
            using var stream = type.Assembly.GetManifestResourceStream(s);
            using var reader = new StreamReader(stream);
            b.Append(reader.ReadToEnd());
        }
    }
    return b.ToString();
}

// In Configure — pass any type from the assembly whose embedded .js resources
// you want bundled (for example, a custom tag helper class you defined):
bundles["site-scripts"].AppendContent(GetScript(typeof(MyCustomTagHelper)), false);
```

`GetScript` pulls every `.js` manifest resource from the assembly containing the given type and concatenates them. Useful for packaging client-side code next to server-side tag helpers when your app defines its own.

### 5.5 Emitting the bundle — `<x-bundle>`

```cshtml
<x-bundle name="site-scripts" minimize="true" />
```

| Attribute | Purpose | Default |
|-----------|---------|---------|
| `name` | Bundle name (must match a `CreateBundle(...)` call) | *required* |
| `minimize` | When true, serves the post-compressor content; when false, serves the full content | `false` |

The tag emits:

- For `BundleType.Scripts`: `<script src="...">...</script>`
- For `BundleType.Styles`: `<link rel="stylesheet" href="...">`

The `src`/`href` is a controller route (`/Bundle/<name>/Content:<digest>?minimize=true`) served by `BundleController` — which the library registers automatically.

### 5.6 Cache busting (content-hash URLs)

Every bundle content change produces a new MD5 digest, which is embedded in the served URL. Browsers see a fresh URL and re-fetch — no manual `?v=N` query-string bumps needed.

If you want the same guarantee for static `<link>`/`<script>` tags that don't go through a bundle, either:

- Bump a query string (`~/css/site.css?v=6`) when you deploy. Cheap, fine for small teams.
- Run the file through a bundle instead. The bundle URL already changes on content change.

Be aware: the bundle content is captured at `Configure` time. If you `AppendFileAsync` a file and then change the file on disk while the app is running, the bundle's content doesn't automatically refresh until the app restarts. For dev-time hot reload, use plain `<link>`/`<script>` instead.

## 6. Per-page assets — `@section`

Use the ASP.NET Core layout section pattern for assets that apply to a single view:

```cshtml
@* _Layout.cshtml, inside <head> *@
<x-lib-includes theme="bootstrap" minimize="true" />
@RenderSection("Header", required: false)
```

```cshtml
@* SpecificView.cshtml *@
@section Header {
    <link rel="stylesheet" href="~/css/reports.css" />
    <script src="~/js/reports.js"></script>
}
```

Place the `@RenderSection` **after** `<x-lib-includes>` so page-specific rules override theme rules.

For page-specific JS that needs to run on ready, prefer `<x-script>` inside the view's body:

```cshtml
<x-script>
    console.log('reports page ready');
</x-script>
```

## 7. Embedded JS via `<x-script>`

`<x-script>` is the preferred way to attach JavaScript to a view or a specific widget. It participates in the library's script aggregation pipeline — scripts are collected and emitted in a coordinated `$(document).ready` block, which avoids ordering pitfalls.

```cshtml
<x-script>
    /* role defaults to "script" → wrapped in $(document).ready */
    kendoConsole.log('ready');
</x-script>

<x-script role="global">
    /* emitted at top level, runs before ready */
    function formatMoney(v) { return kendo.toString(v, 'c'); }
</x-script>

<x-button role="action" label="Click me">
    <x-script role="click">
        /* bound as the click handler for this button */
        alert(this.text());
    </x-script>
</x-button>
```

Full role list and semantics are in `common.md` § "Script roles — concept".

Mixing `<script>` and `<x-script>`:

- Use plain `<script>` in the layout for *unconditional, always-on* helpers (e.g., `site.js`).
- Use `<x-script>` for view-specific behavior and for handlers attached to tag-helper widgets — it has access to `this`, `e`, and the widget instance in a way plain `<script>` can only mimic by looking up elements after the fact.

## 8. Ordering rules — what must come before what

In `_Layout.cshtml`'s `<head>` the correct order is:

```cshtml
<head>
    <!-- 1. Character set / viewport / title first - no surprises -->
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>@ViewBag.Title</title>

    <!-- 2. Kendo + library (defines jQuery and Kendo globals) -->
    <x-lib-includes theme="bootstrap" minimize="true" />

    <!-- 3. Any bundle that depends on jQuery (most do) -->
    <x-bundle name="site-scripts" />

    <!-- 4. Site CSS (applies on top of Kendo theme CSS in the cascade) -->
    <link rel="stylesheet" href="~/css/site.css?v=1" />

    <!-- 5. Per-page additions from the view -->
    @RenderSection("Header", required: false)
</head>
```

Why:

- jQuery must be defined before any script that uses `$`. `<x-lib-includes>` is the source of jQuery on this stack, so it must come before *any* script that expects `$` — including your own `site.js` and any `<x-bundle>` derived from `AppendContent` that uses jQuery.
- Site CSS after Kendo theme CSS means your rules win the cascade.
- `@RenderSection("Header")` last so a view can override either.

Common mistake: putting a `<script src="~/js/site.js">` above `<x-lib-includes>`. It will throw `$ is not defined` on pages where `site.js` uses jQuery.

## 9. When to bypass `<x-lib-includes>`

Prefer `<x-lib-includes>`. Bypass it only when one of these is true:

- The page is a narrow isolated test harness and you don't want the embedded library CSS/JS at all.
- You have a custom CDN strategy that forbids runtime-combined bundles.
- You're rendering a fragment that gets embedded in a page that already has Kendo loaded from a different origin.

If you must bypass, you lose the automatic stitching of the library's own CSS/JS (`taghelper.css`, `formAjax.js`, and the other resources embedded in the assembly) — these have to be included manually, or many `<x-*>` widgets will look wrong or fail to initialize. The maintainable way to do that is still a bundle: build a `BundleFactory` bundle that appends the necessary pieces and reference that via `<x-bundle>`.

## 10. Differences between Kendo 2022 and 2026 for local assets

The `BundleFactory` API, `<x-bundle>` syntax, `<x-script>` semantics, and the `wwwroot/css/`, `wwwroot/js/` conventions are **identical** between the two drivers. Everything in this document works the same for both.

2022-vs-2026 considerations that touch local assets:

- **Underlying jQuery version.** 2022 ships jQuery **1.12.4**; 2026 ships jQuery **4.0.0**. Any app JS that uses `.live()`, `.size()`, `.andSelf()`, event shorthands like `.load(fn)` / `.error(fn)`, `$.browser`, or treats `.attr('checked')` as a boolean will **break** on 2026. See `setup.md` §5.3 and the migration checklist in `troubleshooting.md` → "jQuery or Bootstrap differences".
- **Underlying Bootstrap version.** 2022 ships Bootstrap **3.3.7**; 2026 ships Bootstrap **5.3.8**. Site CSS and Razor markup using `.panel`, `.well`, `.thumbnail`, `.btn-default`, `.glyphicon`, `.hidden-xs`, `.img-responsive`, or `data-toggle="…"` attributes will need to migrate to Bootstrap-5 equivalents (`.card`, `.btn-secondary`, `.d-none`, `.img-fluid`, `data-bs-toggle`, etc.).
- **Bootstrap JS bundle.** 2022 does **not** ship Bootstrap's own JS (apps that use dropdowns/modals typically pull Bootstrap 3 JS from a CDN or bundle it themselves). 2026 ships `script/bootstrap.bundle.min.js` locally — but `<x-lib-includes>` does **not** auto-load it. If the app uses Bootstrap 5 modals/tooltips/dropdowns, include it explicitly: `<script src="~/lib/Kendo.UI/script/bootstrap.bundle.min.js"></script>` or add it to a `<x-bundle>`.
- **Body class convention:** 2022 test app uses `<body class="k-content">`, 2026 test app uses `<body class="k-body">`. If your `site.css` targets the outer body class, match the driver you're on (or drive it from config as shown in `setup.md` §4).
- **Theme file minification:** 2022 theme files are `.min.css` (pre-minimized); 2026 theme files are un-minified `.css`. This has no effect on your local CSS, but it matters if you're stacking an `@import` or a `<link>` *before* the theme CSS — a pre-minimized file can't be restyled with un-closed rules from above.
- **Icons and fonts:** 2026 ships a `css/kendo/font-icons/` subfolder with icon fonts referenced by relative URL from the theme CSS. Don't move or rename that subfolder; `css/kendo/*.css` points at `font-icons/...` with a relative path.

Apart from the jQuery/Bootstrap version change (which is genuinely breaking for legacy app code), writing, bundling, and including local CSS/JS is driver-agnostic.
