---
name: odoo-13.0
description: Odoo 13 development reference for Python models and ORM (search, domain, read_group, compute fields), XML/CSV data and views, legacy JavaScript widgets and QWeb, QWeb reports, security (ACL, record rules, groups), cron and server actions, migrations and module upgrades, tests, i18n, and performance. Use this skill whenever work involves Odoo 13 or custom addons, including traceback diagnosis, addons, manifests, XML errors, HTTP controllers, business rules, feature work, refactoring, or reviews.
---

# Odoo 13 Skill - Master Index

Master index for all Odoo 13 development guides. Read the appropriate guide from `references/` based on your task.

## Quick Reference

| Topic | File | When to Use |
|-------|------|-------------|
| Actions | `references/odoo-13-actions-guide.md` | Creating actions, menus, scheduled jobs, server actions |
| API Decorators | `references/odoo-13-decorator-guide.md` | Using @api decorators, compute fields, validation |
| Controllers | `references/odoo-13-controller-guide.md` | Writing HTTP endpoints, routes, web controllers |
| Data Files | `references/odoo-13-data-guide.md` | XML/CSV data files, records, shortcuts |
| Development | `references/odoo-13-development-guide.md` | Creating modules, manifest, reports, security, wizards |
| Field Types | `references/odoo-13-field-guide.md` | Defining model fields, choosing field types |
| JavaScript | `references/odoo-13-javascript-guide.md` | Legacy web client widgets, registries, RPC, QWeb JS templates |
| Manifest | `references/odoo-13-manifest-guide.md` | __manifest__.py configuration, dependencies, hooks |
| Migration | `references/odoo-13-migration-guide.md` | Upgrading modules, data migration, version changes |
| Mixins | `references/odoo-13-mixins-guide.md` | mail.thread, activities, email aliases, tracking |
| Model Methods | `references/odoo-13-model-guide.md` | Writing ORM queries, CRUD operations, domain filters |
| Performance | `references/odoo-13-performance-guide.md` | Optimizing queries, fixing slow code, preventing N+1 |
| Reports | `references/odoo-13-reports-guide.md` | QWeb reports, PDF/HTML, templates, paper formats |
| Security | `references/odoo-13-security-guide.md` | Access rights, record rules, field permissions |
| Testing | `references/odoo-13-testing-guide.md` | Writing tests, mocking, assertions, browser testing |
| Transactions | `references/odoo-13-transaction-guide.md` | Handling database errors, savepoints, UniqueViolation |
| Translation | `references/odoo-13-translation-guide.md` | Adding translations, localization, i18n |
| Views & XML | `references/odoo-13-view-guide.md` | Writing XML views, actions, menus, QWeb templates |

## File Structure

```
skills/odoo-13.0/
|-- SKILL.md                          # This file - master index
`-- references/                       # Development guides
    |-- odoo-13-actions-guide.md
    |-- odoo-13-controller-guide.md
    |-- odoo-13-data-guide.md
    |-- odoo-13-decorator-guide.md
    |-- odoo-13-development-guide.md
    |-- odoo-13-field-guide.md
    |-- odoo-13-javascript-guide.md
    |-- odoo-13-manifest-guide.md
    |-- odoo-13-migration-guide.md
    |-- odoo-13-mixins-guide.md
    |-- odoo-13-model-guide.md
    |-- odoo-13-performance-guide.md
    |-- odoo-13-reports-guide.md
    |-- odoo-13-security-guide.md
    |-- odoo-13-testing-guide.md
    |-- odoo-13-transaction-guide.md
    |-- odoo-13-translation-guide.md
    `-- odoo-13-view-guide.md
```

## What Is Different In Odoo 13

Odoo 13 predates several APIs that later guides in this repository assume. Keep these
differences in mind before copying patterns from the 14.0+ guides:

- **No OWL.** The web client is built on the legacy widget framework (`odoo.define`,
  `web.Widget`, `web.AbstractField`, `web.field_registry`, `web.view_registry`).
  Use `references/odoo-13-javascript-guide.md`, not the OWL guides.
- **Views use `attrs` and `states`.** There are no expression strings such as
  `invisible="state == 'done'"`; use `attrs="{'invisible': [('state', '=', 'done')]}"`.
- **QWeb uses `t-esc` and `t-raw`.** `t-out` does not exist in 13.0.
- **Assets are declared in XML**, by inheriting `web.assets_backend`,
  `web.assets_frontend`, or `web.assets_common`. The manifest has no `assets` key;
  JavaScript QWeb templates are listed under the manifest `qweb` key.
- **No `Command` class.** Use the legacy command tuples `(0, 0, vals)`, `(1, id, vals)`,
  `(2, id, 0)`, `(3, id, 0)`, `(4, id, 0)`, `(5, 0, 0)`, `(6, 0, ids)`.
- **No `@api.multi`, `@api.one`, `@api.cr`, or `@api.model_cr`.** They were removed in
  13.0. `@api.model_create_multi` is available and is the preferred `create` override.
- **No `fields.Json` and no `Properties` field.** Use `Text` with JSON serialization or a
  relational model.
- **`read_group()` is the public API.** `_read_group()` is a 16.0+ replacement.
- **`name_get()` and `name_search()` still exist.** `_compute_display_name` and
  `_rec_names_search` are later replacements.
- **`track_visibility` is deprecated in favor of `tracking=True`**, but is still read by
  `mail.thread` in 13.0.
- **`decimal_precision` is deprecated.** Use `digits='Precision Name'` strings.
- **Python 3.6+** is the supported runtime.

## Base Code Reference (Odoo 13)

All guides are based on analysis of Odoo 13 source code:
- `odoo/models.py` - ORM implementation
- `odoo/fields.py` - Field types
- `odoo/api.py` - Decorators and Environment
- `odoo/http.py` - HTTP layer and routing
- `odoo/exceptions.py` - Exception types
- `odoo/tests/common.py` - Test cases, Form helper, HttpCase
- `odoo/tools/translate.py` - Translation system
- `odoo/modules/module.py` - Manifest loading and defaults
- `odoo/addons/base/models/ir_ui_view.py` - View validation and modifiers
- `addons/web/static/src/js/core/registry.js` - JavaScript registries
- `addons/web/views/webclient_templates.xml` - Asset bundles

## External Documentation

- [Odoo 13 Official Documentation](https://github.com/odoo/documentation/tree/13.0)
- [Odoo 13 Developer Reference](https://github.com/odoo/documentation/blob/13.0/content/developer/reference/backend/orm.rst)
