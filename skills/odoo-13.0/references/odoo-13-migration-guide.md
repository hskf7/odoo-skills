---
name: odoo-13-migration
description: Complete reference for Odoo 13 module migration - version bumping to 13.0.X.Y.Z, migrations/<version>/{pre,post,end}-*.py scripts with migrate(cr, version), lifecycle hooks, _auto_init and _register_hook, install/update CLI, ir.module.module states, SQL and ORM data migrations, xmlid renames, and the 12.0 -> 13.0 API changes.
globs: "**/migrations/**/*.py"
topics:
  - Migration script discovery and execution in Odoo 13
  - migrate(cr, installed_version) signature
  - Folder layout migrations/13.0.X.Y.Z/{pre,post,end}-*.py
  - The special 0.0.0 folder
  - Module lifecycle hooks (pre_init_hook, post_init_hook, uninstall_hook)
  - _auto_init versus init versus _register_hook
  - Install and update commands with odoo-bin
  - ir.module.module states and version fields
  - Raw SQL patterns, idempotency, and ORM access
  - Renaming fields, models, and XML IDs
  - The 12.0 -> 13.0 API changes
  - Forward-looking notes for 13.0 -> 14.0
when_to_use:
  - Shipping a new version of an Odoo 13 module
  - Renaming a field, model, table, or XML ID during an upgrade
  - Backfilling or transforming data during an upgrade
  - Adding a lifecycle hook to a module
  - Deciding between a migration script and a hook
  - Running install/update commands against a database
---

# Odoo 13 Migration Guide

Reference for Odoo 13 module migration: version bumping, the
`migrations/<version>/{pre,post,end}-*.py` convention, the
`migrate(cr, version)` contract, lifecycle hooks, schema hooks, install and
update commands, `ir.module.module` state, data migration patterns, and the
12.0 -> 13.0 API changes.

