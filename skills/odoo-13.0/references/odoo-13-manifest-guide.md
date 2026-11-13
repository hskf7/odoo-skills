---
name: odoo-13-manifest
description: Complete reference for Odoo 13 module manifest (__manifest__.py) covering every key the 13.0 loader understands, defaults, dependencies, data loading order, the qweb key, XML asset declarations, external dependencies, hooks, auto_install, version scheme, and license identifiers.
globs: "**/__manifest__.py"
topics:
  - Every valid __manifest__.py key in Odoo 13
  - Manifest defaults from load_information_from_description_file
  - Module dependencies and loading order
  - `data` list ordering (security first, views last)
  - The `qweb` key for static JavaScript templates
  - XML asset declarations (no `assets` manifest key in 13.0)
  - External dependencies (python, bin)
  - Hooks (pre_init_hook, post_init_hook, uninstall_hook, post_load)
  - auto_install (bool or list of dependencies)
  - `installable`, `application`, `sequence`, `summary`, `website`, `icon`, `web`, `bootstrap`
  - Version scheme 13.0.X.Y.Z and adapt_version()
  - License identifiers
when_to_use:
  - Authoring a new Odoo 13 module manifest
  - Configuring module dependencies or data loading order
  - Declaring JavaScript QWeb templates with the `qweb` key
  - Adding backend or frontend assets in Odoo 13
  - Declaring external Python or binary dependencies
  - Setting up module lifecycle hooks
  - Deciding between `auto_install=True` and an explicit trigger list
  - Checking whether a manifest key exists in 13.0
---

# Odoo 13 Module Manifest Guide

Complete reference for the Odoo 13 `__manifest__.py`: the filenames the loader accepts, every key
`load_information_from_description_file` understands, the defaults it applies, dependency and data
loading rules, the `qweb` key, XML asset declarations, hooks, and version rules.

Odoo 13 has no `assets` manifest key. Assets are declared in XML templates that inherit the
`web.assets_*` templates. This guide shows the 13.0 pattern and labels later-version constructs
explicitly when they are mentioned.

## Table of Contents

