---
name: odoo-13-development
description: Complete reference for Odoo 13 module authoring - directory layout, manifest keys and defaults, models, views, security, wizards, reports, controllers, asset bundles, i18n, tests, and the install/update CLI.
globs: "**/*.{py,xml,csv}"
topics:
  - Standard Odoo 13 module directory structure
  - __init__.py patterns (root, models/, wizard/, controllers/)
  - Odoo 13 __manifest__.py keys and defaults
  - Models with models.Model and models.TransientModel
  - Views, menus, and window actions
  - Security files (ir.model.access.csv, ir.rule)
  - QWeb reports (ir.actions.report)
  - Asset bundles via XML inheritance (web.assets_backend)
  - HTTP controllers, i18n, tests, and the install/update CLI
when_to_use:
  - Scaffolding a new Odoo 13 module
  - Adding a model, view, wizard, report, or controller
  - Declaring backend/frontend assets for Odoo 13
  - Setting up security or translations for a module
  - Running install, update, and test commands
---

# Odoo 13 Development Guide

Module creation overview for Odoo 13: directory structure, `__init__.py`
patterns, manifest keys, models, views, security, wizards, reports,
controllers, asset bundles, i18n, tests, and the install/update CLI. Every
claim is written against the Odoo 13.0 source tree; the
[Base Code Reference](#base-code-reference) lists the files verified against.

## Table of Contents

1. [Module Structure](#module-structure)
2. [`__init__.py` Patterns](#__init__py-patterns)
3. [`__manifest__.py` Keys & Defaults](#__manifest__py-keys--defaults)
4. [Models](#models)
5. [Views, Menus & Actions](#views-menus--actions)
6. [Security Declaration](#security-declaration)
7. [Wizards & `TransientModel`](#wizards--transientmodel)
8. [Reports](#reports)
9. [Controllers](#controllers)
10. [Asset Bundles](#asset-bundles)
11. [Static Assets Layout](#static-assets-layout)
12. [i18n (Translations)](#i18n-translations)
13. [Tests](#tests)
14. [Install / Update CLI](#install--update-cli)
15. [Base Code Reference](#base-code-reference)

---

## Module Structure

Standard Odoo 13 module skeleton (matches `odoo-bin scaffold`, plus folders
used by real modules):

```
my_module/
|-- __init__.py     # package init (imports submodules)
|-- __manifest__.py # manifest (REQUIRED)
|-- models/         # models.Model subclasses
|-- wizard/         # models.TransientModel subclasses
|-- controllers/    # http.Controller subclasses
|-- views/          # ir.ui.view, actions, menus
|-- security/       # ir.model.access.csv, ir.rule, groups
|-- data/           # ir.sequence, ir.cron, reference data
|-- demo/           # demo data (only in demo mode)
|-- report/         # ir.actions.report + QWeb templates
|-- migrations/13.0.1.0.0/  # pre/post/end-*.py, migrate(cr, version)
|-- i18n/           # <module>.pot + one .po per locale
|-- static/         # description/icon.png + src/{js,xml,scss,...}
`-- tests/
```

- `static/description/icon.png` is discovered automatically; do not declare it.
- `migrations/` scripts live in a per-version subfolder (`13.0.1.0.0/` or a
  majorless `1.0.0/`); no `upgrade_code` CLI command exists in 13.0.
- JS QWeb templates in `static/src/xml/` are declared via the manifest `qweb` key, not via `data`.

---

## `__init__.py` Patterns

### Module root `__init__.py`

```python
# my_module/__init__.py
from odoo import api, SUPERUSER_ID

from . import controllers
from . import models
from . import wizard


def pre_init_hook(cr):
    """Runs before install (fresh installs only). Takes a cursor."""


def post_init_hook(cr, registry):
    """Runs after install (fresh installs only)."""
    env = api.Environment(cr, SUPERUSER_ID, {})
    env['ir.config_parameter'].sudo().set_param('my_module.enabled', 'True')


def uninstall_hook(cr, registry):
    env = api.Environment(cr, SUPERUSER_ID, {})
    env['ir.config_parameter'].sudo().search([('key', '=like', 'my_module.%')]).unlink()


def post_load():
    """Runs once per server process on import. No args, no DB access."""
```

> **13.0 vs later versions:** install hooks receive `(cr, registry)`, NOT a
> single `env` argument.

### Subpackage `__init__.py` files

```python
# models/__init__.py -> from . import my_model, res_partner
# wizard/__init__.py -> from . import my_wizard
# controllers/__init__.py -> from . import main
# tests/__init__.py -> from . import test_my_model
```

Tests are only auto-discovered when `--test-enable` is active.

---

## `__manifest__.py` Keys & Defaults

The manifest is `__manifest__.py`. `__openerp__.py` is accepted as a fallback
(`MANIFEST_NAMES = ('__manifest__.py', '__openerp__.py')`) but is deprecated.

```python
# my_module/__manifest__.py
# -*- coding: utf-8 -*-
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'summary': 'Short description',
    'author': 'UncleCat',
    'license': 'LGPL-3',
    'category': 'Tools',
    'depends': ['base'],
    'data': [
        'security/my_module_groups.xml',
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
        'views/my_module_menus.xml',
    ],
    'demo': ['demo/my_module_demo.xml'],
    'installable': True,
    'application': False,
    'auto_install': False,
}
```

### Default values

When a key is absent, `odoo/modules/module.py`
(`load_information_from_description_file`) fills these defaults:

| Key | Default |
|-----|---------|
| `application` | `False` |
| `author` | `'Odoo S.A.'` |
| `auto_install` | `False` |
| `category` | `'Uncategorized'` |
| `depends` | `[]` |
| `description` | `''` (falls back to README content) |
| `installable` | `True` |
| `license` | `'LGPL-3'` |
| `post_load` | `None` |
| `version` | `'1.0'` (normalized by `adapt_version`) |
| `web` | `False` |
| `sequence` | `100` |
| `summary` | `''` |
| `website` | `''` |

### Key semantics

- `version`: `13.0.X.Y.Z` scheme (`version_info = (13, 0, 0, FINAL, 0, '')`).
- `depends`: technical module names.
- `data`: XML/CSV loaded on install/update. Order matters (security first).
- `demo`: loaded only in demonstration mode.
- `auto_install`: `False`, or a list/set of dependency names (must be a subset
  of `depends`). A truthy non-list becomes `set(depends)`; `[]` = always auto-install.
- `application`: `True` marks the module as an installable App.
- `qweb`: list of `static/src/xml/*.xml` JS QWeb templates (see
  [Asset Bundles](#asset-bundles)).
- `pre_init_hook` / `post_init_hook` / `uninstall_hook`: function names in the
  root `__init__.py`; `post_load` is a no-arg function.
- `bootstrap`: `True` loads `i18n/<lang>.po` for the login screen.

### Not available in 13.0

- No `assets` manifest key (assets are declared in XML).
- No `external_dependencies` key read by core module loading.

---

## Models

```python
from odoo import api, fields, models


class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
```

Super-classes: `models.Model` (persisted), `models.TransientModel` (wizards,
auto-vacuumed), `models.AbstractModel` (alias of `BaseModel` for mixins).

### Fields available in 13.0

`Boolean`, `Integer`, `Float`, `Monetary`, `Char`, `Text`, `Html`, `Date`,
`Datetime`, `Binary`, `Image`, `Selection`, `Reference`, `Many2one`,
`Many2oneReference`, `One2many`, `Many2many`, `Id`.

Not present: `Json`, `Properties`. `oldname` is in `DEPRECATED_ATTRS`
(`odoo/fields.py`) - do not recommend it; use an upgrade script.

```python
amount = fields.Monetary(currency_field='currency_id')
quantity = fields.Float(digits='Product Unit of Measure')
```

`decimal_precision.get_precision()` is deprecated in 13.0: use
`digits='<precision name>'` instead.

### Decorators in `odoo/api.py`

`@api.constrains`, `@api.onchange`, `@api.depends`, `@api.depends_context`,
`@api.returns`, `@api.model`, `@api.model_create_single`,
`@api.model_create_multi`.

Not present: `@api.multi`, `@api.one`, `@api.cr`, `@api.model_cr`,
`@api.ondelete`, `@api.autovacuum`. Methods are multi-record by default in 13.0
(loop over `self`); `@api.multi` is implied, not written.

### Environment helpers

Available: `sudo()`, `with_user()`, `with_context()`, `with_env()`,
`with_prefetch()`; and `env.user`, `env.company`, `env.companies`, `env.lang`,
`env.ref(xml_id)`.

Not available: `with_company()` (14.0+).

### ORM search/grouping

`read_group` exists; `_read_group` does NOT (later private API):

```python
def read_group(self, domain, fields, groupby,
               offset=0, limit=None, orderby=False, lazy=True):
    ...
```

Name resolution: `name_get()`, `name_search()`, `_name_search()`. The
`_compute_display_name` method feeds the `display_name` field. `_rec_names_search`
is NOT available in 13.0.

### X2many write commands (no `Command` class)

`odoo.Command` does not exist in 13.0. Use raw tuples:

| Tuple | Meaning |
|-------|---------|
| `(0, 0, vals)` | create a record |
| `(1, id, vals)` | update the linked record |
| `(2, id, 0)` | unlink and delete |
| `(3, id, 0)` | unlink (detach) |
| `(4, id, 0)` | link an existing record |
| `(5, 0, 0)` | unlink all |
| `(6, 0, ids)` | replace the set |

---

## Views, Menus & Actions

### View types in 13.0

`tree`, `form`, `graph`, `pivot`, `calendar`, `diagram`, `gantt`, `kanban`, `search`, `qweb`. `activity` is added by the `mail` module via `selection_add`.

### `attrs` and `states` (not expression strings)

Dynamic visibility/readonly/required uses `attrs`/`states`, which `ir_ui_view`
converts into `modifiers`:

```xml
<field name="state" invisible="1"/>
<field name="reason"
       attrs="{'invisible': [('state', '=', 'draft')],
               'required': [('state', '=', 'confirmed')]}"/>
<button name="action_cancel" states="draft,confirmed"/>
```

Plain `invisible`/`readonly`/`required` attributes work, but expression strings
such as `invisible="state == 'done'"` are 14.0+ - do NOT use them in 13.0.

### View, action, and menu

```xml
<!-- views/my_model_views.xml -->
<odoo>
    <record id="view_my_model_form" model="ir.ui.view">
        <field name="name">my.model.form</field>
        <field name="model">my.model</field>
        <field name="arch" type="xml">
            <form>
                <header>
                    <button name="action_confirm" type="object" string="Confirm" class="btn-primary"/>
                    <field name="state" widget="statusbar"/>
                </header>
                <sheet>
                    <group>
                        <field name="name"/>
                        <field name="state"/>
                    </group>
                </sheet>
            </form>
        </field>
    </record>

    <record id="action_my_model" model="ir.actions.act_window">
        <field name="name">My Models</field>
        <field name="res_model">my.model</field>
        <field name="view_mode">tree,form</field>
    </record>
</odoo>
```

```xml
<!-- views/my_module_menus.xml -->
<odoo>
    <menuitem id="menu_my_module_root" name="My Module" sequence="20"/>
    <menuitem id="menu_my_model" name="My Models"
              parent="menu_my_module_root" action="action_my_model"/>
</odoo>
```

Menus reference actions, so actions load before the menus that point at them
(menus last in `data`).

---

## Security Declaration

Security files go first in `data`, because views/reports/menus reference the
groups and ACLs they define:

```python
'data': [
    'security/my_module_groups.xml',     # groups first
    'security/ir_rule.xml',              # then record rules
    'security/ir.model.access.csv',      # then model ACL
    # ...views, reports, menus...
],
```

### `ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model manager,model_my_model,my_module.group_my_module_manager,1,1,1,1
```

- `model_id:id` uses the auto-generated `model_<name>` external id.
- An empty `group_id:id` applies to every user (implicit public group).

### Groups

```xml
<!-- security/my_module_groups.xml -->
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="group_my_module_user" model="res.groups">
        <field name="name">User</field>
    </record>

    <record id="group_my_module_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
    </record>
</odoo>
```

### Record rules

```xml
<!-- security/ir_rule.xml -->
<?xml version="1.0" encoding="utf-8"?>
<odoo noupdate="1">
    <record id="my_model_personal_rule" model="ir.rule">
        <field name="name">My Model: own records</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[('user_id', '=', user.id)]</field>
        <field name="groups" eval="[(4, ref('group_my_module_user'))]"/>
    </record>

    <record id="my_model_manager_rule" model="ir.rule">
        <field name="name">My Model: all records</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[(1, '=', 1)]</field>
        <field name="groups" eval="[(4, ref('group_my_module_manager'))]"/>
    </record>
</odoo>
```

`ir.rule` fields in 13.0: `name`, `active`, `model_id`, `groups`,
`domain_force`, `perm_read`/`perm_write`/`perm_create`/`perm_unlink` (each
defaulting to `True`).

---

## Wizards & `TransientModel`

Wizards are short-lived models vacuumed periodically (`_transient_max_hours`
default 1 hour).

```python
# wizard/my_wizard.py
from odoo import api, fields, models
from odoo.exceptions import UserError


class MyWizard(models.TransientModel):
    _name = 'my.wizard'
    _description = 'My Action Wizard'

    date = fields.Date(default=fields.Date.context_today, required=True)
    user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
    record_ids = fields.Many2many('my.model', string='Records')

    @api.model
    def default_get(self, fields_list):
        res = super(MyWizard, self).default_get(fields_list)
        if 'record_ids' in fields_list and self.env.context.get('active_model') == 'my.model':
            res['record_ids'] = [(6, 0, self.env.context.get('active_ids', []))]
        return res

    def action_apply(self):
        self.ensure_one()
        if not self.record_ids:
            raise UserError('Select at least one record.')
        for rec in self.record_ids:
            rec.action_done(self.date)
        return {'type': 'ir.actions.act_window_close'}
```

The wizard view opens in a dialog via `target="new"`:

```xml
<odoo>
    <record id="view_my_wizard_form" model="ir.ui.view">
        <field name="name">my.wizard.form</field>
        <field name="model">my.wizard</field>
        <field name="arch" type="xml">
            <form string="My Wizard">
                <group>
                    <field name="date"/>
                    <field name="user_id"/>
                </group>
                <field name="record_ids" invisible="1"/>
                <footer>
                    <button name="action_apply" type="object" string="Apply" class="btn-primary"/>
                    <button string="Cancel" special="cancel"/>
                </footer>
            </form>
        </field>
    </record>

    <record id="action_my_wizard" model="ir.actions.act_window">
        <field name="name">Run My Wizard</field>
        <field name="res_model">my.wizard</field>
        <field name="view_mode">form</field>
        <field name="target">new</field>
    </record>
</odoo>
```

---

## Reports

Declared in `data` before menus (reports can be referenced from view buttons).

### Action

The core `<report>` shorthand maps `name` -> `report_name` and `file` ->
`report_file`:

```xml
<!-- report/my_report_actions.xml -->
<odoo>
    <report id="action_report_my_model" string="My Model Report"
            model="my.model" report_type="qweb-pdf"
            name="my_module.report_my_model" file="my_module.report_my_model"/>
</odoo>
```

`ir.actions.report` fields in 13.0: `report_type`, `report_name`,
`report_file`, `paperformat_id`, `attachment`, `attachment_use`,
`print_report_name`, `binding_model_id`, `binding_type`, `groups_id`, `multi`.

| `report_type` | Output |
|---------------|--------|
| `qweb-pdf`    | PDF via wkhtmltopdf (default). |
| `qweb-html`   | HTML viewed in the browser. |
| `qweb-text`   | Plain text (labels, barcodes). |

### Template

```xml
<!-- report/my_report_templates.xml -->
<odoo>
    <template id="report_my_model_document">
        <t t-call="web.external_layout">
            <div class="page">
                <h2 t-field="doc.name"/>
                <p t-field="doc.description"/>
            </div>
        </t>
    </template>

    <template id="report_my_model">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="my_module.report_my_model_document"/>
            </t>
        </t>
    </template>
</odoo>
```

Layout templates: `web.external_layout`, `web.internal_layout`, `web.basic_layout`, `web.minimal_layout`, `web.html_container`.

### Server QWeb directives

`t-esc`, `t-raw`, `t-set`, `t-call`, `t-if`/`t-elif`/`t-else`,
`t-foreach`/`t-as`, `t-att-*`, `t-attf-*`, `t-field`, `t-options`,
`t-call-assets` (with `t-js`/`t-css`), `t-lang`, `t-debug`, `t-groups`,
`t-name`.

Not available: `t-out` (14.0+). `t-inherit`/`t-inherits` are NOT server QWeb
directives; `t-inherit` exists only for *static JS* QWeb templates (xpath
inheritance on the client bundle).
---

## Controllers

```python
# controllers/main.py
from odoo import http
from odoo.http import request


class MyController(http.Controller):

    @http.route('/my_module/hello', type='http', auth='public')
    def hello(self, **kw):
        return request.render('my_module.hello', {'message': 'Hello'})

    @http.route('/my_module/objects', type='http', auth='user', website=True)
    def list_objects(self, **kw):
        objects = request.env['my.model'].search([])
        return request.render('my_module.listing', {'objects': objects})

    @http.route('/my_module/do_thing', type='json', auth='user', methods=['POST'])
    def do_thing(self, **kw):
        # type='json': request.jsonrequest holds the body; the returned dict
        # is serialized back as JSON.
        return {'status': 'ok', 'received': request.jsonrequest.get('value')}
```

`@http.route` parameters: `route`, `type` (`'http'` | `'json'`), `auth`
(`'user'` | `'public'` | `'none'`), `methods`, `csrf` (default `True`), `cors`,
`website`, `sitemap`.

`request` exposes `env`, `params`, `jsonrequest` (JSON routes), `session`,
`httprequest`, `csrf_token()`, `render()`, `make_response()`, `redirect()`,
`not_found()`. `request.website` is provided by the `website` module. For CSRF:
`csrf` defaults to `True` for unsafe methods; QWeb forms already carry the token,
and JS-generated forms obtain it via `request.csrf_token()`.

---

## Asset Bundles

There is **no `assets` manifest key** in Odoo 13. Assets are declared as XML
templates that inherit bundle templates; JS QWeb templates go in the manifest
`qweb` key.

### Bundle template ids

Verified in `addons/web/views/webclient_templates.xml` and
`addons/web/views/report_templates.xml`:

| Bundle | Use for |
|--------|---------|
| `web.assets_common` | Assets shared by frontend and backend. |
| `web.assets_backend` | Backend JS/SCSS. |
| `web.assets_frontend` | Website/portal assets. |
| `web.report_assets_common` | SCSS/JS used when rendering QWeb reports. |
| `web.report_assets_pdf` | Extra PDF-only CSS. |

There is **no** `web.assets_qweb` bundle and no `ir.asset` model in 13.0
(both are later additions).

### Adding assets via XML inheritance

Canonical pattern (mirrors `addons/mail/views/mail_templates.xml`):

```xml
<!-- views/assets.xml -->
<odoo>
    <template id="assets_backend" name="my_module assets"
              inherit_id="web.assets_backend">
        <xpath expr="." position="inside">
            <link rel="stylesheet" type="text/scss"
                  href="/my_module/static/src/scss/my_widget.scss"/>
            <script type="text/javascript"
                    src="/my_module/static/src/js/my_widget.js"/>
        </xpath>
    </template>
</odoo>
```

List the file in `data` so it loads on install/update. To target frontend or
reports, inherit `web.assets_frontend`, `web.report_assets_common`, or
`web.report_assets_pdf`.

### JS QWeb templates via the `qweb` key

Client-side QWeb templates are declared in the manifest `qweb` key and
concatenated by the web client:

```python
'qweb': [
    'static/src/xml/my_widget.xml',
],
```

The files use a `<templates>` root (NOT `<odoo>`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
    <t t-name="MyWidget">
        <div class="o_my_widget">
            <t t-esc="widget.title"/>
        </div>
    </t>
</templates>
```

### Legacy AMD JavaScript

13.0 has **no OWL**. Client modules use the legacy AMD loader:

```javascript
odoo.define('my_module.my_widget', function (require) {
    "use strict";

    var Widget = require('web.Widget');
    var field_registry = require('web.field_registry');

    var MyWidget = Widget.extend({
        init: function (parent, options) {
            this._super.apply(this, arguments);
        },
        start: function () {
            this.$el.append('<span/>');
            return this._super.apply(this, arguments);
        },
    });

    field_registry.add('my_widget', MyWidget);
    return MyWidget;
});
```

Common `require()` modules: `web.Widget`, `web.AbstractField`,
`web.field_registry`, `web.view_registry`, `web.core` (`core.qweb`,
`core.action_registry`), `web.rpc`, `web.session`, `web.dom`, `web.utils`.

Widget lifecycle: `init(parent, options)`, `willStart()`, `start()`,
`destroy()`. Instance helpers: `this.$el`, `this.$()`, `this._rpc()`,
`this.do_action()`, `this.trigger_up()`, `this.call('service', 'method', args)`.

---

## Static Assets Layout

```
static/
|-- description/icon.png     # auto-discovered by the Apps UI
`-- src/
    |-- js/                  # AMD modules (odoo.define)
    |-- xml/                 # JS QWeb templates (manifest 'qweb' key)
    |-- scss/  css/  lib/  img/
```

- Backend JS/SCSS files are wired into `web.assets_backend` via XML inheritance;
  `static/src/xml/*.xml` JS templates go in the manifest `qweb` key.
- Referenced URLs are absolute: `/my_module/static/src/js/my_widget.js`.

---

## i18n (Translations)

Translations live in `i18n/` as `.po` (or `.csv`) files, one per locale, plus a
`<module>.pot` template. The `.pot` is not loaded at install; locale
`.po`/`.csv` files load automatically on install/update. `ir.translation`
stores terms in the database; `odoo.tools.translate` provides `_()`, `_lt()`
(`class _lt`), `GettextAlias`.

```bash
# export translatable terms for a module to a .pot
./odoo-bin -d <db> --modules=my_module --i18n-export=my_module/i18n/my_module.pot

# export one language to a .po
./odoo-bin -d <db> -l fr_FR --modules=my_module --i18n-export=my_module/i18n/fr.po

# import a .po/.csv and optionally overwrite existing terms
./odoo-bin -d <db> -l fr_FR --i18n-import=my_module/i18n/fr.po --i18n-overwrite
```

`.pot` skeleton (header carries `Project-Id-Version: Odoo Server 13.0`):

```po
msgid ""
msgstr ""
"Project-Id-Version: Odoo Server 13.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Plural-Forms: \n"

#. module: my_module
#: model:ir.model,name:my_module.model_my_model
msgid "My Model"
msgstr ""
```

---

## Tests

```python
# tests/test_my_model.py
from odoo.tests.common import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestMyModel(TransactionCase):

    def setUp(self):
        super(TestMyModel, self).setUp()
        self.record = self.env['my.model'].create({'name': 'Test'})

    def test_confirm_sets_state(self):
        self.record.action_confirm()
        self.assertEqual(self.record.state, 'confirmed')
```

Test classes: `TransactionCase`, `SingleTransactionCase`, `SavepointCase`,
`HttpCase`, `Form`, `O2MForm`, `M2MProxy`, `assertQueryCount`. Decorators:
`@tagged`, `@at_install`, `@post_install`, `@users`. The default tag is
`standard` plus the module technical name.

`HttpCase` provides `browser_js(url_path, code, ready='', login=None,
timeout=60, **kw)` and the legacy alias `phantom_js = browser_js` (both present
in 13.0). A `ChromeBrowser` class also exists for headless-Chrome runs.

---

## Install / Update CLI

```bash
# install a module (requires a database)
./odoo-bin -d <db> --addons-path=addons,my_addons -i my_module

# update a module
./odoo-bin -d <db> --addons-path=addons,my_addons -u my_module

# install and exit after initialization (CI / scripting)
./odoo-bin -d <db> --addons-path=addons,my_addons -i my_module --stop-after-init

# run tests
./odoo-bin -d <db> --addons-path=addons,my_addons -i my_module \
    --test-enable --test-tags=/my_module

# developer mode
./odoo-bin -d <db> --addons-path=addons,my_addons -u my_module --dev=all
```

### Options (verified in `odoo/tools/config.py`)

| Option | Meaning |
|--------|---------|
| `-i`, `--init` | install modules (comma-separated; `all` = all). Requires `-d`. |
| `-u`, `--update` | update modules (comma-separated). Requires `-d`. |
| `-d`, `--database` | database name. |
| `--addons-path` | comma-separated addons paths. |
| `--stop-after-init` | stop the server after initialization. |
| `--dev` | `all`, `[pudb\|wdb\|ipdb\|pdb]`, `reload`, `qweb`, `werkzeug`, `xml`. |
| `--test-enable` | enable unit tests. |
| `--test-tags` | comma-separated filter specs (implies `--test-enable`). |
| `--test-file` | run one Python test file. |
| `--without-demo` | disable demo data loading. |
| `--log-level` | `critical\|error\|warn\|info\|debug\|debug_rpc\|debug_rpc_answer\|debug_sql`. |
| `--logfile` | log file path. |
| `--upgrade-path` | additional upgrade path (migration scripts). |
| `-l`, `--language` | language for `--i18n-export` / `--i18n-import`. |
| `--i18n-export` | export translations to CSV/PO/TGZ and exit. |
| `--i18n-import` | import translations and exit. |
| `--i18n-overwrite` | overwrite existing translation terms. |
| `--shell-interface` | `ipython\|ptpython\|bpython\|python`. |

### Subcommands

`odoo-bin` exposes `scaffold`, `shell`, `start`, `deploy`, `cloc`, and the
default `server`:

```bash
./odoo-bin scaffold my_module <dest>            # default template
./odoo-bin scaffold -t theme my_theme <dest>    # theme template
```

There is no `upgrade_code` CLI command in 13.0. Schema/data migrations use
`migrations/<version>/pre-*.py`, `post-*.py`, and `end-*.py`, each exposing
`migrate(cr, version)`.

---

## Base Code Reference

- `odoo/modules/module.py` - `MANIFEST_NAMES`; manifest defaults in
  `load_information_from_description_file`; `auto_install` normalization;
  `adapt_version`; `post_load` invocation.
- `odoo/modules/loading.py` - hooks: `pre_init_hook(cr)`, `post_init_hook(cr,
  registry)`, `uninstall_hook(cr, registry)`.
- `odoo/modules/migration.py` - `migrations/<version>/pre|post|end-*.py` with
  `migrate(cr, version)`; `0.0.0` special folder.
- `odoo/api.py` - decorators (`constrains`, `onchange`, `depends`,
  `depends_context`, `returns`, `model`, `model_create_single`,
  `model_create_multi`); `Environment` (`user`, `company`, `companies`, `lang`,
  `ref`).
- `odoo/fields.py` - field classes; `DEPRECATED_ATTRS` (`oldname`);
  `get_digits()` accepting a precision-name string.
- `odoo/models.py` - `Model`, `TransientModel`, `AbstractModel = BaseModel`;
  `read_group`, `name_get`, `name_search`, `_name_search`, `sudo` / `with_user` /
  `with_context` / `with_env` / `with_prefetch`.
- `odoo/http.py` - `route()` parameters (`type`, `auth`, `methods`, `csrf`,
  `cors`); `request` helpers (`render`, `make_response`, `redirect`,
  `not_found`, `csrf_token`).
- `odoo/exceptions.py` - `except_orm` (deprecated), `UserError`,
  `RedirectWarning`, `AccessDenied`, `AccessError`, `CacheMiss`,
  `MissingError`, `ValidationError`, `DeferredException`, `QWebException`.
- `odoo/tests/common.py` - `TransactionCase`, `SingleTransactionCase`,
  `SavepointCase`, `HttpCase`, `Form`, `O2MForm`, `M2MProxy`, `assertQueryCount`,
  `ChromeBrowser`, `tagged`, `at_install`, `post_install`, `users`;
  `phantom_js = browser_js`.
- `odoo/addons/base/models/ir_ui_view.py` - view types; `transfer_node_to_modifiers`
  / `transfer_field_to_modifiers` (the `attrs`/`states` conversion).
- `odoo/addons/base/models/qweb.py` and `ir_qweb.py` - server QWeb directives
  and evaluation order; `t-call-assets` with `t-js`/`t-css`.
- `odoo/addons/base/models/ir_actions_report.py` - `report_type`
  (`qweb-pdf`/`qweb-html`/`qweb-text`), `report_name`, `report_file`,
  `paperformat_id`, `attachment`, `attachment_use`, `binding_model_id`.
- `addons/web/views/webclient_templates.xml` - bundle ids (`web.assets_common`,
  `web.assets_backend`, `web.assets_frontend`).
- `addons/web/views/report_templates.xml` - `web.report_assets_common`,
  `web.report_assets_pdf`, `web.external_layout`, `web.internal_layout`,
  `web.basic_layout`, `web.html_container`.
- `addons/web/controllers/main.py` - `manifest_glob('qweb', ...)` (manifest
  `qweb` key consumption); `HomeStaticTemplateHelpers` (static JS QWeb
  templates, `<templates>` root, `t-inherit`).
- `addons/web/static/src/js/**` - `web.Widget`, `web.AbstractField`,
  `web.field_registry`, `web.view_registry`, `web.core`, `web.rpc`,
  `web.session`, `web.dom`, `web.utils` (legacy AMD, no OWL).
- `addons/mail/models/mail_thread.py` - `tracking` and deprecated
  `track_visibility`; `mail_activity.py` (`mail.activity.mixin`),
  `mail_alias.py` (`mail.alias.mixin`), `mail_cc_mixin.py` (`mail.thread.cc`).
- `odoo/addons/base/models/decimal_precision.py` - `decimal.precision` model;
  deprecated `get_precision`.
- `odoo/addons/base/models/ir_translation.py` - `ir.translation` model.
- `odoo/tools/translate.py` - `_()`, `class _lt`, `GettextAlias`.
- `odoo/tools/config.py` - `-i`/`-u`/`-d`, `--addons-path`, `--stop-after-init`,
  `--dev`, `--test-enable`, `--test-tags`, `--test-file`, `--log-level`,
  `--logfile`, `--upgrade-path`, `-l`, `--i18n-export/import/overwrite`,
  `--shell-interface`.
- `odoo/cli/scaffold.py` and `odoo/cli/templates/default/` - `odoo-bin scaffold`.
- `odoo/release.py` - `version_info = (13, 0, 0, FINAL, 0, '')`,
  `major_version = '13.0'`.
