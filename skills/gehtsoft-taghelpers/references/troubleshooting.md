# Troubleshooting — Gehtsoft.TagHelpers

Common errors, surprising behaviors, and what to check first.

## Contents

- [Setup & wiring](#setup--wiring)
- [Forms & controls](#forms--controls)
- [Grids](#grids)
- [Menus & navigation](#menus--navigation)
- [Scripts & templates](#scripts--templates)
- [Bundles & assets](#bundles--assets)

---

## Setup & wiring

### `<x-form>` renders as literal `<x-form>` HTML

The Razor parser doesn't know about the tag because `_ViewImports.cshtml` is missing the directive.

**Fix:**

```cshtml
@addTagHelper *, Gehtsoft.TagHelpers
```

Add to `Views/_ViewImports.cshtml`. Areas need their own `_ViewImports.cshtml` with the same line.

### Page loads but Kendo widgets never initialize (textboxes look like raw `<input>`)

No driver was registered. The legacy fallback path emits markup but skips Kendo init JS.

**Fix:** in `Program.cs` / `Startup.ConfigureServices`, call **exactly one** driver registration matching the vendored Kendo assets:

```csharp
services.AddTagHelperServices(builder.Environment);
services.AddKendo2022Driver();   // for Kendo UI 2022.x assets
// — OR —
services.AddKendo2026Driver();   // for Kendo UI 2026.x assets
```

See `setup.md` §2 for the full details.

### Page loads but looks broken — unstyled widgets, JS errors, 404s for `/lib/Kendo.UI/css/kendo/...` or `/script/...`

The Razor page itself renders fine (the `<x-lib-includes>` tag just emits `<link>`/`<script>` references — it doesn't fail the page). The problem surfaces when the browser then **fetches** the referenced CSS/JS: those fetches 404 because the driver expects filenames that aren't vendored, or the script bundle is incomplete. Visible symptoms:

- Styles missing (layout collapses to raw HTML flow, Kendo colors/spacing gone).
- Scripts missing (`kendoGrid`/`kendoDropDownList` init never runs, widgets stay as plain inputs).
- DevTools Console: `$ is not defined`, `kendo is not defined`, `Uncaught TypeError: ...(anonymous)... is not a function`.
- DevTools Network: 404 rows for files under `/lib/Kendo.UI/css/kendo/...` or `/lib/Kendo.UI/script/...`.

Common root causes:

- **No driver registered** — neither `AddKendo2022Driver()` nor `AddKendo2026Driver()` was called. See the previous entry.
- **Driver/asset generation mismatch.** The driver requests filenames that only exist in the other generation:
  - 2022 driver expects `kendo.<theme>.min.css`, `kendo.common-<family>.min.css`, `kendo.<theme>.mobile.min.css`, plus four scripts.
  - 2026 driver expects `<family>-<variant>.css` (no `.min.` infix, no separate common/mobile), plus four scripts.

**Fix:** either switch the driver call (`AddKendo2022Driver()` ↔ `AddKendo2026Driver()`) or re-vendor the matching Kendo generation (`setup.md` §5.1 for 2022 layout, §5.2 for 2026 layout).

Quick check: open `wwwroot/lib/Kendo.UI/css/kendo/` — if it contains `kendo.*.min.css` filenames, it's 2022; if it contains `<family>-<variant>.css` filenames (e.g. `default-main.css`, `bootstrap-main.css`), it's 2026. Match the `AddKendoXXXXDriver()` call to what you see.

### jQuery or Bootstrap differences after switching Kendo 2022 ↔ 2026

The two Kendo generations ship **different** jQuery and Bootstrap versions alongside Kendo itself:

| Vendor file | Kendo 2022 ships | Kendo 2026 ships |
|-------------|------------------|------------------|
| `lib/Kendo.UI/script/jquery.min.js` | jQuery **1.12.4** | jQuery **4.0.0** |
| `lib/Kendo.UI/css/bootstrap.min.css` | Bootstrap **3.3.7** | Bootstrap **5.3.8** |
| `lib/Kendo.UI/script/bootstrap.bundle.min.js` | *not shipped* | Bootstrap 5 JS bundle (incl. Popper) |
| `lib/Kendo.UI/script/jquery.ui.min.js` | *not shipped* | jQuery UI (shipped, not loaded by `<x-lib-includes>`) |
| `lib/Kendo.UI/script/angular.min.js`, `kendo.angular.min.js` | shipped (legacy) | *not shipped* |

This is a **breaking change** across the jump — app code that assumed the 1.x jQuery or Bootstrap-3 markup will stop working cleanly on 2026. Watch for:

**Removed in jQuery 3.x / 4.x that 1.12 still has:**
- `.size()` → use `.length`.
- `.load(fn)`, `.unload(fn)`, `.error(fn)` event shorthands → use `.on('load', fn)` etc.
- `.live()` / `.die()` (event delegation) → use `.on(event, selector, fn)` on a parent.
- `.andSelf()` → renamed to `.addBack()`.
- `$.browser` → gone.
- `.toggle(fn1, fn2)` (alternating click handler) → gone.
- Property access via `.attr('checked')` returning boolean → use `.prop('checked')`.

**Breaking CSS from Bootstrap 3 → 5 (via Bootstrap 4):**
- `.panel`, `.panel-default`, `.panel-body`, `.well`, `.thumbnail` — gone; replaced by `.card`.
- `.btn-default` → `.btn-secondary` (Bootstrap 4+).
- `.hidden-xs` / `.visible-xs` etc. → replaced by `.d-none`, `.d-*-block` responsive utilities.
- `.navbar-fixed-top` → `.fixed-top` + `.navbar`.
- `.img-responsive` → `.img-fluid`.
- `.col-xs-*` → `.col-*` (the "xs" breakpoint is the default, no prefix).
- `.col-sm-6 col-md-4` grid syntax works but column behavior at breakpoints has shifted; retest responsive layouts.
- Glyphicons (`.glyphicon`) removed — switch to Kendo's `k-i-*` icons or Bootstrap Icons.
- Data attributes changed: `data-toggle` → `data-bs-toggle`, `data-target` → `data-bs-target`, etc.

**Symptom patterns after a 2022 → 2026 migration:**
- App-specific JS throws `TypeError: $(...).live is not a function` or similar → legacy jQuery API usage, fix per the list above.
- A Bootstrap-styled page layout collapses, buttons look grey/unstyled, navbars break → Bootstrap-3 class names no longer match rules in the 5.3 CSS.
- Modals/dropdowns that used `data-toggle="modal"` silently stop working → missing the `-bs-` infix.

**Fix:** audit `site.js`, `site.css`, and Razor views for the patterns above. Running the page against `<body class="k-body">` + 2026 assets and fixing each DevTools error one-by-one is usually the fastest path.

Note: `<x-lib-includes>` only loads `jquery.min.js`, `jquery.cookie.min.js`, `jszip.min.js`, and `kendo.all.min.js`. `bootstrap.bundle.min.js` and `jquery.ui.min.js` are available in the 2026 tree but **not auto-loaded** — if you want them, add your own `<script>` tag or `<x-bundle>` entry.

### `<x-lib-includes>` emits nothing or browser shows 404s for `kendo.*.min.js`

The Kendo files aren't where the library expects them. Open DevTools → Network and check the failing URLs — they should be under `/lib/Kendo.UI/script/...` and `/lib/Kendo.UI/css/kendo/...`.

**Fix:** ensure `wwwroot/lib/Kendo.UI/` matches the layout in `setup.md` § 5. Common omissions: `jquery.min.js` (must load before `kendo.all.min.js`), the per-theme CSS (`kendo.<theme>.min.css`).

If the project uses `Gehtsoft.Build.ContentDelivery` (`setup.md` § 5a) and `wwwroot/lib/Kendo.UI/` is empty after a fresh checkout, the `Content` target was never invoked — a normal `dotnet build` does not run it. Run it explicitly:

```bash
dotnet build YourApp.csproj -t:CleanContent,Content
```

### `Unable to resolve service for type 'IUrlHelper'`

A control needs the URL helper but the service is missing. `AddTagHelperServices()` registers a fallback, but if you also call `services.AddSingleton<IUrlHelper>(...)` somewhere with bad lifetime semantics it can clash.

**Fix:** ensure `AddTagHelperServices` is called *after* `AddMvc()`/`AddControllersWithViews()`, and don't manually register `IUrlHelper`.

### `AddKendo2022Driver` / `AddKendo2026Driver` exists but tags still don't initialize

The driver registers `ITagLibraryDriver` and `ILibraryProvider` as singletons via `TryAddSingleton` — meaning if you registered something else for those interfaces earlier, the driver is silently skipped.

**Fix:** check for stray `services.AddSingleton<ITagLibraryDriver>(…)` lines in your codebase. There should be only one. Also verify you're not calling both `AddKendo2022Driver()` and `AddKendo2026Driver()` — the second one is a no-op thanks to `TryAddSingleton`, but the intent mismatch usually means the vendored assets are wrong for the active driver (see the "404s in DevTools" entry above).

### Theme value works on one driver but not the other

Driver-specific theme vocabularies (see `setup.md` §6):

- `default-v2`, `office365`, `bootstrap-v4`, `fiori`, `material-v2`, `nova` — valid on Kendo 2022 (map to distinct `.min.css` files); on Kendo 2026 they're remapped to generic families (most collapse to `default-main` / `bootstrap-4` / `material-main` / `classic-main`).
- `classic`, `fluent`, and any variant filename like `default-main-dark`, `bootstrap-3`, `classic-opal` — valid on Kendo 2026 only. Passing these to the 2022 driver will produce a 404 for a file like `kendo.classic.min.css` which doesn't exist.

**Fix:** pick a theme value valid on the driver you're using. If the app needs to support both, drive the theme from configuration and use values that exist on both (`default`, `bootstrap`, `material`).

---

## Forms & controls

### Checkbox label appears twice

In `twocolumn` mode the label takes the left column and the checkbox the right; the checkbox itself also has its own inline label. Looks like duplication.

**Fix:** `suppress-label="true"` to drop the left-column label, OR move to `inline` mode.

```cshtml
<x-check for="@Model.AgreeToTerms" label="I agree to the terms" suppress-label="true" />
```

### Date picker shows "Invalid date" or wrong format

`format` uses **Kendo** date patterns, not .NET. Common mistakes:

- `mm/dd/yyyy` → use `MM/dd/yyyy` (lowercase `mm` is minutes)
- `HH:mm tt` → use `h:mm tt` (lowercase `h` for 12-hour)
- `MMM d, yy` → use `MMM d, yyyy`

```cshtml
<x-date-edit for="@Model.At" format="MM/dd/yyyy h:mm tt" include-time="true" />
```

### Client-side validation never fires

`<x-form-errors>` is missing or `support-on-client-validation` is false.

**Fix:**

```cshtml
<x-form for="@Model" method="post">
    <x-form-errors form-errors="@Model.Errors" support-on-client-validation="true" />
    <!-- controls + <x-validate> rules -->
</x-form>
```

### AJAX form post returns 400 / model is null

`method="ajax"` posts JSON, but nested model property paths produce JSON-unfriendly keys (`"Address.Street"`).

**Fix options:**

1. Switch to `method="post"` for forms with nested models (default form-encoded handles `Address.Street` fine).
2. Flatten the model: hidden fields for ids, scalar properties only.
3. Use `[FromBody]` on the action **and** a flat DTO matching the posted shape.

```csharp
[HttpPost]
public IActionResult Save([FromBody] CustomerSaveDto dto) { ... }
```

### Validation regex with `@` doesn't compile

Razor sees `@` as a code transition.

**Fix:** double it. `@@` becomes a literal `@` in the rendered JS.

```cshtml
<x-validate is-expression="true" error="Bad email">
    /^[^\s@@]+@@[^\s@@]+\.[^\s@@]+$/.test(value)
</x-validate>
```

### Async upload finishes, server saves, but client shows error

The server returned an empty body or wrong shape. For chunked uploads the server **must** return JSON matching `AsyncUploadResult`:

```csharp
return Json(new AsyncUploadResult { IsCompleted = isFinalChunk });
```

For non-chunked sync uploads, return `Content("")` for success or a non-empty error string.

### `for="@Model.X"` shows label "X" instead of the friendly name

The library derives the label from the property name. To override:

```cshtml
<x-edit for="@Model.X" label="Friendly name" />
```

For project-wide friendly names, decorate the model property with `[Display(Name = "Friendly name")]` — the library will pick it up.

### Radio buttons don't bind together

All radios bound to the same property must be inside the same `<x-form>` and use `for="@Model.Same"`. They share the rendered name.

```cshtml
<x-form for="@Model">
    <x-radio for="@Model.Choice" option="A" label="Option A" />
    <x-radio for="@Model.Choice" option="B" label="Option B" />
</x-form>
```

---

## Grids

### Grid shows "no records" but server returned data

The schema's `result-name` doesn't match the JSON envelope.

**Fix:** either return `{ "data": [...], "total": N }` (Kendo defaults), or override:

```cshtml
<x-server-schema result-name="items" total-name="count" />
```

### Pager shows `1 - 20 of 0` and grid is empty

Server returned the page but didn't include `total`. The pager needs both.

**Fix:**

```csharp
return Json(new { data = pageItems, total = totalCount });
```

### Sorting/filtering/paging happens on the client even with large datasets

Server-side modes are off.

**Fix:** set all three on `<x-server-datasource>`:

```cshtml
<x-server-datasource page-size="20"
                     server-paging="true"
                     server-sorting="true"
                     server-filtering="true">
```

### `[DataSourceRequest]` doesn't compile

The `KendoUI.DataRequest` package is not referenced. Either add it, or parse `skip`/`take`/`sort`/`filter` manually from the request:

```csharp
[HttpPost]
public IActionResult List(int skip, int take, string sort, string filter)
{
    // Parse sort/filter JSON if present, then page the data.
}
```

### "Add new" button is missing on an editable grid

A toolbar must be defined explicitly:

```cshtml
<x-grid editable="inline">
    <x-script script-role="byname" script-role-name="toolbar">
        return [{ name: "create", text: "Add", iconClass: "k-i-plus" }];
    </x-script>
    <!-- columns + datasource -->
</x-grid>
```

### Edit/Delete buttons in template column don't work

Kendo wires action buttons by CSS class. Use `k-grid-edit`, `k-grid-delete`, `k-grid-update`, `k-grid-cancel`:

```cshtml
<x-template>
    <a class="k-button k-grid-edit"   href="\#">Edit</a>
    <a class="k-button k-grid-delete" href="\#">Delete</a>
</x-template>
```

(The leading `\#` is a Razor-escaped `#` so it doesn't get interpreted as a Kendo template expression.)

### Dates render as `/Date(1234567890000)/` or as raw ISO strings

Field type is wrong on the schema:

```cshtml
<x-server-field name="createdAt" field-type="date" />
```

For columns add a format:

```cshtml
<x-grid-column title="Created" field="createdAt" format="{0:d}" />
```

### Template column shows "[object Object]" or HTML escapes incorrectly

Forgot `htmlEncode` on user-controlled text, or used `#: e.x #` instead of `#= e.x #`. The latter is the standard interpolation; `#: e.x :#` is the encoding-aware variant in some Kendo versions.

```cshtml
<x-template>
    <span>#= kendo.htmlEncode(e.name) #</span>
</x-template>
```

---

## Menus & navigation

### Menu items ignore `permissions` attribute

No `auth-info` provided, so the menu can't evaluate.

**Fix:** pass an `IAuthInfo` instance:

```csharp
public sealed class AuthContext : IAuthInfo
{
    public AuthContext(ClaimsPrincipal user) { User = user; }
    public ClaimsPrincipal User { get; }
    public bool IsLoggedIn => User?.Identity?.IsAuthenticated ?? false;
    public bool HasPermission(string p) => User?.IsInRole(p) ?? false;
}
```

```cshtml
<x-menu menu-id="main" auth-info="@(new AuthContext(User))">
    <!-- items with permissions="Admin,Editor" -->
</x-menu>
```

### Mobile menu shows but doesn't open

The mobile menu CSS expects the hamburger icon on a dedicated container. Ensure both `menu-id` and `mobile-menu-id` are set, and the rendered mobile container is in the DOM (some layouts hide it via CSS — confirm responsive breakpoints aren't hiding the toggle).

---

## Scripts & templates

### `#= e.x #` template expressions appear literally on the page

The expression is outside an `<x-template>` (or `<x-script role="template">`). The `#=` syntax is parsed only inside template scopes.

**Fix:** wrap in `<x-template>` or move the expression into a `<x-script>` event handler that builds a string.

### `<x-script role="click">` on a button never fires

The button's `role` must be `action` for the click handler to wire. `role="button"` is a presentational button without events.

```cshtml
<x-button role="action" label="Click me">
    <x-script role="click">alert('hi');</x-script>
</x-button>
```

### Scripts referencing `$(...)` fail with "jQuery is not defined"

`<x-lib-includes>` (or the equivalent jQuery `<script>`) is not in the layout, OR a per-page script tag is emitted *above* `<x-lib-includes>`.

**Fix:** keep `<x-lib-includes>` in the `<head>` of the layout; let the library handle script ordering.

---

## Bundles & assets

### `<x-bundle name="X">` emits nothing

The bundle was not created. `BundleFactory` only emits bundles that exist.

**Fix:** create the bundle in `Configure`:

```csharp
public void Configure(IApplicationBuilder app, BundleFactory bundles)
{
    bundles.CreateBundle("X", BundleType.Scripts);
    bundles["X"].AppendContent("/* JS */", false);
}
```

Or add a file:

```csharp
await bundles["X"].AppendFileAsync("path/to/file.js", preMinimized: true, setAsBase: false);
```

### Library inline JS/CSS appears unminified despite `minimize="true"`

`minimize="true"` on `<x-lib-includes>` only affects the inline library payload. Kendo's own `.min.js` files are already minimized and emitted as-is.

If you also need site-wide minification, configure your bundles to use `BundleFactory`'s minification (the library uses `Yahoo.Yui.Compressor` internally).

### Assets are stale after deployment

Browsers cache by URL. The library's bundle URLs include a content hash, so changing the underlying file should produce a new URL. If you're seeing stale content:

- Verify the bundle's content actually changed (hash in URL is stable per content).
- Hard-refresh (Ctrl+Shift+R / Cmd+Shift+R) to confirm it's a cache issue.
- Ensure `UseStaticFiles()` is in the pipeline.

---

## When in doubt

1. **Check `_ViewImports.cshtml` and `Program.cs` first** — 90% of "the tag isn't working" issues are missing wiring.
2. **Open DevTools → Console and Network** — most runtime issues surface as JS errors or 404s for assets.
3. **Compare against `setup.md` § 7 (smoke test)** — does the minimal form work? If yes, the wiring is fine; the issue is specific to the misbehaving feature.
