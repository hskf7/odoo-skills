---
name: odoo-13-reports
description: Complete reference for Odoo 13 QWeb reports covering PDF/HTML/Text report actions, report templates, paper formats, external and internal layouts, report assets, barcodes, translations, and rendering reports from Python and tests.
globs: "**/*.{py,xml}"
topics:
  - QWeb reports (qweb-pdf, qweb-html, qweb-text)
  - ir.actions.report fields and methods
  - report.paperformat records
  - t-call layouts (web.external_layout, web.internal_layout, web.basic_layout)
  - Custom reports with _get_report_values
  - Report assets (web.report_assets_common, web.report_assets_pdf)
  - Barcodes through /report/barcode and the barcode widget
  - Translatable reports (t-lang, t-translation)
  - Rendering reports from Python and in tests
when_to_use:
  - Creating PDF, HTML, or text reports for a model
  - Designing report templates with standard or custom layouts
  - Defining custom paper formats
  - Adding barcodes or QR codes to reports
  - Writing translatable reports
  - Rendering or testing reports from Python
---

# Odoo 13 Reports Guide

Complete reference for Odoo 13 QWeb reports: PDF, HTML, and text reports rendered through QWeb and, for PDF, through `wkhtmltopdf`.

## Table of Contents

1. [Report Basics](#report-basics)
2. [`ir.actions.report` Fields](#iractionsreport-fields)
3. [Report Templates](#report-templates)
4. [Layouts (`t-call`)](#layouts-t-call)
5. [Paper Formats](#paper-formats)
6. [Custom Reports (`_get_report_values`)](#custom-reports-_get_report_values)
7. [Rendering Reports from Python](#rendering-reports-from-python)
8. [Translatable Reports](#translatable-reports)
9. [Barcodes](#barcodes)
10. [Report Assets and Print CSS](#report-assets-and-print-css)
11. [Testing Reports](#testing-reports)
12. [Debugging Reports](#debugging-reports)
13. [Quick Reference](#quick-reference)
14. [Base Code Reference](#base-code-reference)

---

## Report Basics

A QWeb report in Odoo 13 is the combination of:

1. An **`ir.actions.report` record** - the action users trigger; it configures the model, template, paper format, and output type.
2. A **QWeb template** (`ir.ui.view`, usually declared with `<template id="...">`) - the HTML markup rendered for the records.
3. Optionally a Python **`AbstractModel`** named `report.<report_name>` whose `_get_report_values(docids, data)` feeds extra values to the template.

### Report Types (`report_type`)

| Value | Output | How |
|-------|--------|-----|
| `qweb-pdf` | PDF | Renders HTML, then runs `wkhtmltopdf` |
| `qweb-html` | HTML | Served directly (previews and debugging) |
| `qweb-text` | Plain text | Renders QWeb and returns text |

Defined in `odoo/addons/base/models/ir_actions_report.py`:

```python
report_type = fields.Selection([
    ('qweb-html', 'HTML'),
    ('qweb-pdf', 'PDF'),
    ('qweb-text', 'Text'),
], required=True, default='qweb-pdf')
```

### Declaring a Report: The `<report>` Shortcut

The `<report>` tag expands to an `ir.actions.report` record. It does NOT create the template - you still write the `<template>` yourself.

```xml
<odoo>
    <report
        id="action_report_my_model"
        string="My Document"
        model="my.model"
        report_type="qweb-pdf"
        name="my_module.report_my_model_document"
        file="my_module.report_my_model"
        paperformat="my_module.paperformat_my_label"
        print_report_name="'MyDoc-%s' % (object.name or 'draft')"
        attachment="'MyDoc-%s.pdf' % object.name"
        attachment_use="True"
        multi="True"
        groups="base.group_user"
    />

    <template id="report_my_model_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="o">
                <t t-call="web.external_layout">
                    <div class="page">
                        <h2>Document <span t-field="o.name"/></h2>
                    </div>
                </t>
            </t>
        </t>
    </template>
</odoo>
```

Attribute mapping (see `odoo/tools/convert.py`, `_tag_report`):

| Tag attribute | `ir.actions.report` field |
|---------------|---------------------------|
| `string` | `name` |
| `model` | `model` |
| `name` | `report_name` |
| `file` | `report_file` |
| `report_type` | `report_type` |
| `attachment`, `attachment_use` | same-named fields |
| `print_report_name` | `print_report_name` |
| `multi` | `multi` |
| `groups` | `groups_id` |
| `paperformat` | `paperformat_id` |

Note the two 13.0-specific attribute names: `groups` (not `groups_id`) and `paperformat` (not `paperformat_id`). The explicit `<record>` form uses the real field names:

```xml
<record id="action_report_my_model" model="ir.actions.report">
    <field name="name">My Document</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">my_module.report_my_model_document</field>
    <field name="report_file">my_module.report_my_model</field>
    <field name="paperformat_id" ref="my_module.paperformat_my_label"/>
    <field name="binding_model_id" ref="my_module.model_my_model"/>
    <field name="binding_type">report</field>
</record>
```

`binding_model_id` and `binding_type="report"` make the action appear in that model's Print menu. `binding_type` defaults to `'report'` on `ir.actions.report`; `binding_model_id` is inherited from `ir.actions.actions`.

---

## `ir.actions.report` Fields

All fields verified from `odoo/addons/base/models/ir_actions_report.py`:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | Char (translated) | Human-readable name, also shown in the Print menu |
| `type` | Char | Defaults to `'ir.actions.report'` |
| `model` | Char | Target model name (e.g. `"my.model"`) - REQUIRED |
| `model_id` | Many2one `ir.model` | Computed from `model`, searchable |
| `report_type` | Selection | `qweb-html` / `qweb-pdf` / `qweb-text` - REQUIRED, default `qweb-pdf` |
| `report_name` | Char | XML id of the QWeb template (`module.template`) - REQUIRED |
| `report_file` | Char | Path to the main report file; optional and stored |
| `print_report_name` | Char (translated) | Python filename expression; `object` and `time` in scope |
| `attachment` | Char | Python expression for an attachment filename; stores the PDF as an `ir.attachment` |
| `attachment_use` | Boolean | Reuse the stored attachment on later prints |
| `paperformat_id` | Many2one `report.paperformat` | Paper format for this report |
| `groups_id` | Many2many `res.groups` | Groups allowed to run the report |
| `multi` | Boolean | If set, the action is not displayed on the right toolbar of a form view |
| `binding_model_id` | Many2one `ir.model` | Attach the action to a model's menu |
| `binding_type` | Selection | `action` / `report`; defaults to `report` |

### `attachment` vs `attachment_use`

```xml
<field name="attachment">'MyDoc-%s.pdf' % object.name</field>
<field name="attachment_use" eval="True"/>
```

Behavior verified in `ir_actions_report.py`:

- `retrieve_attachment(record)` evaluates `attachment` with `safe_eval(..., {'object': record, 'time': time})` and searches `ir.attachment` by `name`, `res_model`, and `res_id`.
- `postprocess_pdf_report(record, buffer)` creates an `ir.attachment` with `name`, base64 `datas`, `res_model`, `res_id`, and `type='binary'`.
- With `attachment_use=True`, records that already have a matching attachment are not re-rendered; the stored PDF is reused.
- For several records, the PDF is split per record using the PDF outlines, so each document must start with a heading. Splitting also requires `data-oe-model` and `data-oe-id` on the `article` element, otherwise `render_qweb_pdf` raises a `UserError`.

### `print_report_name`

```xml
<field name="print_report_name">'MyDoc - %s' % (object.name or 'Draft')</field>
<field name="print_report_name">'SO-%s-%s' % (object.name, object.state)</field>
```

Variables available: `object` (the current record) and `time` (the Python `time` module). The expression applies only when a single record is printed; otherwise the report name is used.

### Main Methods

| Method | Returns | Notes |
|--------|---------|-------|
| `render_qweb_pdf(res_ids, data=None)` | `(pdf_bytes, 'pdf')` | Runs `wkhtmltopdf`; test mode falls back to HTML unless `force_report_rendering` is in context |
| `render_qweb_html(docids, data=None)` | `(html_bytes, 'html')` | Renders the template only |
| `render_qweb_text(docids, data=None)` | `(text_bytes, 'text')` | Renders the template as text |
| `render(res_ids, data=None)` | tuple | Dispatches on `report_type` |
| `report_action(docids, data=None, config=True)` | dict | Builds an `ir.actions.report` action |
| `_get_report_from_name(report_name)` | recordset | First report whose `report_name` matches |
| `barcode(barcode_type, value, **kwargs)` | PNG bytes | Reportlab barcode rendering |
| `get_paperformat()` | `report.paperformat` | `paperformat_id` or the company's paper format |
| `get_wkhtmltopdf_state()` | string | `install`, `ok`, `upgrade`, `workers`, or `broken` |

In 13.0 these methods have no leading underscore: `render_qweb_pdf`, not `_render_qweb_pdf`. The underscore-prefixed names are a later-version change and are not valid 13.0 API.

---

## Report Templates

### Minimal Skeleton

```xml
<template id="report_my_model_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2 t-field="o.name"/>
                </div>
            </t>
        </t>
    </t>
</template>
```

- `web.html_container` sets up the HTML document and includes report assets.
- `web.external_layout` wraps content with the company header and footer.
- `div.page` is the conventional content wrapper for one document.

### Variables Available in the Template

| Variable | Description |
|----------|-------------|
| `docs` | Recordset of the documents being rendered |
| `doc_ids`, `doc_model` | List of IDs and model name string |
| `data` | Data dict passed to the renderer |
| `report_type` | `'html'`, `'pdf'`, or `'text'` |
| `user`, `res_company` | The printing user and the user's current company |
| `company` | Set by the layout (multi-company aware) |
| `time`, `datetime` | Python `time` and `datetime` modules |
| `context_timestamp` | Converts a UTC datetime to the user timezone |
| `web_base_url` | Base URL from `web.base.url` |
| `image_data_uri` | Converts a base64 image to a `data:` URI |
| `env`, `request` | The environment and the current request, if any |

`render_template()` in `ir_actions_report.py` adds `time`, `context_timestamp`, `user`, `res_company`, `website`, and `web_base_url`; `ir.ui.view._prepare_qcontext()` adds `env`, `datetime`, `image_data_uri`, `request`, and the other view values.

### Core QWeb Directives

```xml
<!-- Loop and condition -->
<t t-foreach="docs" t-as="o">...</t>
<p t-if="o.state == 'draft'">DRAFT</p>
<p t-elif="o.state == 'posted'">Posted</p>
<p t-else="">Other state</p>

<!-- Field rendering (dates, currency, selection labels, relations) -->
<span t-field="o.date_order"/>
<span t-field="o.amount_total"
      t-options='{"widget": "monetary", "display_currency": o.currency_id}'/>

<!-- Escaped and unescaped values -->
<span t-esc="o.name"/>
<div t-raw="o.note"/>

<!-- Call a template, set a variable, bind attributes -->
<t t-call="web.external_layout">...</t>
<t t-set="subtotal" t-value="sum(l.price_subtotal for l in o.line_ids)"/>
<div t-att-class="'text-danger' if o.state == 'cancel' else 'text-success'"/>
<a t-attf-href="{{ web_base_url }}/web#id={{ o.id }}&amp;model={{ doc_model }}">Open</a>

<!-- Translation language (on t-call) -->
<t t-call="my_module.body" t-lang="o.partner_id.lang"/>
```

### `t-esc` vs `t-raw` vs `t-field`

| Directive | Escapes HTML | Uses field metadata | Typical use |
|-----------|--------------|---------------------|-------------|
| `t-esc` | Yes | No (unless `t-options` widget) | Computed strings and numbers |
| `t-raw` | No | No (unless `t-options` widget) | HTML fields and pre-rendered markup |
| `t-field` | Field-dependent | Yes | Record fields: dates, money, selections, relations, HTML |

`t-field` restrictions verified in `qweb.py`:

- It cannot be used on a `<t>` element.
- It cannot be used on `table`, `tbody`, `thead`, `tfoot`, `tr`, `td`, `li`, `ul`, `ol`, `dl`, `dt`, or `dd`.
- The expression must contain a dot, e.g. `o.name`.

`t-out` does not exist in 13.0. Use `t-esc` for escaped output and `t-raw` for unescaped output.

### `t-options` and Widgets

`t-options` passes a dict to the field or value converter; `t-options-xxx` attributes are merged into the same dict.

```xml
<span t-field="o.amount_total"
      t-options='{"widget": "monetary", "display_currency": o.currency_id}'/>

<span t-field="o.date_order" t-options='{"format": "yyyy-MM-dd"}'/>

<span t-field="o.name"
      t-options='{"widget": "barcode", "symbology": "Code128", "width": 300, "height": 80}'/>
```

Available converters in 13.0 (`ir_qweb_fields.py`): `integer`, `float`, `date`, `datetime`, `text`, `selection`, `many2one`, `many2many`, `html`, `image`, `monetary`, `float_time`, `duration`, `relative`, `barcode`, `contact`, `qweb`.

Monetary options: `display_currency`, `from_currency`, `date`, `company_id`. For a Monetary field with a `currency_field`, the currency is taken from the record automatically. Barcode options: `symbology` (default `Code128`), `width` (600), `height` (100), `humanreadable` (0), `quiet` (1), plus `img_*` attributes copied to the generated `<img>`.

### `t-foreach` Loop Variables

For `t-foreach="collection" t-as="item"`, 13.0 exposes `item` (current item), `item_index`, `item_size`, `item_first`, `item_last`, `item_parity` (`'odd'`/`'even'`), `item_even`, `item_odd`, and `item_value` (the value when iterating a mapping).

An integer is iterated as `range(value)`. A mapping is iterated as `items()`, so `item` is the key and `item_value` is the value. Tuple unpacking in `t-as` is not supported in 13.0; iterate `dict.items()` and index the tuple, or use `item_value`.

### `t-set`, `t-call`, and Slots

```xml
<!-- Value form -->
<t t-set="total" t-value="sum(line.price_subtotal for line in o.line_ids)"/>

<!-- Body form: the rendered body becomes the value -->
<t t-set="address_block">
    <div class="address"><span t-field="o.partner_id.name"/></div>
</t>

<!-- Pass content to a called template; the callee renders it with t-raw="0" -->
<t t-call="my_module.report_header"><span t-field="o.name"/></t>
```

`t-call` accepts `t-lang` to switch the translation language for the called template.

---

## Layouts (`t-call`)

Odoo 13 provides these layout templates in `addons/web/views/report_templates.xml`:

| Template | Purpose |
|----------|---------|
| `web.html_container` | Top-level wrapper; always the first `t-call` |
| `web.report_layout` | Private HTML document skeleton used by `html_container` |
| `web.external_layout` | Outer report layout with company header and footer |
| `web.external_layout_standard` | Default standard layout |
| `web.external_layout_background` | Background layout variant |
| `web.external_layout_boxed` | Boxed layout variant |
| `web.external_layout_clean` | Clean layout variant |
| `web.internal_layout` | Minimal layout for internal documents |
| `web.basic_layout` | Wrapper that calls `html_container` and adds an `article` |
| `web.address_layout` | Address and information block used by external layouts |
| `web.minimal_layout` | Used internally by the PDF pipeline; do not call it from report templates |

13.0 has exactly the four external layout variants above. `web.external_layout_bold` and `web.external_layout_striped` do not exist in 13.0.

### Recipe: External Layout

```xml
<template id="report_my_model_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2>Document <span t-field="o.name"/></h2>
                    <p><strong>Partner:</strong> <span t-field="o.partner_id"/></p>
                    <table class="table table-sm o_main_table">
                        <thead>
                            <tr><th>Product</th><th class="text-right">Quantity</th></tr>
                        </thead>
                        <tbody>
                            <tr t-foreach="o.line_ids" t-as="line">
                                <td><span t-field="line.product_id"/></td>
                                <td class="text-right"><span t-field="line.product_uom_qty"/></td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </t>
        </t>
    </t>
</template>
```

### Recipe: Internal and Basic Layouts

```xml
<template id="report_internal_checklist">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="web.internal_layout">
                <div class="page">
                    <h3>Checklist: <t t-esc="o.name"/></h3>
                    <ul><li t-foreach="o.check_ids" t-as="c" t-esc="c.name"/></ul>
                </div>
            </t>
        </t>
    </t>
</template>

<!-- basic_layout already calls html_container; do not wrap it again -->
<template id="report_my_model_basic">
    <t t-call="web.basic_layout">
        <div class="page"><h2 t-field="o.name"/></div>
    </t>
</template>
```

`web.internal_layout` renders a header with the current timestamp, the company name, and page numbers, then an `article` containing your content.

### How `web.external_layout` Chooses a Layout

Verified in `report_templates.xml`:

1. If `o` is not set, it is set to `doc`.
2. If `company` is not set, it is resolved from `company_id`, then `o.company_id.sudo()`, then `res_company`.
3. If `company.external_report_layout_id` is set, the layout calls `company.external_report_layout_id.key`.
4. Otherwise it calls `web.external_layout_standard`.

### Custom External Layout

Define a template with `header`, `article`, and `footer` elements, register it as a `report.layout`, and select it on the company.

```xml
<template id="external_layout_my_company">
    <div class="header">
        <div class="row">
            <div class="col-6">
                <img t-if="company.logo" t-att-src="image_data_uri(company.logo)" alt="Logo"/>
            </div>
            <div class="col-6 text-right"><span t-field="company.name"/></div>
        </div>
    </div>

    <div class="article"
         t-att-data-oe-model="o and o._name"
         t-att-data-oe-id="o and o.id"
         t-att-data-oe-lang="o and o.env.context.get('lang')">
        <t t-raw="0"/>
    </div>

    <div class="footer">
        <div class="text-center">
            <span t-field="company.report_footer"/>
            <div t-if="report_type == 'pdf'">
                Page: <span class="page"/> / <span class="topage"/>
            </div>
        </div>
    </div>
</template>

<record id="report_layout_my_company" model="report.layout">
    <field name="name">My Company</field>
    <field name="view_id" ref="my_module.external_layout_my_company"/>
</record>
```

Then set `company.external_report_layout_id` to `my_module.external_layout_my_company` (for example from the Document Layout wizard).

The `data-oe-model`, `data-oe-id`, and `data-oe-lang` attributes on the `article` are not decorative: the PDF pipeline uses them to split a multi-record PDF into per-record attachments and to select the language of the header and footer.

### How the PDF Pipeline Uses the Layout

`_prepare_html(html)` in `ir_actions_report.py` parses the rendered HTML, extracts every element with class `header` into a header container, every element with class `footer` into a footer container, and every element with class `article` as one PDF body. It reads `data-report-*` attributes from the root `<html>` element as per-render paper format overrides, then renders `web.minimal_layout` for the header and footer and one body per article.

This is why custom layouts must use the `header`, `article`, and `footer` class names.

---

## Paper Formats

`report.paperformat` is defined in `odoo/addons/base/models/report_paperformat.py`.

| Field | Type / values | Default |
|-------|---------------|---------|
| `name` | Char, required | - |
| `default` | Boolean | False |
| `format` | Selection: `A0`-`A9`, `B0`-`B10`, `C5E`, `Comm10E`, `DLE`, `Executive`, `Folio`, `Ledger`, `Legal`, `Letter`, `Tabloid`, `custom` | `A4` |
| `margin_top` | Float (mm) | 40 |
| `margin_bottom` | Float (mm) | 20 |
| `margin_left` | Float (mm) | 7 |
| `margin_right` | Float (mm) | 7 |
| `page_height`, `page_width` | Integer (mm) | False |
| `orientation` | `Landscape` / `Portrait` | `Landscape` |
| `header_line` | Boolean | False |
| `header_spacing` | Integer (mm) | 35 |
| `dpi` | Integer, required | 90 |
| `report_ids` | One2many `ir.actions.report` | - |
| `print_page_width`, `print_page_height` | Float, computed | - |

The model default orientation is `Landscape`; the standard A4 and US Letter records explicitly set `Portrait`. A constraint forbids setting `page_width` or `page_height` when `format` is not `custom`.

### Standard Formats Shipped by Odoo

From `odoo/addons/base/data/report_paperformat_data.xml`:

| XML id | Name | Format | Orientation | Top / Bottom | Left / Right | Header spacing | DPI |
|--------|------|--------|-------------|--------------|--------------|----------------|-----|
| `base.paperformat_euro` | A4 | `A4` | Portrait | 40 / 28 | 7 / 7 | 35 | 90 |
| `base.paperformat_us` | US Letter | `Letter` | Portrait | 40 / 25 | 7 / 7 | 35 | 90 |
| `base.paperformat_batch_deposit` | US Batch Deposit | `Letter` | Portrait | 15 / 10 | 10 / 10 | 0 | 90 |

### Custom Paper Format

```xml
<record id="paperformat_my_label" model="report.paperformat">
    <field name="name">Label 100x150</field>
    <field name="default" eval="False"/>
    <field name="format">custom</field>
    <field name="page_height">150</field>
    <field name="page_width">100</field>
    <field name="orientation">Portrait</field>
    <field name="margin_top">2</field>
    <field name="margin_bottom">2</field>
    <field name="margin_left">2</field>
    <field name="margin_right">2</field>
    <field name="header_line" eval="False"/>
    <field name="header_spacing">2</field>
    <field name="dpi">200</field>
</record>
```

### Attaching and Resolving

```xml
<record id="action_report_my_label" model="ir.actions.report">
    <field name="name">My Label</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">my_module.report_my_label</field>
    <field name="paperformat_id" ref="my_module.paperformat_my_label"/>
</record>
```

Resolution order in `get_paperformat()`: `report.paperformat_id`, then `env.company.paperformat_id`. `res.company.paperformat_id` defaults to `base.paperformat_euro`, so a company normally always has a paper format. If both are empty, `_build_wkhtmltopdf_args` receives an empty recordset and passes no page-size or margin arguments to `wkhtmltopdf`.

### How the Paper Format Reaches `wkhtmltopdf`

`_build_wkhtmltopdf_args` maps the record to command-line arguments:

- `format` (when not `custom`) becomes `--page-size`.
- `page_width` and `page_height` (when `format='custom'`) become `--page-width` and `--page-height` in mm.
- `margin_top`, `margin_bottom`, `margin_left`, `margin_right` become the matching `--margin-*` arguments.
- `dpi` becomes `--dpi`; with `wkhtmltopdf >= 0.12.2` Odoo also adds `--zoom 96/dpi`.
- `header_spacing` becomes `--header-spacing`.
- `orientation` becomes `--orientation` unless landscape is forced.
- `header_line` adds `--header-line`.

The renderer also honors `data-report-margin-top`, `data-report-header-spacing`, `data-report-dpi`, and `data-report-landscape` attributes on the root `<html>` element; these per-render values take priority over the paper format record. `web.report_layout` exposes them through the `data_report_margin_top`, `data_report_header_spacing`, `data_report_dpi`, and `data_report_landscape` values. The `landscape` and `set_viewport_size` context keys are also read by `render_qweb_pdf`.

---

## Custom Reports (`_get_report_values`)

When the template needs more than `docs` (pre-computed totals, grouped data, helper functions), define an `AbstractModel` named `report.<report_name>`.

```python
# my_module/report/report_my_model.py
from odoo import api, models


class ReportMyModel(models.AbstractModel):
    _name = 'report.my_module.report_my_model_document'
    _description = 'My Model Report'

    @api.model
    def _get_report_values(self, docids, data=None):
        docs = self.env['my.model'].browse(docids)
        totals = {
            doc.id: {
                'line_count': len(doc.line_ids),
                'subtotal': sum(line.price_subtotal for line in doc.line_ids),
            }
            for doc in docs
        }
        return {
            'doc_ids': docids,
            'doc_model': 'my.model',
            'docs': docs,
            'data': data or {},
            'totals': totals,
        }
```

Register it by importing the module from `report/__init__.py` and importing `report` from the addon `__init__.py`. Use the extra values in the template:

```xml
<template id="report_my_model_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2>Document <span t-field="o.name"/></h2>
                    <p>Lines: <t t-esc="totals[o.id]['line_count']"/></p>
                </div>
            </t>
        </t>
    </t>
</template>
```

The model name MUST be `report.<report_name>`, where `<report_name>` is exactly the `report_name` field of the `ir.actions.report` record. For example, `report_name = 'my_module.report_my_model_document'` requires `_name = 'report.my_module.report_my_model_document'`. If no such model exists, `_get_rendering_context` falls back to the generic values `doc_ids`, `doc_model`, and `docs`.

---

## Rendering Reports from Python

### Building an Action

```python
records = self.env['my.model'].browse(ids)
action = self.env.ref('my_module.action_report_my_model').report_action(records)
# action['type'] == 'ir.actions.report'
# action['report_name'] == 'my_module.report_my_model_document'
# action['report_type'] == 'qweb-pdf'
```

`report_action` accepts a recordset, an integer, or a list of ids. It puts the ids in `context['active_ids']`.

13.0 quirk: for an admin user whose company has no `external_report_layout_id`, `report_action(..., config=True)` returns the Document Layout configurator action instead of the report action. Pass `config=False` to bypass that check:

```python
action = report.report_action(records, config=False)
```

### Rendering Directly

```python
report = self.env.ref('my_module.action_report_my_model')

html, output_type = report.render_qweb_html(records.ids)   # ('html', bytes)
pdf, output_type = report.render_qweb_pdf(records.ids)     # ('pdf', bytes)
text, output_type = report.render_qweb_text(records.ids)   # ('text', bytes)

# Look up by template name
report = self.env['ir.actions.report']._get_report_from_name(
    'my_module.report_my_model_document'
)

# Pass extra data; available as `data` in the template and to _get_report_values
html, _ = report.render_qweb_html(records.ids, data={'custom_flag': True})
```

### PDF Rendering Details

`render_qweb_pdf`:

1. Sets `data['report_type'] = 'pdf'`.
2. In test mode (`test_enable` or `test_file`) without `force_report_rendering` in context, returns `render_qweb_html` instead.
3. With a `TestCursor`, also returns the HTML version.
4. Otherwise renders HTML, splits it into bodies, headers, and footers, and runs `wkhtmltopdf`.
5. Returns `(pdf_content, 'pdf')`.

`wkhtmltopdf` must be at least 0.12.0. Odoo logs `upgrade` when it is older, `install` when it is missing, `broken` when it does not respond, and `workers` when the server runs with fewer than two workers.

---

## Translatable Reports

### `t-lang` on `t-call`

`t-lang` switches the translation context for a called template.

```xml
<!-- Wrapper: loop and delegate per language -->
<template id="report_my_model">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="my_module.report_my_model_document" t-lang="o.partner_id.lang or user.lang"/>
        </t>
    </t>
</template>

<!-- Document: rendered in the requested language -->
<template id="report_my_model_document">
    <t t-set="o" t-value="o.with_context(lang=o.partner_id.lang)"/>
    <t t-call="web.external_layout">
        <div class="page">
            <h2>Document <span t-field="o.name"/></h2>
        </div>
    </t>
</template>
```

Rules:

- `t-lang` is meaningful on `t-call`; it sets the language option for the called template.
- Re-browse the record with `o.with_context(lang=...)` so translatable Char, Text, and Selection values render in the target language.
- Static strings in the template are extracted into `.po` files like any other module content.

### Per-Document Language in PDFs

The standard layouts put `data-oe-lang` on the `article` element:

```xml
<div class="article"
     t-att-data-oe-model="o and o._name"
     t-att-data-oe-id="o and o.id"
     t-att-data-oe-lang="o and o.env.context.get('lang')">
    <t t-raw="0"/>
</div>
```

`_prepare_html` reads `data-oe-lang` and renders the header and footer for that body in the matching language, so a multi-document PDF can mix languages. To force the header and footer into a fixed language while keeping the body in the partner's language, put `t-lang="'en_US'"` on the `web.external_layout` call and re-browse the body record inside it.

### Excluding Content and Exporting Translations

```xml
<div t-translation="off">Do not translate this</div>
```

This is honored by the translation extractor in `odoo/tools/translate.py`. 13.0 uses the `ir.translation` model and the standard CLI options:

```bash
odoo-bin -d mydb -l fr_FR --i18n-export=fr.po --modules=my_module
odoo-bin -d mydb -l fr_FR --i18n-import=fr.po --modules=my_module
odoo-bin -d mydb -l fr_FR --i18n-import=fr.po --i18n-overwrite --modules=my_module
```

Options verified in `odoo/tools/config.py`: `--i18n-export`, `--i18n-import`, `--i18n-overwrite`, `-l/--language`, and `--modules`. Report action names are translatable because `name` and `print_report_name` are declared with `translate=True`.

---

## Barcodes

Odoo 13 serves barcode images through the controller in `addons/web/controllers/main.py`:

```python
@http.route(['/report/barcode', '/report/barcode/<type>/<path:value>'],
            type='http', auth="public")
def report_barcode(self, type, value, **kwargs):
    ...
```

### URL Forms

```xml
<!-- Short path form -->
<img t-att-src="'/report/barcode/QR/%s' % o.name"/>
<img t-att-src="'/report/barcode/Code128/%s' % o.default_code"/>

<!-- Query-string form; in 13.0 the parameter is type, not barcode_type -->
<img t-att-src="'/report/barcode/?type=%s&amp;value=%s&amp;width=%s&amp;height=%s' % (
    'Code128', o.name, 300, 80
)"/>
```

Remember to escape `&` as `&amp;` in XML.

### Supported Types and Parameters

From the `report_barcode` docstring and `ir.actions.report.barcode()`:

- Linear: `Codabar`, `Code11`, `Code128`, `EAN8`, `EAN13`, `Extended39`, `Extended93`, `FIM`, `I2of5`, `MSI`, `POSTNET`, `Standard39`, `Standard93`, `UPCA`, `USPS_4State`
- 2D: `QR`
- Heuristic: `auto` (8 characters -> `EAN8`, 13 characters -> `EAN13`, otherwise `Code128`)

`UPCA` values of 11, 12, or 13 characters are converted to `EAN13` internally.

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `type` | string | - | Barcode type |
| `value` | string | - | Data to encode |
| `width` | int | 600 | Pixel width |
| `height` | int | 100 | Pixel height |
| `humanreadable` | 0 / 1 | 0 | Adds readable text under the barcode |
| `quiet` | 0 / 1 | 1 | White margins on the sides |
| `barBorder` | int | 4 | QR border size |
| `barLevel` | `L` / `M` / `Q` / `H` | `L` | QR error correction level |

For `QR`, `quiet` is ignored; `barBorder` is used instead.

### Barcode Widget and Reportlab

The `barcode` QWeb widget renders a `data:` URI image directly, without an HTTP request:

```xml
<span t-field="o.name"
      t-options='{"widget": "barcode", "symbology": "Code128", "width": 300, "height": 80}'/>
```

`ir.actions.report.barcode()` calls `reportlab.graphics.barcode.createBarcodeDrawing(barcode_type, value=value, format='png', **kwargs)` and returns PNG bytes. Odoo initializes the reportlab T1 font cache at startup to avoid a thread-safety lock when several barcodes are rendered concurrently.

---

## Report Assets and Print CSS

### Bundles

13.0 has no `assets` key in the manifest. Report assets are QWeb templates:

| Bundle | Contents |
|--------|----------|
| `web.report_assets_common` | Report SCSS (report, layout variants, company styles), fonts, Font Awesome, report JS |
| `web.report_assets_pdf` | `reset.min.css` for the PDF renderer |

`web.report_layout` loads them in the document head:

```xml
<t t-call-assets="web.report_assets_common" t-js="false"/>
<t t-call-assets="web.assets_common" t-css="false"/>
<t t-call-assets="web.report_assets_common" t-css="false"/>
```

`web.minimal_layout`, used for PDF headers and footers, loads both bundles. `t-call-assets` supports `t-css` (default true), `t-js` (default true), `async_load`, `defer_load`, and `lazy_load`, and cannot contain child nodes.

### Adding Custom Report CSS

Inherit `web.report_assets_common` in XML:

```xml
<template id="report_assets_common" inherit_id="web.report_assets_common">
    <xpath expr="." position="inside">
        <link rel="stylesheet" type="text/css"
              href="/my_module/static/src/css/report.css"/>
    </xpath>
</template>
```

Add the XML file to the manifest `data` list. Do not use `web.assets_backend` for report styles: the PDF renderer only loads the report bundles.

### Print CSS Notes

- `wkhtmltopdf` has limited support for `@media print`; Odoo's own report SCSS avoids relying on it.
- Use `page-break-inside: avoid` on table rows and blocks that must not split.
- `data-pagebreak="before"` or `data-pagebreak="after"` on a `<tr>` is handled by `ir.qweb`: the table is split and a `page-break-after: always` div is inserted.
- The `header`, `article`, and `footer` classes are structural for the PDF pipeline, not just styling hooks.
- `div.page` is the conventional content wrapper; it does not by itself force a page break.
- `wkhtmltopdf` is invoked with `--disable-local-file-access`; assets must be reachable through the server URL.
- The PDF renderer sets `debug=False` in the context so asset bundles are not split into separate files, which avoids missing CSS or JavaScript in headers and footers.

`web.report_assets_common` includes `report.scss`, the four `layout_*.scss` files, `asset_styles_company_report.scss`, and `fonts.scss`. Company colors and fonts are generated by `web.styles_company_report` from `company.font`, `company.primary_color`, `company.secondary_color`, and `company.external_report_layout_id`.

---

## Testing Reports

### `force_report_rendering`

In test mode, `render_qweb_pdf` returns the HTML rendering instead of invoking `wkhtmltopdf`:

```python
if (tools.config['test_enable'] or tools.config['test_file']) \
        and not self.env.context.get('force_report_rendering'):
    return self.render_qweb_html(res_ids, data=data)
```

To exercise the real PDF path in a test, set the context key:

```python
pdf, output_type = report.with_context(force_report_rendering=True).render_qweb_pdf(records.ids)
self.assertEqual(output_type, 'pdf')
self.assertTrue(pdf.startswith(b'%PDF'))
```

Without `force_report_rendering`, the same call returns HTML bytes and `output_type == 'html'`. Tests that only assert content should use `render_qweb_html` directly; tests that assert PDF structure need `force_report_rendering` and a working `wkhtmltopdf`.

### Example Test

```python
from odoo.tests.common import TransactionCase


class TestMyReport(TransactionCase):

    def setUp(self):
        super(TestMyReport, self).setUp()
        self.record = self.env['my.model'].create({'name': 'Test Document'})
        self.report = self.env.ref('my_module.action_report_my_model')

    def test_html_rendering(self):
        html, output_type = self.report.render_qweb_html(self.record.ids)
        self.assertEqual(output_type, 'html')
        self.assertIn(b'Test Document', html)

    def test_pdf_rendering(self):
        pdf, output_type = self.report.with_context(
            force_report_rendering=True
        ).render_qweb_pdf(self.record.ids)
        self.assertEqual(output_type, 'pdf')
        self.assertTrue(pdf.startswith(b'%PDF'))
```

When `self.env.cr` is a `TestCursor`, `render_qweb_pdf` returns the HTML version even with `force_report_rendering`, because the test cursor cannot safely open another environment while the transaction is open. Use `render_qweb_html` for content assertions in that situation.

To test `attachment` and `attachment_use`, render once and then search `ir.attachment` by the expected `name`, `res_model`, and `res_id`.

---

## Debugging Reports

### Preview URLs

| URL | Description |
|-----|-------------|
| `/report/html/<report_name>/<id>` | HTML preview, exactly as the PDF renderer sees it |
| `/report/pdf/<report_name>/<id>` | PDF download |
| `/report/text/<report_name>/<id>` | Text output |
| `/report/barcode/<type>/<value>` | Barcode PNG |
| `/report/barcode/?type=...&value=...&width=...&height=...` | Barcode PNG with options |

Example: `/report/html/my_module.report_my_model_document/42`. The HTML preview does not include the header and footer injected by the PDF pipeline; that is expected.

### `wkhtmltopdf` State

`self.env['ir.actions.report'].get_wkhtmltopdf_state()` returns `install`, `ok`, `upgrade`, `workers`, or `broken`.

### Common Pitfalls

- **Blank PDF**: missing `web.html_container`, or content outside `div.page`.
- **Styles missing in PDF**: CSS added to `web.assets_backend` instead of `web.report_assets_common`.
- **Report template not found**: `report_name` does not match the `<template id="...">` including the module prefix.
- **`object` undefined in `print_report_name`**: the expression is evaluated with `object` and `time`, not `obj` or `record`.
- **Attachment split error**: the `article` element lacks `data-oe-model` and `data-oe-id`.
- **Header or footer missing**: the layout does not use the `header` and `footer` class names.
- **Barcode 404 or error**: the query parameter is `type` in 13.0; `barcode_type` is not the 13.0 parameter name.
- **`wkhtmltopdf` too old**: 13.0 requires at least 0.12.0.
- **Fewer than two workers**: PDF rendering logs the `workers` state and may fail in production.

---

## Quick Reference

### Module Layout

```text
my_module/
|-- __manifest__.py
|-- __init__.py
|-- models/my_model.py
|-- report/
|   |-- __init__.py
|   |-- report_my_model.py              # AbstractModel (optional)
|   |-- report_my_model_templates.xml   # <report> and <template>
|   `-- paperformat_data.xml            # Custom report.paperformat (optional)
`-- static/src/css/report.css
```

Add the report XML files to the manifest `data` list. There is no `assets` key in a 13.0 manifest; report assets are added by inheriting `web.report_assets_common` in XML.

### QWeb Cheat Sheet

| Directive | Usage |
|-----------|-------|
| `t-call="web.html_container"` | Report wrapper (mandatory) |
| `t-call="web.external_layout"` | Header, footer, company branding |
| `t-call="web.internal_layout"` | Minimal internal layout |
| `t-call="web.basic_layout"` | HTML container plus article wrapper |
| `t-foreach="docs" t-as="o"` | Iterate records |
| `t-if` / `t-elif` / `t-else` | Conditionals |
| `t-field="o.field"` | Field-aware rendering |
| `t-esc="expr"` | Escaped output |
| `t-raw="expr"` | Unescaped output |
| `t-set="x" t-value="..."` | Local variable |
| `t-options='{"widget": "monetary", ...}'` | Field or value options |
| `t-lang="'en_US'"` | Translation language on `t-call` |
| `t-attf-href="{{ web_base_url }}/..."` | Templated attribute |
| `t-translation="off"` | Exclude a subtree from translation |

### 13.0 API Notes

- Report methods are `render_qweb_pdf`, `render_qweb_html`, and `render_qweb_text`; the underscore-prefixed variants are not 13.0 API.
- The barcode query parameter is `type`; `barcode_type` is not the 13.0 parameter name.
- `t-out` does not exist in 13.0; use `t-esc` or `t-raw`.
- `web.external_layout_bold` and `web.external_layout_striped` do not exist in 13.0.
- There is no `assets` manifest key and no `ir.asset` model in 13.0.
- The `<report>` tag uses `groups` and `paperformat`, not `groups_id` and `paperformat_id`.

---

## Base Code Reference

The APIs and templates documented here were verified against the Odoo 13.0 source:

- `odoo/addons/base/models/ir_actions_report.py` - `ir.actions.report` fields, `render_qweb_pdf`, `render_qweb_html`, `render_qweb_text`, `render`, `report_action`, `_get_report_from_name`, `_get_rendering_context`, `_get_report_values` dispatch, `barcode`, `get_paperformat`, `_build_wkhtmltopdf_args`, `_prepare_html`, `_run_wkhtmltopdf`, `retrieve_attachment`, `postprocess_pdf_report`, `force_report_rendering`.
- `odoo/addons/base/models/report_paperformat.py` - `report.paperformat` fields, paper sizes, `_check_format_or_page`, `_compute_print_page_size`.
- `odoo/addons/base/data/report_paperformat_data.xml` - `paperformat_euro`, `paperformat_us`, `paperformat_batch_deposit`.
- `odoo/addons/base/models/qweb.py` - QWeb directives: `t-esc`, `t-raw`, `t-field`, `t-options`, `t-foreach`, `t-if`/`t-elif`/`t-else`, `t-set`, `t-call`, `t-att-*`, `t-attf-*`, `t-translation`, `foreach_iterator`.
- `odoo/addons/base/models/ir_qweb.py` - `ir.qweb` renderer, `t-call-assets`, `t-lang`, `_get_asset_nodes`, `_get_asset_content`, `_get_field`, `_get_widget`, `data-pagebreak` handling.
- `odoo/addons/base/models/ir_qweb_fields.py` - field converters: `monetary`, `date`, `datetime`, `image`, `barcode`, `contact`, `html`, `many2one`, `selection`, and others.
- `odoo/addons/base/models/ir_ui_view.py` - `render_template`, `_prepare_qcontext` (`env`, `datetime`, `image_data_uri`, `request`, `xmlid`, `viewid`).
- `odoo/addons/base/models/ir_actions.py` - `binding_model_id` and `binding_type` on `ir.actions.actions`.
- `odoo/addons/base/models/res_company.py` - `paperformat_id` default and `external_report_layout_id`.
- `odoo/addons/base/models/report_layout.py` - `report.layout` model (`view_id`, `image`, `pdf`, `name`).
- `odoo/addons/base/wizard/base_document_layout.py` - Document Layout wizard and `report.layout` selection.
- `odoo/addons/base/report/report_base_report_irmodulereference.py` - `_get_report_values` example.
- `odoo/addons/base/report/ir_module_reports.xml` - `<report>` tag example.
- `odoo/tools/convert.py` - `_tag_report` attribute mapping for the `<report>` shortcut.
- `odoo/tools/translate.py` - QWeb translation extraction, `t-translation="off"`, `trans_export`.
- `odoo/tools/config.py` - `--i18n-export`, `--i18n-import`, `--i18n-overwrite`, `--modules`.
- `addons/web/views/report_templates.xml` - `web.report_assets_common`, `web.report_assets_pdf`, `web.report_layout`, `web.html_container`, `web.minimal_layout`, `web.external_layout`, `web.external_layout_standard`, `web.external_layout_background`, `web.external_layout_boxed`, `web.external_layout_clean`, `web.internal_layout`, `web.basic_layout`, `web.address_layout`, `web.styles_company_report`.
- `addons/web/data/report_layout.xml` - `report.layout` records for the standard, background, boxed, and clean layouts.
- `addons/web/controllers/main.py` - `/report/<converter>/<reportname>`, `/report/download`, `/report/barcode`, `/report/check_wkhtmltopdf`.
- `addons/web/static/src/scss/report.scss` and `layout_*.scss` - report print styles.
