---
name: odoo-13-translation
description: Complete reference for Odoo 13 translations and localization. Covers Python translations with _() and _lt(), JavaScript/_t translations, QWeb template translations, translate=True fields, the ir.translation model, PO/POT/CSV structure, translation export/import, and language management.
globs: "**/*.{py,js,xml}"
topics:
  - Python translations (_ and _lt)
  - JavaScript translations (_t and _lt)
  - QWeb template translations
  - Field translations (translate=True, ir.translation storage)
  - PO / POT / CSV structure
  - Translation export/import
  - Language / context switching
when_to_use:
  - Adding translatable strings to Python code
  - Adding translations to JavaScript widgets
  - Creating translatable QWeb templates
  - Setting up translated fields
  - Exporting/importing translations
  - Reading translations in a specific language
---

# Odoo 13 Translation & Localization Guide

Complete guide for translating and localizing Odoo 13 modules.

> **Odoo 13 storage model**: `translate=True` fields keep the English source value in
> the model's own DB column (a plain `varchar`/`text` column). Translated values live
> in the **`ir.translation`** table, keyed by `(type, name, lang, res_id)` for `model`
> terms and `(type, name, lang, res_id, src)` for `model_terms` terms, plus a
> separate unique index for `code` terms. There is **no JSONB per-field column** and
> no `get_field_translations()` / `update_field_translations()` helper - both are
> 16.0+/17.0 APIs and are not available in 13.0.

## Quick Reference

| Context | Function | Example |
|---------|----------|---------|
| Python code (inside a method) | `_()` | `_("Hello World")` |
| Python module/class-level constants | `_lt()` | `STATUS = _lt("Draft")` |
| JavaScript / web client | `_t()` | `_t("Good morning")` |
| JavaScript lazy (returns `{toString}`) | `_lt()` | `_lt("contains")` |
| Field definition | `translate=True` | `name = fields.Char(translate=True)` |
| HTML field term translation | `translate=html_translate` | `body = fields.Html(translate=html_translate)` |

---

## Table of Contents

