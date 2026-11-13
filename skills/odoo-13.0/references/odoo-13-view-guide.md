---
name: odoo-13-view
description: Complete reference for Odoo 13 XML views: form, tree, search, kanban, graph, pivot, calendar, gantt, diagram, and QWeb views, attrs and states modifiers, field widgets, view inheritance with xpath positions, actions, menus, and view validation.
globs: "**/views/**/*.xml"
topics:
  - ir.ui.view records and the 13.0 view type selection
  - Form views: sheet, group, notebook, header, buttons, widgets
  - Tree views: editable, decorations, sums, statusbar
  - Search views: filters, group by, filter_domain
  - Kanban views and kanban-box templates
  - Graph, pivot, calendar, gantt, diagram, and QWeb views
  - attrs and states modifiers (the 13.0 mechanism)
  - View inheritance with inherit_id, xpath, and position values
  - Window actions and menus
  - View validation with valid_view and the RNG schemas
when_to_use:
  - Writing or fixing XML views in an Odoo 13 module
  - Conditionally hiding or disabling fields
  - Extending an existing view by inheritance
  - Adding filters, group by options, or kanban cards
  - Diagnosing view validation errors
---

# Odoo 13 View Guide

Complete reference for Odoo 13 XML views. Odoo 13 uses the classic XML view
system: view records are `ir.ui.view` rows, dynamic visibility is expressed
with `attrs` and `states`, and the list view root tag is `<tree>`.

## Table of Contents

