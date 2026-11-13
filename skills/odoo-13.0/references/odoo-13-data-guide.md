---
name: odoo-13-data
description: Complete reference for Odoo 13 data files - XML and CSV. Covers record/field/delete/function tags, shortcuts (menuitem, template, report, act_window), noupdate, and value resolution (eval/ref/search/type/file).
globs: "**/*.{xml,csv}"
topics:
  - XML data files structure (<odoo>, <data>)
  - record tag (create / update records)
  - field tag value resolution (eval, ref, search, type, file)
  - delete tag
  - function tag (calling methods / module hooks)
  - Shortcuts (menuitem, template, report, act_window)
  - CSV data files and :id / .id suffix references
  - noupdate attribute
  - External IDs and ir.model.data
when_to_use:
  - Authoring v13 data / demo files
  - Loading views, menus, actions, security rules
  - Populating seed data on install
  - Referencing other modules' records
---

# Odoo 13 Data Files Guide

Reference for Odoo 13 data files: XML structure, record/field/delete/function tags, shortcuts, CSV files, and value resolution.

All behaviour below is grounded in the 13.0 loader (`odoo/tools/convert.py`) and the
external-id model (`odoo/addons/base/models/ir_model.py`). Odoo 13.0 has no
`<asset>` tag, no `Command` class in the eval context, and no `:type` CSV column
suffix; where a newer-version alternative exists it is called out explicitly as
"not available in 13.0".

## Table of Contents

