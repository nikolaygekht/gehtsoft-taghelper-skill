# Common Tags Reference — Gehtsoft.TagHelpers

Layout, navigation, scripting, and miscellaneous tags. (Form controls live in `forms.md`; grids in `grids.md`.)

## Contents

- [`<x-lib-includes>`](#x-lib-includes)
- [`<x-bundle>`](#x-bundle)
- [`<x-script>`](#x-script)
- [`<x-template>`](#x-template)
- [`<x-button>`](#x-button)
- [`<x-button-group>`](#x-button-group)
- [`<x-popup>`](#x-popup)
- [`<x-menu>` family](#x-menu-family)
- [`<x-tabstrip>`](#x-tabstrip)
- [`<x-splitter>`](#x-splitter)
- [`<x-notification>`](#x-notification)
- [`<x-link>`](#x-link)
- [`<x-row>` / `<x-col>`](#x-row--x-col)
- [`<x-area>`](#x-area)
- [`<x-barcode>`](#x-barcode)
- [`<x-partial>`](#x-partial)
- [`<x-message>`](#x-message)
- [Script roles — concept](#script-roles--concept)

---

## `<x-lib-includes>`

Renders all `<link>` and `<script>` tags needed to load Kendo UI + the library's bundled CSS/JS. Place inside `<head>` of the shared layout. See **`setup.md`** for the theme list (per driver — 2022 and 2026 have different theme vocabularies) and the expected asset layout, and **`assets.md`** for how this tag composes with your app's own CSS/JS.

```cshtml
<x-lib-includes theme="bootstrap" minimize="true" />
```

| Attribute | Purpose |
|-----------|---------|
| `theme` | Kendo theme name (case-insensitive). On the 2022 driver: one of `default`, `default-v2`, `bootstrap`, `bootstrap-v4`, `fiori`, `material`, `material-v2`, `nova`, `office365`. On the 2026 driver: one of `default`, `bootstrap`, `material`, `classic`, `fluent` — or any native 2026 variant filename (`default-main-dark`, `bootstrap-3`, etc.) passed verbatim. |
| `minimize` | Minimize the inline library JS/CSS (Kendo's own files are always pre-minimized) |

---

## `<x-bundle>`

Emits a `<link>` or `<script>` for a bundle registered with `BundleFactory`. Useful for app-specific assets that should be served as a single optimized request.

```cshtml
<head>
    <x-lib-includes theme="bootstrap" />
    <x-bundle name="site-scripts" />
    <x-bundle name="site-styles" />
</head>
```

The bundle must be created in `Configure`. For a full walkthrough of `BundleFactory` — inline vs file-backed content, embedded resources, minimization, cache-hashed URLs, and how `<x-bundle>` interleaves with `<link>`/`<script>`/`<x-script>` — see **`assets.md`** §5.

| Attribute | Purpose |
|-----------|---------|
| `name` | Bundle name (must match the `CreateBundle` call) |
| `minimize` | Minimize the bundle output |

---

## `<x-script>`

Embedded JavaScript with a *role* that determines when and how it runs. The library accumulates scripts and emits them in the appropriate place — typically wrapped in `$(document).ready()`.

```cshtml
<x-script>
    console.log('Page ready');
</x-script>
```

The default role is "run on document ready". Other roles:

| Role | Purpose |
|------|---------|
| `script` (default) | Run on document ready |
| `global` | Emit at top level (define functions visible globally) |
| `template` | Treat the body as a template function (use with parameter-name) |
| `click`, `change`, `select`, `error`, `databound`, `edit`, `save`, etc. | Event handler attached to the parent control |
| `byname` + `script-role-name` | Custom named role (e.g., grid `toolbar`, column `filter-ui`) |

**Global function:**

```cshtml
<x-script role="global">
    function showInfo(msg) {
        kendoConsole.log(msg);
    }
</x-script>
```

**Event handler on a button:**

```cshtml
<x-button id="alertBtn" role="action" label="Alert me">
    <x-script role="click">
        alert('Clicked');
    </x-script>
</x-button>
```

**Named role (e.g., grid databound):**

```cshtml
<x-grid id="g1">
    <!-- ... -->
    <x-script script-role="byname" script-role-name="databound">
        console.log('rows:', this.dataSource.total());
    </x-script>
</x-grid>
```

Inside event/role scripts, `this` is the parent widget (kendoGrid, kendoButton, etc.) and `e` is the event arg (override the parameter name with `parameter-name="row"`).

---

## `<x-template>`

HTML template with `#= expression #` interpolation, parsed by Kendo's template engine.

```cshtml
<x-template>
    <div class="card">
        <strong>#= kendo.htmlEncode(e.name) #</strong><br />
        <span class="muted">#= kendo.htmlEncode(e.email) #</span>
    </div>
</x-template>
```

Used inside grid columns, dropdown items, autocomplete suggestions, etc. Always `htmlEncode` user data — Kendo templates do not auto-escape.

For complex logic, prefer `<x-script role="template">` (the body is JS, returns the HTML string).

---

## `<x-button>`

Generic button with several roles.

```cshtml
<x-button role="submit"  label="Save" primary="true" />
<x-button role="link"    label="Home" href="@Url.Action("Index")" target="_self" />
<x-button role="button"  label="Just a button" />
<x-button role="action"  label="Alert" sprite="k-i-notification">
    <x-script role="click">alert('Hi');</x-script>
</x-button>
```

| Attribute | Purpose |
|-----------|---------|
| `role` | `submit`, `button`, `link`, `action` |
| `label` | Button text |
| `href`, `target` | For `link` role |
| `primary` | Highlight as primary |
| `sprite` | Icon class (Kendo `k-i-*` or Bootstrap icon) |
| `disabled` | Disable |
| `tool-tip` | Hover tooltip |

`role="action"` is for buttons that fire a `click` event (define the handler with `<x-script role="click">`). `role="button"` is a no-op presentational button.

---

## `<x-button-group>`

Toggleable button bar.

```cshtml
<x-button-group id="viewMode" selected-button="0" multi-select="false">
    <x-button-group-item text="List"   icon-sprite="k-i-group" />
    <x-button-group-item text="Grid"   icon-sprite="k-i-table" />
    <x-button-group-item text="Kanban" icon-sprite="k-i-board" />
</x-button-group>
```

| Attribute | Purpose |
|-----------|---------|
| `selected-button` | Initially selected index |
| `multi-select` | Allow multiple buttons toggled simultaneously |
| `disabled` | Disable group |
| `var` | JS variable for the kendoButtonGroup |
| `items` | In-memory list of items (alternative to child tags) |

Item attributes: `text`, `badge`, `icon-sprite`, `icon-image`, `enabled`.

---

## `<x-popup>`

Modal dialog.

```cshtml
<x-popup id="confirmDialog" title="Confirm" width="400px">
    <p>Are you sure you want to proceed?</p>

    <x-button role="action" label="Yes" primary="true">
        <x-script role="click">
            $('#confirmDialog').data('kendoDialog').close();
            // ... commit action ...
        </x-script>
    </x-button>
    <x-button role="action" label="No">
        <x-script role="click">
            $('#confirmDialog').data('kendoDialog').close();
        </x-script>
    </x-button>
</x-popup>

<x-button role="action" label="Open">
    <x-script role="click">
        $('#confirmDialog').data('kendoDialog').open();
    </x-script>
</x-button>
```

| Attribute | Purpose |
|-----------|---------|
| `id` | DOM id |
| `title` | Title bar text |
| `width` | Width (`px` or `%`) |
| `content-height` | Inner content height |
| `var` | JS variable for the kendoDialog |

The dialog is initially closed; show it via `.open()` on the kendoDialog instance.

---

## `<x-menu>` family

Horizontal, vertical, or accordion-style menus with item types for actions, URLs, and submenus.

```cshtml
<x-menu menu-id="main"
        mobile-menu-id="mainMobile"
        orientation="horizontal"
        logo-image="@Url.Content("~/img/logo.png")"
        logo-href="@Url.Action("Index", "Home")">

    <x-menuitem-action area=""   controller="Home"     action="Index" name="Home" />
    <x-menuitem-action area=""   controller="Products" action="Index" name="Products" sprite="k-i-table" />

    <x-menuitem-submenu name="Reports">
        <x-menuitem-action controller="Reports" action="Sales"     name="Sales" />
        <x-menuitem-action controller="Reports" action="Inventory" name="Inventory" />
    </x-menuitem-submenu>

    <x-menuitem-url url="https://example.com/help" name="Help" target="_blank" />
</x-menu>
```

`<x-menu>` attributes:

| Attribute | Purpose |
|-----------|---------|
| `menu-id` | DOM id |
| `mobile-menu-id` | If set, a hamburger-toggleable mobile variant is generated |
| `orientation` | `horizontal`, `vertical`, `verticalPanel` |
| `width` | For vertical menus |
| `logo-image`, `logo-href`, `logo-height` | Brand logo |
| `menu-name` | Title shown in mobile mode |
| `mobile-menu-sprite` | Hamburger icon class |
| `scrollable` | Allow scroll in vertical menus |
| `panel-item-class` | CSS class for accordion panels (verticalPanel mode) |
| `auth-info` | Authentication context (drives permissions) |

Item types and common attributes:

| Item | Purpose | Key attrs |
|------|---------|-----------|
| `<x-menuitem-action>` | Link to an MVC action | `area`, `controller`, `action`, `name` |
| `<x-menuitem-url>` | Link to an arbitrary URL | `url`, `target`, `name` |
| `<x-menuitem-submenu>` | Container for child items | `name` |
| `<x-menuitem>` | Plain item (rare) | `name` |

Common to all items: `name` (text), `sprite` (icon), `image-url`, `when-logged-in` (`true`/`false`/unset), `permissions` (comma-separated; any-match wins).

For `permissions` to work, supply `auth-info="@authContext"` on the menu — `authContext` implements `IAuthInfo` (`bool IsLoggedIn { get; }`, `bool HasPermission(string)`).

---

## `<x-tabstrip>`

```cshtml
<x-tabstrip id="settingsTabs" orientation="top" animation="true">
    <x-tab id="general"  title="General">
        <x-form for="@Model.General"><!-- ... --></x-form>
    </x-tab>
    <x-tab id="security" title="Security">
        <!-- ... -->
    </x-tab>
    <x-tab id="advanced" title="Advanced">
        <!-- ... -->
    </x-tab>
</x-tabstrip>
```

| `<x-tabstrip>` attr | Purpose |
|---------------------|---------|
| `orientation` | `top` (default), `bottom`, `left`, `right` |
| `animation` | Animate tab switches |
| `collapsible` | Allow collapsing the active tab |
| `var` | JS variable for kendoTabStrip |

`<x-tab>` attrs: `id`, `title`.

---

## `<x-splitter>`

```cshtml
<x-splitter id="main" orientation="horizontal" style="height:100vh">
    <x-splitter-pane id="nav" size="240px" min-size="180px" max-size="320px" collapsible="true">
        <!-- nav -->
    </x-splitter-pane>
    <x-splitter-pane id="content">
        <!-- content -->
    </x-splitter-pane>
</x-splitter>
```

| `<x-splitter>` attr | Purpose |
|---------------------|---------|
| `orientation` | `horizontal` (default) or `vertical` |
| `var` | JS variable |

| `<x-splitter-pane>` attr | Purpose |
|--------------------------|---------|
| `size` | Initial size (`px` or `%`) |
| `min-size`, `max-size` | Bounds |
| `collapsible` | Allow user-collapse |
| `collapsed-size` | Size when collapsed |
| `scrollable` | Add scrollbars when content overflows |

---

## `<x-notification>`

Toast notifications, driven from script.

```cshtml
<x-notification id="toasts" stacking="up" position-top="20" position-right="20"
                auto-hide-after="4000" />

<x-script>
    var toasts = $('#toasts').data('kendoNotification');
    toasts.show('Saved', 'success');
    toasts.show('Could not save', 'error');
    toasts.show('Working…', 'information');
</x-script>
```

| Attribute | Purpose |
|-----------|---------|
| `auto-hide-after` | Auto-hide delay (ms) |
| `allow-hide-after` | Min visible time before user can dismiss (ms) |
| `hide-on-click` | Click to dismiss |
| `disable-animation` | Skip animations |
| `stacking` | `up`, `down`, `left`, `right` |
| `pinned` | Don't move with page scroll |
| `position-top` / `bottom` / `left` / `right` | Anchor offsets (px) |

---

## `<x-link>`

Plain hyperlink (semantic wrapper, not strictly necessary but useful when you want library styling).

```cshtml
<x-link id="details" href="@Url.Action("Details", new { id = Model.Id })" target="_blank">
    View details
</x-link>
```

---

## `<x-row>` / `<x-col>`

Bootstrap-grid layout helpers — emit `<div class="row">` and `<div class="col-…">` with proper Bootstrap classes.

```cshtml
<x-row>
    <x-col md-size="6" sm-size="12">
        <x-edit for="@Model.FirstName" label="First name" />
    </x-col>
    <x-col md-size="6" sm-size="12">
        <x-edit for="@Model.LastName" label="Last name" />
    </x-col>
</x-row>
```

| `<x-col>` attr | Purpose |
|----------------|---------|
| `xs-size`, `sm-size`, `md-size`, `lg-size` | Column width 1–12 at breakpoint |
| `xs-offset`, `sm-offset`, `md-offset`, `lg-offset` | Left offset |
| `xs-visible`, `sm-visible`, `md-visible`, `lg-visible` | Visibility at breakpoint |

`<x-row>` has `suppress-labels="true"` to hide field labels for inline forms.

---

## `<x-area>`

Logical grouping with show/hide via JS. No visual representation by default.

```cshtml
<x-area id="advancedOptions">
    <x-edit for="@Model.SecretCode" label="Secret code" />
</x-area>

<x-button role="action" label="Toggle advanced">
    <x-script role="click">
        tagHelper_manageArea('advancedOptions', 'toggle');
    </x-script>
</x-button>
```

JS API:
- `tagHelper_manageArea(id, 'show')`
- `tagHelper_manageArea(id, 'hide')`
- `tagHelper_manageArea(id, 'toggle')`

---

## `<x-barcode>`

Barcode display via Kendo's BarCode widget.

```cshtml
<x-barcode id="productBarcode" type="EAN13" value="5901234123457" width="240" height="100" />
```

| Attribute | Purpose |
|-----------|---------|
| `type` | `EAN8`, `EAN13`, `UPCE`, `UPCA`, `Code11`, `Code39`, `Code128`, `POSTNET`, etc. |
| `value` | String to encode |
| `width`, `height` | Dimensions |
| `color`, `background-color` | Bar/background colors |
| `padding`, `border-color`, `border-width` | Trim |
| `show-checksum` | Display checksum digit |

---

## `<x-partial>`

Render an MVC partial view, optionally with a model.

```cshtml
<x-partial url="@Url.Action("_AddressEditor")" for="@Model.BillingAddress" />
```

| Attribute | Purpose |
|-----------|---------|
| `url` | Partial URL |
| `for` | Model expression to pass as the partial's model |

---

## `<x-message>`

Override localized messages of a parent widget.

```cshtml
<x-text-editor for="@Model.Body">
    <x-message name="bold" value="Bold" />
    <x-message name="italic" value="Italic" />
</x-text-editor>
```

The `name` keys are widget-specific and align with Kendo's message catalogs.

---

## Script roles — concept

The library aggregates scripts from anywhere in the page (controls, columns, grids, buttons) and emits them in a single coordinated block — typically wrapped in `$(document).ready(function() { … })`. Each script has a *role* that controls:

- **Where** it's emitted (top-level vs inside a control's init function).
- **What** `this` and `e` mean inside it.
- **When** it runs (immediately on ready vs on a specific event).

Common roles by context:

| Parent | Role names |
|--------|-----------|
| `<x-script>` standalone | `script` (default), `global` |
| `<x-button role="action">` | `click` |
| `<x-grid>` | `databound`, `change`, `edit`, `save`, `cancel`, `cellClose`, `toolbar` |
| `<x-grid-column>` | `template`, `filter-ui`, `filter-operators`, `editor` |
| `<x-form>` | `submit`, `error` |
| `<x-control>` (most input controls) | `change`, `select`, `dataBound`, etc. |

If you use `script-role="byname"` you must also set `script-role-name="<name>"`. The named role is then matched to whatever the parent widget expects (e.g., `databound` on a grid).
