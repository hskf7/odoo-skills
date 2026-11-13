---
name: odoo-13-security
description: Complete reference for Odoo 13 security covering access rights (ACL via ir.model.access.csv), record rules (ir.rule), user groups (res.groups with implied_ids), field-level permissions, multi-company rules, sudo() / with_user(), check_access_rights / check_access_rule and common security pitfalls.
globs: "**/*.{py,xml,csv}"
topics:
  - Access rights (ir.model.access.csv)
  - Record rules (ir.rule)
  - Field-level access (groups attribute)
  - User groups (res.groups, implied_ids)
  - Multi-company record rules
  - sudo() / with_user() / bypass patterns
  - check_access_rights / check_access_rule
  - Portal and public user access
  - ir.attachment access
  - Security pitfalls (SQL injection, XSS, eval)
when_to_use:
  - Configuring security for new models
  - Setting up access rights CSV
  - Creating record rules
  - Preventing security vulnerabilities
  - Understanding multi-company security
  - Implementing field-level permissions
  - Auditing portal / public user access
---

# Odoo 13 Security Guide

Complete reference for Odoo 13 security: access rights, record rules, field access, groups, multi-company and common pitfalls.

## Table of Contents

1. [Security Overview](#security-overview)
2. [User Groups](#user-groups)
3. [Access Rights (ACL)](#access-rights-acl)
4. [Record Rules](#record-rules)
5. [Field-Level Access](#field-level-access)
6. [Multi-Company](#multi-company)
7. [sudo / with_user / bypass patterns](#sudo--with_user--bypass-patterns)
8. [Access Checks in Code](#access-checks-in-code)
9. [Portal and Public Access](#portal-and-public-access)
10. [ir.attachment Access](#irattachment-access)
11. [Security Pitfalls](#security-pitfalls)
12. [Security Review Checklist](#security-review-checklist)

---

## Security Overview

### Two-Layer Security

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| 1 | Access Rights (ACL) | Grants CRUD access on an **entire model** |
| 2 | Record Rules | Restricts **which records** of the model a user can access |

Both layers are linked to users through **groups** (`res.groups`). A user belongs to multiple groups and permissions are the union of them.

### Access Control Flow

```
User operation (read/write/create/unlink)
        |
        v
Is at least one of user's groups allowed by ir.model.access?
   |-- no --> AccessError
   |
   v
Apply ir.rule for (model, operation, user's groups):
   - Global rules (no `groups`)         -> AND together  (all must match)
   - Group rules (have `groups`)        -> OR together   (one match is enough)
   - Final domain = (AND of globals) AND (OR of group rules)
        |
        v
Does record match the final domain?
   |-- no --> AccessError
   |
   v
Access granted
```

### Bypass Rules

- Superuser (`self.env.su`, OdooBot / `__system__`) bypasses record rules.
- `sudo()` returns a recordset whose `env.su` is True.
- `with_user(user)` switches the user but **does not** bypass anything.
- ACL checks are skipped entirely in superuser mode (`ir.model.access.check` returns True when `self.env.su`).

---

## User Groups

### res.groups

Groups are the foundation of Odoo security. In Odoo 13, `res.groups` has `category_id` pointing at `ir.module.category`. Inherited groups are declared through `implied_ids` (provided by the `GroupsImplied` extension in `res_users.py`); there is no `privilege_id` in Odoo 13.

### Defining a Group

```xml
<!-- security/security_groups.xml -->
<odoo>
    <!-- Module category (optional, groups appear in Settings > Users) -->
    <record id="module_category_trip_management" model="ir.module.category">
        <field name="name">Trip Management</field>
        <field name="description">Manage business trips</field>
        <field name="sequence">20</field>
    </record>

    <record id="group_trip_user" model="res.groups">
        <field name="name">User</field>
        <field name="category_id" ref="module_category_trip_management"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <record id="group_trip_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="module_category_trip_management"/>
        <field name="implied_ids" eval="[(4, ref('my_module.group_trip_user'))]"/>
    </record>
</odoo>
```

### res.groups Fields

| Field | Description |
|-------|-------------|
| `name` | Display name (translate=True) |
| `category_id` | Many2one on `ir.module.category` - groups appear in that app's section |
| `implied_ids` | Groups implied by this one (declared in the `GroupsImplied` extension) |
| `users` | Many2many on `res.users` |
| `model_access` | One2many on `ir.model.access` |
| `rule_groups` | Many2many on `ir.rule` |
| `menu_access` | Menus accessible to the group |
| `view_access` | Views accessible to the group |
| `comment` | Description (translate=True) |
| `color` | Color index for the UI |
| `full_name` | Computed `category / name` display string |
| `share` | True if this group is only for sharing (portal-like) |

A SQL constraint enforces `unique (category_id, name)` - two groups in the same category cannot share a name.

### Group Inheritance via `implied_ids`

If `group_manager` has `implied_ids` pointing to `group_user`, every user in `group_manager` is automatically added to `group_user` (transitively). Use this to build hierarchies:

```
group_manager  >>  group_user  >>  base.group_user  (internal user)
```

The `GroupsImplied.write()` method propagates the implied groups to all current users of the group via a recursive SQL query over `res_groups_implied_rel`.

### Checking Groups in Code

```python
# Is current user in a group?
if self.env.user.has_group('my_module.group_trip_manager'):
    ...

# has_group requires a fully-qualified external ID (module.ext_id)
# and respects implied_ids: a manager is also a user automatically.
if self.env.user.has_group('base.group_user'):
    ...

# user_has_groups parses a comma-separated list with optional `!` negation:
# True if in one of the positive groups and in none of the negated groups.
if self.user_has_groups('base.group_user,!base.group_portal'):
    ...
```

`has_group()` is delegated to `_has_group()` which runs a cached SQL lookup on `res_groups_users_rel` joined against `ir_model_data` using the external ID.

---

## Access Rights (ACL)

### ir.model.access - Model-Level Access

Access rights grant CRUD operations on an entire model. They are normally declared via a CSV file loaded at module install/update.

```
my_module/
  security/
    ir.model.access.csv
```

### CSV Columns (Odoo 13)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
```

Odoo 13 uses exactly these 8 columns, in this order. There is no `active` column in the CSV; the `active` field defaults to True and is only toggled manually or via XML.

The CSV is processed by `convert_csv_import()` in `odoo/tools/convert.py`, which derives the target model from the filename prefix before the first `-` (`ir.model.access.csv` -> `ir.model.access`) and calls `env[model].load(fields, datas)`. The `:id` suffix in `model_id:id` and `group_id:id` tells the loader to resolve those values as external IDs. In init mode the CSV **must** contain an `id` column.

### Example `ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_business_trip_user,business.trip.user,model_business_trip,my_module.group_trip_user,1,1,1,0
access_business_trip_manager,business.trip.manager,model_business_trip,my_module.group_trip_manager,1,1,1,1
access_business_trip_portal,business.trip.portal,model_business_trip,base.group_portal,1,0,0,0
access_business_expense_all,business.expense.all,model_business_expense,,1,0,0,0
```

### Fields Explained

| Column | Description |
|--------|-------------|
| `id` | Unique external ID for this ACL row |
| `name` | Human-readable label |
| `model_id:id` | External ID of the `ir.model` row - convention is `model_<model_name_with_underscores>` |
| `group_id:id` | External ID of the `res.groups` row; **empty means all users** (including portal/public) |
| `perm_read` | `1` or `0` - allow read |
| `perm_write` | `1` or `0` - allow write |
| `perm_create` | `1` or `0` - allow create |
| `perm_unlink` | `1` or `0` - allow unlink (delete) |

### ACL Evaluation

`ir.model.access.check(model, mode, raise_exception)` decides access: if `self.env.su`, return True; otherwise it checks whether any of the user's groups grants the permission (SQL over `ir_model_access` joined to `res_groups_users_rel`), then falls back to the generic rule (`group_id IS NULL`), and finally raises `AccessError` listing the groups that do have access. ACLs are **additive**: a user gets the union of CRUD flags from every matching group, using `MAX(CASE WHEN perm_<mode> ...)`.

### ACL Rules of Thumb

- An **empty `group_id`** grants access to **everyone** (internal, portal, public). Use sparingly.
- A model with **no ACL row** is inaccessible to everyone except the superuser.
- ACLs bypass nothing on their own - they are the first barrier, record rules are the second.
- A `TransientModel` (wizard) needs no ACL row: `ir.model.access.check` returns True for transient models. Only the implicit "creator-only" access rule applies.

### Model ID Naming Convention

For a model with `_name = 'business.trip'`, the `ir.model` external ID is `model_business_trip` in the module that declares the model. When referencing a model declared in another module, prefix: `other_module.model_business_trip`.

---

## Record Rules

### ir.rule - Record-Level Security

Record rules filter which records of a model a given user can access.

```xml
<record id="trip_personal_rule" model="ir.rule">
    <field name="name">Business Trip: own records</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('my_module.group_trip_user'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

### ir.rule Fields (Odoo 13)

| Field | Description |
|-------|-------------|
| `name` | Description of the rule |
| `active` | Disable without deleting |
| `model_id` | Many2one on `ir.model` (required, ondelete cascade) |
| `groups` | Many2many on `res.groups` via `rule_group_rel`; empty = global rule |
| `domain_force` | **Text containing a Python list literal** for the domain (not XML) |
| `perm_read`, `perm_write`, `perm_create`, `perm_unlink` | When this rule applies (all default True) |
| `global` | Computed/stored boolean: `not groups` |

`domain_force` must be a string that evaluates to a domain list - e.g. `"[('user_id', '=', user.id)]"`. It is parsed with `safe_eval` at rule evaluation time.

A SQL constraint requires at least one of `perm_read/write/create/unlink` to be checked. Record rules cannot be applied to `TransientModel` records or to `ir.rule` itself (both raise `ValidationError`).

### Variables Available in `domain_force`

`ir.rule._eval_context()` returns the namespace handed to `safe_eval`:

| Variable | Description |
|----------|-------------|
| `user` | Current user record, browsed with an empty context (`self.env.user.with_context({})`) |
| `user.id` | Current user ID |
| `user.partner_id` | Partner linked to user |
| `user.company_id` | User's main company |
| `user.company_ids` | Companies the user can access |
| `time` | Python `time` module |
| `company_id` | Current allowed company (`self.env.company.id`) |
| `company_ids` | IDs of allowed companies (`self.env.companies.ids`) |

Attribute access on `user` (e.g. `user.partner_id.id`, `user.commercial_partner_id.id`) is available for any `res.users` field.

### Global vs Group Rules

`ir.rule._compute_domain()` evaluates all matching rules, then combines:

| Type | How they combine |
|------|------------------|
| **Global** (`groups` empty) | **AND** together - all global rules must match |
| **Group** (has `groups`) | **OR** together - any matching-group rule is enough |
| Combined | `AND(global_domains + [OR(group_domains)])` |

The result is cached via `@tools.ormcache` keyed on uid, su flag, model name, mode, and the `allowed_company_ids` context value (from `_compute_domain_keys()`).

Use global rules for company separation; use group rules to broaden access for managers vs. employees.

### Classic Patterns

#### Own records only (employees)

```xml
<record id="trip_user_rule" model="ir.rule">
    <field name="name">Trip: own records</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('my_module.group_trip_user'))]"/>
</record>
```

#### All records (managers)

```xml
<record id="trip_manager_rule" model="ir.rule">
    <field name="name">Trip: all records (manager)</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('my_module.group_trip_manager'))]"/>
</record>
```

#### Portal users (own partner's records)

```xml
<record id="trip_portal_rule" model="ir.rule">
    <field name="name">Trip: portal own</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">[('partner_id', '=', user.partner_id.id)]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="False"/>
    <field name="perm_create" eval="False"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

#### Time-sensitive access

```xml
<record id="archive_rule" model="ir.rule">
    <field name="name">Only this year's records</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">[('create_date', '&gt;=', time.strftime('%Y-01-01'))]</field>
    <field name="groups" eval="[(4, ref('my_module.group_trip_user'))]"/>
</record>
```

---

## Field-Level Access

### Restricting Fields to Groups

The `groups=` attribute on a field limits who can read/write it. The field is hidden from `fields_get`, removed from views, and raises `AccessError` on direct read/write for users who lack the group.

```python
class BusinessTrip(models.Model):
    _name = 'business.trip'

    name = fields.Char()                                       # all users
    internal_notes = fields.Text(groups='base.group_user')     # internal users only
    secret_code = fields.Char(groups='my_module.group_trip_manager')
    salary = fields.Float(groups='base.group_system')          # admin only

    # Multiple groups: comma-separated = OR;
    # `!` prefix negates a group (must NOT be a member).
    estimate = fields.Monetary(groups='base.group_user,!base.group_portal')
```

### Field-Level Group Behaviour

1. `fields_get()` skips the field entirely for users lacking the group (`field.groups and not self.user_has_groups(field.groups)`).
2. The field is removed from **views** at load time (`ir_ui_view._apply_group` removes the node).
3. `check_field_access_rights(operation, fields)` raises `AccessError` for restricted fields; `read()` calls it internally.
4. The `!` prefix in a comma-separated `groups` string means the field is hidden from members of that group.

### Related Fields with Groups

```python
partner_email = fields.Char(related='partner_id.email', groups='base.group_user')
```

The field-level group on the related alias hides the field from users outside the group; the check lives on the alias field, so the source `partner_id.email` is governed by its own (usually unrestricted) access.

---

## Multi-Company

### Record Rules and `allowed_company_ids`

The company switcher is carried in the context key `allowed_company_ids`. Odoo validates that list against `self.user.company_ids` when resolving `env.company` / `env.companies` (an invalid or unauthorized list raises `AccessError`).

`ir.rule._eval_context()` exposes:

- `company_id` = `self.env.company.id` (the first allowed company)
- `company_ids` = `self.env.companies.ids` (the user's currently allowed companies)

The rule cache key includes `allowed_company_ids` so switching companies recomputes the rule domains.

### Multi-Company Record Rule

Always add a **global** rule for company filtering on any model with `company_id`:

```xml
<record id="business_trip_company_rule" model="ir.rule">
    <field name="name">Business Trip: multi-company</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">
        ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
    </field>
</record>
```

Use `company_ids` (the user's **currently allowed** companies) rather than `user.company_ids` to respect the company switcher.

### `_check_company_auto` Flag

Set on a model to validate on every `create()` / `write()` that Many2one fields marked `check_company=True` point to records whose `company_id` is compatible with the record's own company.

```python
class SaleOrder(models.Model):
    _name = 'sale.order'
    _check_company_auto = True

    company_id = fields.Many2one('res.company', default=lambda s: s.env.company, required=True)
    partner_id = fields.Many2one('res.partner', check_company=True)
    pricelist_id = fields.Many2one('product.pricelist', check_company=True)
```

`models.py` triggers `_check_company()` from `create()`/`write()` when `_check_company_auto` is set; a `UserError` (not `AccessError`) is raised for incompatible records.

---

## sudo / with_user / bypass patterns

### `sudo()` - Elevate to Superuser

`sudo()` returns a new recordset whose environment runs with `su=True`. Record rules, field-level groups, and ACL checks are all bypassed.

```python
# Read a field the current user can't see (e.g. internal audit log)
internal_ref = record.sudo().internal_reference

# Create a record the user isn't allowed to create directly
self.env['mail.mail'].sudo().create({...})
```

Use sparingly - `sudo()` is a **bypass of permissions** and a common source of vulnerabilities.

### `with_user(user)` - Switch User Without Superuser

Run code as a different user; **all permissions still enforced**. The resulting environment is forced to `su=False` unless the target user is the superuser.

```python
res = self.with_user(self.env.ref('base.user_demo')).search([])
```

Passing a user object to `sudo()` is deprecated in 13.0 (it redirects to `with_user()` with a warning). Use `with_user()` explicitly.

### `with_context()` / `with_env()` - No `with_company()` in 13.0

Odoo 13 has `with_context()`, `with_env()`, and `with_user()`. There is **no `with_company()`** method (that is a later-version API, not available in 13.0). Switch company by setting the context key:

```python
records = self.with_context(allowed_company_ids=[other_company.id]).search([])
```

### Typical `sudo()` Patterns

```python
# GOOD: sudo to read one safe related field, then keep processing as the real user
partner_country = record.partner_id.sudo().country_id
if partner_country.code == 'US':
    record.action_apply_us_logic()          # still runs with ACL/rules

# GOOD: manual gatekeeping before sudo - guarantees the caller is allowed
def action_validate(self):
    if not self.env.user.has_group('my_module.group_trip_manager'):
        raise AccessError(_("Only Trip Managers can validate."))
    self.sudo().write({'state': 'validated'})

# BAD: blanket sudo() lets any user mutate any record
def action_unsafe(self):
    self.sudo().write({'active': False})   # No caller check - dangerous
```

### Checking Whether We Run as Superuser

```python
if self.env.su:
    # We are running as superuser (record rules bypass)
    ...

# Environment helpers available in 13.0:
#   self.env.is_superuser()  -> self.su
#   self.env.is_admin()      -> su or has group "Access Rights" (group_erp_manager)
#   self.env.is_system()     -> su or has group "Settings" (group_system)
```

---

## Access Checks in Code

### `check_access_rights(operation, raise_exception=True)`

Checks the ACL only - model-level permission. Delegates to `self.env['ir.model.access'].check(self._name, operation, raise_exception)`.

```python
# Silent check
can_read = self.env['business.trip'].check_access_rights('read', raise_exception=False)

# Raising check
self.env['business.trip'].check_access_rights('write')  # AccessError if denied
```

Valid operations: `'read'`, `'write'`, `'create'`, `'unlink'`.

### `check_access_rule(operation)`

Checks the record rules only on a specific recordset. Returns immediately in superuser mode.

```python
# Will raise AccessError if any record in self is filtered out
self.check_access_rule('write')
```

Internally it computes `invalid = self - self._filter_access_rules_python(operation)` and, if any forbidden record exists, raises the `AccessError` built by `ir.rule._make_access_error()`. For transient models it raises the owner-only error; for records deleted from the database it raises `MissingError` (except read/unlink on already-deleted records).

### Combined Check

There is **no single `check_access` method** in Odoo 13 - combine the two explicitly:

```python
try:
    self.check_access_rights('write')
    self.check_access_rule('write')
except AccessError:
    raise UserError(_("You cannot modify this record."))
```

### Filtering by Rules (Non-Raising)

```python
# Filtering by rules (non-raising)
# Returns the subset of self for which the operation is allowed
allowed_read = self._filter_access_rules('read')          # SQL-based
allowed_write = self._filter_access_rules_python('write') # Python domain based

# Field-level group check for an explicit field list
self.check_field_access_rights('read', ['secret_code', 'name'])  # AccessError
```

---

## Portal and Public Access

### Base Groups

| Group external ID | Meaning |
|-------------------|---------|
| `base.group_user` | Internal user (employees) |
| `base.group_portal` | Portal user |
| `base.group_public` | Public user (logged-out access) |
| `base.group_system` | Settings / technical admin |
| `base.group_erp_manager` | Access Rights admin |
| `base.group_no_one` | Technical Features |

`base.public_user` is the user record that carries `base.group_public`. A new portal signup clones it. `base.template_portal_user_id` is the disabled template holding `base.group_portal`.

### Granting Portal/Public Access

Portal and public users are still governed by the same two layers - give them an ACL row (often with empty `group_id` or `base.group_portal`) **plus** a group-specific record rule that narrows them to their own records.

```xml
<!-- ACL: read-only for the portal group -->
<!-- (in ir.model.access.csv) -->
access_trip_portal,trip.portal,model_business_trip,base.group_portal,1,0,0,0

<!-- Record rule: only the portal user's own partner records -->
<record id="trip_portal_rule" model="ir.rule">
    <field name="name">Trip: portal own</field>
    <field name="model_id" ref="model_business_trip"/>
    <field name="domain_force">[('partner_id', '=', user.partner_id.id)]</field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>
```

### Res.partner Sharing Rules

`base.security/base_security.xml` defines `res_partner_portal_public_rule`, which gives portal/public read access only to their commercial partner subtree (`[('id', 'child_of', user.commercial_partner_id.id)]`). When you add a portal-facing model, mirror this pattern rather than granting blanket partner read access.

---

## ir.attachment Access

### Model-Level ACL

`base/security/ir.model.access.csv` defines:

```csv
access_ir_attachment_group_user,ir_attachment group_user,model_ir_attachment,base.group_user,1,1,1,1
access_ir_attachment_group_portal_public,ir_attachment group_portal_public,model_ir_attachment,,0,0,0,0
```

Basic attachment CRUD is ACL-controlled. But `ir.attachment` adds a layer: access also depends on the **referred record** (`res_model` / `res_id`).

### `ir.attachment.check(mode, values=None)`

Overrides the standard `check`: for each attachment, it verifies the user can access the linked record:

- If `public` and `mode == 'read'`, allow.
- If the attachment has a `res_field` (binary field) or is orphaned (`res_id` false) and not created by the user, deny unless the user is admin/system.
- For each attached `(res_model, res_id)`, call `records.check_access_rights(mode)` and `records.check_access_rule(mode)` on the parent record.

The same logic is re-applied in the overridden `_search()` so a `search()` / `search_read()` silently drops attachments whose parent record the user cannot read.

### Access Token

```python
access_token = fields.Char('Access Token', groups="base.group_user")
```

`access_token` is field-restricted to internal users; external access to a specific attachment is granted by the token generated through `generate_access_token()`, passed on the `/web/content` URL.

### Serving Binary Attachments

A `type='binary'` attachment with a `url` set can be served by `ir.http`'s dispatch fallback. Only groups returned by `get_serving_groups()` (default `['base.group_system']`) may create/write such records - enforced by `_check_serving_attachments`.

---

## Security Pitfalls

### 1. Public Methods Are Callable via RPC

Any method that does **not** start with an underscore can be invoked by any authenticated user via JSON-RPC / XML-RPC. Always gate-keep:

```python
# BAD: any user can call this
def action_force_done(self, new_state):
    self.write({'state': new_state})

# GOOD: private worker + controlled public entry point
def action_force_done(self):
    if not self.env.user.has_group('my_module.group_trip_manager'):
        raise AccessError(_("Not allowed."))
    self._set_state('done')

def _set_state(self, state):
    self.sudo().write({'state': state})
```

Note: Odoo 13 has **no `@api.private` decorator** (that is a later-version API, not available in 13.0). The leading-underscore convention is the only standard mechanism to keep a method non-RPC-callable; rename the method if you must.

### 2. Bypassing the ORM with Raw SQL

Raw SQL bypasses ACLs, record rules, field-level access, translations, and computed fields.

```python
# BAD
self.env.cr.execute("SELECT id FROM business_trip WHERE user_id = %s", (user_id,))
rows = self.env.cr.fetchall()

# GOOD
trips = self.env['business.trip'].search([('user_id', '=', user_id)])
```

When SQL is truly necessary, always use parameter substitution (`%s` placeholders) and post-filter results through `browse().exists()`.

### 3. SQL Injection

```python
# VERY BAD: concatenation/formatting opens SQL injection
self.env.cr.execute("SELECT id FROM my_table WHERE name = '" + user_input + "'")

# GOOD: parameterised query
self.env.cr.execute("SELECT id FROM my_table WHERE name = %s", (user_input,))
```

### 4. XSS via `t-raw`

Odoo 13 QWeb has `t-esc` (escapes) and `t-raw` (renders raw HTML). There is **no `t-out`** in 13.0.

```xml
<!-- BAD: renders user content as HTML -->
<div t-raw="message"/>

<!-- GOOD: auto-escaped -->
<div t-esc="message"/>
```

In Python, escape before wrapping in markup:

```python
from markupsafe import Markup, escape

# BAD: f-strings insert content before escaping
body = Markup(f"<p>{user_input}</p>")

# GOOD: % formatting escapes the content
body = Markup("<p>%s</p>") % user_input
```

### 5. Dangerous `eval`

```python
from ast import literal_eval
from odoo.tools import safe_eval

# VERY BAD
domain = eval(self.filter_domain)

# BAD (can still execute many functions with default globals)
domain = safe_eval(self.filter_domain)

# GOOD for simple literal expressions
domain = literal_eval(self.filter_domain)
```

`ir.rule.domain_force` is inherently `safe_eval`-ed by the framework - keep rule domains to plain field/value domain terms.

### 6. `getattr` on Arbitrary Field Names

```python
# BAD: can access private methods / attributes
value = getattr(record, user_supplied_field)

# GOOD: goes through the ORM, respects field-level groups
value = record[user_supplied_field]   # AccessError if field is restricted
```

### 7. Skipping `check_company`

Forgetting `check_company=True` on Many2one fields in multi-company models allows cross-company references, leaking data across companies.

```python
# BAD
partner_id = fields.Many2one('res.partner')

# GOOD
partner_id = fields.Many2one('res.partner', check_company=True)
```

And set `_check_company_auto = True` on the model to enforce checks on every write/create.

### 8. Overusing `sudo()`

```python
# BAD: wide open
def action_archive(self):
    for rec in self:
        rec.sudo().write({'active': False})

# GOOD: only elevate where required, after gatekeeping
def action_archive(self):
    self.check_access_rights('write')
    self.check_access_rule('write')
    self.sudo().write({'active': False})
```

### 9. Validation Must Survive the ORM

Use database constraints (`_sql_constraints`) or `@api.constrains` to enforce invariants, not just pre-checks in a controller or method.

```python
_sql_constraints = [
    ('amount_positive', 'CHECK (amount >= 0)', 'Amount must be positive'),
]
```

### 10. Field Whitelisting on Public Endpoints

HTTP controllers receiving user values must restrict which fields they forward to `write()`:

```python
ALLOWED_FIELDS = {'name', 'note', 'partner_id'}

def _apply_form(self, record, values):
    cleaned = {k: v for k, v in values.items() if k in ALLOWED_FIELDS}
    record.sudo().write(cleaned)
```

Controller routes use `auth='public'|'user'|'none'`; a `type='json'` route is reachable without CSRF in some cases, so never trust `request.jsonrequest` values without validation.

---

## Security Review Checklist

| Item | Done |
|------|------|
| Every non-transient, non-abstract model has at least one `ir.model.access.csv` row | |
| `ir.model.access.csv` uses exactly the 8 columns `id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink` | |
| Groups declared with `category_id` and `implied_ids` hierarchy | |
| Record rules for own/all access per group | |
| Global rule for `company_id` in multi-company models (using `company_ids`) | |
| `check_company=True` on Many2one + `_check_company_auto = True` in multi-company models | |
| Sensitive fields restricted with `groups=` | |
| Public methods gated with `has_group()` / `check_access_rights()` | |
| No raw SQL with string concatenation | |
| No `t-raw` with user-provided content | |
| `sudo()` used sparingly, never to skip a permission the caller should have | |
| Portal/public access narrowed by group-specific record rule on top of ACL | |
| Attachments rely on parent-record access (`ir.attachment.check`) | |

### Minimal Security Scaffold

`security/security_groups.xml`:

```xml
<odoo>
    <record id="module_category_my_module" model="ir.module.category">
        <field name="name">My Module</field>
        <field name="sequence">20</field>
    </record>

    <record id="group_my_user" model="res.groups">
        <field name="name">User</field>
        <field name="category_id" ref="module_category_my_module"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <record id="group_my_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="module_category_my_module"/>
        <field name="implied_ids" eval="[(4, ref('my_module.group_my_user'))]"/>
    </record>

    <record id="my_model_user_rule" model="ir.rule">
        <field name="name">My Model: own</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[('user_id', '=', user.id)]</field>
        <field name="groups" eval="[(4, ref('my_module.group_my_user'))]"/>
    </record>

    <record id="my_model_manager_rule" model="ir.rule">
        <field name="name">My Model: all</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[(1, '=', 1)]</field>
        <field name="groups" eval="[(4, ref('my_module.group_my_manager'))]"/>
    </record>

    <record id="my_model_company_rule" model="ir.rule">
        <field name="name">My Model: multi-company</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
    </record>
</odoo>
```

`security/ir.model.access.csv`:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,my_module.group_my_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_my_manager,1,1,1,1
```

Register both in the manifest (order matters - the groups XML must load before the CSV references them):

```python
# __manifest__.py
{
    'author': 'Example',
    'data': [
        'security/security_groups.xml',
        'security/ir.model.access.csv',
        ...
    ],
}
```

---

## Base Code Reference

The guide is based on the Odoo 13.0 source tree. Reference files:

| File | Contents |
|------|----------|
| `odoo/addons/base/models/ir_model.py` | `ir.model.access` fields and `check()` ACL evaluation |
| `odoo/addons/base/models/ir_rule.py` | `ir.rule` fields, `_eval_context`, `_compute_domain` global/group combination, `_get_rules`, `_make_access_error` |
| `odoo/addons/base/models/res_users.py` | `res.groups` (with `implied_ids`), `has_group`, `user_has_groups`, `with_user` |
| `odoo/models.py` | `check_access_rights`, `check_access_rule`, `check_field_access_rights`, `_filter_access_rules`, `_check_company`, `_check_company_auto`, `sudo`, `with_user`, `fields_get` |
| `odoo/api.py` | `Environment.su`, `is_superuser`, `is_admin`, `is_system`, `env.company`, `env.companies`, `env.lang` |
| `odoo/addons/base/models/ir_ui_view.py` | view `groups_id` and view/field group filtering |
| `odoo/addons/base/models/ir_ui_menu.py` | menu `groups_id` |
| `odoo/addons/base/models/ir_actions.py` | action `groups_id` |
| `odoo/addons/base/models/ir_attachment.py` | attachment `check()`, access token, serving groups |
| `odoo/addons/base/security/base_groups.xml` | `group_user`, `group_portal`, `group_public`, `public_user` |
| `odoo/addons/base/security/base_security.xml` | portal/public partner rules, multi-company rules |
| `odoo/tools/convert.py` | `convert_csv_import` CSV loading |
| `odoo/exceptions.py` | `AccessError`, `UserError`, `ValidationError`, `MissingError` |