1. [Data File Structure](#data-file-structure)
2. [External IDs](#external-ids)
3. [`<record>` Tag](#record-tag)
4. [`<field>` Tag Values (eval / ref / search / type / file)](#field-tag-values)
5. [Relational Field Commands](#relational-field-commands)
6. [`<delete>` Tag](#delete-tag)
7. [`<function>` Tag](#function-tag)
8. [Shortcuts (`<menuitem>`, `<template>`, `<report>`, `<act_window>`)](#shortcuts)
9. [CSV Data Files](#csv-data-files)
10. [`noupdate` Attribute](#noupdate-attribute)
11. [Loading Order & Modes](#loading-order--modes)
12. [Common Pitfalls](#common-pitfalls)
13. [Quick Reference](#quick-reference)

---

## Data File Structure

Every XML data file is a well-formed XML document whose root is `<odoo>`
(`<data>` and the legacy `<openerp>` are accepted as roots too). The loader's
`DATA_ROOTS` are exactly `['odoo', 'data', 'openerp']`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <!-- (Re-)loaded at install and at every module upgrade -->
    <record id="always_updatable" model="my.model">
        <field name="name">Value</field>
    </record>

    <!-- Loaded on install; skipped on upgrade -->
    <data noupdate="1">
        <record id="user_editable" model="my.model">
            <field name="name">Default</field>
        </record>
    </data>
</odoo>
```

### Accepted Top-Level Tags

The loader (`odoo/tools/convert.py::xml_import.__init__`) recognises exactly these
children of `<odoo>` / `<data>` / `<openerp>`:

| Tag | Purpose |
|-----|---------|
| `<record>` | Create or update a record of an arbitrary model |
| `<delete>` | Delete records by id or by search domain |
| `<function>` | Call an arbitrary model method |
| `<menuitem>` | Shortcut to create an `ir.ui.menu` |
| `<template>` | Shortcut to create a QWeb `ir.ui.view` |
| `<report>` | Shortcut to create an `ir.actions.report` |
| `<act_window>` | Shortcut to create an `ir.actions.act_window` |

Any other tag at the top level is silently ignored (the loader looks the tag up in
`_tags` and `continue`s when it is absent). There is **no `<asset>` tag** in 13.0
(that arrives later with the `ir.asset` model); assets are declared with a
`<template id="assets_backend" inherit_id="web.assets_backend">` view instead.

### File Locations

Convention-based, declared via the `data:` / `demo:` keys of `__manifest__.py`
(`__openerp__.py` is also accepted but deprecated):

| Directory | When Loaded |
|-----------|-------------|
| `data/` | Always, at install and upgrade (from `data:` in the manifest) |
| `demo/` | Only when demo data is enabled (from `demo:` in the manifest) |
| `security/` | `ir.model.access.csv` and record-rule XML (loaded from `data:`) |
| `views/` | Views/menus/actions (loaded from `data:`) |

13.0 manifests have **no `assets` key**; the manifest keys that pull data are
`data`, `demo`, `test`, and the deprecated `init_xml`, `update_xml`, `demo_xml`.
JavaScript QWeb templates are declared with the `qweb` manifest key (a list of
`static/src/xml/*.xml` paths), not in `data`.

---

## External IDs

Every data record is addressed by an **external ID** (aka XML-id), stored as a row
in `ir.model.data`.

Format: `module.name`

- Inside the same module, the module prefix can be omitted: `ref="admin"` is
  equivalent to `ref="<current_module>.admin"` (the loader's `make_xml_id`
  prepends the installing module when the string contains no dot).
- Across modules always use the full form: `ref="base.user_admin"`.
- Exactly **one dot** is allowed: `module.identifier` (the identifier itself must
  not contain a dot). `convert.py::_test_xml_id` asserts this and additionally
  asserts the referenced module is installed.

```xml
<record id="my_partner" model="res.partner">
    <field name="name">Corp</field>
</record>

<record id="my_contact" model="res.partner">
    <field name="name">Jane</field>
    <field name="parent_id" ref="my_partner"/>            <!-- same module -->
    <field name="country_id" ref="base.us"/>              <!-- cross module -->
</record>
```

The `ir.model.data` model exposes the resolver methods the loader relies on:
`xmlid_lookup`, `xmlid_to_res_model_res_id`, `xmlid_to_res_id`,
`xmlid_to_object`, `get_object_reference`, and `_load_xmlid`
(all in `odoo/addons/base/models/ir_model.py`). Its `noupdate` column is what
protects records from being overwritten on upgrade (see the noupdate section).

---

## `<record>` Tag

```xml
<record id="partner_acme" model="res.partner">
    <field name="name">ACME Corp</field>
    <field name="is_company" eval="True"/>
    <field name="country_id" ref="base.us"/>
</record>
```

### `<record>` Attributes

| Attribute | Purpose |
|-----------|---------|
| `id` | External ID (recommended; required to make the record updatable and referenceable) |
| `model` | Target model (required) |
| `context` | Extra context dict (a Python literal) passed to create/write |
| `uid` | External ID of a user to run as (defaults to SUPERUSER) |
| `forcecreate` | `"0"`/`"false"` skips creation when the record is missing during an update run inside a `noupdate="1"` block |

### Create vs. Update

The loader decides based on the existence of the external ID:

- First time the file is loaded -> `create`.
- Subsequent upgrades -> `write` (unless the record is protected by `noupdate`,
  in which case it is left untouched).

The decision lives in `models.py::BaseModel._load_records`: records whose XML-id
is missing go into `to_create`; records whose XML-id exists go into `to_update`
(unless the XML-id's own `noupdate` flag is set, in which case they are kept as
is). Re-declaring only a subset of fields writes only those fields:

```xml
<!-- Later XML (same id) only updates `email` -->
<record id="partner_acme" model="res.partner">
    <field name="email">info@acme.test</field>
</record>
```

An empty `<record>` (no `<field>` children) produces an empty `write` and changes
nothing.

---

## `<field>` Tag Values

A `<field>` sets one attribute of the record. `convert.py::_tag_record` resolves a
value in this priority order:

1. `search="..."` - evaluate a domain, use the first match (all matches for many2many).
2. `ref="..."` - resolve an external ID to a database id.
3. `eval="..."` - evaluate a Python expression.
4. Inline text, interpreted according to `type="..."` (default `char`).

The base value starts as `False`, so an empty field writes `False`:

```xml
<field name="partner_id"/>   <!-- Sets to False -->
```

### Direct Value

```xml
<field name="name">ACME Corp</field>
<field name="ref">ACME-001</field>
```

Numeric/boolean coercion is driven by the destination field's ORM type. After
resolving `eval`/inline text, `_tag_record` coerces: `many2one` -> `int`, `integer`
-> `int`, `float`/`monetary` -> `float`, `boolean` string -> `str2bool`. So
`<field name="active">1</field>` writes `True` on a Boolean, and
`<field name="user_id">1</field>` writes integer `1` on a Many2one.

### `type="..."` - Interpretation Hints

| Type | Semantics |
|------|-----------|
| `char` (default) | Plain string |
| `int` | Integer (`None` maps to Python `None`) |
| `float` | Float |
| `xml` / `html` | Serialise child XML/HTML into the field (`xml` wraps multiple roots in a `<data>` container and prefixes an XML declaration) |
| `file` | Validate a module-relative path and store it as `"module,path"` |
| `base64` | Base64-encode the bytes read from `file=` |
| `list` / `tuple` | Build a list/tuple from nested `<value>` children |

```xml
<!-- HTML body -->
<field name="description" type="html">
    <p>Read the <a href="https://odoo.com">docs</a>.</p>
</field>

<!-- Binary file loaded from a module path -->
<field name="data" type="base64" file="my_module/static/src/img/logo.png"/>

<!-- File path reference: the PATH is the TEXT CONTENT, not a file= attribute -->
<field name="image_path" type="file">my_module/static/src/img/photo.jpg</field>

<!-- Integer -->
<field name="priority" type="int">10</field>

<!-- List -->
<field name="tags" type="list">
    <value>alpha</value>
    <value>beta</value>
    <value eval="'gamma'"/>
</field>
```

Note the difference between `type="base64"` and `type="file"`:

- `type="base64" file="module/path"` reads the file bytes and returns base64.
- `type="file"` reads its **text content** as a module-relative path, validates it
exists via `module.get_module_resource`, and stores `"module,path"`.

### `eval="..."` - Python Expression

Evaluated with `safe_eval`. The evaluation context (built in
`convert.py::_get_idref`, over a restricted-builtins sandbox from
`tools/safe_eval.py`) includes:

| Name | Description |
|------|-------------|
| `True`, `False`, `None` | Python literals (plus the restricted builtins: `int`, `float`, `str`, `list`, `dict`, `tuple`, `set`, `range`, `len`, `min`, `max`, `sum`, `sorted`, `map`, `filter`, `enumerate`, `zip`, `isinstance`, ...) |
| `ref` | `ref('module.xmlid')` -> database integer id (bound to `id_get`) |
| `obj` | `obj('res.partner')` -> `env['res.partner'].browse` (only present when a `model` attribute is given) |
| `time`, `datetime`, `DateTime`, `timedelta`, `relativedelta` | Date/time helpers |
| `pytz` | Timezone helper |
| `version` | `odoo.release.major_version`, e.g. `"13.0"` |

There is **no `Command` name** in the 13.0 eval context (`odoo.fields.Command`
does not exist); use the legacy tuple form instead (see Relational Field
Commands).

```xml
<field name="active" eval="True"/>
<field name="amount" eval="19.95"/>
<field name="date" eval="(datetime.date.today() + relativedelta(days=30)).strftime('%Y-%m-%d')"/>
<field name="groups_id" eval="[(6, 0, [ref('base.group_user')])]"/>
```

### `ref="..."` - External ID Reference

Resolves an external ID to its database integer id. Works on Many2one and on any
field expecting an id.

```xml
<field name="user_id" ref="base.user_admin"/>
<field name="country_id" ref="base.us"/>
```

On a **reference** (polymorphic) field, `ref` stores `"model,id"` automatically
(`_tag_record` calls `model_id_get`, which returns `(model, res_id)`, then joins
them with a comma):

```xml
<field name="resource_ref" ref="base.user_admin"/>
<!-- Stored as "res.users,<id>" -->
```

### `search="..."` - Domain Search

Evaluate an ORM domain (with `safe_eval`) and take the result.

```xml
<record id="demo_partner" model="res.partner">
    <field name="name">Demo</field>
    <field name="country_id" search="[('code','=','US')]"/>   <!-- first match -->
</record>
```

Rules (from `_tag_record` / `_eval_xml`):

- Many2one: uses the first matching record.
- Many2many: uses the full set, written as `(6, 0, [ids])`.
- Use `use="..."` to pick a field other than `id` from the matched records.
- On a search field you must also give `model="..."` when the loader cannot infer
  the comodel (it infers `comodel_name` automatically for a named ORM field).

```xml
<field name="country_id" model="res.country" search="[('code','=','US')]"/>
```

### Placeholder Substitution (`%(...)d` / `%(...)s`)

Inline field text supports legacy positional substitution: `%(module.xmlid)d` or
`%(module.xmlid)s` is replaced by the referenced record's database id. This is how
older `type="xml"` view arches interpolated record ids. The matching is done by
`convert.py::_eval_xml` (the `_process` helper). Prefer `ref=` for new code, but
the substitution is still valid 13.0.

```xml
<field name="arch" type="xml">
    <field name="category_id" domain="[('id','child_of',%(module.root_cat)d)]"/>
</field>
```

---

## Relational Field Commands

Odoo 13.0 uses **tuple commands only** - there is no `Command` class. The tuple
forms are the canonical way to express x2many mutations in `eval`.

| Tuple form | Effect |
|------------|--------|
| `(0, 0, {values})` | Create a related record |
| `(1, id, {values})` | Update the linked record |
| `(2, id, 0)` | Delete the record and unlink |
| `(3, id, 0)` | Unlink only (remove the relation) |
| `(4, id, 0)` | Link an existing record |
| `(5, 0, 0)` | Clear all links |
| `(6, 0, [ids])` | Replace the link set |

Examples:

```xml
<!-- Replace tags (M2M) -->
<field name="category_id" eval="[(6, 0, [ref('cat_customer'), ref('cat_supplier')])]"/>

<!-- Add a tag without removing existing -->
<field name="category_id" eval="[(4, ref('cat_vip'))]"/>

<!-- Clear -->
<field name="line_ids" eval="[(5, 0, 0)]"/>

<!-- Create inline child (O2M) -->
<field name="line_ids" eval="[
    (0, 0, {'name': 'Line A', 'price': 10.0}),
    (0, 0, {'name': 'Line B', 'price': 20.0}),
]"/>
```

### Inline O2M

There is no nested `<record>`-inside-`<record>` syntax in 13.0: the loader only
reads `<field>` children of a `<record>` (`convert.py::_tag_record` uses
`rec.findall('./field')`). To give child records their own external ids, create
them as separate records and point the inverse many2one back at the parent:

```xml
<record id="order_demo" model="sale.order">
    <field name="partner_id" ref="base.res_partner_2"/>
</record>

<record id="order_demo_line_1" model="sale.order.line">
    <field name="order_id" ref="order_demo"/>
    <field name="product_id" ref="product.product_product_1"/>
    <field name="product_uom_qty">1</field>
</record>

<record id="order_demo_line_2" model="sale.order.line">
    <field name="order_id" ref="order_demo"/>
    <field name="product_id" ref="product.product_product_2"/>
    <field name="product_uom_qty">2</field>
</record>
```

For children without external ids, the `(0, 0, {...})` tuple command on the
parent's one2many field is the alternative (see above).

---

## `<delete>` Tag

Remove records at load time. `id` and `search` are mutually exclusive.

```xml
<!-- By external ID -->
<delete model="res.partner" id="legacy.partner_old"/>

<!-- By domain (removes all matching) -->
<delete model="ir.ui.menu" search="[('name','=','Obsolete')]"/>
```

Behaviour (from `convert.py::_tag_delete`):

- With `search`, the domain is evaluated with `safe_eval`; on `ValueError` the
  loader logs a warning and skips the deletion.
- With `id`, a missing external ID logs a warning and continues (it does not
  raise).
- Matched records are `unlink()`ed.

---

## `<function>` Tag

Invoke a method on a model.

```xml
<!-- With eval -> positional args list -->
<function model="res.partner" name="create"
          eval="[{'name': 'From XML', 'email': 'x@example.com'}]"/>

<!-- With explicit <value> children -->
<function model="my.module" name="setup_defaults">
    <value>arg1</value>
    <value eval="ref('base.user_admin')"/>
    <value name="company_id" eval="ref('base.main_company')"/>   <!-- becomes kwarg -->
</function>

<!-- Nested function: the inner call's result is passed as args -->
<function model="res.partner" name="write">
    <function model="res.partner" name="search"
              eval="[[('vip','=',True)]]"/>
    <value eval="{'category_id': [(4, ref('cat_vip'))]}"/>
</function>
```

Notes:

- The `<value>` children are positional arguments unless they carry a `name`
  attribute, in which case they become keyword arguments.
- `context` is always merged into the call kwargs (`convert.py::_eval_xml`
  function branch).
- Inside a `<data noupdate="1">` block, `<function>` is **skipped** except during
  an `init` install (`convert.py::_tag_function` returns early when
  `self.noupdate and self.mode != 'init'`). Use it for one-shot post-install
  hooks, cache clearing, or data migration at install time.

---

## Shortcuts

### `<menuitem>` - ir.ui.menu Shortcut

```xml
<menuitem id="menu_root" name="My Module" sequence="10"
          web_icon="my_module,static/description/icon.png"/>

<menuitem id="menu_records" name="Records" parent="menu_root"
          action="action_my_records" sequence="1"/>

<!-- Security groups (prefix "-" to remove) -->
<menuitem id="menu_admin_only" name="Administration"
          parent="menu_root" groups="base.group_system"/>
```

| Attribute | Description |
|-----------|-------------|
| `id` | External ID |
| `name` | Label (defaults to `id` when omitted) |
| `parent` | Parent menu external ID |
| `action` | External ID of any `ir.actions.*` record |
| `sequence` | Integer; lower sorts first |
| `groups` | Comma-separated groups; prefix `-` to exclude |
| `web_icon` | `module,path/to/icon.png` - used for top-level app menus |
| `active` | `"True"` / `"False"` (defaults to `True`) |

`convert.py::_tag_menuitem` resolves `action` to `"<type>,<id>"`, and when `name`
is omitted it copies the action's `name` if the action type ends with one of
`act_window`, `wizard`, `url`, `client`, `server`. `groups` becomes `(4, group_id)`
link commands (or `(3, group_id)` for `-`-prefixed exclusions).

### `<template>` - QWeb ir.ui.view Shortcut

```xml
<template id="landing_page" name="Landing Page">
    <div class="container">
        <h1 t-esc="title"/>
    </div>
</template>

<!-- Inheritance -->
<template id="landing_page_extra" inherit_id="my_module.landing_page">
    <xpath expr="//h1" position="after">
        <p>Extended</p>
    </xpath>
</template>

<!-- Primary (a clone rather than a modifier) -->
<template id="landing_alternative"
          inherit_id="my_module.landing_page"
          primary="True"/>
```

The shortcut (`convert.py::_tag_template`) expands to an `ir.ui.view` record
(`theme.ir.ui.view` when the module name starts with `theme_`) with
`type="qweb"`, `key` set to the fully-qualified `<module>.<id>`, and the inner
XML stored in the `arch` field.

Supported attributes: `id`/`t-name`, `name`, `inherit_id`, `primary` (clones
instead of extending), `priority`, `track`, `key`, `website_id`, `active`,
`customize_show`, `groups`, `forcecreate`, `context`.

**Assets in 13.0.** There is no `<asset>` tag and no `assets` manifest key.
Bundles are declared by inheriting the bundle templates in XML:

```xml
<template id="assets_backend" inherit_id="web.assets_backend">
    <xpath expr="." position="inside">
        <script type="text/javascript" src="/my_module/static/src/js/widget.js"/>
        <link rel="stylesheet" type="text/scss" href="/my_module/static/src/scss/style.scss"/>
    </xpath>
</template>
```

Client-side QWeb templates go in the manifest `qweb` key, e.g.
`'qweb': ['static/src/xml/widget.xml']`.

### `<report>` Shortcut (ir.actions.report)

```xml
<report id="action_report_my_model" string="My Report" model="my.model"
        report_type="qweb-pdf" name="my_module.my_report_template"
        file="my_module.my_report" attachment_use="True"
        attachment="'Report-' + (object.name or '').replace('/', '_') + '.pdf'"
        print_report_name="'Report-%s' % (object.name)"/>
```

`convert.py::_tag_report` maps attributes onto `ir.actions.report` fields:
`string`->`name`, `model`->`model`, `name`->`report_name`, `file`->`report_file`,
plus optional `report_type`, `attachment`, `attachment_use`, `usage`, `parser`,
`print_report_name`, `auto`, `header`, `multi`, `groups`, `paperformat`
(`paperformat` -> `paperformat_id`), and `menu`. When `menu` is truthy (the
default), the loader calls `report.create_action()` to register the contextual
"Print" action.

### `<act_window>` Shortcut (ir.actions.act_window)

```xml
<act_window id="action_my_model" name="My Model" res_model="my.model"
            view_mode="tree,form" domain="[('active','=',True)]"
            context="{'default_user_id': uid}"/>
```

Attributes (from `convert.py::_tag_act_window`): `id`, `name`, `res_model`,
`domain`, `context`, `view_id`, `view_mode` (default `tree,form`), `target`,
`groups`, `limit`, `usage`, and the binding attributes `binding_model`,
`binding_type`, `binding_views`. `domain` and `context` are evaluated at load time
with an eval context that includes `uid`, `active_id`, `active_ids`,
`active_model`, `view_id`, `res_model`, `src_model`, `view_mode`, `usage`,
`limit`, `xml_id`, and `name`. Domains that only make sense client-side are kept
as literal strings when they fail to parse server-side.

---

## CSV Data Files

CSV files are the go-to format for flat, bulk data (access rights, translations,
country lookups, etc.). Loading goes through `convert.py::convert_csv_import`,
which derives the model from the file name and then calls `env[model].load(...)`.

### Naming

The model is the file name **before the first `-`** (dots kept as-is). By
convention the file is named `<model_with_dots_as_underscores>.csv`:

| Model | File |
|-------|------|
| `ir.model.access` | `ir.model.access.csv` |
| `res.country.state` | `res.country.state.csv` |

### Structure

- Header row: field names. `id` references the external id.
- Each subsequent row is one record.
- Use `:id` on a column to look up related records by their external id.

```csv
id,country_id:id,name,code
state_us_ca,base.us,California,CA
state_us_ny,base.us,New York,NY
```

### Reference Suffixes

Odoo 13.0 normalises import/export column names via
`models.py::fix_import_export_id_paths`, which rewrites `x.id` to `x/.id` and
`x:id` to `x/id`, then splits on `/`. The supported reference forms for a
relational column are therefore:

| Column | Meaning |
|--------|---------|
| `field` | Match by the comodel's `name_get` / `name_search` (or a literal for non-relational fields) |
| `field:id` (or `field/id`) | Resolve an external id in `ir.model.data` |
| `field.id` (or `field/.id`) | Use the database id directly |

A relational column without any suffix is resolved by `name_search` first; if
`name_create_enabled_fields` is active for that field the loader may
`name_create` a missing record. `field:id`/`field/id` resolve the external id
(module-prefixed with the importing module when the value has no dot);
`field.id`/`field/.id` take a raw database id.

There is **no `:type` suffix** in 13.0 (that form does not exist in this version;
type coercion is always driven by the destination ORM field). Do not write
`amount:float` - the column is just `amount` and the ORM float field handles the
text value.

### Many2many in CSV

A Many2many column is a single cell with comma-separated references:

```csv
id,name,category_id:id
partner_vip,ACME,"customer,vip"
```

The `_str_to_many2many` converter splits the cell on `,` and resolves each
reference with the given suffix. Without `update_many2many` in context the result
replaces the whole link set (`(6, 0, ids)`).

### One2many in CSV

One2many values span rows using `<field>/<subfield>` columns. A row with only the
subfield values (no other values) attaches to the previous record. The convention:

```csv
id,name,line_ids/name,line_ids/price
order_1,Order One,Line A,10
,,Line B,20
```

### Typical Use Cases

#### `ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,my_module.model_my_model,base.group_user,1,0,0,0
access_my_model_manager,my.model.manager,my_module.model_my_model,my_module.group_manager,1,1,1,1
```

#### Simple Master Data

```csv
id,name,code,active
tag_vip,VIP,VIP,True
tag_new,New Customer,NEW,True
```

### CSV vs XML

| CSV | XML |
|-----|-----|
| Simpler for bulk seed data | Supports all features (eval, search, type, relational commands) |
| No Python evaluation | Required for HTML blocks, complex relationships |
| Only `:id` / `.id` reference shorthand | Full external-id + domain search |
| Single model per file | Multiple models per file |

---

## `noupdate` Attribute

`noupdate` controls whether records in a block are refreshed on `-u`
(module update).

```xml
<odoo>
    <!-- Always refreshed on upgrade -->
    <record id="view_x_form" model="ir.ui.view">
        <field name="model">x</field>
        <field name="arch" type="xml">...</field>
    </record>

    <!-- Only inserted once (install), never overwritten -->
    <data noupdate="1">
        <record id="default_company_website" model="res.company">
            <field name="website">https://example.com</field>
        </record>
    </data>
</odoo>
```

### How It Works

- The `<data>`/`<odoo>` root can carry `noupdate="1"`; the loader pushes it onto a
  `_noupdate` stack so it applies to every nested element.
- In update mode, `_tag_record` returns early for a `noupdate`-protected record
  that already exists (it records the existing id in `idref` but does not write).
- The per-record `ir.model.data.noupdate` flag is set from the same value and is
  checked again inside `_load_records`.
- `_update_xmlids` only refreshes the `date_update` timestamp `WHERE NOT
  ir_model_data.noupdate`, so protected XML-ids keep their original timestamp.

### `noupdate` Defaults

`modules/loading.py::load_data` computes the default: `noupdate = False`, except
for demo data (kind `demo`/`demo_xml`) and for `.csv` files loaded in init mode,
where it is `True`.

| Record Kind | `noupdate` |
|-------------|-----------|
| Views, menus, actions, reports | `0` (default) |
| Scheduled jobs users may reconfigure | `1` |
| Default records (sequences, warehouses, journals) | `1` |
| Demo data | `1` (and inside `<data noupdate="1">`) |
| Access-control records (`ir.model.access`) | `0` - keep in sync |
| Record rules | `0` - keep in sync |

### `forcecreate`

Inside a `noupdate="1"` block, a record referenced by id that does **not** exist
is still created on update. To opt out, add `forcecreate="0"` on the `<record>`:

```xml
<data noupdate="1">
    <record id="optional_cron" model="ir.cron" forcecreate="0">
        <field name="name">Optional Job</field>
        ...
    </record>
</data>
```

`_tag_record` checks `nodeattr2bool(rec, 'forcecreate', True)`: when the XML-id
is absent, `forcecreate` is false, and we are not in init mode, the record is
skipped rather than created.

---

## Loading Order & Modes

- Files are loaded in the order declared in `__manifest__.py` (`data:` first,
  then `demo:`; the `test:` key is for test-only files).
- Within a file, operations execute top-to-bottom. A later record can reference
  an earlier external id, but not the other way around.
- Init mode (first install) runs everything, including `<function>` in
  `noupdate="1"` blocks.
- Update mode (`-u my_module`) re-runs every `data:` file but skips
  `noupdate="1"` blocks and `<function>` calls inside them.

### Demo Data

Demo files load only when demo data is enabled. `modules/graph.py::should_have_demo`
requires the package to have demo files (or a pending demo flag) **and** all
parent modules to have loaded demo data. The CLI flag is `--without-demo` (its
absence enables demo on a fresh database). `load_demo` wraps the load in a
`cr.savepoint(flush=False)` and, on failure, records an `ir.demo_failure` instead
of aborting the install. Multiple `<data>` blocks may be freely mixed in one
file, as in the intro example.

---

## Common Pitfalls

### Duplicate XML-ids

Two records with the same `module.name` collide on the unique `(module, name)`
index of `ir.model.data`. `_update_xmlids` uses `ON CONFLICT (module, name)` to
update rather than insert, so a duplicate is not an error - the second load
silently takes over the id. The real risk is reusing the id for a **different
model**: `_load_records` raises a `ValidationError` then. Keep ids unique per
module and keep the model stable.

### noupdate Trap

Two related gotchas. First, records inside `<data noupdate="1">` are never
re-written on upgrade, so edits to their XML values do not reach existing
databases - decide up front whether a record is module-owned (`noupdate="0"`)
or user-owned (`noupdate="1"`). Second, a `<function>` inside a `noupdate="1"`
block only runs on init; put per-upgrade work outside the block.

### eval Syntax

`eval` is `safe_eval`, not full Python: the builtins are a restricted subset, and
attribute names containing `__` are rejected. Expressions must be a single
expression (no statements, no `if`/`for` at statement level - comprehensions and
`lambda` are allowed). `obj(...)` is only available when a `model` attribute is
set on the `<field>`. There is **no `Command` helper** in 13.0; use tuples:

```xml
<field name="category_id" eval="[Command.set([ref('a')])]"/>  <!-- NameError: not 13.0 -->
<field name="category_id" eval="[(6, 0, [ref('a')])]"/>       <!-- correct -->
```
### ref to a Not-Yet-Loaded Record

`ref` resolves immediately against `ir.model.data`. Referencing an id defined
later in the same file (or in a later file of the manifest) fails. Order the
manifest `data` list and the records top-to-bottom so every `ref`/`search`
dependency is loaded first. For a computed value that needs a just-created id,
use the `%(module.xmlid)d` placeholder inside field text instead.

### External ID Dots

`module.name.with.dots` is invalid: `_test_xml_id` asserts at most one dot. And
`<field name="country_id" ref="us"/>` in module `my_module` resolves to
`my_module.us`, not `base.us` - write `ref="base.us"` for cross-module refs. The
same shortening applies to `action=`/`parent=` attributes.

### `type="file"` vs `type="base64"`

Covered in the `<field>` section above: `type="file"` stores the element's text
content as a validated module path; `type="base64"` base64-encodes the bytes
named by the `file=` attribute.

### Whitespace and Numeric Coercion

Inline text keeps leading/trailing whitespace for `char` fields. Numbers and
booleans are coerced by the destination field type, so a Boolean field written as
`<field name="active">1</field>` becomes `True`, and an integer field written as
`<field name="count">01</field>` becomes `1`. Rely on the field type rather than
pre-formatting strings.

---

## Quick Reference

```xml
<!-- record + field -->
<record id="external_id" model="model.name" context="{}" forcecreate="1">
    <field name="char_field">plain string</field>
    <field name="boolean" eval="True"/>
    <field name="m2o_field" ref="module.other_xmlid"/>
    <field name="m2o_field" search="[('code','=','X')]"/>
    <field name="m2m_field" eval="[(6, 0, [ref('a'), ref('b')])]"/>
    <field name="html_field" type="html"><p>HTML content</p></field>
    <field name="binary" type="base64" file="module/static/file.bin"/>
    <field name="file_ref" type="file">module/static/x.png</field>
    <field name="count" type="int">42</field>
</record>

<!-- shortcuts -->
<menuitem id="m1" name="Label" parent="m_root" action="act_x" sequence="10"
          groups="base.group_user" web_icon="module,static/description/icon.png"/>
<template id="tpl" name="My Template" inherit_id="parent.tpl" priority="16">
    <xpath expr="//div" position="inside"><p>Extra</p></xpath>
</template>
<report id="act_rep_x" string="My Report" model="my.model" report_type="qweb-pdf"
        name="my_module.tpl_report_x" file="my_module.report_x"/>
<act_window id="act_x" name="My Model" res_model="my.model" view_mode="tree,form"/>

<!-- delete / function -->
<delete model="res.partner" id="obsolete.legacy_partner"/>
<delete model="ir.ui.menu" search="[('name','=','Old')]"/>
<function model="res.partner" name="unlink_inactive"/>
<function model="my.model" name="post_install_hook" eval="[ref('base.main_company')]"/>
```

Relational commands: `(0,0,{...})` create, `(1,id,{...})` update, `(2,id,0)`
delete, `(3,id,0)` unlink, `(4,id,0)` link, `(5,0,0)` clear, `(6,0,[ids])` replace.

---

## Base Code Reference

- `odoo/tools/convert.py` - XML data loader and all tag handlers
  (`_tag_record`, `_tag_delete`, `_tag_function`, `_tag_menuitem`,
  `_tag_template`, `_tag_report`, `_tag_act_window`), the eval context
  (`_get_idref`), field value resolution (`_eval_xml`), CSV import
  (`convert_csv_import`), and `DATA_ROOTS`.
- `odoo/addons/base/models/ir_model.py` - `ir.model.data` and the external-id
  resolver (`xmlid_lookup`, `xmlid_to_res_model_res_id`, `xmlid_to_res_id`,
  `xmlid_to_object`, `get_object_reference`, `_load_xmlid`, `_update_xmlids`).
- `odoo/models.py` - `BaseModel.load`, `_load_records`, `_extract_records`,
  `_convert_records`, and `fix_import_export_id_paths` (CSV `:id`/`.id` handling).
- `odoo/addons/base/models/ir_fields.py` - `IrFieldsConverter.db_id_for` and the
  `_str_to_*` converters for CSV relational resolution.
- `odoo/tools/safe_eval.py` - the restricted builtins and opcode whitelist behind
  every `eval`/`search`/`domain`.
- `odoo/modules/module.py` - manifest loading and defaults
  (`load_information_from_description_file`), `get_module_resource`.
- `odoo/modules/loading.py` - `load_data` / `load_demo` and the noupdate defaults.
- `odoo/import_xml.rng` - the RelaxNG schema defining valid tags and attributes.
