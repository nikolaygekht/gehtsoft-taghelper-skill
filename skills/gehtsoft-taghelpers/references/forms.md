# Forms Reference — Gehtsoft.TagHelpers

The `<x-form>` family: forms, controls, validation, AJAX behavior, layout.

## Contents

- [Mental model](#mental-model)
- [`<x-form>`](#x-form)
- [Form modes (layouts)](#form-modes-layouts)
- [Common control attributes](#common-control-attributes)
- [Text input](#text-input)
   - [`<x-edit>`](#x-edit) · [`<x-masked-edit>`](#x-masked-edit) · [`<x-password>`](#x-password) · [`<x-text-area>`](#x-text-area)
- [Numeric input](#numeric-input)
   - [`<x-numeric-edit>`](#x-numeric-edit)
- [Date / time input](#date--time-input)
   - [`<x-date-edit>`](#x-date-edit)
- [Choice controls](#choice-controls)
   - [`<x-select>`](#x-select) · [`<x-editable-select>`](#x-editable-select) · [`<x-autocomplete-edit>`](#x-autocomplete-edit) · [`<x-check>`](#x-check) · [`<x-check-switch>`](#x-check-switch) · [`<x-radio>`](#x-radio)
- [Tree controls](#tree-controls)
   - [`<x-tree-control>`](#x-tree-control) · [`<x-tree-select>`](#x-tree-select)
- [Specialized controls](#specialized-controls)
   - [`<x-upload>`](#x-upload) · [`<x-text-editor>`](#x-text-editor) · [`<x-signature>`](#x-signature) · [`<x-color-picker>`](#x-color-picker) · [`<x-listbox>`](#x-listbox)
- [Layout helpers](#layout-helpers)
   - [`<x-form-group>`](#x-form-group) · [`<x-static>`](#x-static) · [`<x-hidden>`](#x-hidden)
- [Validation](#validation)
   - [`<x-validate>`](#x-validate) · [`<x-enable>`](#x-enable) · [`<x-form-errors>`](#x-form-errors)
- [AJAX form behavior](#ajax-form-behavior)
- [Controller side — receiving form posts](#controller-side--receiving-form-posts)

---

## Mental model

A form has three concerns: **structure** (what controls exist and how they lay out), **binding** (which model field each control reads/writes), and **validation** (rules to enforce client-side and server-side). The library expresses all three declaratively.

```
<x-form>                           ← structure: container + layout mode + submit target
  <x-form-errors />                ← validation: bridges server errors and enables client validation
  <x-control for="@Model.X" />     ← binding: control linked to a model property
    <x-validate>...</x-validate>   ← validation: per-control rule
  </x-control>
  <x-button role="submit" />       ← submit
</x-form>
```

`for="@Model.Property"` is the canonical binding mechanism — it drives the label text (humanized property name), the posted field name, the validation provider lookup, and the AJAX serializer. Use bare `name="..."` only for fields with no model backing.

---

## `<x-form>`

Container that wraps controls, defines layout, and configures submission.

| Attribute | Purpose |
|-----------|---------|
| `id` | Form element id (used by JS API and AJAX helpers) |
| `for` | Model expression naming the **submodel** that controls bind to. Inner `for=` paths become relative to this. |
| `action` | Submit URL |
| `method` | `post` (default), `get`, `ajax`, `ajaxManual` — see [AJAX form behavior](#ajax-form-behavior) |
| `mode` | Layout: `twocolumn` (default), `horizontal`, `inline`, `custom` |
| `title` | Optional title rendered above the form |
| `block` | Apply Kendo's `k-block` styling |

**Example — basic form:**

```cshtml
<x-form id="customerForm"
        action="@Url.Action("Save")"
        method="post"
        for="@Model.Customer"
        mode="twocolumn">

    <x-edit for="@Model.Customer.Name" label="Name" />
    <x-edit for="@Model.Customer.Email" label="Email" />

    <x-button role="submit" label="Save" primary="true" />
</x-form>
```

When `for="@Model.Customer"` is set on the form, the inner controls are still safest written with the full path (`@Model.Customer.Name`) — Razor needs the full expression for compile-time checking. The `for=` on the form drives the *posted* shape: server receives a `Customer` object, not a wrapper.

## Form modes (layouts)

| Mode | Visual | Use for |
|------|--------|---------|
| `twocolumn` | Labels left, controls right, one control per row | Standard data entry forms |
| `horizontal` | Multiple controls per row, label above each | Wider inputs, dashboards |
| `inline` | All on one line | Filter bars, search rows |
| `custom` | No layout — caller arranges children | When using your own grid system |

Within `twocolumn`, control attributes that affect flow:

- `whole-row="true"` — the control spans both columns (use for text areas, tables)
- `suppress-label="true"` — drop the label cell entirely (rare)

## Common control attributes

Every control derives from a base that provides:

| Attribute | Purpose |
|-----------|---------|
| `for` | Model expression — drives binding, label, name, validation lookup |
| `label` | Override default label (otherwise derived from the property name) |
| `id` | DOM id (auto-generated if omitted) |
| `name` | Posted field name (use `for=` instead unless non-model) |
| `value` | Initial value (rare — `for=` reads from the model) |
| `placeholder` | Placeholder text |
| `tooltip` | Hover tooltip |
| `disabled` | Disable the control |
| `read-only` | Render as read-only |
| `hidden` | Render but hide via CSS |
| `width` | Explicit width (`"200px"` or `"50%"`) |
| `auto-focus` | Focus on page load |
| `tab-index` | Tab order |
| `whole-row` | (twocolumn mode) span both columns |
| `suppress-label` | Hide the label cell |

These appear on every control type below; they are not repeated in each section.

---

## Text input

### `<x-edit>`

Plain textbox.

```cshtml
<x-edit for="@Model.FirstName" label="First name" />
<x-edit for="@Model.Email" label="Email" placeholder="name@example.com" size="40" />
```

`size` — visible characters (HTML `size` attribute).

### `<x-masked-edit>`

Textbox with input mask.

```cshtml
<x-masked-edit for="@Model.Phone" label="Phone" mask="(999) 999-9999" />
<x-masked-edit for="@Model.Ssn"   label="SSN"   mask="999-99-9999" prompt-char="_" />
```

`mask` follows Kendo's mask syntax (`9` digit, `L` letter, `&` any char). `prompt-char` is the placeholder shown for empty positions (default `_`).

### `<x-password>`

Masked password input. Same shape as `<x-edit>`.

```cshtml
<x-password for="@Model.Password" label="Password" />
```

### `<x-text-area>`

Multi-line text.

```cshtml
<x-text-area for="@Model.Notes" label="Notes" rows="5" columns="60" max-length="500" whole-row="true" />
```

`rows`, `columns`, `height`, `max-length` — sizing and length cap.

---

## Numeric input

### `<x-numeric-edit>`

```cshtml
<x-numeric-edit for="@Model.Quantity" label="Quantity" spinners="true" spin-min="0" spin-max="999" spin-step="1" />
<x-numeric-edit for="@Model.Price"    label="Price"    format="c2" spinners="false" />
```

| Attribute | Purpose |
|-----------|---------|
| `spinners` | Show up/down buttons |
| `spin-min`, `spin-max`, `spin-step` | Bounds + increment |
| `format` | Kendo numeric format (`n2` two decimals, `c` currency, `p` percent) |

---

## Date / time input

### `<x-date-edit>`

```cshtml
<x-date-edit for="@Model.BirthDate" label="Birth date" format="MM/dd/yyyy" />
<x-date-edit for="@Model.AppointmentAt" label="Appointment"
             include-time="true" format="MM/dd/yyyy h:mm tt" interval="15" />
```

| Attribute | Purpose |
|-----------|---------|
| `include-time` | Enable time picker (default false) |
| `format` | **Kendo date format** — uppercase `MM` for month, lowercase `mm` for minutes, `tt` for AM/PM. Not the same as .NET format strings. |
| `min`, `max` | Date bounds |
| `interval` | Time picker minute step (default 15) |
| `show-week-number` | Show ISO week numbers |
| `calendar-start`, `calendar-depth` | `month` / `year` / `decade` / `century` for initial view and drill-depth |
| `use-formatted-input` | Apply a mask matching `format` |

---

## Choice controls

### `<x-select>`

Dropdown. Can be single-select (default) or multi-select. Items can be defined statically (child `<x-select-option>`), programmatically (`options=` attribute), or fetched from the server (`source-url=`).

**Static:**

```cshtml
<x-select for="@Model.Status" label="Status">
    <x-select-option value="active"   title="Active" />
    <x-select-option value="inactive" title="Inactive" />
    <x-select-option value="pending"  title="Pending" />
</x-select>
```

**Programmatic:**

```cshtml
@{
    var statusList = new[] {
        new { value = "active",   text = "Active" },
        new { value = "inactive", text = "Inactive" },
    };
}

<x-select for="@Model.Status" label="Status" options="@statusList" />
```

`options` accepts any `IEnumerable` whose items have `value` and `text` (case-insensitive) — anonymous types or DTOs both work.

**Server-fetched:**

```cshtml
<x-select for="@Model.CategoryId" label="Category"
          source-url="@Url.Action("CategoryOptions")" />
```

The endpoint must return:

```json
{ "data": [
    { "Id": 1, "Value": "Books" },
    { "Id": 2, "Value": "Music" }
] }
```

| Attribute | Purpose |
|-----------|---------|
| `multi-select` | Allow multiple selections |
| `options` | In-memory data |
| `source-url` | Server endpoint for items |
| `source-url-filtering` | Send filter text to server (`true` = server-side filtering) |

### `<x-editable-select>`

Like `<x-select>` but the user can type a value not in the list. Posts back as `[selectedId, displayText]` — handle accordingly on the server.

```cshtml
<x-editable-select for="@Model.Tag" label="Tag" options="@Model.TagList" />
```

### `<x-autocomplete-edit>`

Type-ahead textbox with server-driven suggestions. Posts only the typed text (no id).

```cshtml
<x-autocomplete-edit for="@Model.SearchTerm" label="Search"
                     autocomplete-action="@Url.Action("Suggest")"
                     autocomplete-min-length="2"
                     autocomplete-delay="300" />
```

| Attribute | Purpose |
|-----------|---------|
| `autocomplete-action` | Server URL — receives `q` query parameter |
| `autocomplete-min-length` | Minimum chars before query fires |
| `autocomplete-delay` | Debounce in ms |
| `autocomplete-ignore-case` | Case-insensitive |
| `autocomplete-separator` | Treat the typed text as a list with this separator |

Server returns the same `{ data: [{ Id, Value }] }` shape as `<x-select>`.

### `<x-check>`

Checkbox.

```cshtml
<x-check for="@Model.IsActive" label="Active" />
<x-check for="@Model.AgreeToTerms" label="I agree to the terms" suppress-label="true" />
```

In `twocolumn` mode the label appears in the left column and the checkbox in the right. To put the label *next to* the checkbox (no left column) use `suppress-label="true"`.

### `<x-check-switch>`

Checkbox styled as an iOS-style on/off switch.

```cshtml
<x-check-switch for="@Model.IsEnabled" label="Enabled"
                true-label="On" false-label="Off"
                switch-alignment="right" />
```

### `<x-radio>`

Radio button. One per option, all bound to the same property.

```cshtml
<x-form for="@Model">
    <x-radio for="@Model.Gender" label="Male"   option="M" />
    <x-radio for="@Model.Gender" label="Female" option="F" />
</x-form>
```

`option` is the value posted when selected.

---

## Tree controls

### `<x-tree-control>`

Hierarchical tree with local items (child `<x-tree-item>`s) or server data.

**Local:**

```cshtml
<x-tree-control id="org" label="Organization" for="@Model.SelectedDept" height="300">
    <x-tree-item id="1" value="Sales">
        <x-tree-item id="2" value="Inside Sales" />
        <x-tree-item id="3" value="Outside Sales" />
    </x-tree-item>
    <x-tree-item id="4" value="Engineering">
        <x-tree-item id="5" value="Frontend" />
        <x-tree-item id="6" value="Backend" />
    </x-tree-item>
</x-tree-control>
```

**Server-driven:**

```cshtml
<x-tree-control id="categories" label="Categories">
    <x-server-datasource page-size="50">
        <x-server-transport read-url="@Url.Action("TreeNodes")" />
        <x-server-schema>
            <x-server-tree-model id="id" has-children="hasChildren">
                <x-server-field name="id"           field-type="number" />
                <x-server-field name="text"         field-type="text" />
                <x-server-field name="hasChildren"  field-type="boolean" />
            </x-server-tree-model>
        </x-server-schema>
    </x-server-datasource>
</x-tree-control>
```

The endpoint receives an `id` parameter for each expansion (root call has no `id`) and returns:

```json
[
  { "id": 1, "text": "Books",  "hasChildren": true },
  { "id": 2, "text": "Music",  "hasChildren": false }
]
```

### `<x-tree-select>`

Dropdown that opens a tree picker.

```cshtml
<x-tree-select for="@Model.DepartmentId" label="Department"
               source-url="@Url.Action("DepartmentTree")" />
```

---

## Specialized controls

### `<x-upload>`

File upload, synchronous or chunked async.

**Synchronous (whole file in one POST):**

```cshtml
<x-upload for="@Model.Documents" label="Documents" multiple="true" show-files="true" />
```

The form's submit POST will include the files.

**Async (independent of form submit):**

```cshtml
<x-upload for="@Model.Avatar" label="Profile picture">
    <x-upload-async save-url="@Url.Action("SaveAvatarChunk")"
                    remove-url="@Url.Action("RemoveAvatar")"
                    auto-upload="true"
                    concurrent="false"
                    chunk-size="1048576"
                    initial-files="@Model.ExistingAvatars" />
</x-upload>
```

Server side for async (chunked) — the POST body has both the file chunk and a `metaData` JSON string:

```csharp
[HttpPost]
public async Task<IActionResult> SaveAvatarChunk(IEnumerable<IFormFile> files, string metaData)
{
    var meta = JsonConvert.DeserializeObject<AsyncUploadChunkData>(metaData);
    // meta.UploadUid, meta.FileName, meta.ChunkIndex, meta.TotalChunks, meta.ContentType

    // ... append chunk to a temp file keyed by meta.UploadUid ...

    bool isFinalChunk = meta.ChunkIndex == meta.TotalChunks - 1;
    return Json(new AsyncUploadResult { IsCompleted = isFinalChunk });
}
```

For non-chunked sync upload, return `Content("")` for success or a non-empty error string to signal failure.

### `<x-text-editor>`

Rich-text WYSIWYG editor with a configurable toolbar.

```cshtml
<x-text-editor for="@Model.Article.Body" label="Body" rows="10" columns="80" encoded="true" whole-row="true">
    <x-text-editor-command tool="bold" />
    <x-text-editor-command tool="italic" />
    <x-text-editor-command tool="createLink" />
    <x-text-editor-command tool="insertImage" />
    <x-text-editor-command tool="viewHtml" />
</x-text-editor>
```

`encoded="true"` HTML-encodes the model value before rendering (use when the model holds raw text). Set `encoded="false"` (or omit) when the model holds HTML.

Tool names mirror Kendo's: `bold`, `italic`, `underline`, `strikethrough`, `subscript`, `superscript`, `fontName`, `fontSize`, `foreColor`, `backColor`, `justifyLeft`/`Center`/`Right`/`Full`, `insertUnorderedList`, `insertOrderedList`, `indent`, `outdent`, `createLink`, `unlink`, `insertImage`, `insertFile`, `tableWizard`, `createTable`, `addColumnLeft`/`Right`, `addRowAbove`/`Below`, `deleteRow`, `deleteColumn`, `formatting`, `cleanFormatting`, `insertHtml`, `viewHtml`, `print`, `pdf`.

### `<x-signature>`

Touch/mouse signature pad — captures as a data URL.

```cshtml
<x-signature for="@Model.SignatureDataUrl" label="Signature"
             width="400" height="150"
             pen-color="black" background-color="white" pen-size="2" />
```

The model field receives a `data:image/png;base64,...` string.

### `<x-color-picker>`

```cshtml
<x-color-picker for="@Model.BrandColor" label="Brand color"
                palette="basic" clear-button="true" preview="true" />
```

`palette` — `websafe` (232), `basic` (16), or a comma-separated list of hex values.

### `<x-listbox>`

Selectable list with optional drag-drop reorder and a side toolbar.

```cshtml
<x-listbox for="@Model.SelectedItems" label="Items"
           options="@Model.AvailableItems"
           draggable="true"
           post="content">
    <x-listbox-toolbar position="right">
        <x-listbox-toolbar-button button="moveUp" />
        <x-listbox-toolbar-button button="moveDown" />
        <x-listbox-toolbar-button button="remove" />
    </x-listbox-toolbar>
</x-listbox>
```

`post`: `none` (don't post), `selection` (post selected only), `content` (post the whole list).
Toolbar buttons: `moveUp`, `moveDown`, `remove`, `transferTo`, `transferFrom`, `transferAllTo`, `transferAllFrom`. Use `connect-with="otherListboxId"` for transfer ops.

---

## Layout helpers

### `<x-form-group>`

Group of controls treated as one row in the layout flow. Use this — not raw `<div>` — to group controls in `twocolumn`/`horizontal`/`inline` modes.

```cshtml
<x-form-group label="Address" whole-row="true">
    <x-edit for="@Model.Street" label="Street" />
    <x-edit for="@Model.City"   label="City" />
    <x-edit for="@Model.Zip"    label="ZIP" />
</x-form-group>
```

**Control strip** — controls flow horizontally, one of them flexes to fill remaining width:

```cshtml
<x-form-group label="Search" control-strip="true">
    <x-control-strip>
        <x-edit for="@Model.Query" placeholder="Search…" />
    </x-control-strip>
    <x-control-strip fill="true">
        <x-button role="submit" label="Go" />
    </x-control-strip>
</x-form-group>
```

### `<x-static>`

Read-only display value (not an `<input>`).

```cshtml
<x-static for="@Model.CreatedAt" label="Created" static-type="information" icon="k-i-clock" />
<x-static value="Processing…" static-type="success" icon="k-i-check" />
```

`static-type`: `static` (plain), `information`, `success`, `error`.

### `<x-hidden>`

Hidden field carried along with the form post.

```cshtml
<x-hidden for="@Model.Id" />
<x-hidden for="@Model.RowVersion" />
```

---

## Validation

The library validates twice: on the client before submit, and on the server after submit. Server-side errors come back through `<x-form-errors>`.

### `<x-validate>`

A rule attached to a single control or to the form as a whole (cross-field).

**Per-control:**

```cshtml
<x-edit for="@Model.Email" label="Email">
    <x-validate error="Enter a valid email" is-expression="true">
        /^[^\s@@]+@@[^\s@@]+\.[^\s@@]+$/.test(value)
    </x-validate>
</x-edit>
```

`is-expression="true"` means the body is a boolean expression (returns true if valid). Without `is-expression`, the body is a function body (must `return` a boolean).

Inside the expression:
- `value` — the current control's value
- `reference('OtherFieldName')` — the value of another control by its posted name

**Cross-field (loose inside the form):**

```cshtml
<x-form for="@Model" method="post">
    <x-password for="@Model.Password"        label="Password" />
    <x-password for="@Model.ConfirmPassword" label="Confirm" />

    <x-validate error="Passwords do not match" is-expression="true">
        reference('Password') === reference('ConfirmPassword')
    </x-validate>
</x-form>
```

**Multiple rules** — list multiple `<x-validate>` children; they evaluate top-to-bottom. Use `cancel-next="true"` to stop the chain when one fails.

> **Razor escape note:** literal `@` in JavaScript regex bodies must be written as `@@` so Razor doesn't interpret it.

### `<x-enable>`

Conditional enable/disable, same shape as `<x-validate>` but the expression returns `true` to *enable* the control.

```cshtml
<x-edit for="@Model.PromoCode" label="Promo code">
    <x-enable is-expression="true">
        reference('UsePromo') === true
    </x-enable>
</x-edit>
```

### `<x-form-errors>`

Bridges server errors and enables client-side validation execution. Place it **inside** the form, near the top.

```cshtml
<x-form for="@Model.Customer" method="post" action="@Url.Action("Save")">
    <x-form-errors form-errors="@Model.Errors"
                   support-on-client-validation="true" />

    <!-- controls... -->
</x-form>
```

| Attribute | Purpose |
|-----------|---------|
| `form-errors` | `IEnumerable<string>` or `IStructValidationResult` from the server |
| `support-on-client-validation` | If `true` (default false), client validation runs before submit |
| `validator` | Provide a custom `IFormValidationProvider` (rare) |

Without `<x-form-errors>` the form submits without client-side validation and server errors won't render.

---

## AJAX form behavior

`<x-form method="ajax">` posts the form body as **JSON**, not as `application/x-www-form-urlencoded`. The server receives a JSON object whose properties match the controls' posted names. Use `[FromBody]` on the controller action.

```csharp
[HttpPost]
public IActionResult Save([FromBody] Customer customer)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    repo.Save(customer);
    return Ok(new { id = customer.Id });
}
```

**`method="ajax"` constraints:**

- Posted property names must be valid JS identifiers — no `.` or `[]` in names. Nested model properties (`@Model.Address.Street`) post as the literal name `"Address.Street"`, which JSON parsers will reject. For nested models prefer `method="post"` (form-encoded) **or** flatten into hidden fields and reassemble server-side.
- Files (`<x-upload>`) cannot be posted via the AJAX path; the upload runs as its own background POST.

**`method="ajaxManual"`** — the form provides no automatic submit. Drive it from script using helpers attached to the form id:

```cshtml
<x-button role="action" label="Save">
    <x-script role="click">
        tagHelper_ajaxPost('customerForm', '@Url.Action("Save")');
    </x-script>
</x-button>
```

Helpers available on AJAX-mode forms:
- `tagHelper_ajaxPost(formId, url)` — POST the form as JSON.
- `tagHelper_getForm(url)` — fetch a JSON object and load it into the form.
- `tagHelper_loadForm(formId, obj)` — load a JS object into the form.
- `tagHelper_saveForm(formId)` — read the form back as a JS object.
- `tagHelper_formToJson(formId)` — alias of the above.

---

## Controller side — receiving form posts

**Standard (`method="post"`)** — model-bound from form-encoded body:

```csharp
[HttpPost]
public IActionResult Save(Customer customer)
{
    if (!ModelState.IsValid)
    {
        var vm = new CustomerEditViewModel { Customer = customer, Errors = ModelStateErrors() };
        return View("Edit", vm);
    }
    repo.Save(customer);
    return RedirectToAction("Index");
}

private IEnumerable<string> ModelStateErrors() =>
    ModelState.Values.SelectMany(v => v.Errors).Select(e => e.ErrorMessage);
```

**AJAX (`method="ajax"`)** — JSON body, return a JSON envelope on failure so the form can render errors:

```csharp
[HttpPost]
public IActionResult Save([FromBody] Customer customer)
{
    if (!ModelState.IsValid)
        return BadRequest(new { errors = ModelStateErrors() });

    repo.Save(customer);
    return Ok(new { id = customer.Id });
}
```

The form's built-in AJAX handler will display error messages from the response body.