1. [Manifest Basics](#manifest-basics)
2. [Default Values](#default-values)
3. [Core Identification Fields](#core-identification-fields)
4. [Version Scheme (13.0.X.Y.Z)](#version-scheme-130xyz)
5. [License Identifiers](#license-identifiers)
6. [Dependencies](#dependencies)
7. [Data Loading](#data-loading)
8. [QWeb Templates (`qweb` key)](#qweb-templates-qweb-key)
9. [Assets in Odoo 13](#assets-in-odoo-13)
10. [External Dependencies](#external-dependencies)
11. [Hooks](#hooks)
12. [Other Fields](#other-fields)
13. [Complete Example](#complete-example)
14. [Common Patterns](#common-patterns)
15. [Base Code Reference](#base-code-reference)

---

## Manifest Basics

### File Name

Odoo 13 looks for these filenames, in this order, via `MANIFEST_NAMES` in `odoo/modules/module.py`:

```python
MANIFEST_NAMES = ('__manifest__.py', '__openerp__.py')
```

Always use `__manifest__.py` for new modules. `__openerp__.py` is still accepted by the 13.0
loader for legacy modules, but it is not the preferred name. `module_manifest()` checks
`MANIFEST_NAMES` in order and returns the first file that exists.

### Manifest Shape

The file is read as bytes, decoded to text, and evaluated with `ast.literal_eval`. It must contain
exactly one Python literal expression that evaluates to a `dict`. No imports, no function calls, no
names, no expressions:

```python
# -*- coding: utf-8 -*-
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'depends': ['base'],
    'data': [],
    'installable': True,
    'license': 'LGPL-3',
    'author': 'UncleCat',
}
```

`ast.literal_eval` accepts strings, bytes, numbers, tuples, lists, dicts, sets, booleans, and
`None`. It does not accept `datetime`, `Path`, or any computed value. Keep the manifest a plain dict
literal.

### Minimal Installable Manifest

```python
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'license': 'LGPL-3',
    'author': 'UncleCat',
    'depends': ['base'],
    'installable': True,
}
```

### What the Loader Does With It

`load_information_from_description_file(module, mod_path=None)` starts from the built-in defaults,
updates them with the literal dict, falls back to `README.rst`, `README.md`, or `README.txt` when
`description` is empty, normalizes `auto_install` into `False` or a `set` of module names, and
normalizes `version` through `adapt_version()`. The resulting dict is stored on `ir.module.module`
and is also available to the web client through the addons manifest.

---

## Default Values

Every key the 13.0 loader understands appears in the `info` dict built inside
`load_information_from_description_file` (`odoo/modules/module.py`). Keys not present in your
manifest fall back to these defaults:

| Key | Default | Notes |
|-----|---------|-------|
| `name` | - | **Required.** No default is supplied. |
| `version` | `'1.0'` | Normalized by `adapt_version()` to `'13.0.1.0'`. |
| `license` | `'LGPL-3'` | Applied silently when missing. |
| `author` | `'Odoo S.A.'` | Override in every custom module. |
| `category` | `'Uncategorized'` | |
| `summary` | `''` | One-line description. |
| `description` | `''` | RST; falls back to `README.rst/md/txt`. |
| `website` | `''` | |
| `sequence` | `100` | Ordering in the Apps list. |
| `depends` | `[]` | |
| `data` | `[]` | |
| `demo` | `[]` | |
| `demo_xml` | `[]` | Legacy alias, avoid. |
| `init_xml` | `[]` | Legacy alias, avoid. |
| `update_xml` | `[]` | Legacy alias, avoid. |
| `test` | `[]` | Legacy key, not used for test discovery. |
| `installable` | `True` | |
| `application` | `False` | |
| `auto_install` | `False` | `True`, `False`, or an iterable of module names. |
| `icon` | computed | `/module/static/description/icon.png` or `/base/static/description/icon.png`. |
| `web` | `False` | Internal flag, do not set. |
| `post_load` | `None` | Name of a function in the module package. |

Keys that are not in the defaults but are read by the 13.0 loader or by `ir.module.module`:

| Key | Read by | Notes |
|-----|---------|-------|
| `external_dependencies` | `ir.module.module._check_external_dependencies` | `python` and `bin` lists. |
| `pre_init_hook` | `odoo/modules/loading.py` | Called with `cr`. |
| `post_init_hook` | `odoo/modules/loading.py` | Called with `cr, registry`. |
| `uninstall_hook` | `odoo/modules/loading.py` | Called with `cr, registry`. |
| `qweb` | `addons/web/controllers/main.py` | Static QWeb template paths. |
| `bootstrap` | `addons/web/controllers/main.py` | Used by the `web` module only. |
| `maintainer` | `ir.module.module.get_values_from_terp` | Stored as metadata. |
| `contributors` | `ir.module.module.get_values_from_terp` | List joined with commas. |
| `url` | `ir.module.module.get_values_from_terp` | Falls back to `live_test_url`. |
| `live_test_url` | `ir.module.module.get_values_from_terp` | Fallback for `url`. |

Keys not listed here are accepted as plain metadata. They are stored in the manifest dict but are
not interpreted by the 13.0 loader.

> **Not a 13.0 manifest key:** `assets`. The 13.0 loader has no `assets` handling, there is no
> `ir.asset` model, and there is no `web.assets_qweb` bundle. See
> [Assets in Odoo 13](#assets-in-odoo-13).

---

## Core Identification Fields

| Key | Default | Notes |
|-----|---------|-------|
| `name` | required | Human-readable name in the Apps list. |
| `summary` | `''` | One-line description under the module title. |
| `description` | `''` | RST; falls back to `README.rst/md/txt`. |
| `author` | `'Odoo S.A.'` | Person, company, or comma-separated list. |
| `maintainer` | absent | Stored as metadata on `ir.module.module`. |
| `contributors` | absent | List joined with `, ` by `get_values_from_terp`. |
| `website` | `''` | Module homepage or project page. |
| `category` | `'Uncategorized'` | Slashes create a hierarchy. |
| `icon` | computed | `static/description/icon.png` or `/base/static/description/icon.png`. |
| `sequence` | `100` | Apps list ordering; `ir.module.module` orders by `sequence, name`. |
| `application` | `False` | Marks the module as an application. |
| `installable` | `True` | `False` stores state `uninstallable` and hides it from Apps. |
| `web` | `False` | Internal flag; leave it unset in custom modules. |
| `bootstrap` | absent | Used by the `web` module for login-screen translations. |

```python
{
    'name': 'Business Trip Management',
    'summary': 'Plan business trips and track expenses',
    'description': """
Business Trip Management
========================
Manage business trips, per-diems, and expense reports.
""",
    'author': 'UncleCat, Acme Corp.',
    'maintainer': 'UncleCat',
    'contributors': ['Alice', 'Bob'],
    'website': 'https://example.com',
    'category': 'Sales/Subscriptions',
    'icon': '/my_module/static/description/icon.png',
    'sequence': 20,
    'application': True,
    'installable': True,
    'web': False,
    'bootstrap': False,
}
```

`category` values such as `Sales/Subscriptions` create a hierarchy. Common roots include
`Accounting`, `Human Resources`, `Inventory`, `Manufacturing`, `Marketing`, `Point of Sale`,
`Productivity/Discuss`, `Project`, `Sales`, `Services`, `Tools`, `Warehouse`, `Website`, and
`Hidden` for technical-only modules.

When `icon` is omitted, `get_module_icon()` looks for `static/description/icon.png` inside the
module and falls back to `/base/static/description/icon.png`. The value is stored on
`ir.module.module.icon`.

`installable: False` stores the module with state `uninstallable` instead of `uninstalled`; the
manifest is still parsed. `bootstrap` is read by the `/web/webclient/bootstrap_translations`
controller, which loads the module's `i18n/<lang>.po` when the flag is true. Do not set it in
ordinary business modules.

---

## Version Scheme (13.0.X.Y.Z)

### Format

Odoo 13 normalizes `version` through `adapt_version()` in `odoo/modules/module.py`:

```python
def adapt_version(version):
    serie = release.major_version
    if version == serie or not version.startswith(serie + '.'):
        version = '%s.%s' % (serie, version)
    return version
```

`release.major_version` is `'13.0'` (`odoo/release.py`, `version_info = (13, 0, 0, FINAL, 0, '')`).
Consequences:

```python
'version': '1.0'          # becomes '13.0.1.0'
'version': '1.0.0'        # becomes '13.0.1.0.0'
'version': '13.0.1.0.0'   # stays  '13.0.1.0.0'
'version': '13.0'         # becomes '13.0.13.0'
```

The last case is a quirk of the exact `version == serie` check. Do not write a bare `'13.0'`;
write the full five-part version.

### Recommended

Always write the full form so the manifest is self-documenting and so migration folders line up
with the declared version:

```python
'version': '13.0.1.0.0',
```

Interpretation (convention, not enforced): `13.0` is the target Odoo series, `1` is the module
MAJOR (incompatible schema changes), `0` is MINOR (backwards-compatible features), and `0` is PATCH
(bug fixes).

### Bumping for Migrations

The migration manager (`odoo/modules/migration.py`) reads folders under `migrations/` and runs
Python files whose names start with `pre-`, `post-`, or `end-`. Each script must define
`migrate(cr, installed_version)`.

A folder may be named with the module version alone (`1.1.0`) or with the server and module version
(`13.0.1.1.0`). The manager prepends `release.major_version` when the folder name has fewer than
two dots. The special folder `0.0.0` runs on any version change.

Bump the manifest version before committing a migration script. A script under
`migrations/13.0.1.1.0/` only runs when the installed version is lower than the new version.

---

## License Identifiers

The 13.0 loader defaults `license` to `'LGPL-3'` and does not warn when the key is missing. Always
declare it explicitly. The accepted values are the selection values on `ir.module.module.license`
(`odoo/addons/base/models/ir_module.py`):

| Identifier | Description |
|------------|-------------|
| `GPL-2` | GNU General Public License v2. |
| `GPL-2 or any later version` | GPL v2 or later. |
| `GPL-3` | GNU General Public License v3. |
| `GPL-3 or any later version` | GPL v3 or later. |
| `AGPL-3` | GNU Affero General Public License v3. |
| `LGPL-3` | GNU Lesser General Public License v3 (Odoo default). |
| `Other OSI approved licence` | Any OSI-approved license. |
| `OEEL-1` | Odoo Enterprise Edition License v1.0. |
| `OPL-1` | Odoo Proprietary License v1.0. |
| `Other proprietary` | Closed-source custom license. |

Use the exact strings, including the spelling `licence` in `Other OSI approved licence`.

---

## Dependencies

### `depends`

```python
'depends': ['base', 'mail', 'product'],
```

List of modules that must be installed before this one. Odoo resolves the dependency graph and
installs or loads them first. Default `[]`.

- Always include `base` explicitly, even when it is transitive.
- Depend on `mail` when you use `mail.thread` or `mail.activity.mixin`.
- Depend on `web` when you add backend assets or JavaScript QWeb templates.
- Avoid circular dependencies; the graph loader rejects them.

### `auto_install`

The 13.0 loader accepts three shapes:

```python
# 1) Opt-in (default)
'auto_install': False,

# 2) Auto-install once every module in `depends` is installed
'auto_install': True,

# 3) Auto-install once this specific subset is installed
#    (the subset must be a subset of `depends`)
'auto_install': ['sale', 'purchase'],
```

The normalization in `load_information_from_description_file` is:

```python
auto_install = info.get('auto_install', info.get('active', False))
if isinstance(auto_install, collections.abc.Iterable):
    info['auto_install'] = set(auto_install)
    non_dependencies = info['auto_install'].difference(info['depends'])
    assert not non_dependencies, ...
elif auto_install:
    info['auto_install'] = set(info['depends'])
else:
    info['auto_install'] = False
```

Notes:

- `True` becomes `set(depends)`, so the module auto-installs when all its dependencies are installed.
- A list becomes a set and must be a subset of `depends`. A non-dependency raises an
  `AssertionError` naming the offending modules.
- An empty list `[]` becomes an empty set, which means the module auto-installs regardless of its
  dependencies. This is the documented way to always auto-install a module.
- The legacy key `active` is read as a fallback when `auto_install` is absent.

Bridge modules typically declare both the dependency and the trigger:

```python
{
    'name': 'Sale - Purchase Bridge',
    'version': '13.0.1.0.0',
    'depends': ['sale', 'purchase'],
    'auto_install': ['sale', 'purchase'],
    'installable': True,
}
```

---

## Data Loading

### `data`

```python
'data': [
    # 1. Security FIRST (groups, ACL, record rules)
    'security/res_groups.xml',
    'security/ir_rules.xml',
    'security/ir.model.access.csv',
    # 2. Reference data (sequences, parameters, templates)
    'data/ir_sequence_data.xml',
    'data/mail_template_data.xml',
    # 3. Wizards
    'wizard/my_wizard_views.xml',
    # 4. Views
    'views/my_model_views.xml',
    'views/res_partner_views.xml',
    # 5. Reports (templates + actions)
    'report/my_report_templates.xml',
    'report/my_report_actions.xml',
    # 6. Menus LAST - they reference window actions defined above
    'views/my_menus.xml',
],
```

`load_data(cr, idref, mode, kind='data', ...)` in `odoo/modules/loading.py` expands `kind='data'`
to the keys `['init_xml', 'update_xml', 'data']` and loads the files in that order. For new
modules, put everything in `data`.

**Ordering rules:**

1. **Security first.** `ir.model.access.csv` and groups must exist before any view or record that
   references them.
2. **Records before their references.** A view that sets `action_id` must come after the action.
3. **Menus last.** They usually reference actions defined earlier.
4. **Wizards before view bindings** that embed wizard buttons.

Paths are relative to the module root. The loader resolves them through `tools.convert_file`, which
supports `.xml` (`convert_xml_import`), `.csv` (`convert_csv_import`), and `.sql`
(`convert_sql_import`). `.js` files are accepted but ignored by the data loader. Any other
extension raises `ValueError`.

### `demo`

```python
'demo': [
    'demo/my_module_demo.xml',
],
```

Demo files are loaded by `load_demo()` after `data`, and only when the database is running in demo
mode. `load_demo()` checks `package.should_have_demo()`, which requires the package to be in demo
mode and all parent packages to have demo data. A production database does not load `demo` files.

| Key | When loaded |
|-----|-------------|
| `data` | Install and update. |
| `demo` | Only when demo mode is active. |

### `demo_xml`, `init_xml`, `update_xml`

Legacy aliases from older Odoo versions. The 13.0 loader still reads them: `kind='data'` expands to
`['init_xml', 'update_xml', 'data']`, and `kind='demo'` expands to `['demo_xml', 'demo']`. When a
file is loaded from one of these legacy keys, the loader logs a warning:

```
module <name>: key 'init_xml' is deprecated in favor of 'data' for file '...'
```

Do not use them in new 13.0 modules.

### `test`

`test` is present in the defaults as an empty list, but the 13.0 test runner does not use it to
discover tests. Tests are discovered from the module's `tests/` package by `get_test_modules()`. Do
not declare a `test` key.

---

## QWeb Templates (`qweb` key)

Odoo 13 has no `assets` manifest key, but it does have a `qweb` key for static JavaScript QWeb
templates. The key is a list of XML files, usually under `static/src/xml/`:

```python
'qweb': [
    'static/src/xml/my_widget.xml',
    'static/src/xml/my_dialog.xml',
],
```

The web client serves these files through `/web/webclient/qweb/<unique>`. The controller calls
`manifest_glob('qweb', ...)`, which reads `manifest.get('qweb', [])` for each installed addon and
resolves each path relative to the addon directory.

The `web` module itself declares its templates this way (`addons/web/__manifest__.py`):

```python
'qweb': [
    "static/src/xml/base.xml",
    "static/src/xml/chart.xml",
    "static/src/xml/kanban.xml",
    "static/src/xml/menu.xml",
    "static/src/xml/notification.xml",
    "static/src/xml/rainbow_man.xml",
    "static/src/xml/report.xml",
    "static/src/xml/web_calendar.xml",
],
```

### Template File Shape

A 13.0 JavaScript template file uses a `<templates>` root and `<t t-name="...">` elements. There is
no `<odoo>` root tag in these files:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
    <t t-name="MyWidget">
        <div class="my-widget">
            <span t-esc="widget.title"/>
        </div>
    </t>
</templates>
```

The template name is the key used by `require('web.qweb')` or by a widget's `template` property.
Keep names unique across installed modules.

### `qweb` vs `data`

- `qweb` files are static templates for the JavaScript client. They are not loaded into the database
  and are not rendered by the server-side QWeb engine.
- `data` XML files are loaded into the database as records, including server-side QWeb view
  templates (`ir.ui.view`).

Do not put server-side view templates in `qweb`, and do not put JavaScript templates in `data`.

---

## Assets in Odoo 13

### No `assets` Manifest Key

Odoo 13 does not read an `assets` key from `__manifest__.py`. There is no `ir.asset` model and no
`web.assets_qweb` bundle. If you add an `assets` key, the loader stores it as plain metadata and
nothing loads it.

Assets are declared in XML by inheriting one of the `web.assets_*` templates and inserting
`<script>` or `<link>` elements. The canonical 13.0 pattern is:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data>
        <template id="assets_backend" name="my_module assets"
                  inherit_id="web.assets_backend">
            <xpath expr="." position="inside">
                <link rel="stylesheet" type="text/scss"
                      href="/my_module/static/src/scss/my_module.scss"/>
                <script type="text/javascript"
                        src="/my_module/static/src/js/my_widget.js"></script>
            </xpath>
        </template>
    </data>
</odoo>
```

Key points:

- `inherit_id` is the bundle template, for example `web.assets_backend`.
- `xpath expr="." position="inside"` appends to the bundle.
- `script` uses `type="text/javascript"` and an absolute URL beginning with `/my_module/`.
- `link` uses `rel="stylesheet"` and `type="text/scss"` for SCSS, or `type="text/css"` for plain
  CSS.
- The XML file must be listed in the manifest `data` list.

### Bundle Templates Available in 13.0

Verified against `addons/web/views/webclient_templates.xml` and
`addons/web/views/report_templates.xml`:

| Template | Purpose |
|----------|---------|
| `web.assets_common` | Common assets used by backend and website. |
| `web.assets_common_minimal_js` | Minimal common JavaScript. |
| `web.assets_common_lazy` | Lazily loaded common assets. |
| `web.assets_backend` | Backend web client assets. |
| `web.assets_frontend` | Website and portal assets. |
| `web.assets_frontend_minimal_js` | Minimal frontend JavaScript. |
| `web.report_assets_common` | Assets for QWeb report rendering. |
| `web.report_assets_pdf` | Extra assets for PDF report rendering. |

These are XML template external IDs, not manifest keys. To add assets, inherit the template; do not
add a manifest entry.

### Real Example: `mail`

`addons/mail/views/mail_templates.xml` shows the 13.0 pattern:

```xml
<template id="assets_backend" name="mail assets"
          inherit_id="web.assets_backend">
    <xpath expr="." position="inside">
        <script type="text/javascript"
                src="/mail/static/src/js/many2many_tags_email.js"></script>
        <script type="text/javascript"
                src="/mail/static/src/js/services/mail_manager.js"></script>
    </xpath>
</template>
```

The file is listed in `addons/mail/__manifest__.py` under `data`, not under an `assets` key.

### Frontend and Report Assets

For website or portal assets, inherit `web.assets_frontend` with the same `<xpath expr="."
position="inside">` pattern. For report-specific CSS or JavaScript, inherit
`web.report_assets_common` or `web.report_assets_pdf`.

### What Does Not Exist in 13.0

The following are later-version constructs and are **not available in 13.0**: the `assets` manifest
key, the `ir.asset` model, the `web.assets_qweb` bundle, manifest asset operations such as
`('include', ...)`, `('remove', ...)`, `('prepend', ...)`, `('replace', ...)`, `('after', ...)`,
and `('before', ...)`, and glob patterns in a manifest asset list. Use XML inheritance and explicit
file paths instead.

---

## External Dependencies

### `external_dependencies`

```python
'external_dependencies': {
    'python': ['requests', 'pyjwt', 'python-dateutil'],
    'bin': ['wkhtmltopdf', 'ghostscript'],
},
```

The 13.0 check lives in `ir.module.module._check_external_dependencies`
(`odoo/addons/base/models/ir_module.py`):

- `python`: each name is checked with `pkg_resources.get_distribution(name)`. If that raises
  `DistributionNotFound`, the loader tries `importlib.import_module(name)` as a
  backward-compatibility fallback. A missing package raises
  `Exception('Python library not installed: ...')`.
- `bin`: each name is checked with `tools.find_in_path(binary)`. A missing binary raises
  `Exception("Unable to find '<name>' in path")`.

`check_external_dependencies()` wraps these errors in a `UserError`:

```
Unable to install module "<name>" because an external dependency is not met: ...
```

The check runs when a module is installed or upgraded, not when the manifest is merely parsed.

Use the distribution name as installed by pip, not necessarily the import name. For example,
`python-dateutil` is the distribution name for the `dateutil` import. The fallback import check
accepts the import name when the distribution name is not a valid PyPI package, but a warning is
logged. For binaries, use the executable name as found on `PATH`, for example `wkhtmltopdf`, not an
absolute path.

---

## Hooks

### Manifest Entries

```python
'pre_init_hook':   'pre_init_hook',
'post_init_hook':  'post_init_hook',
'uninstall_hook':  'uninstall_hook',
'post_load':       'post_load',
```

Each value is a function name looked up in the module package
(`sys.modules['odoo.addons.<module_name>']`). The functions live in the module's `__init__.py` or
are imported there.

### Signatures in Odoo 13

Verified against `odoo/modules/loading.py`:

```python
# __init__.py

def pre_init_hook(cr):
    """Runs before the module's models and data are loaded, on fresh install."""
    ...

def post_init_hook(cr, registry):
    """Runs after install and data load, on fresh install."""
    ...

def uninstall_hook(cr, registry):
    """Runs when the module is being removed."""
    ...
```

Call sites:

- `pre_init_hook(cr)` at `loading.py` line ~187, only when `package.state == 'to install'`.
- `post_init_hook(cr, registry)` at `loading.py` line ~240, only on a new install.
- `uninstall_hook(cr, registry)` at `loading.py` line ~496, during module removal.

Odoo 13 does **not** pass an `env` to these hooks. That signature change is a later-version
construct and is **not available in 13.0**. Build an environment inside the hook when you need one:

```python
from odoo import api, SUPERUSER_ID

def post_init_hook(cr, registry):
    env = api.Environment(cr, SUPERUSER_ID, {})
    env['ir.config_parameter'].sudo().set_param('my_module.enabled', 'True')
```

### `post_load`

`post_load` is different: it runs at module import time, once per server process, not per database.
`load_openerp_module()` calls it with no arguments:

```python
def post_load():
    # No arguments. Called once per worker.
    # Use for server-wide monkey-patching or global registration.
    ...
```

The loader reads `info['post_load']` and calls
`getattr(sys.modules['odoo.addons.' + module_name], info['post_load'])()`. The default is `None`.

### Example `__init__.py`

```python
# my_module/__init__.py
from odoo import api, SUPERUSER_ID

from . import models
from . import wizard


def pre_init_hook(cr):
    cr.execute("CREATE TABLE IF NOT EXISTS my_module_audit (id SERIAL PRIMARY KEY, message TEXT)")


def post_init_hook(cr, registry):
    env = api.Environment(cr, SUPERUSER_ID, {})
    env['ir.config_parameter'].sudo().set_param('my_module.enabled', 'True')


def uninstall_hook(cr, registry):
    cr.execute("DROP TABLE IF EXISTS my_module_audit")
```

---

## Other Fields

### `url` and `live_test_url`

```python
'url': 'https://example.com/my_module',
```

`get_values_from_terp` stores `url` on `ir.module.module.url`. When `url` is absent, it falls back
to `live_test_url`. Neither key is in the loader defaults.

### `maintainer` and `contributors`

```python
'maintainer': 'UncleCat',
'contributors': ['Alice', 'Bob'],
```

Stored as metadata on `ir.module.module`. `contributors` is joined with `, `.

### `images` and `countries`

Odoo 13 does not read an `images` manifest key. The module icon is discovered from
`static/description/icon.png` by `get_module_icon()`; use the `icon` key for a custom path. Odoo 13
also does not read a `countries` manifest key; localization modules are identified by their `l10n_*`
naming and category. Both are later-version conventions and are **not available in 13.0**.

### `assets`

Not a 13.0 manifest key. See [Assets in Odoo 13](#assets-in-odoo-13).

---

## Complete Example

```python
# -*- coding: utf-8 -*-
{
    # == Identification ==
    'name': 'Business Trip Management',
    'version': '13.0.1.0.0',
    'summary': 'Plan business trips, track per-diems and expenses',
    'description': """
Business Trip Management
========================
Features:

* Plan trips with itinerary, per-diem rates, and approvers.
* Track expenses and link receipts.
* Generate PDF reports per trip.
* Email templates for approval and reimbursement.
""",
    'category': 'Human Resources/Expenses',
    'author': 'UncleCat',
    'maintainer': 'UncleCat',
    'website': 'https://example.com/business_trip',
    'license': 'LGPL-3',
    'icon': '/business_trip/static/description/icon.png',

    # == Dependencies ==
    'depends': ['base', 'mail', 'hr', 'hr_expense', 'product', 'web'],
    'auto_install': False,

    # == Data (order matters) ==
    'data': [
        # Security first
        'security/business_trip_groups.xml',
        'security/ir_rule.xml',
        'security/ir.model.access.csv',
        # Reference data
        'data/ir_sequence_data.xml',
        'data/mail_template_data.xml',
        'data/ir_cron_data.xml',
        # Wizards
        'wizard/business_trip_cancel_views.xml',
        # Views
        'views/business_trip_views.xml',
        'views/business_trip_expense_views.xml',
        'views/hr_employee_views.xml',
        # Reports
        'report/business_trip_report_templates.xml',
        'report/business_trip_report_actions.xml',
        # Asset declarations (XML, not a manifest key)
        'views/assets.xml',
        # Menus (last)
        'views/business_trip_menus.xml',
    ],

    # == Demo ==
    'demo': ['demo/business_trip_demo.xml'],

    # == JavaScript QWeb templates ==
    'qweb': ['static/src/xml/business_trip_widget.xml'],

    # == External requirements ==
    'external_dependencies': {
        'python': ['python-dateutil'],
        'bin': ['wkhtmltopdf'],
    },

    # == Hooks ==
    'pre_init_hook': 'pre_init_hook',
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',

    # == Flags ==
    'application': True,
    'installable': True,
    'sequence': 20,
}
```

The matching `views/assets.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data>
        <template id="assets_backend" name="business_trip assets"
                  inherit_id="web.assets_backend">
            <xpath expr="." position="inside">
                <link rel="stylesheet" type="text/scss"
                      href="/business_trip/static/src/scss/trip_form.scss"/>
                <script type="text/javascript"
                        src="/business_trip/static/src/js/trip_widget.js"></script>
            </xpath>
        </template>
    </data>
</odoo>
```

The matching `static/src/xml/business_trip_widget.xml` uses the `<templates>` root shown in
[QWeb Templates](#qweb-templates-qweb-key).

---

## Common Patterns

### Bridge / link module

```python
{
    'name': 'Sale - Purchase Bridge',
    'version': '13.0.1.0.0',
    'author': 'UncleCat',
    'license': 'LGPL-3',
    'category': 'Hidden',
    'depends': ['sale', 'purchase'],
    'auto_install': ['sale', 'purchase'],
    'data': ['views/sale_order_views.xml'],
    'installable': True,
}
```

### Website / portal module

```python
{
    'name': 'My Portal Pages',
    'version': '13.0.1.0.0',
    'author': 'UncleCat',
    'license': 'LGPL-3',
    'depends': ['portal', 'website'],
    'data': ['security/ir.model.access.csv', 'views/portal_templates.xml', 'views/assets.xml'],
    'installable': True,
}
```

`views/assets.xml` inherits `web.assets_frontend` and adds the portal JavaScript and SCSS.

### Backend widget module

```python
{
    'name': 'My Backend Widget',
    'version': '13.0.1.0.0',
    'author': 'UncleCat',
    'license': 'LGPL-3',
    'depends': ['web'],
    'data': ['views/assets.xml'],
    'qweb': ['static/src/xml/my_widget.xml'],
    'installable': True,
}
```

`views/assets.xml` inherits `web.assets_backend` and adds the widget JavaScript and SCSS. The
`qweb` key registers the widget template. For `external_dependencies`, hooks, and `auto_install: []`
(always auto-install), see the dedicated sections above.

---

## Base Code Reference

All behavior documented above is derived from the Odoo 13.0 source:

- `odoo/modules/module.py` - `MANIFEST_NAMES`, `module_manifest()`,
  `load_information_from_description_file()`, the default `info` dict, `ast.literal_eval` parsing,
  README fallback, `auto_install` normalization, `adapt_version()`, `get_module_icon()`, and
  `load_openerp_module()` (which invokes `post_load`).
- `odoo/modules/loading.py` - `load_data()` (the `data`/`demo`/legacy key expansion), `load_demo()`,
  and the `pre_init_hook`, `post_init_hook`, and `uninstall_hook` call sites with their 13.0
  signatures.
- `odoo/modules/migration.py` - `migrations/` folder layout, `pre-`/`post-`/`end-` script naming,
  `migrate(cr, installed_version)`, and the `0.0.0` special folder.
- `odoo/modules/db.py` - how `installable`, `auto_install`, `web`, `license`, `application`, `icon`,
  `sequence`, and `summary` are written to `ir.module.module`.
- `odoo/addons/base/models/ir_module.py` - the `license` selection values, `get_values_from_terp()`,
  `_check_python_external_dependency()`, and `_check_external_dependencies()`.
- `odoo/release.py` - `version_info = (13, 0, 0, FINAL, 0, '')` and `major_version = '13.0'`.
- `addons/web/__manifest__.py` - the canonical `qweb` key and the `bootstrap` flag.
- `addons/web/views/webclient_templates.xml` - `web.assets_common`, `web.assets_backend`,
  `web.assets_frontend`, and the related bundle templates.
- `addons/web/views/report_templates.xml` - `web.report_assets_common` and
  `web.report_assets_pdf`.
- `addons/web/controllers/main.py` - `manifest_glob('qweb', ...)`, the
  `/web/webclient/qweb/<unique>` route, and the `bootstrap` translation check.
- `addons/mail/views/mail_templates.xml` - a real 13.0 XML asset declaration inheriting
  `web.assets_backend`.
- `addons/mail/__manifest__.py` - a real 13.0 manifest that lists the asset XML file under `data`.
