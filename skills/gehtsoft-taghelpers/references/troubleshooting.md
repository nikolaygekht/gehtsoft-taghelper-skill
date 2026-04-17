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

`AddKendo2022Driver()` was not called. The legacy fallback path emits markup but skips Kendo init JS.

**Fix:** in `Program.cs` / `Startup.ConfigureServices`:

```csharp
services.AddTagHelperServices(builder.Environment);
services.AddKendo2022Driver();
```

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

### `AddKendo2022Driver` exists but tags still don't initialize

The driver registers `ITagLibraryDriver` and `ILibraryProvider` as singletons via `TryAddSingleton` — meaning if you registered something else for those interfaces earlier, the driver is silently skipped.

**Fix:** check for stray `services.AddSingleton<ITagLibraryDriver>(…)` lines in your codebase. There should be only one.

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
