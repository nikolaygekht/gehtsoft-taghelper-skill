---
name: gehtsoft-taghelpers
description: Use this skill whenever you write or modify ASP.NET Core code that involves the Gehtsoft.TagHelpers Razor library — installing it into a project, registering its services in Program.cs/Startup.cs, placing Kendo UI assets under wwwroot/lib/Kendo.UI/, or authoring Razor views that use any x-* tag (x-form, x-control, x-edit, x-numeric-edit, x-date-edit, x-select, x-check, x-grid, x-popup, x-menu, x-bundle, x-lib-includes, x-script, x-validate, x-form-errors, x-tabstrip, x-splitter, x-button, x-tree-control, x-upload, etc.). Trigger this skill whenever a Razor file contains any x-* tag, when @addTagHelper *, Gehtsoft.TagHelpers appears in _ViewImports.cshtml, when a .csproj references the Gehtsoft.TagHelpers package, or when the user asks how to add forms, grids, popups, menus, or Kendo widgets to an ASP.NET Core MVC view using Gehtsoft tag helpers — even if they don't explicitly say "Gehtsoft" or "TagHelpers". Gehtsoft.TagHelpers is a custom library; do not assume Kendo UI MVC helpers, Telerik UI for ASP.NET Core, or vanilla Razor TagHelpers — the syntax and DI wiring are specific to this library.
---

# Gehtsoft.TagHelpers Skill

A Razor Tag Helper library that wraps Kendo UI 2022 widgets and Bootstrap layout into custom `<x-*>` tags. It targets server-rendered ASP.NET Core MVC apps and emphasizes a declarative, composable form/grid model with built-in client+server validation and AJAX form posting.

This skill helps you do three things:

1. **Install** the library cleanly into a fresh or existing ASP.NET Core MVC project.
2. **Author** Razor markup using the library's `<x-*>` tag families.
3. **Diagnose** common errors (missing assets, model-binding edge cases, AJAX pitfalls).

## Critical context (read once, internalize)

