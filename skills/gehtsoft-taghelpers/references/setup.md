# Setup Reference — Gehtsoft.TagHelpers

Detailed installation and wiring of the library into an ASP.NET Core MVC project.

## Contents

- [1. NuGet package](#1-nuget-package)
- [2. DI registration](#2-di-registration)
- [3. `_ViewImports.cshtml`](#3-_viewimportscshtml)
- [4. `_Layout.cshtml` head](#4-_layoutcshtml-head)
- [5. Kendo UI assets in `wwwroot/lib/Kendo.UI/`](#5-kendo-ui-assets-in-wwwrootlibkendoui)
   - [5a. Bower (preferred)](#5a-bower-preferred)
   - [5b. Manual placement](#5b-manual-placement)
- [6. Themes](#6-themes)
- [7. Verification — minimal smoke test](#7-verification--minimal-smoke-test)
- [8. Default ASP.NET MVC template — what to change](#8-default-aspnet-mvc-template--what-to-change)

---

## 1. NuGet package

Add to the project's `.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="Gehtsoft.TagHelpers" Version="*" />
</ItemGroup>
```

Or via CLI:

```bash
dotnet add package Gehtsoft.TagHelpers
```

Optional companion packages:

| Package | Purpose | Install when |
|---------|---------|--------------|
| `Gehtsoft.TagHelpers.Utils` | Adapters for `Gehtsoft.Validator` / `Gehtsoft.Mapper` | The project already uses those validators |

## 2. DI registration

In `Program.cs` (minimal hosting model) or `Startup.ConfigureServices`:

```csharp
using Gehtsoft.TagHelpers;
using Gehtsoft.TagHelpers.Driver.Kendo2022;

// ...

builder.Services.AddTagHelperServices(builder.Environment);
builder.Services.AddKendo2022Driver();
```

Both calls are required. Order does not matter relative to each other, but they should be added before `builder.Build()`.

What each does:

- **`AddTagHelperServices(env)`** registers: `IWebHostEnvironment` (singleton), `IActionContextAccessor`, `IUrlHelper`, `XClassDictionary`, `ITagHelperResourceProvider`, `IBundleFileCache`, and `BundleFactory`. Safe to call after MVC is added.
- **`AddKendo2022Driver()`** registers: `ILibraryProvider` and `ITagLibraryDriver` as singletons. Without this, `<x-lib-includes>` falls back to a legacy code path and individual control tags will not render their JS initialization.

The `Startup.cs` form is identical:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();                        // or AddControllersWithViews()
    // ...
    services.AddTagHelperServices(CurrentEnvironment);
    services.AddKendo2022Driver();
}
```

`CurrentEnvironment` is the `IWebHostEnvironment` captured in the `Startup` constructor.

### Optional — site-wide script bundle

If the application needs a global script bundle (e.g., to inject helpers used by inline scripts), create it in `Configure`:

```csharp
public void Configure(IApplicationBuilder app, BundleFactory bundles)
{
    bundles.CreateBundle("site-scripts", BundleType.Scripts);
    bundles["site-scripts"].AppendContent("/* shared inline JS */", false);
    // ...
}
```

Then reference it from the layout:

```html
<x-bundle name="site-scripts" />
```

## 3. `_ViewImports.cshtml`

Add to `Views/_ViewImports.cshtml` (and to any `Areas/<Area>/Views/_ViewImports.cshtml` if the project uses areas):

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, Gehtsoft.TagHelpers
```

The Microsoft line is usually already present from the project template; the Gehtsoft line is what you're adding. Without it, Razor sees `<x-form>` as plain HTML and silently emits it as-is.

If you want to use the C# types directly in views (rare), also add:

```cshtml
@using Gehtsoft.TagHelpers
```

## 4. `_Layout.cshtml` head

Add inside `<head>`:

```cshtml
<x-lib-includes theme="bootstrap" minimize="true" />
```

For development, set `minimize="false"` to keep emitted JS/CSS readable in DevTools.

A typical minimal layout looks like:

```cshtml
@using Microsoft.Extensions.Configuration
@inject IConfiguration Config

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>@ViewBag.Title</title>

    <x-lib-includes theme="@(Config["UI:Theme"] ?? "bootstrap")" minimize="true" />

    @RenderSection("Header", required: false)
</head>
<body class="k-content">
    @RenderBody()
</body>
</html>
```

`<x-lib-includes>` emits, in order:

1. A `<link>` to a common-styles bundle (Bootstrap CSS where applicable).
2. A `<link>` to a theme-styles bundle (Kendo theme CSS + the library's own `taghelper.css`).
3. A `<script>` to a scripts bundle (jQuery, Kendo, the library's helper scripts).

Repeating this tag per page is wasteful — keep it in the shared layout only.

## 5. Kendo UI assets in `wwwroot/lib/Kendo.UI/`

The driver's library provider expects this exact tree:

```
wwwroot/
└── lib/
    └── Kendo.UI/
        ├── css/
        │   ├── bootstrap.min.css                    (only required if theme=bootstrap)
        │   └── kendo/
        │       ├── kendo.common.min.css             (default themes use this)
        │       ├── kendo.common-bootstrap.min.css   (theme=bootstrap)
        │       ├── kendo.common-fiori.min.css       (theme=fiori)
        │       ├── kendo.common-material.min.css    (theme=material)
        │       ├── kendo.common-nova.min.css        (theme=nova)
        │       ├── kendo.<theme>.min.css            (every theme)
        │       └── kendo.<theme>.mobile.min.css     (every theme except *-v2 variants)
        └── script/
            ├── jquery.min.js
            ├── jquery.cookie.min.js
            ├── jszip.min.js
            └── kendo.all.min.js
```

You don't need every file for every theme — only the files matching the theme you serve. For `theme="bootstrap"` that's:

- `css/bootstrap.min.css`
- `css/kendo/kendo.common-bootstrap.min.css`
- `css/kendo/kendo.bootstrap.min.css`
- `css/kendo/kendo.bootstrap.mobile.min.css`
- All four scripts.

For `theme="default-v2"` (and any `*-v2` theme) the `common` and `mobile` files are skipped — only `kendo.<theme>.min.css` plus the scripts.

### 5a. Bower (preferred)

The library originated in an era when Bower was standard for client assets, and the library docs and examples assume it. If the project doesn't already have Bower set up:

1. Install Bower if needed: `npm install -g bower`.

2. At the project root, create `.bowerrc`:

   ```json
   {
     "directory": "wwwroot/lib"
   }
   ```

3. At the project root, create `bower.json`. The Kendo distribution typically comes from a private feed:

   ```json
   {
     "name": "myapp",
     "private": true,
     "dependencies": {
       "Kendo.UI": "https://your-internal-feed/kendo.ui.2022.3.1109.zip"
     }
   }
   ```

   Replace the URL with whatever Kendo distribution your organization uses. The version must be a Kendo UI 2022.x build to match the `Kendo2022` driver.

4. Run:

   ```bash
   bower install
   ```

   This unpacks Kendo into `wwwroot/lib/Kendo.UI/` — matching the layout above.

5. Add `wwwroot/lib/` to `.gitignore` if you don't want the large vendor tree in source control.

### 5b. Manual placement

If Bower isn't available (or the user prefers not to install it):

1. Obtain a Kendo UI 2022.x distribution (Telerik download portal, internal artifact server, or a vendored copy from another project).

2. Copy the files into `wwwroot/lib/Kendo.UI/` matching the tree shown above. Only the files for your chosen theme are strictly required.

3. Verify minified filenames exactly: the driver looks for `.min.css` and `.min.js` filenames. If a vendor archive ships only un-minified versions, either rename them or download the minified set.

### Why not `<script src="...kendo.all.min.js">` directly?

You *could* — and the `index.ds` documentation in the library shows that legacy approach — but it duplicates what `<x-lib-includes>` already does, and it skips the library's own embedded CSS/JS (`taghelper.css`, `formAjax.js`, etc.) that get stitched into the same bundle. Use `<x-lib-includes>` unless you have a specific need to hand-roll.

## 6. Themes

Allowed values for the `theme` attribute on `<x-lib-includes>`:

| Value | Notes |
|-------|-------|
| `default` | Default Kendo theme |
| `default-v2` | Default v2 — no separate `common` or `mobile` CSS file |
| `bootstrap` | Pulls in `bootstrap.min.css` and Kendo's bootstrap-compat common styles |
| `bootstrap-v4` | No separate `common` or `mobile` CSS file |
| `fiori` | SAP Fiori theme |
| `material` | Material Design |
| `material-v2` | No separate `common` or `mobile` CSS file |
| `nova` | Nova theme |
| `office365` | Note: maps to `kendo.officer365` for the common-styles file in this version |

Theme values are matched case-insensitively. The selected theme drives both the CSS chosen and a slight CSS variant emitted by the library (`taghelper-bootstrap.css` for `bootstrap`, `taghelper-other.css` for everything else).

To make the theme configurable, drive it from `appsettings.json`:

```json
{
  "UI": { "Theme": "bootstrap" }
}
```

```cshtml
@inject IConfiguration Config
<x-lib-includes theme="@(Config["UI:Theme"] ?? "default")" minimize="true" />
```

## 7. Verification — minimal smoke test

Drop this into any view and load the page:

```cshtml
@{
    var sample = new { Name = "" };
}

<x-form for="@sample" mode="twocolumn">
    <x-edit for="@sample.Name" label="Name" />
    <x-button role="submit" label="OK" />
</x-form>
```

Expected behavior:

- Page loads with no JavaScript console errors.
- The textbox is styled per the chosen theme (Kendo's signature look).
- Submitting the form posts back without a 500.
- DevTools Network tab shows the bundles being served (`/lib/Kendo.UI/...`).

If any of these fail, jump to **`troubleshooting.md`**.

## 8. Default ASP.NET MVC template — what to change

When starting from `dotnet new mvc`, the diff from the default template is:

| File | Change |
|------|--------|
| `<project>.csproj` | Add `<PackageReference Include="Gehtsoft.TagHelpers" Version="*" />`. |
| `Program.cs` | After `var builder = WebApplication.CreateBuilder(args);` and before `builder.Build()`, add `builder.Services.AddTagHelperServices(builder.Environment);` and `builder.Services.AddKendo2022Driver();`. Add `using Gehtsoft.TagHelpers;` and `using Gehtsoft.TagHelpers.Driver.Kendo2022;`. |
| `Views/_ViewImports.cshtml` | Add `@addTagHelper *, Gehtsoft.TagHelpers`. |
| `Views/Shared/_Layout.cshtml` | Inside `<head>`, add `<x-lib-includes theme="bootstrap" minimize="true" />`. The default template's existing `<link>` to `bootstrap.min.css` and its `<script>` to `bootstrap.bundle.min.js` can be removed once `<x-lib-includes>` is in place — Kendo's bootstrap-compat CSS replaces the former, and Kendo doesn't depend on Bootstrap's JS. |
| `wwwroot/lib/` | Populate `Kendo.UI/` per [§5](#5-kendo-ui-assets-in-wwwrootlibkendoui). |
| `.bowerrc`, `bower.json` | Create if using Bower (see [§5a](#5a-bower-preferred)). |

The default template's site CSS (`wwwroot/css/site.css`) and JS (`wwwroot/js/site.js`) can stay or go — they don't conflict with the library.