Every claim was checked against the Odoo 13.0 source tree in
[Base Code Reference](#base-code-reference). Anything from 14.0 or later is
explicitly labelled **not available in 13.0**.

## Table of Contents

1. [When Migrations Run](#when-migrations-run)
2. [Version Scheme and Manifest Bump](#version-scheme-and-manifest-bump)
3. [Folder Layout](#folder-layout)
4. [`migrate(cr, version)` Signature](#migratecr-version-signature)
5. [Stages: pre-, post-, end-](#stages-pre--post--end-)
6. [The Special 0.0.0 Folder](#the-special-000-folder)
7. [Lifecycle Hooks vs Migration Scripts](#lifecycle-hooks-vs-migration-scripts)
8. [`_auto_init`, `init`, and `_register_hook`](#_auto_init-init-and-_register_hook)
9. [Install and Update Commands](#install-and-update-commands)
10. [`ir.module.module` States](#irmodulemodule-states)
11. [Raw SQL, Idempotency, and ORM Access](#raw-sql-idempotency-and-orm-access)
12. [Recipes](#recipes)
13. [The 12.0 -> 13.0 API Changes](#the-120---130-api-changes)
14. [Forward-Looking Notes: 13.0 -> 14.0](#forward-looking-notes-130---140)
15. [Testing and Version Management](#testing-and-version-management)
16. [Base Code Reference](#base-code-reference)

---

## When Migrations Run

Migrations run when a module is **updated** with `odoo-bin -u`. The dispatcher
is `odoo.modules.migration.MigrationManager.migrate_module`:

```python
state = pkg.state if stage in ('pre', 'post') else getattr(pkg, 'load_state', None)
if not (hasattr(pkg, 'update') or state == 'to upgrade') or state == 'to install':
    return
```

A fresh install (`state == 'to install'`) never runs migrations; use
`post_init_hook` for initial data. A plain server start without `-u` does not
run them, but a module marked `to upgrade` in `ir.module.module` does.

For each folder, a script runs when
`installed_version < version_folder <= manifest_version`; `0.0.0` runs
whenever `installed_version < manifest_version`.

`installed_version` is `pkg.load_version` when set, otherwise
`pkg.installed_version`. The 13.0 field names are confusing:
`installed_version` is the version **on disk**, `latest_version` is the
version recorded in the database.

---

## Version Scheme and Manifest Bump

Odoo 13 uses `13.0.X.Y.Z`. `odoo/release.py` defines
`version_info = (13, 0, 0, FINAL, 0, '')` and `major_version = '13.0'`.

The manifest `version` defaults to `'1.0'`, normalized by `adapt_version()`:
a value not starting with `13.0` is prefixed with `13.0.`, so `'1.0'` becomes
`'13.0.1.0'`, while `'13.0.1.2.0'` is used as written. Bump the manifest
before adding the folder that should fire:

```python
# __manifest__.py
{
    'name': 'My Module',
    'version': '13.0.1.1.0',   # activates migrations/13.0.1.1.0/*
    'depends': ['base'],
    'license': 'LGPL-3',
    'installable': True,
}
```

Typical bumps: feature `13.0.1.1.0`, bug fix `13.0.1.0.1`. The database
version must be strictly lower than the new manifest version, or the folder is
skipped.

---

## Folder Layout

```
my_module/
|-- __manifest__.py                  # 'version': '13.0.1.2.0'
|-- migrations/
|   |-- 13.0.1.0.0/
|   |   |-- pre-migrate.py           # before schema sync
|   |   |-- post-migrate_data.py     # after schema sync and data files
|   |   `-- end-cleanup.py           # after all modules are updated
|   |-- 13.0.1.1.0/post-rename_fields.py
|   `-- 0.0.0/                       # runs on every version increase
|       `-- pre-always.py
`-- upgrades/13.0.1.1.0/pre-migrate.py   # optional second root
```

Discovery rules (`MigrationManager._get_files`):

1. Both `migrations/` and `upgrades/` inside the module are scanned.
2. Every subdirectory is a version folder. 13.0 has **no** `VERSION_RE`;
   names are parsed by `convert_version()` and `parse_version()`. A name with
   fewer than two dots is prefixed with `13.0.` (`1.1` -> `13.0.1.1`).
3. Only `.py` files whose basename starts with `pre-`, `post-`, or `end-` are
   executed. Other files are ignored.
4. Within a version and stage, files are sorted by basename; use numeric
   prefixes when order matters (`pre-10-drop.py`, `pre-20-rename.py`).
5. There is no special handling of a `tests/` folder in 13.0. Do not put
   `pre-*.py` / `post-*.py` / `end-*.py` files there, or they run.
6. The dispatcher also scans `odoo.upgrade.__path__` for the same module name
   (enterprise upgrade-service layout). Community modules normally use only
   the module-local `migrations/` folder.

The directory is not declared in `__manifest__.py`; it is discovered on disk.

---

## `migrate(cr, version)` Signature

Every script must expose a module-level `migrate`:

```python
# migrations/13.0.1.1.0/post-rename_state.py

def migrate(cr, version):
    """cr: raw cursor. version: previously installed version or None."""
    if version is None:
        return
    cr.execute("UPDATE my_model SET state = %s WHERE state = %s",
               ('confirmed', 'validated'))
```

The dispatcher calls `migrate(self.cr, installed_version)`. Parameter names are
not enforced; the two positional arguments are. A missing function logs
`Each ...-migration file must have a "migrate(cr, installed_version)" function`
and skips the file; an `ImportError` aborts the upgrade.

Accepted variants are `migrate(cr, version)`, `migrate(cr, _version)`, and
`migrate(_cr, version)`. `version` is the **previously installed** version,
not the new manifest version; use it only to split behavior by origin version.

---

## Stages: pre-, post-, end-

For each module being updated, `odoo/modules/loading.py` runs:

```
1. pre-*.py        before the module is imported and before schema sync
2. load_openerp_module(package.name)
3. registry.load(cr, package)
4. registry.init_models(...)   -> _auto_init() then init() on each model
5. load_data(...)              -> XML/CSV data files
6. post-*.py                   after schema sync and data files
7. post_init_hook (fresh install only)
8. tests (when enabled)
9. end-*.py                    after ALL modules in the graph are updated
10. _register_hook() on every model
```

**pre-**: module Python is not imported and the old schema is still in place;
new columns do not exist. Use it to rename a column before the ORM creates a
new one, convert a column type, drop a column the new code no longer declares,
or pre-compute data the new schema consumes.

**post-**: schema sync is done, data files are loaded, models are available.
Use it for backfills, ORM transformations, recomputing stored computes, and
XML ID renames after the new data files load.

**end-**: runs after every module in the update graph, for cross-module
invariants. In 13.0 `migrate_module(package, 'end')` uses `pkg.load_state`, so
a package that was only loaded does not run its `end-` scripts.

---

## The Special 0.0.0 Folder

`migrations/0.0.0/` runs on every version increase. In
`_get_migration_versions`, `0.0.0` is moved **first** for `pre` and **last**
for `post` and `end`; the gate is
`version == "0.0.0" and parsed_installed_version < current_version`.

Use it for always-run sanity checks, dropping a legacy column that may exist
from any prior version, or emitting a log marker. It must be fully idempotent
because it runs on every bump.
---

## Lifecycle Hooks vs Migration Scripts

Hooks live in `__manifest__.py` and are called from
`odoo/modules/loading.py`. Their 13.0 signatures differ from later versions:

| Hook | Signature in 13.0 | When it runs |
|------|-------------------|--------------|
| `pre_init_hook` | `pre_init_hook(cr)` | Fresh install, after `load_openerp_module`, before `registry.load` |
| `post_init_hook` | `post_init_hook(cr, registry)` | Fresh install, after schema sync and data files, before `_register_hook` |
| `uninstall_hook` | `uninstall_hook(cr, registry)` | Uninstall, before `module_uninstall()` deletes records and tables |

```python
# __manifest__.py
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'depends': ['base'],
    'pre_init_hook': 'pre_init_hook',
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',
}
```

```python
# hooks.py
from odoo import api, SUPERUSER_ID

def pre_init_hook(cr):
    cr.execute("ALTER TABLE res_partner ADD COLUMN IF NOT EXISTS x_legacy_ref VARCHAR")

def post_init_hook(cr, registry):
    api.Environment(cr, SUPERUSER_ID, {})['ir.config_parameter'].sudo().set_param(
        'my_module.installed', '1')

def uninstall_hook(cr, registry):
    api.Environment(cr, SUPERUSER_ID, {})['ir.config_parameter'].sudo().search([
        ('key', '=like', 'my_module.%'),
    ]).unlink()
```

Rules: hooks run on install or uninstall, not on upgrade, so upgrade work
belongs in `migrations/<version>/`. In 13.0 both `post_init_hook` and
`uninstall_hook` receive `(cr, registry)`; `post_init_hook(env)` is a later
signature and is **not available in 13.0**. The function must be importable
from the module root, because it is looked up by name on the module package.

---

## `_auto_init`, `init`, and `_register_hook`

### `_auto_init(self)`

Called by `Registry.init_models()` for every model of the module being
installed or updated, before `init()`. The docstring in `odoo/models.py` is
explicit:

> Note: you should not override this method. Instead, you can modify the
> model's database schema by overriding method `init`, which is called right
> after this one.

It creates the table, creates/updates columns, initializes new columns, adds
SQL constraints and indexes, and queues post-init work. Core overrides it
occasionally (for example `IrActionsActWindowView._auto_init` in
`ir_actions.py`), but module code should prefer `init()`.

### `init(self)`

Called immediately after `_auto_init()` on install and update. This is the
supported place for schema changes the ORM cannot infer: functional or partial
indexes, triggers, views, or a backfill that must exist before the first ORM
read.

```python
from odoo import models, tools


class MyModel(models.Model):
    _name = 'my.model'

    def init(self):
        super(MyModel, self).init()
        tools.create_index(self._cr, 'my_model_state_date_index',
                           self._table, ['state', 'date'])
```

`init()` runs on every install and update, so it must be idempotent.
`tools.create_index` and `tools.create_unique_index` are idempotent; raw
`CREATE INDEX` is not.

### `_register_hook(self)`

Called once after the registry is fully loaded:

```python
# odoo/modules/loading.py, STEP 8
for model in env.values():
    model._register_hook()
```

It is re-run by `Registry.setup_models()` when the registry is already ready.
Use it for runtime wiring that depends on the final registry: registering a
callback, patching a method after all modules load, or building caches that
depend on other modules. Do not put DDL or bulk data changes there; it can run
on every registry load and is not part of the upgrade transaction contract in
the same way as `init()`.

---

## Install and Update Commands

Options from `odoo/tools/config.py`:

```text
-i, --init        install modules (comma-separated, "all"); requires -d
-u, --update      update modules (comma-separated, "all"); requires -d
-d, --database    database name
--stop-after-init stop the server after initialization
--dev             all, reload, qweb, werkzeug, xml, pudb, wdb, ipdb, pdb
```

```bash
odoo-bin -c odoo.conf -d mydb -i my_module --stop-after-init
odoo-bin -c odoo.conf -d mydb -u my_module --stop-after-init
odoo-bin -c odoo.conf -d mydb -u all --stop-after-init
odoo-bin -c odoo.conf -d mydb -u my_module --dev=all
```

`--dev=all` enables `reload`, `qweb`, `werkzeug`, and `xml` reloading. It does
not change when migrations run: they still require `-u`. For migration
debugging add `--log-handler=odoo.modules.migration:DEBUG`.

The 13.0 CLI has no `upgrade_code` command; that tool is **not available in
13.0**. Test by installing the old code, swapping the code on disk, and
running `-u`.

---

## `ir.module.module` States

`odoo/addons/base/models/ir_module.py` defines:

```python
STATES = [
    ('uninstallable', 'Uninstallable'),
    ('uninstalled', 'Not Installed'),
    ('installed', 'Installed'),
    ('to upgrade', 'To be upgraded'),
    ('to remove', 'To be removed'),
    ('to install', 'To be installed'),
]
```

- `uninstalled` -> `to install` with `-i` or `button_immediate_install()`.
- `installed` -> `to upgrade` with `-u` or `button_immediate_upgrade()`.
- `installed` -> `to remove` with `button_immediate_uninstall()`.
- After a successful load the module is `installed` again.

Version fields, with the misleading names documented in the source:

| Field | Meaning in 13.0 |
|-------|-----------------|
| `installed_version` | Computed "Latest Version": version found on disk |
| `latest_version` | "Installed Version": version recorded in the database |
| `published_version` | Version available on the repository, if configured |

A migration compares the database value in `latest_version` with the new
manifest version on disk. The dispatcher itself uses `pkg.installed_version`
and `pkg.load_version` from the module graph.

Useful methods: `button_immediate_install()`,
`button_immediate_upgrade()`, `button_immediate_uninstall()`,
`module_uninstall()`. The `button_immediate_*` methods commit, reset the
environment, and reload the registry; never call them inside a migration.

---

## Raw SQL, Idempotency, and ORM Access

Raw SQL is preferred for bulk data: it avoids per-record ORM overhead, skips
`compute`, `tracking`, `_log_access`, and `ir.rule` filtering, and can perform
DDL.

```python
def migrate(cr, version):
    cr.execute(
        "UPDATE my_model SET state = %s WHERE state = %s",
        ('confirmed', 'validated'),
    )
    cr.execute("SELECT id, name FROM my_model WHERE state = 'draft'")
    for row_id, name in cr.fetchall():
        pass
```

Odoo manages the surrounding transaction. Do not call `cr.commit()` in a short
migration; it breaks all-or-nothing upgrade and can leave a half-migrated
database. Commit only for long migrations where checkpointing is worth it.

Schema skeletons:
```sql
ALTER TABLE my_model ADD COLUMN IF NOT EXISTS new_field VARCHAR;
ALTER TABLE my_model DROP COLUMN IF EXISTS old_field;
ALTER TABLE my_model RENAME COLUMN old_name TO new_name;
ALTER TABLE my_model
    ALTER COLUMN amount TYPE DOUBLE PRECISION
    USING amount::DOUBLE PRECISION;
INSERT INTO ir_config_parameter (key, value)
VALUES ('my_module.version', '13.0.1.1.0')
ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value;
```

```python
from odoo import tools

def migrate(cr, version):
    if tools.column_exists(cr, 'my_model', 'legacy_ref'):
        cr.execute("ALTER TABLE my_model DROP COLUMN legacy_ref")
```

A script must be safe to run twice; Odoo does not guarantee exactly-once
execution. Guard DDL with `IF (NOT) EXISTS`, guard DML with a predicate that
becomes false after the first run, check existence before acting, and use
`ON CONFLICT` for inserts:

```python
def migrate(cr, version):
    cr.execute("ALTER TABLE my_model ADD COLUMN IF NOT EXISTS new_field VARCHAR")
    cr.execute("UPDATE my_model SET new_field = old_field "
               "WHERE new_field IS NULL AND old_field IS NOT NULL")
    cr.execute("INSERT INTO ir_config_parameter (key, value) VALUES (%s, %s) "
               "ON CONFLICT (key) DO NOTHING", ('my_module.feature_x', 'enabled'))
```

`migrate()` receives only a cursor. Build an environment explicitly:

```python
from odoo import api, SUPERUSER_ID

def migrate(cr, version):
    env = api.Environment(cr, SUPERUSER_ID, {})
    partner = env.ref('base.main_partner', raise_if_not_found=False)
    if partner:
        partner.write({'comment': 'migrated'})
```

Caveats:

- In `pre-`, module Python is not imported and the new schema is not applied;
  ORM access may see the old registry. Prefer SQL.
- In `post-`, models are fully loaded; this is the right place for ORM work.
- `env.ref()` takes `raise_if_not_found=True` by default.
- `env.company` and `env.companies` exist and respect `allowed_company_ids`.
- Avoid user-action side effects: mail tracking, activity creation, and
  computed writes can create records that were never in the original data.
  Use SQL for those tables.

---

## Recipes

### Rename a field

Rename the column in `pre-` so the ORM does not create an empty new column
next to the old one, then update metadata:

```python
# migrations/13.0.1.1.0/pre-rename_customer.py

def migrate(cr, version):
    if tools.column_exists(cr, 'sale_order', 'customer_id') \
            and not tools.column_exists(cr, 'sale_order', 'partner_id'):
        cr.execute("ALTER TABLE sale_order RENAME COLUMN customer_id TO partner_id")
    cr.execute("UPDATE ir_model_fields SET name = 'partner_id' "
               "WHERE model = 'sale.order' AND name = 'customer_id'")
    cr.execute("UPDATE ir_translation "
               "SET name = REPLACE(name, ',customer_id', ',partner_id') "
               "WHERE name LIKE 'sale.order,customer_id%'")
```

Do not use `oldname=` in 13.0. `odoo/fields.py` lists it in
`DEPRECATED_ATTRS` as `("oldname", "use an upgrade script instead.")`; it logs
a warning and does not perform the rename.

### Change a field type

Alter with an explicit cast:
`ALTER TABLE my_model ALTER COLUMN amount TYPE DOUBLE PRECISION USING
amount::DOUBLE PRECISION`. When the cast can fail, stage through a temporary
column: add `amount_int INTEGER`, fill it with a `CASE` that casts only valid
values, drop the old column, then rename the temporary one.

### Rename a model and its table

Model renames touch several metadata tables. In 13.0 the relevant tables are
`ir_model`, `ir_model_fields`, `ir_model_data`, `ir_attachment`,
`ir_ui_view`, `ir_act_window`, `mail_message`, `mail_followers`, and
`mail_activity`.

```python
# migrations/13.0.1.1.0/pre-rename_model.py

def migrate(cr, version):
    cr.execute("ALTER TABLE IF EXISTS old_model RENAME TO new_model")
    cr.execute("UPDATE ir_model SET model = 'new.model' WHERE model = 'old.model'")
    cr.execute("UPDATE ir_model_fields SET model = 'new.model' WHERE model = 'old.model'")
    cr.execute("UPDATE ir_model_fields SET relation = 'new.model' WHERE relation = 'old.model'")
    cr.execute("UPDATE ir_model_data SET model = 'new.model' WHERE model = 'old.model'")
    cr.execute("UPDATE ir_attachment SET res_model = 'new.model' WHERE res_model = 'old.model'")
    cr.execute("UPDATE ir_ui_view SET model = 'new.model' WHERE model = 'old.model'")
    cr.execute("UPDATE ir_act_window SET res_model = 'new.model' WHERE res_model = 'old.model'")
    # mail_message uses "model"; mail_followers and mail_activity use "res_model"
    cr.execute("UPDATE mail_message SET model = 'new.model' WHERE model = 'old.model'")
    cr.execute("UPDATE mail_followers SET res_model = 'new.model' WHERE res_model = 'old.model'")
    cr.execute("UPDATE mail_activity SET res_model = 'new.model' WHERE res_model = 'old.model'")
```

Notes: many2many relation tables keep their old names; `ir_model_data.name`
usually stays the same (only `model` changes); update the module's XML data in
the same release, or the next `-u` will recreate stale records.

### Drop an obsolete field or model

```python
def migrate(cr, version):
    cr.execute("ALTER TABLE my_model DROP COLUMN IF EXISTS legacy_ref")
    cr.execute("DELETE FROM ir_model_fields WHERE model = 'my.model' AND name = 'legacy_ref'")
    cr.execute("DELETE FROM ir_model_data WHERE model = 'my.model' AND name LIKE 'legacy_%'")
```

If the field is still declared in Python, the ORM recreates the column on the
next `-u`. Remove the declaration first, then ship the migration.

### Recompute a stored computed field

When the formula changes, mark it for recomputation in `post-`:

```python
from odoo import api, SUPERUSER_ID


def migrate(cr, version):
    env = api.Environment(cr, SUPERUSER_ID, {})
    Order = env['sale.order']
    Order.invalidate_cache(['margin'])
    orders = Order.search([])
    orders._compute_margin()
    orders.flush(['margin'])
```

In 13.0 the cache API is `invalidate_cache()` and the flush API is `flush()`
(`Order.invalidate_cache(['margin'])`, `records.flush(['margin'])`).
There is no `invalidate_model()` / `flush_model()` pair; those are later names
and are **not available in 13.0**. A cheaper alternative is
`UPDATE sale_order SET margin = NULL`, letting the ORM recompute lazily.

### Rename an XML ID

`ir.model.data` stores the external ID as `(module, name)`. Rename it in
`post-` after the new data files load:

```python
def migrate(cr, version):
    cr.execute("SELECT id FROM ir_model_data WHERE module = %s AND name = %s",
               ('my_module', 'old_record'))
    if not cr.fetchone():
        return
    cr.execute("UPDATE ir_model_data SET name = %s WHERE module = %s AND name = %s",
               ('new_record', 'my_module', 'old_record'))
```

If other records store the XML ID as text, update them too: typical places are
`ir_config_parameter.value`, `ir_default.json_value`, and custom `Char`
fields.

`ir.model.data._process_end()` runs at the end of loading and deletes records
whose XML ID is no longer loaded from the updated module's data files, unless
the record is marked `noupdate`. Update the XML data and the XML ID in the
same release.

---

## The 12.0 -> 13.0 API Changes

### Removed decorators: `@api.multi`, `@api.one`, `@api.cr`, `@api.model_cr`

`odoo/api.py` in 13.0 defines only `constrains`, `onchange`, `depends`,
`depends_context`, `returns`, `model`, `model_create_single`, and
`model_create_multi`. There is no `multi`, `one`, `cr`, or `model_cr`.

- Drop `@api.multi`; methods operate on recordsets by default.
- Drop `@api.one`; iterate `for record in self:` and assign on the record.
- Replace `@api.cr` with a method taking `cr` explicitly, or move the logic
  into a model method.
- Replace `@api.model_cr` with `@api.model` for registry-level hooks, or move
  schema work into `init()`.

`@api.model_create_multi` **is** available in 13.0 and is used by core models.
`BaseModel.create` is decorated with it and accepts either a dict or a list.

### `track_visibility` -> `tracking`

`addons/mail/models/mail_thread.py` reads
`getattr(field, 'tracking', None) or getattr(field, 'track_visibility', None)`.
Use `tracking=True` in 13.0:

```python
state = fields.Selection(
    [('draft', 'Draft'), ('done', 'Done')],
    string='Status',
    tracking=True,
)
```

`track_visibility` is deprecated and is removed in 14.0. Do not introduce it
in new code.

### `oldname` removed

`oldname` is in `DEPRECATED_ATTRS` in `odoo/fields.py` as
`("oldname", "use an upgrade script instead.")`. The ORM logs a warning and
does not rename the column. Use the rename-field recipe instead.

### `view_type` removed from window actions

`ir.actions.act_window` in 13.0 has `view_mode`, `view_id`, and `view_ids`,
but no `view_type`. Remove XML such as `<field name="view_type">form</field>`
and use `<field name="view_mode">tree,form</field>` instead.

### Action `multi` -> `binding_view_types`

`ir.actions.actions` in 13.0 has
`binding_view_types = fields.Char(default='list,form')`, the comma-separated
list of view types on which a bound action appears.

The legacy `multi` boolean still exists on `ir.actions.act_window.view`
(`multi = fields.Boolean(string='On Multiple Doc.', ...)`), but the
action-level binding switch is `binding_view_types`. Replace the old `multi`
flag with `binding_view_types` on the action record:

```xml
<field name="binding_model_id" ref="model_my_model"/>
<field name="binding_view_types">list,form</field>
```

### `decimal_precision` module -> `digits` strings

The standalone `decimal_precision` addon is gone in 13.0. The
`decimal.precision` model remains in `base` with `precision_get(name)`.
Declare precision on the field with a string:

```python
price = fields.Float(string='Price', digits='Product Price')
```

`odoo/fields.py` documents `digits` as a pair `(total, decimal)` or a string
referencing a `decimal.precision` record name; `Float.get_digits()` resolves
the string through `env['decimal.precision'].precision_get()`. Port
`digits_compute='product_price'` to `digits='Product Price'`.
`digits_compute` is listed in `RENAMED_ATTRS` and is no longer supported.

### `env.company` and `env.companies`

Both exist on `odoo.api.Environment`: `env.company` returns a `res.company`
record and `env.companies` a recordset. `env.company` reads
`allowed_company_ids[0]` and falls back to `env.user.company_id`;
`env.companies` returns all allowed companies, or the user's companies when
the key is absent.

There is no `with_company()` on recordsets in 13.0. Use
`with_context(allowed_company_ids=[company.id])` instead.

### Mail template `format_tz` -> `format_datetime`

`addons/mail/models/mail_template.py` defines
`format_datetime(env, dt, tz=False, dt_format='medium', lang_code=False)` and
exposes it in the rendering context. There is no `format_tz` in 13.0. Port
`${format_tz(object.date, tz=user.tz, format='%Y-%m-%d')}` to
`${format_datetime(object.date, tz=user.tz, dt_format='yyyy-MM-dd')}`.

### jQuery promises -> native Promise

The 13.0 web client exposes native Promises from RPC: `web.rpc.query()` in
`addons/web/static/src/js/core/rpc.js` returns `ajax.rpc(...)`, which wraps
`$.ajax` in `new Promise(...)`. Port `.done()`, `.fail()`, and `.always()`
chains to `.then()` and `.catch()`:

```javascript
this._rpc({model: 'my.model', method: 'my_method'}).then(function (result) {
    ...
}).catch(function (error) {
    ...
});
```

A few legacy widgets still create `$.Deferred()` objects in 13.0 (for example
`name_and_signature.js`). Treat those as leftovers; new code should use native
Promises.

### Other 12.0 -> 13.0 reminders

- `read_group(...)` is the public API; there is no public `_read_group`.
- `name_get()`, `name_search()`, and `_name_search()` remain the supported
  overrides.
- `sudo()`, `with_user()`, `with_context()`, and `with_env()` all exist.
- There is no `Command` class: use `(0, 0, vals)`, `(1, id, vals)`,
  `(2, id, 0)`, `(3, id, 0)`, `(4, id, 0)`, `(5, 0, 0)`, `(6, 0, ids)`.
- There is no `Json` field and no `Properties` field; use `Text`, `Char`, or a
  relational model.

---

## Forward-Looking Notes: 13.0 -> 14.0

The following belong to Odoo 14.0 or later. They are **not available in 13.0**
and must not be used in a 13.0 module. They are listed so a module being
prepared for a later port does not paint itself into a corner.

### `t-raw` -> `t-out`

13.0 QWeb uses `t-raw` for unescaped output and `t-esc` for escaped output;
`odoo/addons/base/models/qweb.py` checks
`set(['t-esc', 't-raw', 't-field']) & set(el.attrib)` and compiles `t-raw` via
`_compile_widget`. `t-out` does not exist in 13.0. In 14.0, `t-out` replaces
both `t-esc` and `t-raw` and applies escaping according to markup safety.
Keep the distinction explicit in 13.0:

```xml
<span t-esc="record.name"/>
<div t-raw="record.description"/>
```

Do not write `t-out` in 13.0 templates; it is **not available in 13.0**.

### `@api.model` create and batch warning

In 13.0, `BaseModel.create` is already decorated with
`@api.model_create_multi` and accepts a dict or a list. If you override
`create` with `@api.model` and a single `vals` dict, callers that pass a list
break. Write batch-safe overrides now:

```python
@api.model_create_multi
def create(self, vals_list):
    records = super(MyModel, self).create(vals_list)
    for record in records:
        record._after_create()
    return records
```

14.0 tightens this contract and warns about overrides that do not handle
batch creation. The warning is a 14.0 behavior; the batch-safe pattern is
already valid in 13.0.

### `ir.asset` model and `<asset>` directives

13.0 has no `ir.asset` model. Assets are declared by inheriting an asset
bundle template in XML:

```xml
<template id="assets_backend" inherit_id="web.assets_backend">
    <xpath expr="." position="inside">
        <script type="text/javascript" src="/my_module/static/src/js/widget.js"/>
        <link rel="stylesheet" type="text/css" href="/my_module/static/src/css/style.css"/>
    </xpath>
</template>
```

The `ir.asset` model and `<asset>` XML directives are introduced in 14.0.
They are **not available in 13.0**. Do not add `ir.asset` records or
`<asset>` tags to a 13.0 module.

### Manifest `assets` key

13.0 manifests have no `assets` key.
`load_information_from_description_file` does not read one, and asset bundles
are XML templates. The manifest key is a later-version feature (15.0) and is
**not available in 13.0**. A 13.0 manifest declares only `qweb` for
JavaScript QWeb templates:

```python
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'depends': ['web'],
    'qweb': ['static/src/xml/widget.xml'],
}
```

Do not add an `assets` list to a 13.0 manifest; it is ignored.

### Later-version APIs to avoid in 13.0

| API | Status in 13.0 |
|-----|----------------|
| `t-out` | Not available; use `t-esc` / `t-raw` |
| `ir.asset` model | Not available; use XML asset bundle inheritance |
| manifest `assets` key | Not available; use `qweb` and XML templates |
| `with_company()` | Not available; use `with_context(allowed_company_ids=[...])` |
| `Command` class | Not available; use tuple commands |
| `Json` field | Not available; use `Text` or a relational model |
| `Properties` field | Not available |
| `upgrade_code` CLI | Not available; install old code and run `-u` |

---

## Testing and Version Management

Odoo 13.0 has no dedicated upgrade-test CLI. Replay the upgrade: install the
old code with `-i my_module --stop-after-init`, replace the code on disk, then
run `-u my_module --stop-after-init` with
`--log-handler=odoo.modules.migration:DEBUG`.

Checklist: the upgrade exits 0; the log shows
`Running migration 13.0.1.1.0 [>13.0.1.1.0]` for each expected script; no
missing-`migrate` error appears; target rows have the expected values; a
second `-u` is a no-op and a forced re-run is idempotent.

Migration scripts are plain Python and can be loaded with `importlib` from a
`TransactionCase` to call `migrate(self.cr, '13.0.1.0.0')` twice and assert
idempotency. The test runs inside the test transaction and rolls back with it.

Use `odoo.tools.parse_version` when one script must split behavior by origin
version (`if parse_version(version) < parse_version('13.0.1.1.0'): ...`). The
version folder already gates execution; compare only when needed. Read the
installed version from SQL:

```sql
SELECT name, state, latest_version
  FROM ir_module_module
 WHERE name = 'my_module';
```

Remember the 13.0 naming quirk: `latest_version` is the database version;
`installed_version` is the on-disk version.

---

## Base Code Reference

Verified against the Odoo 13.0 source tree:

- `odoo/modules/migration.py` - `MigrationManager._get_files` (scans
  `migrations/`, `upgrades/`, `odoo.upgrade.__path__`; no `VERSION_RE`),
  `migrate_module`, `convert_version`, `_get_migration_versions`,
  `_get_migration_files`, `load_script`.
- `odoo/modules/loading.py` - `load_module_graph` stage order, the three hook
  call sites, STEP 8 `_register_hook()`.
- `odoo/modules/module.py` - `load_information_from_description_file`,
  `adapt_version`; `odoo/modules/registry.py` - `Registry.init_models`
  (`_auto_init()` then `init()`), `Registry.setup_models`.
- `odoo/models.py` - `_auto_init` (line 2487), `init` (line 2561),
  `_register_hook` (line 5015), `create` (line 3726), `read_group`,
  `name_get`, `name_search`, `_name_search`, `sudo`, `with_user`,
  `with_context`, `with_env`.
- `odoo/api.py` - `constrains`, `onchange`, `depends`, `depends_context`,
  `returns`, `model`, `model_create_single`, `model_create_multi`;
  `Environment.company` (line 528), `Environment.companies` (line 558),
  `Environment.ref` (line 502); no `multi`, `one`, `cr`, `model_cr`.
- `odoo/fields.py` - `DEPRECATED_ATTRS` with `oldname` (line 36);
  `RENAMED_ATTRS` with `digits_compute` -> `digits` (line 35);
  `Float.get_digits`.
- `odoo/tools/config.py` - `-i/--init`, `-u/--update`, `-d/--database`,
  `--stop-after-init`, `--dev`; `odoo/tools/sql.py` - `table_exists`,
  `column_exists`; `odoo/tools/parse_version.py` - `parse_version`.
- `odoo/release.py` - `version_info = (13, 0, 0, FINAL, 0, '')`,
  `major_version = '13.0'`.
- `odoo/addons/base/models/ir_module.py` - `STATES`, `installed_version`,
  `latest_version`, `button_immediate_*`, `module_uninstall`.
- `odoo/addons/base/models/ir_actions.py` - `binding_view_types` (line 40),
  no `view_type` on `ir.actions.act_window`, `multi` (line 265).
- `odoo/addons/base/models/ir_model.py` - `ir.model.data`, `_process_end`;
  `ir_translation.py` - `ir.translation`; `decimal_precision.py` -
  `decimal.precision`, `precision_get`; `qweb.py` - `t-esc`, `t-raw`,
  `t-field`, no `t-out`.
- `addons/mail/models/mail_thread.py` - `tracking` / `track_visibility`
  lookup (line 549); `mail_template.py` - `format_datetime` (line 293), no
  `format_tz`; `mail_message.py`, `mail_followers.py`, `mail_activity.py` -
  `model` versus `res_model` columns.
- `addons/web/static/src/js/core/rpc.js`, `ajax.js` - native Promise RPC;
  `addons/web/views/webclient_templates.xml` - `web.assets_common`,
  `web.assets_backend` XML asset bundles.