- **Target framework / version.** The current package (`Gehtsoft.TagHelpers` 0.7.1) targets `net10.0`. The consuming project must target `net10.0` (or a compatible newer TFM); older `net8.0`/`net9.0` projects must be retargeted before adding the package.
- **Two registrations are required** in `ConfigureServices`: `services.AddTagHelperServices(env)` registers the core; `services.AddKendo2022Driver()` registers the *driver* that actually emits HTML. **Without the driver, no `<x-*>` tag renders correctly** — the legacy fallback path is incomplete and should not be relied on.
- **Kendo UI is not bundled.** The library expects Kendo UI assets at `wwwroot/lib/Kendo.UI/{script,css}/...`. You install Kendo separately. The current canonical method is the `Gehtsoft.Build.ContentDelivery` MSBuild package (used by the library's own `TestApp`); Bower and manual placement remain supported. The `<x-lib-includes>` tag emits `<script>`/`<link>` tags pointing into this directory.
- **`<x-lib-includes>` and `_ViewImports.cshtml` are both mandatory.** Without `@addTagHelper *, Gehtsoft.TagHelpers` the parser silently treats `<x-form>` as raw HTML; without `<x-lib-includes>` (or equivalent bundles) the page loads but Kendo widgets never initialize.
- **Examples below use generic models** (`Customer`, `Product`, `Invoice`). Substitute the user's real model — never copy these names verbatim into the user's code.

## Setup at a glance

A complete install touches five places. Detailed steps and rationale: **`references/setup.md`**.

```
┌──────────────────────────────┬────────────────────────────────────────────────┐
│ Project                      │ <TargetFramework>net10.0</TargetFramework>     │
│   .csproj                    │ <PackageReference Include="Gehtsoft.TagHelpers"│
│                              │     Version="0.7.1" />                         │
├──────────────────────────────┼────────────────────────────────────────────────┤
│ Program.cs / Startup.cs      │ services.AddTagHelperServices(env);            │
│   (in ConfigureServices)     │ services.AddKendo2022Driver();                 │
├──────────────────────────────┼────────────────────────────────────────────────┤
│ Views/_ViewImports.cshtml    │ @addTagHelper *, Gehtsoft.TagHelpers           │
├──────────────────────────────┼────────────────────────────────────────────────┤
│ Views/Shared/_Layout.cshtml  │ <x-lib-includes theme="bootstrap"              │
│   (inside <head>)            │                  minimize="true" />            │
├──────────────────────────────┼────────────────────────────────────────────────┤
│ wwwroot/lib/Kendo.UI/        │ Kendo UI scripts + theme CSS                   │
│   (via Gehtsoft.Build.       │ (jquery.min.js, kendo.all.min.js, kendo.<theme>│
│   ContentDelivery MSBuild    │  .min.css, etc. — see references/setup.md)     │
│   target, Bower, or manual)  │                                                │
└──────────────────────────────┴────────────────────────────────────────────────┘
```

Supported themes for `<x-lib-includes theme="...">`: `default`, `default-v2`, `bootstrap`, `bootstrap-v4`, `fiori`, `material`, `material-v2`, `nova`, `office365`.

## Tag map — where to look

The library has ~50 tags. Group them by purpose and load the matching reference file when you need detail:

| You need to… | Load this reference | Key tags |
|--------------|---------------------|----------|
| Install or wire up the library | **`references/setup.md`** | (DI calls, `_ViewImports`, `_Layout`, MSBuild content delivery / Bower) |
| Build a form with input controls and validation | **`references/forms.md`** | `x-form`, `x-control`, `x-edit`, `x-numeric-edit`, `x-date-edit`, `x-select`, `x-check`, `x-radio`, `x-text-area`, `x-tree-control`, `x-upload`, `x-validate`, `x-form-errors`, `x-form-group` |
| Render a data grid (paged/sorted/filterable, optionally editable) | **`references/grids.md`** | `x-grid`, `x-grid-column`, `x-server-datasource`, `x-server-transport`, `x-server-schema`, `x-server-model` |
| Add navigation, popups, layout widgets, scripts, bundles | **`references/common.md`** | `x-bundle`, `x-lib-includes`, `x-script`, `x-template`, `x-popup`, `x-menu` family, `x-tabstrip`, `x-splitter`, `x-button`, `x-button-group`, `x-notification`, `x-link`, `x-row`/`x-col`, `x-area` |
| Diagnose an error or unexpected behavior | **`references/troubleshooting.md`** | (pitfalls per area) |

> **Don't load all references at once.** Load the one(s) the current task needs. Each reference file has a TOC at the top so you can jump.

## Minimal end-to-end example

A complete CRUD-ish page with a header bar, a form, and a grid — illustrating the library's three biggest tag families at once. Use this as a shape to fit the user's task into; the references give you the deeper attribute knowledge.

```cshtml
@model CustomerEditViewModel

@{
    ViewBag.Title = "Customers";
}

@section Header {
    <link rel="stylesheet" href="~/css/site.css" />
}

<x-menu menu-id="main" orientation="horizontal">
    <x-menuitem-action controller="Home" action="Index" name="Home" />
    <x-menuitem-action controller="Customers" action="Index" name="Customers" />
</x-menu>

<x-form id="customerForm"
        for="@Model.Customer"
        method="ajax"
        action="@Url.Action("Save")"
        mode="twocolumn">

    <x-form-errors form-errors="@Model.Errors" support-on-client-validation="true" />

    <x-hidden for="@Model.Customer.Id" />

    <x-edit for="@Model.Customer.Name" label="Name" />
    <x-edit for="@Model.Customer.Email" label="Email">
        <x-validate error="Enter a valid email" is-expression="true">
            /^[^\s@@]+@@[^\s@@]+\.[^\s@@]+$/.test(value)
        </x-validate>
    </x-edit>
    <x-date-edit for="@Model.Customer.JoinedOn" label="Joined" format="MM/dd/yyyy" />
    <x-select for="@Model.Customer.Status" label="Status"
              source-url="@Url.Action("StatusOptions")" />

    <x-button role="submit" label="Save" primary="true" />
</x-form>

<x-grid id="customerGrid" pageable="true" sortable="true" filterable="true" height="400px">
    <x-grid-column title="Name"  field="Name"  width="200px" />
    <x-grid-column title="Email" field="Email" width="240px" />
    <x-grid-column title="Joined" field="JoinedOn" format="{0:d}" width="120px" />

    <x-server-datasource page-size="20" server-paging="true" server-sorting="true">
        <x-server-transport read-url="@Url.Action("List")" />
        <x-server-schema>
            <x-server-model id="id">
                <x-server-field name="id"       field-type="number" />
                <x-server-field name="name"     field-type="text" />
                <x-server-field name="email"    field-type="text" />
                <x-server-field name="joinedOn" field-type="date" />
            </x-server-model>
        </x-server-schema>
    </x-server-datasource>
</x-grid>
```

Notes on this example:

- `for="@Model.Customer"` on `<x-form>` makes child `<x-control>`s bind to that submodel via shorter `for=` paths.
- `method="ajax"` causes the form to post as JSON without page navigation. For complex nested models prefer `method="post"` — see **`references/forms.md`** under "AJAX form gotchas".
- The `@@` in the validation expression is Razor-escaping `@`, which appears literally in the emitted JavaScript regex.
- `<x-server-transport>` URLs return Kendo-shaped JSON: `{ data: [...], total: N }`. See **`references/grids.md`** for the request/response contract and a controller-side parser.

## Behavioral guidance

- **Always use `for="@Model.X"`** for model binding rather than `name="X"`. The `for` form drives the validation provider, the AJAX serializer, and the label text. Bare `name=` is an escape hatch for non-model fields only.
- **Group controls with `<x-form-group>`** instead of raw `<div>`s. The form layout engine will not honor `<div>` boundaries; `<x-form-group>` participates in `twocolumn` / `horizontal` / `inline` flow correctly.
- **Per-control validation goes inside the `<x-control>`**; cross-field validation goes loose inside `<x-form>` and uses `reference('OtherField')` to reach values.
- **Never invent attributes**. If the user asks for behavior not documented here or in the references, say so — don't fabricate `<x-edit auto-trim="true">` style attributes that may not exist.
- **`<x-lib-includes>` belongs in the `<head>` of the layout**, not in individual views. Repeating it per view loads Kendo twice.
- **Themes are case-sensitive in the file path** (`kendo.bootstrap.min.css`, lowercase) but accepted in any case as the `theme` attribute value.

## Required vs. optional dependencies

Required by the library at runtime:
- `.NET 10` (`net10.0` TFM) — the package targets `net10.0` only
- `Microsoft.AspNetCore.App` (framework reference, present by default in MVC projects)
- jQuery (loaded via the Kendo bundle)
- Kendo UI 2022.x assets in `wwwroot/lib/Kendo.UI/`

Optional but commonly used:
- `Gehtsoft.TagHelpers.Utils` — bridge to a Validator/Mapper stack. Skip unless the user already uses those.
- `Newtonsoft.Json` — needed only for AJAX upload chunk metadata parsing. The MVC project typically has it already.

## When you finish a task

After producing markup or wiring code, briefly state which assumptions you made about the user's project (theme choice, model class names, controller actions invented, etc.) so the user can correct them. Don't invent plausible-but-fake things silently.
