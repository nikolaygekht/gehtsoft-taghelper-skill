# Setup Reference — Gehtsoft.TagHelpers

Detailed installation and wiring of the library into an ASP.NET Core MVC project. The library supports two Kendo UI driver generations — **Kendo 2022** and **Kendo 2026** — and the choice affects DI registration, vendored asset layout, and theme names. Every step below calls out the difference where it matters.

## Contents

- [1. NuGet package](#1-nuget-package)
- [2. DI registration (Kendo 2022 vs Kendo 2026)](#2-di-registration-kendo-2022-vs-kendo-2026)
- [3. `_ViewImports.cshtml`](#3-_viewimportscshtml)
- [4. `_Layout.cshtml` head](#4-_layoutcshtml-head)
- [5. Kendo UI assets in `wwwroot/lib/Kendo.UI/`](#5-kendo-ui-assets-in-wwwrootlibkendoui)
   - [5.1 Kendo 2022.x layout](#51-kendo-2022x-layout)
   - [5.2 Kendo 2026.x layout](#52-kendo-2026x-layout)
   - [5.3 Vendored jQuery and Bootstrap versions — important](#53-vendored-jquery-and-bootstrap-versions--important)
   - [5a. Gehtsoft.Build.ContentDelivery — MSBuild target (preferred)](#5a-gehtsoftbuildcontentdelivery--msbuild-target-preferred)
   - [5b. Bower (legacy)](#5b-bower-legacy)
   - [5c. Manual placement](#5c-manual-placement)
- [6. Themes](#6-themes)
   - [6.1 Kendo 2022 themes](#61-kendo-2022-themes)
   - [6.2 Kendo 2026 themes](#62-kendo-2026-themes)
- [7. Verification — minimal smoke test](#7-verification--minimal-smoke-test)
- [8. Default ASP.NET MVC template — what to change](#8-default-aspnet-mvc-template--what-to-change)
- [9. Picking 2022 vs 2026](#9-picking-2022-vs-2026)

---

## 1. NuGet package

The current package is **`Gehtsoft.TagHelpers` 0.7.1**, targeting **`net10.0`**. The consuming project's `.csproj` must use a compatible target framework — typically:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
</PropertyGroup>
```

If the project still targets `net8.0` or `net9.0`, retarget it before adding the package — the package will not restore against an older TFM.

Add to the project's `.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="Gehtsoft.TagHelpers" Version="0.7.1" />
</ItemGroup>
```

Or via CLI:

```bash
dotnet add package Gehtsoft.TagHelpers --version 0.7.1
```

Optional companion packages:

| Package | Purpose | Install when |
|---------|---------|--------------|
| `Gehtsoft.TagHelpers.Utils` | Adapters for `Gehtsoft.Validator` / `Gehtsoft.Mapper` | The project already uses those validators |
| `Gehtsoft.Build.ContentDelivery` | MSBuild target that downloads & unpacks the Kendo UI archive into `wwwroot/lib/Kendo.UI/` (see [§5a](#5a-gehtsoftbuildcontentdelivery--msbuild-target-preferred)) | You want a one-command install of Kendo from a private feed/HTTP URL |

## 2. DI registration (Kendo 2022 vs Kendo 2026)

Every project must call **exactly one** `AddKendoXXXXDriver()`. The driver determines which `ILibraryProvider` is registered, which writer classes the tag helpers discover, and which CSS filename patterns are emitted by `<x-lib-includes>`. **Do not call both; do not omit the driver.**

### 2.a Kendo 2022 driver

```csharp
using Gehtsoft.TagHelpers;
using Gehtsoft.TagHelpers.Driver.Kendo2022;

// in Program.cs (minimal hosting):
builder.Services.AddTagHelperServices(builder.Environment);
builder.Services.AddKendo2022Driver();

// or in Startup.ConfigureServices:
services.AddTagHelperServices(CurrentEnvironment);
services.AddKendo2022Driver();
```

### 2.b Kendo 2026 driver

```csharp
using Gehtsoft.TagHelpers;
using Gehtsoft.TagHelpers.Driver.Kendo2026;

// in Program.cs (minimal hosting):
builder.Services.AddTagHelperServices(builder.Environment);
builder.Services.AddKendo2026Driver();

// or in Startup.ConfigureServices:
services.AddTagHelperServices(CurrentEnvironment);
services.AddKendo2026Driver();
```

### Common notes

- Both `AddTagHelperServices(env)` + the driver are required. Order does not matter relative to each other, but they should be added before `builder.Build()`.
- `CurrentEnvironment` in the `Startup.cs` form is the `IWebHostEnvironment` captured in the `Startup` constructor.

What each call does:

- **`AddTagHelperServices(env)`** registers: `IWebHostEnvironment` (singleton), `IActionContextAccessor`, `IUrlHelper`, `XClassDictionary`, `ITagHelperResourceProvider`, `IBundleFileCache`, and `BundleFactory`. Safe to call after MVC is added.
- **`AddKendo2022Driver()` / `AddKendo2026Driver()`** registers: `ILibraryProvider` and `ITagLibraryDriver` as singletons (via `TryAddSingleton`). Without this, `<x-lib-includes>` falls back to a legacy code path and individual control tags will not render their JS initialization.
- The two drivers use the **same `<x-*>` tag surface** — switching from 2022 to 2026 does not require rewriting views, only re-vendoring assets (§5) and possibly adjusting `theme=` on `<x-lib-includes>` (§6).

### Optional — site-wide script bundle

If the application needs a global script bundle (e.g., to inject helpers used by inline scripts), create it in `Configure`. This is **independent of the driver** — the same code works for 2022 and 2026:

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

For richer patterns (file-backed bundles, per-page scripts, site.css wiring), see **`assets.md`**.

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

A typical minimal layout (works for both 2022 and 2026):

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
    <x-bundle name="site-scripts" />                         <!-- optional -->
    <link rel="stylesheet" href="~/css/site.css?v=1" />      <!-- optional app CSS -->
    <script src="~/js/site.js?v=1"></script>                 <!-- optional app JS -->

    @RenderSection("Header", required: false)
</head>
<body class="@(Config["UI:BodyClass"] ?? "k-body")">
    @RenderBody()
</body>
</html>
```

Body-class convention used by the library's own test apps:

- **Kendo 2022 test app:** `<body class="k-content">`
- **Kendo 2026 test app:** `<body class="k-body">`

Either works functionally; match the convention of the Kendo generation you're on, or factor it through configuration as shown above. See **`assets.md`** for more on mixing tag-helper output with project-owned CSS/JS.

`<x-lib-includes>` emits, in order:

1. A `<link>` to a common-styles bundle (Bootstrap CSS where applicable).
2. A `<link>` to a theme-styles bundle (Kendo theme CSS + the library's own `taghelper.css` — on 2022 also `kendo.common-*.min.css` and `kendo.<theme>.mobile.min.css` when applicable).
3. A `<script>` to a scripts bundle (jQuery, Kendo, the library's helper scripts).

Repeating this tag per page is wasteful — keep it in the shared layout only.

## 5. Kendo UI assets in `wwwroot/lib/Kendo.UI/`

The driver's library provider expects specific filenames under `wwwroot/lib/Kendo.UI/`. The layout differs between the two Kendo generations — pick the section matching your driver.

### 5.1 Kendo 2022.x layout

```
wwwroot/
└── lib/
    └── Kendo.UI/
        ├── css/
        │   ├── bootstrap.min.css                    (always emitted as common-styles <link>)
        │   └── kendo/
        │       ├── kendo.common.min.css             (default themes use this)
        │       ├── kendo.common-bootstrap.min.css   (theme=bootstrap)
        │       ├── kendo.common-fiori.min.css       (theme=fiori)
        │       ├── kendo.common-material.min.css    (theme=material)
        │       ├── kendo.common-nova.min.css        (theme=nova)
        │       ├── kendo.officer365.min.css         (theme=office365 — note: "officer365", not "office365")
        │       ├── kendo.<theme>.min.css            (every theme — pre-minimized)
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

For `theme="default-v2"`, `theme="bootstrap-v4"`, or `theme="material-v2"` the `common` and `mobile` files are skipped — only `kendo.<theme>.min.css` plus the scripts.

### 5.2 Kendo 2026.x layout

Kendo 2026 ships **self-contained theme files** (no separate `common` or `mobile` bundle) and the `/css/kendo/` folder holds un-minified `.css` files named `{family}-{variant}.css`:

```
wwwroot/
└── lib/
    └── Kendo.UI/
        ├── css/
        │   ├── bootstrap.min.css                    (always emitted as common-styles <link>)
        │   └── kendo/
        │       ├── default-main.css                 (theme=default → this)
        │       ├── default-main-dark.css            (variant)
        │       ├── default-blue.css / default-green.css / default-purple.css / ...
        │       ├── default-ocean-blue.css / default-ocean-blue-a11y.css
        │       ├── bootstrap-main.css               (theme=bootstrap → this)
        │       ├── bootstrap-main-dark.css
        │       ├── bootstrap-3.css / bootstrap-3-dark.css
        │       ├── bootstrap-4.css / bootstrap-4-dark.css (theme=bootstrap-v4 → bootstrap-4)
        │       ├── bootstrap-nordic.css / bootstrap-urban.css / bootstrap-vintage.css
        │       ├── bootstrap-turquoise.css / bootstrap-turquoise-dark.css
        │       ├── material-main.css                (theme=material → this; also maps material-v2, fiori)
        │       ├── material-main-dark.css
        │       ├── classic-main.css                 (theme=classic → this; also maps nova)
        │       ├── classic-main-dark.css
        │       ├── classic-green.css / classic-opal.css / classic-lavender.css / classic-metro.css / classic-moonlight.css / classic-silver.css / classic-uniform.css (and *-dark variants)
        │       ├── fluent-main.css                  (theme=fluent → this)
        │       ├── fluent-main-dark.css / fluent-1.css / fluent-1-dark.css
        │       ├── kendo-theme-utils.css
        │       └── font-icons/                      (subfolder with icon fonts)
        └── script/
            ├── jquery.min.js
            ├── jquery.cookie.min.js
            ├── jquery.ui.min.js                     (shipped, not required by library)
            ├── jszip.min.js
            ├── kendo.all.min.js
            ├── bootstrap.bundle.min.js              (shipped, optional)
            └── pako_deflate.min.js
```

Only four scripts are actually required at runtime: `jquery.min.js`, `jquery.cookie.min.js`, `jszip.min.js`, `kendo.all.min.js`. The rest are part of the 2026 distribution but not referenced by `<x-lib-includes>`.

### 5.3 Vendored jQuery and Bootstrap versions — important

The two Kendo generations ship **different** jQuery and Bootstrap versions. These are pulled in from the same `wwwroot/lib/Kendo.UI/` tree, so switching the Kendo driver/assets also switches jQuery and Bootstrap under your app:

| Vendor file | Kendo 2022 ships | Kendo 2026 ships |
|-------------|------------------|------------------|
| `script/jquery.min.js` | jQuery **1.12.4** | jQuery **4.0.0** |
| `css/bootstrap.min.css` | Bootstrap **3.3.7** | Bootstrap **5.3.8** |
| `script/bootstrap.bundle.min.js` | *not shipped* | Bootstrap 5 JS bundle (includes Popper) |
| `script/jquery.ui.min.js` | *not shipped* | jQuery UI (shipped, not auto-loaded) |
| `script/angular.min.js`, `script/kendo.angular.min.js` | shipped (legacy) | *not shipped* |

What this means in practice:

- **App code that uses jQuery 1.x-only APIs breaks on 2026.** Examples: `.live()`, `.size()`, `.andSelf()`, event shorthands like `.load(fn)` / `.error(fn)`, `$.browser`. See `troubleshooting.md` for the migration checklist.
- **App markup that uses Bootstrap 3 / 4 classes breaks on 2026.** `.panel`, `.well`, `.thumbnail`, `.btn-default`, `.glyphicon`, `.hidden-xs`, `.img-responsive`, `data-toggle="…"` all change or go away in Bootstrap 5. See `troubleshooting.md` for the mapping.
- **`<x-lib-includes>` still loads only the four required scripts.** `bootstrap.bundle.min.js` and `jquery.ui.min.js` exist in the 2026 tree but are **not auto-loaded** — include them manually with a `<script>` tag or `<x-bundle>` if your app needs them.
- **The `bootstrap.min.css` served as the "common-styles" bundle is Bootstrap 3 on 2022 and Bootstrap 5 on 2026.** Your app's own CSS rules that style `.btn`, `.nav`, `.container`, etc. must be compatible with the version underneath.

For `theme="bootstrap"` that's:

- `css/bootstrap.min.css`
- `css/kendo/bootstrap-main.css`
- The four required scripts.

You can also pass any 2026 **variant filename** as `theme=` directly — e.g. `theme="default-main-dark"`, `theme="bootstrap-3"`, `theme="classic-opal"` — and the driver uses it verbatim as the CSS file name. Legacy 2022 theme names are remapped (see §6.2).

### 5a. Gehtsoft.Build.ContentDelivery — MSBuild target (preferred)

This is the method used by the library's own `TestApp` and is the recommended approach for new projects. An MSBuild package adds a `GetContent` task that downloads a zip archive and unpacks it into the configured directory. It runs only when its target is invoked explicitly (not on every build), so it doesn't slow down day-to-day compilation.

1. Add the package and the two MSBuild targets to the project's `.csproj`:

   ```xml
   <ItemGroup>
     <PackageReference Include="Gehtsoft.Build.ContentDelivery"
                       Version="0.1.11"
                       IncludeAssets="build" />
   </ItemGroup>

   <Target Name="CleanContent">
     <ItemGroup>
       <FilesToDelete Include="wwwroot/lib/Kendo.UI/**/*" />
     </ItemGroup>
     <Delete Files="@(FilesToDelete)" />
     <RemoveDir Directories="wwwroot/lib/Kendo.UI" />
   </Target>

   <Target Name="Content">
     <!-- Kendo 2022 (pairs with AddKendo2022Driver): -->
     <GetContent Source="https://www.myget.org/F/gehtsoft-public/bower/packages/Kendo.UI/2022.3.1109.zip"
                 Destination="$(MSBuildProjectDirectory)/wwwroot/lib/Kendo.UI"
                 Unzip="true" />
     <!-- OR Kendo 2026 (pairs with AddKendo2026Driver):
     <GetContent Source="https://www.myget.org/F/gehtsoft-public/bower/packages/Kendo.UI/2026.1.325.zip"
                 Destination="$(MSBuildProjectDirectory)/wwwroot/lib/Kendo.UI"
                 Unzip="true" />
     -->
   </Target>
   ```

   `IncludeAssets="build"` keeps the package as a build-time dependency only — it does not flow to consumers of the project. Replace the `Source` URL with the exact Kendo UI distribution URL your team uses. The **Kendo version in the URL must match the driver** chosen in §2 — a 2022 archive unpacked into a project that calls `AddKendo2026Driver()` will 404 at page load because the expected CSS filenames are different (§5.1 vs §5.2).

2. Trigger the targets the first time, and any time the source archive changes:

   ```bash
   dotnet build YourApp.csproj -t:CleanContent,Content
   ```

   `CleanContent` wipes `wwwroot/lib/Kendo.UI/`; `Content` downloads the zip from `Source` and unpacks it into `Destination`. Together they refresh the local Kendo tree from scratch.

   A small `updateKendo.bat` (or shell script) at the project root makes this routine reproducible:

   ```bat
   dotnet build YourApp.csproj -t:CleanContent,Content
   ```

3. Add `wwwroot/lib/` to `.gitignore` so the vendored Kendo tree stays out of source control — the build command above is the canonical way to recreate it on a fresh checkout.

4. Verify the unpacked layout matches §5 above (the archive your team publishes should already be packed in that exact `script/` + `css/kendo/` shape).

### 5b. Bower (legacy)

The library originated in an era when Bower was standard for client assets, and many projects in the ecosystem still vendor Kendo this way. **Bower works for both Kendo 2022 and Kendo 2026** — the Gehtsoft public Bower feed publishes a Kendo.UI package for each generation:

- Kendo 2022 example: `https://www.myget.org/F/gehtsoft-public/bower/packages/Kendo.UI/2022.3.1109.zip`
- Kendo 2026 example: `https://www.myget.org/F/gehtsoft-public/bower/packages/Kendo.UI/2026.1.325.zip`

(Exact version numbers change; use the URL your team publishes.) New projects should still prefer [§5a](#5a-gehtsoftbuildcontentdelivery--msbuild-target-preferred) because it's a single MSBuild target rather than an extra global tool — but Bower is fully supported when preferred.

1. Install Bower if needed: `npm install -g bower`.

2. At the project root, create `.bowerrc`:

   ```json
   {
     "directory": "wwwroot/lib"
   }
   ```

3. At the project root, create `bower.json`. The Kendo distribution comes from the Gehtsoft public Bower feed (or any equivalent mirror):

   For Kendo 2022 (paired with `AddKendo2022Driver()`):

   ```json
   {
     "name": "myapp",
     "private": true,
     "dependencies": {
       "Kendo.UI": "https://www.myget.org/F/gehtsoft-public/bower/packages/Kendo.UI/2022.3.1109.zip"
     }
   }
   ```

   For Kendo 2026 (paired with `AddKendo2026Driver()`):

   ```json
   {
     "name": "myapp",
     "private": true,
     "dependencies": {
       "Kendo.UI": "https://www.myget.org/F/gehtsoft-public/bower/packages/Kendo.UI/2026.1.325.zip"
     }
   }
   ```

   Adjust the exact version number to whichever build your team has standardized on. The **driver and the Kendo version must match** — a 2022 bower package with `AddKendo2026Driver()` (or vice versa) will 404 for theme CSS files at page load.

4. Run:

   ```bash
   bower install
   ```

   This unpacks Kendo into `wwwroot/lib/Kendo.UI/` — matching the layout above.

5. Add `wwwroot/lib/` to `.gitignore` if you don't want the large vendor tree in source control.

### 5c. Manual placement

If neither MSBuild content delivery nor Bower is available (or the user prefers not to install either):

1. Obtain a Kendo UI distribution matching your driver — **2022.x** for `AddKendo2022Driver()`, **2026.x** for `AddKendo2026Driver()`. Sources: Telerik download portal, internal artifact server, or a vendored copy from another project.

2. Copy the files into `wwwroot/lib/Kendo.UI/` matching the tree shown in §5.1 (2022) or §5.2 (2026). Only the files for your chosen theme are strictly required.

3. **Filename case matters:**
   - **2022:** the driver looks for `.min.css` and `.min.js` exactly. If a vendor archive ships only un-minified CSS, either rename to `.min.css` or obtain the minified set. Theme file name is always lowercase (e.g. `kendo.bootstrap.min.css`).
   - **2026:** theme files are **un-minified `.css`** (no `.min.` infix). The script files are still minified (`jquery.min.js`, `kendo.all.min.js`, etc.).

### Why not `<script src="...kendo.all.min.js">` directly?

You *could* — and legacy projects sometimes do — but it duplicates what `<x-lib-includes>` already does and skips the library's own embedded CSS/JS (`taghelper.css`, `formAjax.js`, etc.) that get stitched into the same bundle. Use `<x-lib-includes>` unless you have a specific need to hand-roll. (If you must, also include the embedded bundle manually; see `assets.md` § "When to bypass `<x-lib-includes>`".)

## 6. Themes

Theme values on `<x-lib-includes theme="…">` are **case-insensitive**. The selected theme drives both which Kendo CSS is linked and which of the library's own stylesheets is inlined (`taghelper-bootstrap.css` for bootstrap-family themes, `taghelper-other.css` for everything else).

### 6.1 Kendo 2022 themes

Supported values:

| Value | Notes |
|-------|-------|
| `default` | Default Kendo theme |
| `default-v2` | Default v2 — no separate `common` or `mobile` CSS file |
| `bootstrap` | Pulls in `bootstrap.min.css` and Kendo's `kendo.common-bootstrap` common styles |
| `bootstrap-v4` | No separate `common` or `mobile` CSS file |
| `fiori` | SAP Fiori theme — uses `kendo.common-fiori` |
| `material` | Material Design — uses `kendo.common-material` |
| `material-v2` | No separate `common` or `mobile` CSS file |
| `nova` | Nova theme — uses `kendo.common-nova` |
| `office365` | Note: the "common" file must actually be present as `kendo.officer365.min.css` (legacy typo preserved) |

### 6.2 Kendo 2026 themes

The 2026 driver supports a small set of native theme families plus any variant filename passed through verbatim. Legacy 2022 names are remapped for easy migration.

Native theme families (pass as `theme=`):

| Value | File name used |
|-------|----------------|
| `default` | `default-main.css` |
| `bootstrap` | `bootstrap-main.css` |
| `material` | `material-main.css` |
| `classic` | `classic-main.css` |
| `fluent` | `fluent-main.css` |

Legacy 2022 names — remapped:

| Legacy (2022) value | Mapped 2026 file |
|---------------------|------------------|
| `default-v2` | `default-main.css` |
| `office365` | `default-main.css` |
| `bootstrap-v4` | `bootstrap-4.css` |
| `material-v2` | `material-main.css` |
| `fiori` | `material-main.css` |
| `nova` | `classic-main.css` |

Native-variant passthrough: any other value is used verbatim as the CSS file name (lowercased). Examples:

| Value | File name used |
|-------|----------------|
| `default-main-dark` | `default-main-dark.css` |
| `bootstrap-3` / `bootstrap-3-dark` | `bootstrap-3.css` / `bootstrap-3-dark.css` |
| `bootstrap-nordic` / `bootstrap-urban` / `bootstrap-vintage` | `bootstrap-*.css` |
| `bootstrap-turquoise` / `bootstrap-turquoise-dark` | `bootstrap-turquoise*.css` |
| `classic-opal` / `classic-lavender` / `classic-metro` / `classic-moonlight` / `classic-silver` / `classic-uniform` | `classic-*.css` |
| `fluent-1` / `fluent-1-dark` | `fluent-*.css` |
| `default-blue` / `default-green` / `default-purple` / `default-turquoise` / `default-urban` / `default-ocean-blue` / `default-ocean-blue-a11y` | `default-*.css` |

Empty or unrecognized → `default-main.css`.

### Making the theme configurable

Drive it from `appsettings.json`:

```json
{
  "UI": { "Theme": "bootstrap" }
}
```

```cshtml
@inject IConfiguration Config
<x-lib-includes theme="@(Config["UI:Theme"] ?? "default")" minimize="true" />
```

This pattern lets a single codebase serve either Kendo 2022 or 2026 — change the driver registration (§2) and the theme value, and the same Razor views keep working.

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

When starting from `dotnet new mvc -f net10.0`, the diff from the default template is:

| File | Change |
|------|--------|
| `<project>.csproj` | Ensure `<TargetFramework>net10.0</TargetFramework>`. Add `<PackageReference Include="Gehtsoft.TagHelpers" Version="0.7.1" />`. If using the MSBuild Kendo install path, also add `Gehtsoft.Build.ContentDelivery` plus the `CleanContent` / `Content` targets — see [§5a](#5a-gehtsoftbuildcontentdelivery--msbuild-target-preferred). |
| `Program.cs` | After `var builder = WebApplication.CreateBuilder(args);` and before `builder.Build()`, add `builder.Services.AddTagHelperServices(builder.Environment);` and **either** `builder.Services.AddKendo2022Driver();` (with `using Gehtsoft.TagHelpers.Driver.Kendo2022;`) **or** `builder.Services.AddKendo2026Driver();` (with `using Gehtsoft.TagHelpers.Driver.Kendo2026;`). Also `using Gehtsoft.TagHelpers;`. (Equivalent in `Startup.ConfigureServices`: same two calls, with `CurrentEnvironment` instead of `builder.Environment` — see [§2](#2-di-registration-kendo-2022-vs-kendo-2026).) |
| `Views/_ViewImports.cshtml` | Add `@addTagHelper *, Gehtsoft.TagHelpers`. |
| `Views/Shared/_Layout.cshtml` | Inside `<head>`, add `<x-lib-includes theme="bootstrap" minimize="true" />`. The default template's existing `<link>` to `bootstrap.min.css` and its `<script>` to `bootstrap.bundle.min.js` can be removed once `<x-lib-includes>` is in place — Kendo's bootstrap-compat CSS replaces the former, and Kendo doesn't depend on Bootstrap's JS. |
| `wwwroot/lib/` | Populate `Kendo.UI/` per [§5](#5-kendo-ui-assets-in-wwwrootlibkendoui) matching your driver (2022 → §5.1; 2026 → §5.2). Preferred path is the MSBuild target ([§5a](#5a-gehtsoftbuildcontentdelivery--msbuild-target-preferred)); Bower ([§5b](#5b-bower-legacy-2022-only), 2022 only) and manual placement ([§5c](#5c-manual-placement)) remain options. |
| `wwwroot/css/site.css`, `wwwroot/js/site.js` | Keep if you want them. The template's default references remain valid — just confirm they appear *after* `<x-lib-includes>` in the `<head>` so Kendo styles/scripts load first. See **`assets.md`** for the recommended ordering and cache-busting options. |
| `.gitignore` | Add `wwwroot/lib/` so the vendored Kendo tree isn't checked in. |

## 9. Picking 2022 vs 2026

Use this to resolve "should this project be on 2022 or 2026?" when it isn't already decided:

| Situation | Pick |
|-----------|------|
| Existing project with populated `wwwroot/lib/Kendo.UI/css/kendo/` full of `kendo.<theme>.min.css` files | **2022** — match what's vendored |
| Existing project whose `wwwroot/lib/Kendo.UI/css/kendo/` contains `<family>-<variant>.css` files (no `.min.` infix) | **2026** — match what's vendored |
| Greenfield project, no constraint to match existing customer CSS overrides | **2026** — it's the current Kendo generation, themes are self-contained, file layout is simpler |
| Migrating a project from 2022 to 2026 | Change `AddKendo2022Driver()` → `AddKendo2026Driver()`, re-vendor Kendo UI 2026.x assets (§5.2), and update the `theme=` value if it was a 2022-only name. Views don't need to change. |

The two drivers share the same `<x-*>` tag surface and the same HTML model-binding/validation behavior, so switching mostly touches wiring and CSS — not authoring.
