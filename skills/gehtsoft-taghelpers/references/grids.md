# Grids Reference — Gehtsoft.TagHelpers

The `<x-grid>` family: tabular data with paging, sorting, filtering, optional editing, bound to in-memory data or a server endpoint via Kendo's request/response contract.

## Contents

- [Mental model](#mental-model)
- [`<x-grid>`](#x-grid)
- [`<x-grid-column>`](#x-grid-column)
- [Column templates](#column-templates)
- [Custom filter UI](#custom-filter-ui)
- [Data sources](#data-sources)
   - [`<x-server-datasource>`](#x-server-datasource) · [`<x-server-transport>`](#x-server-transport) · [`<x-server-schema>`](#x-server-schema) · [`<x-server-model>`](#x-server-model) · [`<x-server-field>`](#x-server-field) · [`<x-server-parameter>`](#x-server-parameter)
- [Server-side request/response contract](#server-side-requestresponse-contract)
- [Editable grids](#editable-grids)
- [Toolbars and commands](#toolbars-and-commands)
- [Grid events](#grid-events)
- [Tree grids](#tree-grids)

---

## Mental model

A grid has three layers:

```
<x-grid>                       ← presentation: paging, sorting, selection, sizing
  <x-grid-column ... />        ← per-column display rules
  <x-server-datasource>        ← what data and how
    <x-server-transport ... /> ← URLs for read/create/update/destroy
    <x-server-schema>          ← shape of the response and the records
      <x-server-model>
        <x-server-field />     ← per-field type, editability, defaults
      </x-server-model>
    </x-server-schema>
  </x-server-datasource>
</x-grid>
```

Server-paging/sorting/filtering is the recommended default. The data source sends Kendo's standard parameters (`skip`, `take`, `sort`, `filter`); you parse them on the server, return only the page, and report `total` for the pager.

---

## `<x-grid>`

```cshtml
<x-grid id="productGrid"
        height="500px"
        pageable="true"
        sortable="true"
        filterable="true"
        selectable="true"
        editable="incell">
    <!-- columns + datasource go inside -->
</x-grid>
```

| Attribute | Purpose |
|-----------|---------|
| `id` | DOM id (also used by JS API: `$('#productGrid').data('kendoGrid')`) |
| `var` | Optional JS variable name to receive the kendoGrid instance |
| `height` | Grid height (`px`, `%`, `vh`) |
| `pageable` | Show pager |
| `sortable` | Allow column sort |
| `filterable` | Show filter row/menu |
| `selectable` | Allow row selection (`true`, `single`, `multiple, row`, `multiple, cell`) |
| `editable` | `none` (default), `incell` (per-cell editing), `inline` (full row), or `popup` |
| `show-header` | Show or hide header row (default true) |

---

## `<x-grid-column>`

```cshtml
<x-grid-column title="ID"    field="id"   width="60px"  sortable="true" />
<x-grid-column title="Name"  field="name" width="200px" sortable="true" sort-by-default="asc" />
<x-grid-column title="Price" field="price" width="100px" format="{0:c2}" />
<x-grid-column title="Created" field="createdAt" width="140px" format="{0:d}" />
```

| Attribute | Purpose |
|-----------|---------|
| `title` | Header text |
| `field` | Source field on the model |
| `width` | Column width (`px`, `%`) — set explicitly to avoid layout thrashing |
| `format` | Kendo format string. `{0:c}` currency, `{0:c2}` two-decimal currency, `{0:n2}` number, `{0:d}` short date, `{0:G}` general date+time, `{0:p}` percent |
| `sortable` | Per-column sort toggle |
| `sort-by-default` | `no`, `asc`, `desc` — initial sort direction |
| `can-unsort` | Allow user to remove the sort |
| `filterable` | Per-column filter toggle |
| `class` | CSS class applied to data cells |
| `header-class` | CSS class applied to header cell |
| `row-class` | CSS class applied to the whole row when this column is the trigger |

A column with no `field` is presumed to be template-driven (see below) — it doesn't bind to a record property.

## Column templates

Use a template column for links, badges, action buttons, or computed display.

**Inline template (HTML with `#= e.field #` expressions):**

```cshtml
<x-grid-column title="Name" width="200px">
    <x-template>
        <a href="/Products/Edit/#= e.id #">#= kendo.htmlEncode(e.name) #</a>
    </x-template>
</x-grid-column>
```

**Function template (when you need real JavaScript):**

```cshtml
<x-grid-column title="Status" width="100px">
    <x-script role="template" parameter-name="row">
        var color = row.active ? 'green' : 'gray';
        return '<span style="color:' + color + '">' +
               kendo.htmlEncode(row.statusText) +
               '</span>';
    </x-script>
</x-grid-column>
```

**Action column (delete + edit):**

```cshtml
<x-grid-column title="Actions" width="120px">
    <x-template>
        <a class="k-button" href="/Products/Edit/#= e.id #">Edit</a>
        <a class="k-button k-button-icontext" onclick="confirmDelete(#= e.id #); return false;">Delete</a>
    </x-template>
</x-grid-column>
```

Always wrap user-controlled text in `kendo.htmlEncode(...)` inside templates — Kendo doesn't auto-encode.

## Custom filter UI

Replace the default filter input with a dropdown, date picker, etc.

```cshtml
<x-grid-column title="Status" field="status" filterable="true">
    <x-script script-role="byname" script-role-name="filter-ui" parameter-name="e">
        e.kendoDropDownList({
            dataValueField: 'value',
            dataTextField: 'text',
            optionLabel: 'Any',
            dataSource: [
                { value: 'active',   text: 'Active' },
                { value: 'inactive', text: 'Inactive' }
            ]
        });
    </x-script>
    <x-script script-role="byname" script-role-name="filter-operators">
        return { string: { eq: 'Equals', neq: 'Not equals' } };
    </x-script>
</x-grid-column>
```

---

## Data sources

### `<x-server-datasource>`

```cshtml
<x-server-datasource page-size="20"
                     server-paging="true"
                     server-sorting="true"
                     server-filtering="true">
    <x-server-transport read-url="@Url.Action("List")" />
    <x-server-schema>
        <x-server-model id="id">
            <x-server-field name="id"        field-type="number" />
            <x-server-field name="name"      field-type="text" />
            <x-server-field name="price"     field-type="number" />
            <x-server-field name="createdAt" field-type="date" />
        </x-server-model>
    </x-server-schema>
</x-server-datasource>
```

| Attribute | Purpose |
|-----------|---------|
| `page-size` | Records per page |
| `page` | Initial page (1-based) |
| `server-paging` | Server returns the requested page only |
| `server-sorting` | Server applies sort |
| `server-filtering` | Server applies filter |

Set all three `server-*` attributes to `true` for any dataset over a few hundred rows.

### `<x-server-transport>`

```cshtml
<x-server-transport read-url="@Url.Action("List")"
                    create-url="@Url.Action("Create")"
                    update-url="@Url.Action("Update")"
                    destroy-url="@Url.Action("Delete")"
                    read-method="POST"
                    update-method="POST" />
```

| Attribute | Purpose |
|-----------|---------|
| `read-url` | Read endpoint |
| `create-url`, `update-url`, `destroy-url` | Write endpoints (only needed for editable grids) |
| `read-method`, `update-method` | `GET` (default) or `POST` |
| `result-type` | `json` (default), `xml`, `jsonp` |

### `<x-server-schema>`

```cshtml
<x-server-schema total-name="total" result-name="data" errors-name="errors">
    <!-- model -->
</x-server-schema>
```

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `result-name` | `data` | Field on the JSON response holding the array |
| `total-name` | `total` | Field holding the total count (for pager) |
| `errors-name` | (none) | Field holding error info |
| `result-type` | `json` | `json` or `xml` |

If the server returns a different envelope shape, override these. The defaults match the Kendo MVC convention.

### `<x-server-model>`

```cshtml
<x-server-model id="id">
    <x-server-field name="id"        field-type="number" nullable="false" />
    <x-server-field name="name"      field-type="text"   editable="true"  />
    <x-server-field name="price"     field-type="number" editable="true"  default-value="0" />
    <x-server-field name="active"    field-type="boolean" editable="true" default-value="true" />
    <x-server-field name="createdAt" field-type="date"   editable="false" />
</x-server-model>
```

`id` names the primary-key field — the data source uses it to track records across operations.

### `<x-server-field>`

| Attribute | Purpose |
|-----------|---------|
| `name` | Field name (must match the JSON property) |
| `field-type` | `text`, `number`, `boolean`, `date`, `variant`, `unknown` |
| `editable` | `true` if user can edit this field (only meaningful for editable grids) |
| `nullable` | Allow nulls |
| `default-value` | Used when creating a new record |

### `<x-server-parameter>`

Send extra parameters with each read/write call.

**Constant value:**

```cshtml
<x-server-parameter operation="read" parameter-name="includeArchived" value="true" />
```

**Computed at request time (script body returns the value):**

```cshtml
<x-server-parameter operation="read" parameter-name="categoryId">
    <x-script>
        return $('#categoryFilter').val();
    </x-script>
</x-server-parameter>
```

`operation` is one of `read`, `create`, `update`, `destroy`.

---

## Server-side request/response contract

The data source sends a request like:

```
POST /Products/List
Content-Type: application/x-www-form-urlencoded

take=20&skip=40&page=3&pageSize=20&sort[0][field]=name&sort[0][dir]=asc
&filter[logic]=and&filter[filters][0][field]=name&filter[filters][0][operator]=startswith
&filter[filters][0][value]=Acme
```

(Or as a query string for GET.)

The server should return:

```json
{
  "data": [
    { "id": 1, "name": "Widget A", "price": 9.99,  "createdAt": "2025-08-01T00:00:00Z" },
    { "id": 2, "name": "Widget B", "price": 14.99, "createdAt": "2025-08-02T00:00:00Z" }
  ],
  "total": 153
}
```

A simple controller for read-only:

```csharp
[HttpPost]
public IActionResult List(int skip, int take, string sort, string filter)
{
    IQueryable<Product> q = db.Products;

    // Apply filter (parse the filter JSON if present)
    // Apply sort (parse the sort JSON if present)

    int total = q.Count();
    var page = q.Skip(skip).Take(take).ToList();

    return Json(new { data = page, total });
}
```

For full Kendo request parsing (filter trees, multi-sort, etc.), use a helper such as `KendoUI.DataRequest`:

```csharp
[HttpPost]
public IActionResult List([DataSourceRequest] DataSourceRequest request)
{
    var result = db.Products.AsQueryable().ApplyDataSourceRequest(request);
    return Json(result);
}
```

(`KendoUI.DataRequest` is a sibling project commonly used with this library — verify its package name in the project's `.csproj` before assuming it's available. If not present, parse `skip`/`take`/`sort`/`filter` manually.)

Date fields should serialize as ISO-8601 strings (`"2025-08-01T00:00:00Z"`); the data source parses them per the field's `field-type="date"`.

---

## Editable grids

Set `editable` on `<x-grid>` and provide the write URLs on the transport:

```cshtml
<x-grid id="productGrid" pageable="true" editable="incell">
    <x-grid-column title="Name"  field="name"  width="200px" />
    <x-grid-column title="Price" field="price" width="100px" format="{0:c2}" />
    <x-grid-column title="" width="100px">
        <x-template>
            <a class="k-button k-grid-delete" href="\#">Delete</a>
        </x-template>
    </x-grid-column>

    <x-server-datasource page-size="20" server-paging="true" server-sorting="true">
        <x-server-transport read-url="@Url.Action("List")"
                            create-url="@Url.Action("Create")"
                            update-url="@Url.Action("Update")"
                            destroy-url="@Url.Action("Delete")" />
        <x-server-schema>
            <x-server-model id="id">
                <x-server-field name="id"    field-type="number" />
                <x-server-field name="name"  field-type="text"   editable="true" />
                <x-server-field name="price" field-type="number" editable="true" default-value="0" />
            </x-server-model>
        </x-server-schema>
    </x-server-datasource>
</x-grid>
```

Editing modes:

- `incell` — cell becomes an editor on click; commit on blur.
- `inline` — entire row becomes editable on edit-button click; commit/cancel buttons appear.
- `popup` — opens a modal for the row.

**Server-side** — for create/update, expect a model bound from the request body; return the saved entity (so client-generated ids get replaced):

```csharp
[HttpPost]
public IActionResult Create(Product p)
{
    repo.Insert(p);
    return Json(new { data = new[] { p } });
}

[HttpPost]
public IActionResult Update(Product p)
{
    repo.Update(p);
    return Json(new { data = new[] { p } });
}

[HttpPost]
public IActionResult Delete(Product p)
{
    repo.Delete(p.Id);
    return Json(new { data = new[] { p } });
}
```

## Toolbars and commands

Add a toolbar (e.g., an "Add new" button) with a named script:

```cshtml
<x-grid id="productGrid" editable="inline">
    <x-script script-role="byname" script-role-name="toolbar">
        return [{ name: "create", text: "Add Product", iconClass: "k-i-plus" }];
    </x-script>

    <x-grid-column title="Name"  field="name"  width="200px" />
    <x-grid-column title="Price" field="price" width="100px" format="{0:c2}" />
    <x-grid-column title="Commands" width="200px">
        <x-template>
            <a class="k-button k-grid-edit" href="\#">Edit</a>
            <a class="k-button k-grid-delete" href="\#">Delete</a>
        </x-template>
    </x-grid-column>

    <!-- datasource... -->
</x-grid>
```

The Kendo CSS classes `k-grid-edit`, `k-grid-delete`, `k-grid-update`, `k-grid-cancel` wire the buttons to the grid's editing actions.

---

## Grid events

Hook into grid lifecycle:

```cshtml
<x-grid id="productGrid">
    <!-- columns + datasource -->

    <x-script script-role="byname" script-role-name="databound">
        console.log('Grid loaded', this.dataSource.total(), 'rows');
    </x-script>

    <x-script script-role="byname" script-role-name="change">
        var selectedRow = this.select();
        var item = this.dataItem(selectedRow);
        console.log('Selected', item);
    </x-script>

    <x-script script-role="byname" script-role-name="edit">
        // e.model — the record being edited
        // e.container — jQuery wrapper for the row/cell
    </x-script>
</x-grid>
```

Common events: `dataBound`, `change`, `beforeEdit`, `edit`, `cellClose`, `cancel`, `save`, `remove`, `navigate`, `page`, `filter`. Use `script-role-name` matching the lowercase event name.

---

## Tree grids

For hierarchical data, swap `<x-server-model>` for `<x-server-tree-model>` with `has-children`:

```cshtml
<x-server-tree-model id="id" has-children="hasChildren">
    <x-server-field name="id"           field-type="number" />
    <x-server-field name="name"         field-type="text" />
    <x-server-field name="hasChildren"  field-type="boolean" />
</x-server-tree-model>
```

The grid will request children lazily by passing the parent `id`. (Tree grid presentation may require `kendoTreeList` instead of `kendoGrid` — see the library source if you need this and the rendering looks off.)