1. [Python Translations](#python-translations)
2. [Field Translations](#field-translations)
3. [XML / HTML Translation Helpers](#xml--html-translation-helpers)
4. [QWeb Template Translations](#qweb-template-translations)
5. [JavaScript Translations](#javascript-translations)
6. [The ir.translation Model](#the-irtranslation-model)
7. [Module Translation Structure](#module-translation-structure)
8. [Translation Export / Import](#translation-export--import)
9. [Language Management](#language-management)
10. [Translation Types](#translation-types)
11. [Best Practices](#best-practices)
12. [Anti-Patterns](#anti-patterns)
13. [Testing Translations](#testing-translations)

---

## Python Translations

### Standard translation function `_()`

`_` is a `GettextAlias` singleton defined in `odoo/tools/translate.py` and re-exported
from the `odoo` package (`from odoo.tools.translate import _, _lt`). Calling `_("...")`
looks up the string in the in-DB `ir.translation` table for the current context
language; when nothing matches it returns the source string unchanged.

```python
from odoo import _                        # re-export of odoo.tools.translate._

# Simple translation
message = _("Hello World")
```

**Formatting is applied *after* `_()` returns.** In 13.0 `GettextAlias.__call__` accepts
a single `source` argument only - it does not pass extra arguments through to a
formatter. Use `%` on the translated result:

```python
# Positional formatting
message = _("Hello, %s") % user.name

# Named formatting (keep names inside the string so translators see them)
message = _("Hello %(name)s, you have %(count)d messages") % {
    'name': user.name,
    'count': len(records),
}
```

The source string passed to `_()` must be a **string literal**. Dynamic-looking source
strings are not extractable:

```python
# BAD - the extractor cannot see the source string
_(f"Hello {user.name}")
_("Hello" + user.name)

# GOOD - a single extractable source with a placeholder
_("Hello %(name)s") % {'name': user.name}
```

### Lazy translation `_lt()` for module-level constants

`_lt` is a class (`class _lt:` with `@functools.total_ordering`) in
`odoo/tools/translate.py`. Each `_lt("...")` instance stores the source in `_source` and
resolves it only when the object is turned into text via `__str__`, via `+`
(`__add__` / `__radd__`), or through concatenation. This makes it safe for values
evaluated at import time, before any request hits the server.

```python
from odoo.tools.translate import _lt

STATUS_DRAFT = _lt("Draft")
STATUS_DONE = _lt("Done")

class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'

    def get_status_label(self, status):
        return {'draft': STATUS_DRAFT, 'done': STATUS_DONE}[status]
```

```python
# Concatenation resolves lazily on both sides
label = "do you speak " + STATUS_DRAFT    # __radd__
label = STATUS_DRAFT + ", I speak it"     # __add__
```

There is no `LazyTranslate(__name__)` constructor in 13.0 (that shape appears in later
versions); the only lazy helper is `_lt`. Do not use `_()` at module scope: it would
translate once, at import time, with no meaningful language context.

### Where `_()` resolves the language from

`GettextAlias._get_lang(frame)` walks the calling frame looking, in order, for:

1. `context.get('lang')` in a local `context` dict
2. `kwargs['context'].get('lang')`
3. `self.env.lang`
4. `self.localcontext.get('lang')` (QWeb rendering)
5. `request.env.lang` (HTTP request)
6. last resort: `res.users.context_get()['lang']` for the current user

So `_()` "just works" inside model methods, controllers, and wizards that have `self`.
For helper functions without `self`, wrap the call with `with_context(lang=...)` or use
`_lt` and let it resolve later.

---

## Field Translations

### Declaring a translatable field

```python
from odoo import fields, models
from odoo.tools.translate import html_translate

class Product(models.Model):
    _name = 'my.product'
    _description = 'Product'

    # Whole-value translation: the full value is replaced per language
    name = fields.Char(string='Name', translate=True)

    # Same for Text
    summary = fields.Text(translate=True)

    # HTML translated term-by-term (each paragraph / inline piece is a term)
    description = fields.Html(translate=html_translate)
```

`fields.Html(translate=True)` with default sanitization is automatically rewritten to
`html_translate` in `fields.py` (the `_String` / `Html` init path), because a sanitized
HTML value must be translated term-by-term, not as one opaque string.

### Storage: source in the column, translations in `ir.translation`

In 13.0 the model column stays the ordinary string column (`varchar` for `Char`,
`text` for `Text`/`Html`). The English (source) value lives there. Translated values
are rows in `ir.translation`:

- `type = 'model'` for whole-value translation (`translate=True`)
- `type = 'model_terms'` for callable translation (`html_translate`, `xml_translate`)
- `name = "<model>,<field>"` (for example `my.product,name`)
- `res_id = <record id>`

Reading a translatable field picks the value for the current `env.lang` (or the source
column when the language is `en_US` / missing). Writing updates the cache, the source
column (for source-value writes), and the matching `ir.translation` row.

**Not available in 13.0:** `BaseModel.get_field_translations()`,
`BaseModel.update_field_translations()`, `BaseModel._update_field_translations()`, and
JSONB column storage. Those are 16.0+ APIs. Use the context pattern below instead.

### Reading in a specific language

```python
name_fr = partner.with_context(lang='fr_FR').name
name_vi = partner.with_context(lang='vi_VN').name
```

### Writing translations programmatically

```python
# Set the value for the current language only
partner.with_context(lang='fr_FR').name = 'Nom'
```

The low-level read/write helpers on `ir.translation` in 13.0 are
`_get_ids(name, tt, lang, ids)` and `_set_ids(name, tt, lang, ids, value, src=None)`
(used internally by the field `write`/`read` path), and `_get_source(name, types, lang,
source=None, res_id=None)` for code terms. Prefer the `with_context(lang=...)` field
API over calling those directly.

### Field `string` and `help` are automatic

The `string` label and `help` tooltip are already marked translatable by Odoo's
extractor. You do not need to wrap them in `_()`:

```python
state = fields.Selection(
    [('draft', 'Draft'), ('done', 'Done')],
    string='Status',
    help='Current document state',
)
```

`'Status'`, `'Current document state'`, and the selection labels `'Draft'`/`'Done'`
all end up in the `.pot` file automatically (selection labels are exported against
`ir.model.fields.selection`).

### Term-by-term translation via `html_translate`

Whole-value translation is wrong for HTML. `html_translate` walks the HTML and calls
the translation callback once per visible text node, so translators see paragraphs and
inline pieces, not a block of markup:

```python
from odoo.tools.translate import html_translate

description = fields.Html(translate=html_translate)
```

The extracted source terms are individual text fragments; they are stored as
`type='model_terms'` rows and recomposed on read.

---

## XML / HTML Translation Helpers

`odoo/tools/translate.py` provides two callable translators usable as a `translate=`
value on fields:

- `xml_translate(callback, value)` - parses the value as XML, translates translatable
  text and attribute values, and re-serializes as XML. Falls back to an HTML parse with
  a wrapping `<div>` (then unwraps) if the XML is malformed.
- `html_translate(callback, value)` - parses the value as an HTML fragment, translates
  text nodes and translatable attributes, and re-serializes as HTML.

Both use `translate_xml_node()`, which honors:

- `TRANSLATED_ELEMENTS` - inline elements such as `b`, `i`, `span`, `strong`, `small`,
  `code`, `em`, `sub`, `sup`, and `br`.
- `TRANSLATED_ATTRS` - attributes that get translated: `string`, `help`, `sum`, `avg`,
  `confirm`, `placeholder`, `alt`, `title`, `aria-label`, `aria-keyshortcuts`,
  `aria-placeholder`, `aria-roledescription`, `aria-valuetext`, `value_label`, plus the
  QWeb `t-attf-<attr>` variants.
- `t-translation="off"` - skips translation of that element (`t-translation` exists in
  13.0 as the translation-off switch).
- Skipped elements `script`, `style`, `title`, and comment/PI node types.

```python
from odoo.tools.translate import xml_translate

arch = fields.Text(translate=xml_translate)
```

---

## QWeb Template Translations

### Text content is auto-translatable

Any text node inside a QWeb template (`.xml` under `static/src/xml/`) is extracted as a
translation term, including the literal text around `t-*` directives:

```xml
<templates>
    <t t-name="my_module.ThanksPage">
        <div>
            <h2>Thank you for your order</h2>
            <p>We will process it within 24 hours.</p>
        </div>
    </t>
</templates>
```

13.0 QWeb lives in `odoo/tools/qweb.py` (server side) and `web.QWeb`
(`addons/web/static/src/js/core/qweb.js`) on the client. Directive set for 13.0:
`t-esc`, `t-raw`, `t-set`, `t-call`, `t-if`/`t-elif`/`t-else`, `t-foreach`/`t-as`,
`t-att-*`/`t-attf-*`, `t-field`, `t-options`, `t-call-assets`, `t-js`, `t-css`.

> **Not available in 13.0:** the `t-out` directive (14.0+). Use `t-esc` (escaped) or
> `t-raw` (unescaped) in 13.0.

### Translatable attributes

The extractor considers `title`, `alt`, `label`, `placeholder`, and `aria-label`
translatable on QWeb elements (`_extract_translatable_qweb_terms`). View `string=`,
`help=`, `confirm=`, and related attributes on record/field nodes follow the
`TRANSLATED_ATTRS` set above:

```xml
<field name="email" string="Email" placeholder="name@example.com"/>
<button string="Save" confirm="Save changes?" type="object" name="action_save"/>
```

### Dynamic content and placeholders

Keep the full sentence as one translatable string, not concatenated pieces. In 13.0
the JS translation function is `_t()` (see the JavaScript section); use `sprintf`-style
`%s` / `%(...)s` placeholders:

```xml
<!-- BAD: grammatically untranslatable in many languages -->
<t t-esc="user.name"/> has <t t-esc="count"/> messages.

<!-- GOOD: single translatable sentence with a placeholder -->
<t t-esc="_.str.sprintf(_t('Hello %(name)s, you have %(count)d messages'),
                         {name: user.name, count: count})"/>
```

### Disabling translation

The switch used in 13.0 is `t-translation="off"`:

```xml
<span t-translation="off">v13.0</span>
<span t-translation="off">user_id</span>
```

---

## JavaScript Translations

### Legacy AMD modules, no OWL

13.0 has no OWL and no `@web` package. Every frontend file is a legacy AMD module:

```javascript
odoo.define('my_module.my_widget', function (require) {
"use strict";

var core = require('web.core');

var _t = core._t;
var _lt = core._lt;

var MyWidget = require('web.Widget').extend({
    init: function () {
        this._super.apply(this, arguments);
        this.label = _t("Saved successfully");
    },
});

return MyWidget;
});
```

### Translation function `_t()`

`_t` is defined in `addons/web/static/src/js/core/translation.js` as module
`web.translation`, and re-exported by `web.core` (`services/core.js`) as `core._t`.
It is an **eager** function: it resolves against the in-memory translation database
immediately (returns the source string when no translation is loaded yet):

```javascript
var _t = require('web.core')._t;

// Simple
var message = _t("Good morning");

// sprintf placeholders via Underscore's _.str.sprintf
var msg = _.str.sprintf(_t("Hello %s, you have %d messages"), userName, count);
```

### `_lt()` in JavaScript

The JS `_lt` returns a plain `{toString: function () { return _t(s); }}` object, so the
translation happens only when the value is coerced to a string (for example when it is
inserted into a template). Use it for strings evaluated before the translation bundle
is loaded, in particular class/object literals:

```javascript
// addons/web/static/src/js/widgets/domain_selector.js (real 13.0 usage)
var FIELD_OPERATORS = {
    "=": _lt("is equal to"),
    "!=": _lt("is not ="),
    "ilike": _lt("contains"),
    "child_of": _lt("child of"),
};
```

### Where JS translations come from

The web client loads translations through the controller
`/web/webclient/translations/<unique>` (`addons/web/controllers/main.py`), which calls
`ir.translation.get_translations_for_webclient(mods, lang)`. Only rows whose `comments`
contain `openerp-web` and whose `value` is non-empty are shipped. JS and client QWeb
terms are extracted with the `openerp-web` comment marker so the server can find them.

`TranslationDataBase.load_translations()` in `translation.js` fetches the bundle once
per session before widgets render (`web.Session` calls it during init). There is no
`translationIsReady` deferred in 13.0; `_t` simply returns the source when the bundle
has not arrived yet.

---

## The `ir.translation` Model

Defined in `odoo/addons/base/models/ir_translation.py`:

| Field | Type | Meaning |
|-------|------|---------|
| `name` | Char (required) | `<model>,<field>` for model terms, file path for code terms |
| `res_id` | Integer (index) | record id (model), or source line number (code) |
| `lang` | Selection | language code, `validate=False` |
| `type` | Selection | `model` / `model_terms` / `code` |
| `src` | Text | source string, kept for backward compatibility |
| `value` | Text | the translated value |
| `module` | Char (index) | module owning the term |
| `state` | Selection | `to_translate` / `inprogress` / `translated` |
| `comments` | Text (index) | extracted comments; `openerp-web` marks web terms |

Unique constraints (created in `_auto_init`):

- `ir_translation_code_unique` on `(type, lang, md5(src))` where `type = 'code'`
- `ir_translation_model_unique` on `(type, lang, name, res_id)` where `type = 'model'`
- `ir_translation_unique` on `(type, name, lang, res_id, md5(src))` for `model_terms`

### Key API methods

| Method | Purpose |
|--------|---------|
| `_get_source(name, types, lang, source=None, res_id=None)` | Return a translated string (used by `_()` lookups) |
| `_get_ids(name, tt, lang, ids)` | Bulk-read `type='model'` values for record ids |
| `_set_ids(name, tt, lang, ids, value, src=None)` | Bulk-write `type='model'` values |
| `_get_terms_translations(field, records)` | Read `model_terms` grouped by record |
| `_get_terms_query(field, records)` | Build the `model_terms` lookup query |
| `_sync_terms_translations(field, records)` | Re-map terms after an English value changed |
| `insert_missing(field, records)` | Insert empty `to_translate` rows for all active languages |
| `_upsert_translations(vals_list)` / `_update_translations(vals_list)` | Bulk upsert / update |
| `_get_import_cursor()` | Return the import cursor used by PO/CSV loading |
| `translate_fields(model, id, field=None)` | Return an action to translate a record's fields |
| `get_translations_for_webclient(mods, lang)` | Build the JS translation bundle |
| `get_technical_translations(model_name)` | Return an action listing a model's field labels/selections |

The fast import path uses `IrTranslationImport`, a temporary SQL table
(`tmp_ir_translation_import`) that is populated in batches at commit time.

---

## Module Translation Structure

### Directory layout

```
my_module/
|-- i18n/
|   |-- my_module.pot      # Template - extracted source strings
|   |-- fr.po              # French translation
|   |-- de.po              # German translation
|   `-- i18n_extra/
|       `-- fr_BE.po       # Belgian French override layered over fr.po
|-- models/
|   `-- my_model.py        # Python with _() / _lt()
|-- views/
|   `-- my_model_views.xml # XML with string=, placeholder=, help=
|-- static/
|   `-- src/
|       |-- js/
|       |   `-- my_widget.js   # JS with _t() / _lt()
|       `-- xml/
|           `-- my_widget.xml  # client QWeb template
`-- __manifest__.py
```

The `.pot` is the source of truth; regenerate it after changing extractable strings and derive each locale `.po` from it.

### `__manifest__.py` example (13.0)

```python
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'license': 'LGPL-3',
    'depends': ['base'],
    'data': [
        'views/my_model_views.xml',
    ],
    'qweb': [
        'static/src/xml/my_widget.xml',
    ],
    'installable': True,
}
```

There is no `assets` key in 13.0 manifests. Client JS/CSS goes into XML asset bundles
(`web.assets_backend` etc.) and client QWeb templates go in the `qweb` manifest key.

### PO file format

`i18n/fr.po`:

```po
# Translation of Odoo Server.
# This file contains the translation of the following modules:
# 	* my_module
#
msgid ""
msgstr ""
"Project-Id-Version: Odoo Server 13.0\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2020-07-23 15:12+0000\n"
"PO-Revision-Date: 2020-07-23 15:12+0000\n"
"Last-Translator: \n"
"Language-Team: \n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: \n"
"Plural-Forms: \n"

#. module: my_module
#: code:addons/my_module/models/my_model.py:0
#, python-format
msgid "Hello %(name)s"
msgstr "Bonjour %(name)s"

#. module: my_module
#: model:ir.model.fields,field_description:my_module.field_my_model__name
msgid "Name"
msgstr "Nom"

#. module: my_module
#: model_terms:ir.ui.view,arch_db:my_module.my_view
msgid "Thank you for your order"
msgstr "Merci pour votre commande"
```

Comment / occurrence markers to know:

- `#: code:<path>:<line>` - a Python (`_`/`_lt`) or JS (`_t`/`_lt`) source string
- `#, python-format` - flag added by the writer when the entry is a `code` term
- `#: model:ir.model.fields,field_description:<module>.<field>` - field label
- `#: model:ir.model.fields,help:<module>.<field>` - field help
- `#: model:ir.model,name:<module>.<model>` - the `_description`
- `#: model:ir.model.fields.selection,name:<module>.selection__<model>__<field>__<value>`
  - a Selection label
- `#: model_terms:ir.ui.view,arch_db:<module>.<view>` - view template text
- `#. openerp-web` - term must be loaded by the web client (JS / client QWeb)
- `#. module: <module>` - originating module (may list several comma-separated modules)

### `i18n_extra` and layered loading

`ir.translation._load_module_terms()` loads, for a locale like `fr_BE`:

1. `i18n/fr.po` (the base language) first
2. `i18n_extra/fr.po` (base-language overrides)
3. `i18n/fr_BE.po` (the locale), overriding base terms
4. `i18n_extra/fr_BE.po`

`overwrite` context is set to `True` after the base files so the locale/extra terms win.

### CSV file format

`i18n/<module>.csv` (older format still read/written by `trans_load` / `trans_export`):

```csv
module,type,name,res_id,src,value,comments
my_module,code,addons/my_module/models/my_model.py,3,Accounting,samva,
my_module,model,my.model,name,my_module.my_record,Name,Nom,
```

Columns, in order: `module`, `type`, `name`, `res_id`, `src`, `value`, `comments`.

- `type` is `code`, `model`, or `model_terms`
- `name` for `model`/`model_terms` is `<model>,<field>`; for `code` it is the display
  path (`addons/<module>/<path>`)
- `res_id` is either a numeric id/line, or `<module>.<external_id>` which the importer
  resolves to a record id and then imports `module`/`imd_name` separately
- duplicate `code` entries with the same `src` are skipped on import

---

## Translation Export / Import

The command options live in `odoo/cli/server.py` `main()` (config declared in
`odoo/tools/config.py`). Use `./odoo-bin` or `python odoo-bin`:
`--load-language`, `-l/--language`, `--i18n-export`, `--i18n-import`,
`--i18n-overwrite`, `--modules`.

### Generate / update the `.pot`

```bash
./odoo-bin -c odoo.conf \
    -d <your_db> \
    --modules=my_module \
    --i18n-export=addons/my_module/i18n/my_module.pot \
    --addons-path=addons,custom_addons
```

The file extension selects the writer: `.po`, `.csv`, or `.tgz` (and `.pot` is written
as PO with no translation values). Omit `--modules` to export everything.

### Export a translated `.po`

```bash
./odoo-bin -c odoo.conf \
    -d <your_db> \
    --modules=my_module \
    --i18n-export=addons/my_module/i18n/fr.po \
    -l fr_FR
```

### Import a `.po` / `.csv` into the database

```bash
./odoo-bin -c odoo.conf \
    -d <your_db> \
    --i18n-import=addons/my_module/i18n/fr.po \
    -l fr_FR \
    --i18n-overwrite
```

`-l` is mandatory for import. `--i18n-overwrite` replaces existing non-`noupdate`
terms instead of only inserting missing ones.

### Programmatic load

Reloading PO files for a module is handled by `ir.translation._load_module_terms()`,
which the module-update machinery calls (via `ir.module.module._update_translations()`)
after a module is installed/updated:

```python
self.env['ir.translation']._load_module_terms(
    modules=['my_module'],
    langs=['fr_FR'],
)
```

### UI export / import

- **Settings -> Translations -> Import / Export Translations** (or the legacy
  General Settings language path): choose language, format (PO/CSV/TGZ), and modules,
  then import/export. The wizard actions live under `base/wizard/`.

---

## Language Management

### The `res.lang` model

Defined in `odoo/addons/base/models/res_lang.py`. Relevant fields:

```python
class Lang(models.Model):
    _name = "res.lang"
    _description = "Languages"

    name = fields.Char(required=True)
    code = fields.Char(string='Locale Code', required=True)     # 'fr_FR'
    iso_code = fields.Char(string='ISO code')                   # 'fr'
    url_code = fields.Char(string='URL Code', required=True)    # 'fr'
    active = fields.Boolean()
    direction = fields.Selection([('ltr', 'Left-to-Right'),
                                  ('rtl', 'Right-to-Left')], required=True, default='ltr')
    date_format = fields.Char(string='Date Format', required=True, default='%m/%d/%Y')
    time_format = fields.Char(string='Time Format', required=True, default='%H:%M:%S')
    week_start = fields.Selection([...], ...)
    grouping = fields.Char(string='Separator Format', required=True, default='[]')
    decimal_point = fields.Char(string='Decimal Separator', required=True, default='.')
    thousands_sep = fields.Char(string='Thousands Separator', default=',')
```

### Loading / activating a language in 13.0

There is **no `_activate_lang()` / `_create_lang()`** in 13.0 (those are later-version
helpers). Use `res.lang.load_lang()`:

```python
# Load a language code (creates it from locale data if missing, activates if present)
self.env['res.lang'].load_lang('fr_FR')

# Also available: _lang_get(code) for the active record, get_installed() for (code, name)
installed = self.env['res.lang'].get_installed()
# [(code, name), ...] for active languages only

current_lang = self.env.lang or 'en_US'
```

Activating a language in the UI runs `_update_translations()`, which loads that
language's PO files for every installed module (`res.lang.toggle_active()`).

### Bootstrap with `--load-language`

`--load-language=fr_FR,de_DE` makes the database bootstrap load the listed languages
(the first one becomes the default partner language). `tools.load_language(cr, lang)`
is called from `odoo/modules/loading.py` and delegates to the `base.language.install`
wizard.

### Switching language per operation

```python
# Read a translated field in a specific language
name_fr = record.with_context(lang='fr_FR').name

# Run logic as if the user were French (any _() inside resolves fr_FR)
record_fr = record.with_context(lang='fr_FR')
record_fr.action_confirm()
```

---

## Translation Types

`ir.translation.type` is a `Selection` of exactly three values (defined at the top of
`ir_translation.py`):

- `model` - whole-value field translation. Source: `fields.Char(translate=True)` /
  `fields.Text(translate=True)`. `name = "<model>,<field>"`, `res_id = record id`.
- `model_terms` - structured field translation (callable `translate`, e.g.
  `html_translate`/`xml_translate`). Each translator-visible text fragment is one row
  with a `src`. Unique on `(type, name, lang, res_id, md5(src))`.
- `code` - Python `_()` / `_lt()` and JS `_t()` / `_lt()` strings. Stored with the
  source path as `name`, unique on `(type, lang, md5(src))`, and looked up directly by
  source string (no record id).

`trans_generate()` in `translate.py` scans installed modules: `*.py` with Babel
keywords `_`/`_lt`, `static/src/js*/*.js` with `_t`/`_lt` (tagged `openerp-web`), and
`static/src/xml*/*.xml` with the QWeb extractor (also tagged `openerp-web`).

---

## Best Practices

### DO use named placeholders for dynamic content

```python
_("Hello %(name)s, welcome to %(app)s") % {'name': user.name, 'app': 'Odoo'}
```

### DON'T concatenate translated strings

```python
# BAD - unusable in languages with different word order / grammar
msg = _("Hello") + ", " + user.name + _("welcome")

# GOOD
msg = _("Hello %(name)s, welcome") % {'name': user.name}
```

### DO keep full sentences in one `_()`

```python
# GOOD
msg = _("Delete the selected records?")

# BAD - translators cannot keep the two fragments grammatical
msg = _("Delete") + _("the selected records")
```

### DO use `_lt()` for module/class-level constants

```python
STATUS_DRAFT = _lt("Draft")

class MyModel(models.Model):
    _name = 'my.model'
    def display(self):
        return STATUS_DRAFT   # resolves when coerced to str
```

### DON'T use `_()` at module scope

`_()` resolves immediately at import time, with no meaningful language context.

### DO use `t-translation="off"` for literal code/tokens in QWeb

```xml
<code t-translation="off">my_module.field_name</code>
```

---

## Anti-Patterns

### Dynamic source strings

```python
message = _(f"User {user.name} created")   # BAD
message = _("User %(name)s created") % {'name': user.name}   # GOOD
```

### Extra positional arguments to `_()`

`GettextAlias.__call__` takes only `source` in 13.0; `_("Hello %s", name)` raises
`TypeError`. Format after translation: `_("Hello %s") % name`.

### Translating technical ids

```python
xml_id = _("my_module.my_record")   # BAD
xml_id = 'my_module.my_record'      # GOOD
```

### Branching inside translation

```python
msg = _("Success" if ok else "Failure")   # BAD - evaluates as one _() call on a bool
msg = _("Success") if ok else _("Failure")   # GOOD
```

### Translation inside hot loops

```python
for rec in records:
    label = _("Name")            # BAD - lookup per iteration
    # ...

label = _("Name")                # GOOD - hoist out
for rec in records:
    # ...
```

### Using 16.0+/17.0 field-translation APIs

```python
# BAD in 13.0 - these methods do not exist
record.get_field_translations('name')
record.update_field_translations('name', {'fr_FR': 'Nom'})

# GOOD in 13.0
name_fr = record.with_context(lang='fr_FR').name
record.with_context(lang='fr_FR').name = 'Nom'
```

---

## Testing Translations

```python
from odoo.tests.common import TransactionCase

class TestTranslation(TransactionCase):

    def setUp(self):
        super().setUp()
        self.env['res.lang'].load_lang('fr_FR')

    def test_translate_field(self):
        product = self.env['my.product'].create({'name': 'Chair'})
        product.with_context(lang='fr_FR').name = 'Chaise'

        self.assertEqual(product.with_context(lang='en_US').name, 'Chair')
        self.assertEqual(product.with_context(lang='fr_FR').name, 'Chaise')

    def test_code_translation(self):
        # Load the module terms for the target language, then resolve in context
        self.env['ir.translation']._load_module_terms(
            modules=['my_module'], langs=['fr_FR'])
        msg = self.env['my.model'].with_context(lang='fr_FR')._hello()
        self.assertEqual(msg, 'Bonjour')
```

Use `TransactionCase` / `SingleTransactionCase` / `SavepointCase` as in the rest of 13.0.

---

## Quick Checklist

When adding translatable content to an Odoo 13 module:

- [ ] Python runtime strings wrapped in `_()`, formatting applied with `%` after the call
- [ ] Python module/class-level strings wrapped in `_lt()`
- [ ] JavaScript strings wrapped in `_t()` (or `_lt()` for top-level literals)
- [ ] Translatable fields declared with `translate=True` / `translate=html_translate`
- [ ] View `string=` / `help=` / `placeholder=` attributes present (auto-extracted)
- [ ] `i18n/<module>.pot` present and up to date
- [ ] Every shipped locale has a `.po` in `i18n/` derived from the `.pot`
- [ ] Dynamic content uses `%s` / `%(name)s` placeholders, never f-string / `+`
- [ ] No `get_field_translations` / `update_field_translations` (16.0+) usage
- [ ] Tests cover at least one non-`en_US` language switch

---

## Key Files Reference

| Purpose | File |
|---------|------|
| Core translation logic, `_`, `_lt`, `GettextAlias` | `odoo/tools/translate.py` |
| `html_translate` / `xml_translate` / `translate_xml_node` | `odoo/tools/translate.py` |
| QWeb term extraction (`babel_extract_qweb`) | `odoo/tools/translate.py` |
| `_`/`_lt` re-export at package level | `odoo/__init__.py` |
| Field `translate` handling (`_String`, `Html`) | `odoo/fields.py` |
| `ir.translation` model + `IrTranslationImport` | `odoo/addons/base/models/ir_translation.py` |
| `_load_module_terms` / module term loading | `odoo/addons/base/models/ir_translation.py` + `ir_module.py` |
| Language model (`res.lang`, `load_lang`, `get_installed`) | `odoo/addons/base/models/res_lang.py` |
| JS translation functions (`_t`, `_lt`, `TranslationDataBase`) | `addons/web/static/src/js/core/translation.js` |
| `web.core` re-export of `_t`/`_lt` | `addons/web/static/src/js/services/core.js` |
| JS translation loading (`web.Session`) | `addons/web/static/src/js/core/session.js` |
| Translations HTTP controller | `addons/web/controllers/main.py` |
| CLI export/import (`trans_export`, `trans_load`) | `odoo/cli/server.py` |
| CLI option declarations (`--i18n-*`, `--load-language`) | `odoo/tools/config.py` |

---

## Base Code Reference

- `odoo/tools/translate.py` - `GettextAlias` / `_`, `_lt`, `html_translate`, `xml_translate`,
  `translate_xml_node`, `trans_export`, `trans_load`, `trans_load_data`,
  `TranslationFileReader`/`Writer`, `CSVFileReader`/`Writer`, `PoFileReader`/`Writer`,
  `TarFileWriter`, `babel_extract_qweb`, `load_language`, `translate()`
- `odoo/__init__.py` - `from odoo.tools.translate import _, _lt`
- `odoo/fields.py` - `_String.translate`, `get_trans_terms`, `get_trans_func`,
  field read/write translation handling, `Html` auto-`html_translate`
- `odoo/addons/base/models/ir_translation.py` - `IrTranslation`, `TRANSLATION_TYPE`,
  `_get_source`, `_get_ids`, `_set_ids`, `_get_terms_translations`,
  `_sync_terms_translations`, `_load_module_terms`, `get_translations_for_webclient`,
  `IrTranslationImport`
- `odoo/addons/base/models/ir_module.py` - `Module._update_translations()`
- `odoo/addons/base/models/res_lang.py` - `Lang.load_lang`, `Lang.get_installed`,
  `Lang._lang_get`, locale/format fields
- `addons/web/static/src/js/core/translation.js` - `_t`, `_lt`, `TranslationDataBase`,
  `load_translations`
- `addons/web/static/src/js/services/core.js` - `core._t` / `core._lt` re-export
- `addons/web/static/src/js/core/session.js` - translation bundle loading on session init
- `addons/web/controllers/main.py` - `/web/webclient/translations/<unique>` controller
- `odoo/cli/server.py` - `export_translation()`, `import_translation()`
- `odoo/tools/config.py` - `--i18n-export`, `--i18n-import`, `--i18n-overwrite`,
  `--modules`, `-l/--language`, `--load-language`