1. [View Records](#view-records)
2. [View Types](#view-types)
3. [Form Views](#form-views)
4. [Tree Views](#tree-views)
5. [Search Views](#search-views)
6. [Kanban Views](#kanban-views)
7. [Other View Types](#other-view-types)
8. [Field Widgets](#field-widgets)
9. [attrs and states](#attrs-and-states)
10. [Static Attributes](#static-attributes)
11. [View Inheritance](#view-inheritance)
12. [Actions and Menus](#actions-and-menus)
13. [QWeb Templates](#qweb-templates)
14. [View Validation](#view-validation)
15. [Common Pitfalls](#common-pitfalls)
16. [Base Code Reference](#base-code-reference)

---

## View Records

A view is a regular data record:

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="type">form</field>
    <field name="arch" type="xml">
        <form>
            <field name="name"/>
        </form>
    </field>
</record>
```

Relevant fields on `ir.ui.view` (`odoo/addons/base/models/ir_ui_view.py`):

| Field | Type | Notes |
|-------|------|-------|
| `name` | Char | Human-readable name, not an external id |
| `model` | Char | Model the view applies to |
| `type` | Selection | See [View Types](#view-types) |
| `arch` | Text | Computed/inverse over `arch_db`; the XML is written here |
| `inherit_id` | Many2one | Parent view, `ondelete='restrict'` |
| `mode` | Selection | `primary` or `extension` |
| `priority` | Integer | Default 16; lower values are applied first |
| `groups_id` | Many2many | Restricts the view to groups |
| `active` | Boolean | Default `True` |
| `key` | Char | Required for `qweb` views |
| `website_id` | Many2one | Added by the website module |

The `arch` field is translatable and is read through `_compute_arch`, which
also honours `dev_xml` mode. Writing `arch` in a view record is the normal path.

---

## View Types

The 13.0 selection is fixed in `ir.ui.view.type`:

```python
[('tree', 'Tree'), ('form', 'Form'), ('graph', 'Graph'), ('pivot', 'Pivot'),
 ('calendar', 'Calendar'), ('diagram', 'Diagram'), ('gantt', 'Gantt'),
 ('kanban', 'Kanban'), ('search', 'Search'), ('qweb', 'QWeb')]
```

Notes specific to 13.0:

- The list view tag is `<tree>`, not `<list>`.
- There is no `activity` entry in the selection. Activity views are
  enterprise-side and not a standard 13.0 community view type.
- `qweb` is a server-side template view type, not a client view.

---

## Form Views

```xml
<form string="Order" create="true" edit="true" delete="false">
    <header>
        <button name="action_confirm" type="object" string="Confirm"
                class="oe_highlight" states="draft"/>
        <button name="action_cancel" type="object" string="Cancel"
                states="draft,confirmed"/>
        <field name="state" widget="statusbar"
               statusbar_visible="draft,confirmed,done"/>
    </header>
    <sheet>
        <div class="oe_title">
            <h1><field name="name" placeholder="Reference"/></h1>
        </div>
        <group>
            <group string="Customer">
                <field name="partner_id" domain="[('is_company', '=', True)]"/>
                <field name="date_order"/>
            </group>
            <group string="Amounts">
                <field name="currency_id" invisible="1"/>
                <field name="amount_total"/>
            </group>
        </group>
        <notebook>
            <page string="Lines" name="lines">
                <field name="line_ids">
                    <tree editable="bottom">
                        <field name="product_id"/>
                        <field name="quantity"/>
                        <field name="price_unit"/>
                        <field name="subtotal" sum="Total"/>
                    </tree>
                </field>
            </page>
            <page string="Notes" name="notes">
                <field name="note" placeholder="Internal notes"/>
            </page>
        </notebook>
    </sheet>
</form>
```

Key containers:

| Tag | Purpose |
|-----|---------|
| `<header>` | Status bar and workflow buttons |
| `<sheet>` | White paper area of the form |
| `<div class="oe_title">` | Title block |
| `<group>` | Two-column label/field layout; nested groups split columns |
| `<notebook>` / `<page>` | Tabbed sections; `page name` stabilises inheritance |
| `<separator string="..."/>` | Section separator |
| `<newline/>` | Forces a line break inside a group |

Form-level attributes: `create`, `edit`, `delete`, `duplicate`,
`js_class`, `string`.

Buttons accept `name`, `type` (`object`, `action`), `string`, `class`,
`states`, `attrs`, `groups`, `confirm`, `special`, `icon`, `context`.
Use `class="oe_highlight"` for the primary action and `class="oe_link"` for
secondary ones.

---

## Tree Views

```xml
<tree string="Orders" editable="bottom" create="false" delete="false"
      default_order="date_order desc"
      decoration-muted="state == 'cancel'"
      decoration-danger="state == 'draft' and amount_total &gt; 1000"
      decoration-bf="state == 'done'">
    <field name="name"/>
    <field name="partner_id"/>
    <field name="date_order"/>
    <field name="amount_untaxed" sum="Untaxed"/>
    <field name="amount_total" sum="Total"/>
    <field name="state"/>
</tree>
```

Tree attributes:

| Attribute | Values | Notes |
|-----------|--------|-------|
| `editable` | `top`, `bottom` | Inline editing |
| `create`, `edit`, `delete`, `duplicate`, `import`, `export_xlsx` | `true`/`false` | Toolbar and row actions |
| `multi_edit` | `true`/`false` | 13.0 multi-edit support |
| `default_order` | field list | e.g. `sequence, name desc` |
| `decoration-*` | domain string | `bf`, `it`, `danger`, `info`, `muted`, `primary`, `success`, `warning` |
| `js_class` | string | Custom JS class |
| `limit` | integer | Page size |

Field attributes inside a tree: `sum`, `avg`, `width`, `optional`
(`show`/`hide`), `widget`, `readonly`, `invisible`, `groups`.

`decoration-*` attributes use a plain Python domain string evaluated against
the row, for example `decoration-danger="state == 'cancel'"`. This is the one
place where an expression string is normal in 13.0; it is a decoration domain,
not an `invisible` modifier.

---

## Search Views

```xml
<search string="Orders">
    <field name="name" string="Order"
           filter_domain="['|', ('name', 'ilike', self), ('ref', '=', self)]"/>
    <field name="partner_id"/>
    <filter name="filter_draft" string="Draft" domain="[('state', '=', 'draft')]"/>
    <filter name="filter_my" string="My Orders"
            domain="[('user_id', '=', uid)]"/>
    <separator/>
    <filter name="group_state" string="State"
            context="{'group_by': 'state'}"/>
    <filter name="group_partner" string="Partner"
            context="{'group_by': 'partner_id'}"/>
</search>
```

- `<field>` adds a searchable input. `filter_domain` builds a compound domain;
  `self` is the typed value and `uid` is the current user id.
- `<filter>` with `domain` adds a toggle. With `context="{'group_by': ...}"`
  it adds a Group By entry.
- `<separator/>` groups filters visually.
- `name` is required on filters you intend to inherit or address by xpath;
  never rely on `string` for selectors.

---

## Kanban Views

```xml
<kanban default_group_by="state" default_order="date_order desc"
        class="o_kanban_small_column" quick_create="false" group_create="false">
    <field name="name"/>
    <field name="partner_id"/>
    <field name="amount_total"/>
    <field name="state"/>
    <templates>
        <t t-name="kanban-box">
            <div class="oe_kanban_global_click o_kanban_record_has_image_fill">
                <div class="o_kanban_image">
                    <img t-att-src="kanban_image('res.partner', 'image', record.partner_id.raw_value)"
                         alt="Partner"/>
                </div>
                <div class="oe_kanban_details">
                    <strong class="o_kanban_record_title"><field name="name"/></strong>
                    <div><field name="partner_id"/></div>
                    <div class="text-right"><field name="amount_total"/></div>
                    <t t-if="record.state.raw_value == 'done'">
                        <span class="badge badge-success">Done</span>
                    </t>
                </div>
            </div>
        </t>
    </templates>
</kanban>
```

13.0 facts:

- The card template name is `kanban-box`. (`kanban-card` is a later rename.)
- Every field used in the template must be declared with `<field>` before
  `<templates>`.
- `record.<field>` exposes `raw_value` and `value`; use `raw_value` for
  comparisons and `value` for display.
- `t-esc` escapes and `t-raw` inserts HTML. Prefer `<field>` for model values.
- `oe_kanban_global_click` makes the whole card open the record.
- Kanban attributes: `default_group_by`, `default_order`, `group_create`,
  `group_edit`, `group_delete`, `quick_create`, `class`, `js_class`.

---

## Other View Types

### Graph

```xml
<graph string="Orders" type="bar" stacked="True">
    <field name="partner_id"/>
    <field name="date_order" interval="month"/>
    <field name="amount_total" type="measure"/>
</graph>
```

`type` is `bar`, `line`, or `pie`. Fields default to `row`; mark the
measured field with `type="measure"`.

### Pivot

```xml
<pivot string="Orders">
    <field name="partner_id" type="row"/>
    <field name="state" type="col"/>
    <field name="amount_total" type="measure"/>
</pivot>
```

### Calendar

```xml
<calendar string="Orders" date_start="date_order" date_stop="date_done"
          color="user_id">
    <field name="name"/>
    <field name="partner_id"/>
</calendar>
```

### Gantt

The `gantt` type is part of the 13.0 view selection, but the community web
client ships no gantt renderer: the view is provided by the enterprise
`web_gantt` module. Do not rely on it in a community-only addon.

### Diagram

```xml
<diagram string="Workflow" create="false">
    <node object="my.model" shape="box" bgcolor="#dddddd">
        <field name="name"/>
    </node>
    <arrow object="my.transition" source="state_from" destination="state_to"/>
</diagram>
```

The diagram vocabulary is `node` (`object`, `shape`, `bgcolor`,
`form_view_ref`) and `arrow` (`object`, `source`, `destination`,
`form_view_ref`, `label`). Diagram views are legacy and rarely used in 13.0.

---

## Field Widgets

`widget="..."` selects a client-side renderer. The 13.0 registry
(`addons/web/static/src/js/fields/field_registry.js`) registers:

| Group | Widgets |
|-------|---------|
| Basic | `input`, `integer`, `boolean`, `date`, `datetime`, `daterange`, `domain`, `text`, `html`, `float`, `char`, `email`, `phone`, `url`, `monetary`, `percentage`, `priority`, `image`, `binary`, `pdf_viewer`, `handle`, `link_button`, `label_selection`, `state_selection`, `boolean_favorite`, `boolean_toggle`, `statinfo`, `percentpie`, `float_time`, `float_factor`, `float_toggle`, `progressbar`, `toggle_button`, `ace`, `color`, `many2one_reference` |
| Relational | `selection`, `radio`, `selection_badge`, `many2one`, `many2one_barcode`, `many2many`, `many2many_binary`, `many2many_tags`, `many2many_tags_avatar`, `many2many_checkboxes`, `one2many`, `statusbar`, `reference`, `font` |
| Special | `timezone_mismatch`, `report_layout` |

Common helpers:

```xml
<field name="state" widget="statusbar" statusbar_visible="draft,done"/>
<field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
<field name="partner_id" options="{'no_create': True, 'no_open': True}"/>
<field name="content" widget="html"/>
<field name="file" filename="file_name" widget="binary"/>
<field name="sequence" widget="handle"/>
```

`options` is a JSON-ish dict consumed by the widget. `domain` and `context`
on a relational field restrict and seed the selection:

```xml
<field name="partner_id"
       domain="[('customer', '=', True), ('company_id', 'in', company_id)]"
       context="{'default_customer': True}"/>
```

---

## attrs and states

Odoo 13 evaluates `attrs` and `states` on the server and turns them into
client-side modifiers (`ir_ui_view.transfer_node_to_modifiers`). This is the
canonical 13.0 mechanism for conditional visibility and editability.

```xml
<field name="partner_id"
       attrs="{'invisible': [('state', '=', 'draft')],
               'readonly': [('state', '!=', 'draft')],
               'required': [('type', '=', 'out_invoice')]}"/>
<field name="invoice_date" attrs="{'invisible': [('state', 'in', ['done', 'cancel'])]}"/>
<button name="action_confirm" type="object" string="Confirm"
        attrs="{'invisible': [('state', '!=', 'draft')]}"/>
```

`attrs` is a dict with up to three keys:

| Key | Value |
|-----|-------|
| `invisible` | Domain; the element is hidden when the domain matches |
| `readonly` | Domain; the field is read-only when the domain matches |
| `required` | Domain; the field is required when the domain matches |

`states` is a comma-separated state list, kept for compatibility:

```xml
<field name="date_done" states="done,cancel"/>
<button name="action_done" type="object" string="Done" states="confirmed"/>
```

`states="a,b"` becomes `invisible: [('state', 'not in', ['a', 'b'])]`. It is
equivalent to `attrs="{'invisible': [('state', 'not in', ['a', 'b'])]}"` and
only applies to models that actually have a `state` field.

Do not write field-name expressions in `invisible`/`readonly`/`required`
attributes. `invisible="state == 'done'"` is not a 13.0 construct; the
attribute is evaluated with `safe_eval` against `context` only, so a bare
field name raises a `NameError`. Use `attrs` instead.

---

## Static Attributes

These attributes accept literals and are not domains:

| Attribute | Applies to | Meaning |
|-----------|------------|---------|
| `invisible="1"` | any element | Always hidden (use `groups` for access control) |
| `readonly="1"` | fields | Always read-only |
| `required="1"` | fields | Always required in the client |
| `nolabel="1"` | fields | Hide the label |
| `groups="base.group_user"` | fields, buttons, pages, menus, views | Access restriction |
| `colspan`, `col` | group/label layout | Grid spans |
| `force_save="1"` | fields | Save a read-only computed value |

`groups` is enforced server-side. `invisible="1"` is a UI hint only: the
value is still readable through the ORM unless a record rule or access right
prevents it.

---

## View Inheritance

Inherit an existing view with `inherit_id` and override `arch`:

```xml
<record id="view_my_model_form_inherit" model="ir.ui.view">
    <field name="name">my.model.form.inherit</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="my_module.view_my_model_form"/>
    <field name="priority" eval="20"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='partner_id']" position="after">
            <field name="partner_ref"/>
        </xpath>
        <xpath expr="//field[@name='note']" position="attributes">
            <attribute name="placeholder">Updated placeholder</attribute>
        </xpath>
    </field>
</record>
```

Position values (`odoo/addons/base/rng/common.rng`):

| Position | Effect |
|----------|--------|
| `before` | Insert as previous sibling |
| `after` | Insert as next sibling |
| `inside` | Append inside the matched element |
| `replace` | Replace the matched element |
| `attributes` | Edit attributes with `<attribute name="...">value</attribute>` |

Shortcuts: a top-level element with a `name` and `position` attribute works
like an xpath, for example:

```xml
<field name="partner_id" position="after">
    <field name="partner_ref"/>
</field>
<button name="action_confirm" position="attributes">
    <attribute name="string">Confirm order</attribute>
</button>
```

Guidance:

- Use `name` selectors (`//field[@name='x']`) rather than `string`
  selectors; `string` is translatable and unstable across languages.
- `priority` decides the order sibling extensions apply. Lower runs first.
- `mode="primary"` creates an independent view derived from the parent
  instead of extending it. Use it for a different form of the same model.
- Never match on a field added by another module unless you depend on it.

---

## Actions and Menus

Window actions live in `ir.actions.act_window`:

```xml
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">Orders</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">tree,form</field>
    <field name="domain">[]</field>
    <field name="context">{'search_default_filter_draft': 1}</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create your first order</p>
    </field>
</record>
```

To use explicit views, populate the `view_ids` one2many. The `views` field is
a computed binary in 13.0 and is not writable:

```xml
<field name="view_ids" eval="[(0, 0, {'view_mode': 'tree'}),
                              (0, 0, {'view_mode': 'form', 'view_id': ref('view_my_model_form')})]"/>
```

Menus:

```xml
<menuitem id="menu_my_root" name="My App" sequence="10"/>
<menuitem id="menu_my_orders" name="Orders" parent="menu_my_root"
          action="action_my_model" sequence="10"
          groups="base.group_user"/>
```

See `references/odoo-13-actions-guide.md` for binds, server actions, and crons.

---

## QWeb Templates

Report and website templates are `ir.ui.view` records with `type="qweb"`
and a `key`. In 13.0 they are usually written with the `<template>` shortcut:

```xml
<template id="report_my_model_document" name="My Model Report">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2><span t-field="doc.name"/></h2>
                    <p t-field="doc.partner_id"/>
                </div>
            </t>
        </t>
    </t>
</template>
```

QWeb directives in 13.0: `t-call`, `t-esc`, `t-raw`, `t-set`, `t-if`,
`t-elif`, `t-else`, `t-foreach`, `t-as`, `t-att-*`, `t-field`,
`t-options`, `t-call-assets`, `t-js`, `t-css`, `t-translation`.
There is no `t-out` in 13.0; use `t-esc` for escaped output and `t-raw`
only for trusted HTML.

JavaScript templates live in `static/src/xml/*.xml` and are declared in the
manifest `qweb` key:

```xml
<templates>
    <t t-name="MyWidget">
        <div class="my_widget">
            <span t-esc="widget.title"/>
        </div>
    </t>
</templates>
```

The root tag is `<templates>`, not `<odoo>`.

---

## View Validation

Views are validated at load time by `odoo.tools.view_validation.valid_view`
against the RNG schemas in `odoo/addons/base/rng/`
(`common.rng`, `tree_view.rng`, `search_view.rng`, `graph_view.rng`,
`pivot_view.rng`, `calendar_view.rng`, `activity_view.rng`,
`diagram_view.rng`).

Typical errors and causes:

| Error | Cause |
|-------|-------|
| `Field "x" used in view but not found in model` | Typo, missing dependency, or field not yet defined |
| `Invalid field "x" in tree view` | Field not present on the model |
| `Element <foo> cannot be located in parent view` | xpath does not match; the parent changed |
| `Invalid view type` | `type` not in the 13.0 selection |
| `Field "x" is not a valid field in this model` | Stale inheritance after a model rename |

Validation happens when the view is loaded or when the module is updated, so
run `odoo-bin -u my_module --stop-after-init` after editing views.

---

## Common Pitfalls

1. **Using `<list>` instead of `<tree>`.** The tag is `<tree>` in 13.0.
2. **Expression strings.** `invisible="state == 'done'"` fails; use
   `attrs="{'invisible': [('state', '=', 'done')]}"`.
3. **Missing field declaration in kanban.** Fields used in a kanban template
   must be declared before `<templates>`.
4. **xpath on `string`.** Use `name` attributes for selectors.
5. **Inheriting a view from a module you do not depend on.** Add the dependency
   to `depends`.
6. **Duplicate `name` on filters.** `name` is the stable inheritance anchor.
7. **`groups` on a field is not security.** Use ACLs and record rules for
   access control; `groups` only hides the node.
8. **Editing `arch` in dev mode.** Changes made in the UI developer mode are
   not in source control; reproduce them in XML.
9. **Forgetting `<field name="model">`** on a new view record.
10. **Using a widget that does not exist.** Check the 13.0 registry table above.

---

## Base Code Reference

- `odoo/addons/base/models/ir_ui_view.py` - view records, type selection,
  `transfer_node_to_modifiers` (attrs/states), `locate_node`,
  `apply_inheritance_specs`, `_apply_group`.
- `odoo/addons/base/rng/common.rng` - field, group, label, button, and
  `position` vocabulary.
- `odoo/addons/base/rng/tree_view.rng` - tree attributes including
  `editable`, `decoration-*`, `default_order`.
- `odoo/addons/base/rng/search_view.rng` - search view grammar.
- `odoo/addons/base/rng/graph_view.rng`, `pivot_view.rng`,
  `calendar_view.rng`, `activity_view.rng`, `diagram_view.rng`.
- `odoo/tools/view_validation.py` - `valid_view`.
- `addons/web/static/src/js/fields/field_registry.js` - the 13.0 widget
  registry.
- `addons/web/static/src/js/views/` - tree/form/kanban/search controllers.
- `addons/web/views/webclient_templates.xml` - asset bundle templates.
- `odoo/tools/convert.py` - `<menuitem>`, `<report>`, `<act_window>` and
  `<template>` shortcut tags.
