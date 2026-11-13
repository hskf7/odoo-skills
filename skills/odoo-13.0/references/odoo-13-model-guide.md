---
name: odoo-13-model
description: Complete reference for Odoo 13 ORM model methods, model attributes, CRUD operations, domain syntax, recordset handling, and environment context. Use this guide when writing model methods, ORM queries, search operations, or working with recordsets.
globs: "**/models/**/*.py"
topics:
  - Model attributes (_name, _inherit, _inherits, _table, _order, _rec_name)
  - Recordset basics (browse, ids, exists, ensure_one, empty recordsets)
  - Search methods (search, search_count, search_read, read)
  - Aggregation with read_group
  - CRUD operations (create, write, unlink, copy, default_get)
  - Display name methods (name_get, name_search, _name_search)
  - Domain syntax and operators
  - Environment context (with_context, sudo, with_user, with_env)
  - Recordset utilities (mapped, filtered, filtered_domain, sorted)
  - Common pitfalls (N+1, prefetch, singleton errors)
when_to_use:
  - Writing model methods in Odoo 13
  - Performing ORM queries and CRUD operations
  - Building and debugging search domains
  - Aggregating data with read_group
  - Iterating recordsets efficiently
  - Changing user, company, or context on a recordset
---

# Odoo 13 Model Guide

Complete reference for Odoo 13 ORM model methods, model attributes, CRUD
operations, domain syntax, and recordset handling. Every signature here was
verified against the Odoo 13.0 source; later-version differences are called out
explicitly.

## Table of Contents

1. [Model Attributes](#model-attributes)
2. [Recordset Basics](#recordset-basics)
3. [Search Methods](#search-methods)
4. [read_group Aggregation](#read_group-aggregation)
5. [CRUD Operations](#crud-operations)
6. [Display Name Methods](#display-name-methods)
7. [Domain Syntax](#domain-syntax)
8. [Environment Context](#environment-context)
9. [Recordset Utility Methods](#recordset-utility-methods)
10. [Iteration Patterns and Prefetch](#iteration-patterns-and-prefetch)
11. [Common Pitfalls](#common-pitfalls)
12. [Base Code Reference](#base-code-reference)

---

## Model Attributes

```python
from odoo import api, fields, models


class SaleOrder(models.Model):
    _name = 'sale.order'                  # registry key, dot notation (required)
    _description = 'Sales Order'          # human label
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'date_order DESC, id DESC'   # default ORDER BY for search()
    _rec_name = 'name'                    # field used by name_get()
    _check_company_auto = True            # validate check_company fields

    name = fields.Char(required=True)
```

| Attribute | Type | 13.0 default | Purpose |
|-----------|------|--------------|---------|
| `_name` | str | `None` | Model name; required for concrete models |
| `_description` | str | `_name` | Informal label used in logs and UI |
| `_inherit` | str or list | `None` | Parent models to extend or mix in |
| `_inherits` | dict | `{}` | Delegation: `{'parent.model': 'm2o_field'}` |
| `_table` | str | `_name.replace('.', '_')` | SQL table name when `_auto` is true |
| `_sequence` | str | `<table>_id_seq` | SQL sequence for the `id` column |
| `_order` | str | `'id'` | Default ordering for `search()` |
| `_rec_name` | str | auto-detected | Field used to label records |
| `_auto` | bool | `True` on `Model` | Whether a table is created |
| `_log_access` | bool | `_auto` | Maintain `create_date`, `create_uid`, `write_date`, `write_uid` |
| `_abstract` | bool | `False` on `Model` | No table, not registered |
| `_transient` | bool | `False` on `Model` | Wizard model, vacuum-cleaned |
| `_parent_store` | bool | `False` | Store `parent_path` for hierarchy queries |
| `_parent_name` | str | `'parent_id'` | Many2one used as hierarchy parent |
| `_sql_constraints` | list | `[]` | `[(name, sql_definition, message), ...]` |
| `_check_company_auto` | bool | `False` | Call `_check_company()` on create/write |
| `_register` | bool | `False` on base classes | Whether the model enters the registry |
| `_custom` | bool | `False` | Marks studio-created models |

### _name and _inherit

```python
# Extend in place: no _name
class ResPartner(models.Model):
    _inherit = 'res.partner'
    loyalty_code = fields.Char()


# New model: _name + _inherit copies all fields into its own table
class CustomPartner(models.Model):
    _name = 'custom.partner'
    _inherit = 'res.partner'
    extra_note = fields.Text()
```

A list of `_inherit` values mixes in several models.

### _inherits

`_inherits = {'parent.model': 'm2o_field'}` exposes a parent model's fields
through a required Many2one; values stay on the parent record.

```python
class ResUsers(models.Model):
    _name = 'res.users'
    _inherits = {'res.partner': 'partner_id'}

    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
```

Reading `user.name` reads `user.partner_id.name`; writing it writes to the
partner. If two inherited models define the same field, the last one in the
`_inherits` order wins.

### _order and _rec_name

`_order` is a comma-separated list of field names with optional `ASC` / `DESC`,
validated by `regex_order` (`odoo/models.py:69`); SQL expressions are not
accepted. For a Many2one, `_generate_m2o_order_by()` uses the comodel's
`_order`, falling back to its `_rec_name` when that `_order` is too complex.
An invalid field raises `ValueError` at query time, not at registry load.

```python
_order = 'priority DESC, date_order ASC, id DESC'
```

`_rec_name` names the field used by `name_get()` and `display_name`. When unset,
13.0 auto-selects `'name'` if present, then `'x_name'` for custom models;
`_rec_name_fallback()` otherwise returns `'id'`, and `name_get()` returns
`"model.name,id"` strings. There is no `_rec_names_search` in 13.0:
`name_search()` searches the single `_rec_name` field. To search several
fields, override `_name_search()`.

### _auto, _log_access, _abstract, _transient

- `Model`: `_auto = True`, `_abstract = False`, `_transient = False`.
- `TransientModel`: `_transient = True`; records are vacuum-cleaned.
- `AbstractModel` is an alias of `BaseModel` with `_abstract = True`: no table,
  never registered.
- `_auto = False` means no table; override `init()` to create it.
- `_log_access` defaults to `_auto` (`cls._log_access = cls._auto`); `False`
  removes the four audit columns.

```python
class ReportWizard(models.TransientModel):
    _name = 'my.report.wizard'
```

### _parent_store

`_parent_store = True` requires an indexed `parent_path` Char field. Odoo
maintains a materialized path (`/1/5/17/`), making `child_of` / `parent_of`
indexed `LIKE` queries instead of recursive ones. A missing `parent_path` is
logged as an error and hierarchy queries fall back to recursive SQL.

```python
class ProductCategory(models.Model):
    _name = 'product.category'
    _parent_store = True
    _order = 'parent_path'

    name = fields.Char(required=True)
    parent_id = fields.Many2one('product.category', ondelete='cascade')
    parent_path = fields.Char(index=True)
```

### _sql_constraints and _check_company_auto

```python
_sql_constraints = [
    ('code_uniq', 'UNIQUE(code)', 'The code must be unique.'),
    ('qty_positive', 'CHECK(qty >= 0)', 'Quantity must be positive.'),
]
```

The constraint name is `<table>_<name>`. On module update,
`_add_sql_constraints()` drops and re-adds any constraint whose definition
changed.

With `_check_company_auto = True`, `create()` and `write()` call
`_check_company()` for every field declared `check_company=True`, raising
`UserError` when a related record's `company_id` is neither `False` nor the
record's own company.

```python
class AccountMove(models.Model):
    _name = 'account.move'
    _check_company_auto = True

    company_id = fields.Many2one('res.company', required=True)
    journal_id = fields.Many2one('account.journal', check_company=True)
```

---

## Recordset Basics

```python
record = self.browse(1)             # one-record recordset
records = self.browse([1, 2, 3])    # several records
empty = self.browse()               # empty recordset
empty = self.env['res.partner']     # also an empty recordset
```

`browse()` never queries the database; non-existent ids stay until a field is
read or `exists()` is called.

```python
record.id       # single record only; raises on a multi-record set
records.ids     # [1, 2, 3] in recordset order
len(records)    # number of records
bool(records)   # False when empty
records[0]      # first record
records[1:3]    # slice -> recordset
```

Field access on an empty recordset returns the field's null value (`False`,
`0`, empty recordset for relational fields). Field access on a multi-record set
raises `ValueError: Expected singleton`, except x2many fields, which return the
union of related records.

### exists() and ensure_one()

```python
valid = self.browse([1, 999, 1000]).exists()   # drops missing ids

order = self.env['sale.order'].search([('name', '=', ref)], limit=1)
order.ensure_one()   # ValueError("Expected singleton: ...") otherwise
```

`exists()` runs one `SELECT id ... WHERE id IN (...)` and marks deleted ids in
the cache; new (unsaved) records are returned as existing. `ensure_one()` is
cheap and belongs at the top of methods that operate on exactly one record.

---

## Search Methods

### search()

Signature in 13.0 (`odoo/models.py:1639`):

```python
def search(self, args, offset=0, limit=None, order=None, count=False)
```

```python
records = self.search([('state', '=', 'draft')])
records = self.search(
    [('state', '=', 'draft')], limit=10, order='date_order DESC, id DESC',
)
records = self.search([('state', '=', 'draft')], offset=20, limit=10)
count = self.search([('state', '=', 'draft')], count=True)   # int
all_records = self.search([])                                # empty domain = all
```

`search()` applies `_order` when `order` is omitted, and applies the implicit
`active_test` filter: models with an `active` field exclude `active = False`
records unless the domain mentions `active` or the context sets
`active_test=False`.

```python
all_partners = self.env['res.partner'].with_context(active_test=False).search([])
```

### search_count()

Signature in 13.0 (`odoo/models.py:1626`):

```python
def search_count(self, args)
```

```python
count = self.search_count([('state', '=', 'draft')])
```

It calls `search(args, count=True)`. There is no `limit` parameter in 13.0;
Odoo 17's `search_count(domain, limit=...)` is not available.

### search_read()

Signature in 13.0 (`odoo/models.py:4955`):

```python
def search_read(self, domain=None, fields=None, offset=0, limit=None, order=None)
```

```python
rows = self.search_read(
    [('state', '=', 'done')], ['name', 'amount_total'], limit=10,
)   # [{'id': 1, 'name': 'SO001', 'amount_total': 100.0}, ...]
```

In 13.0 this is literally `search()` followed by `read()`; it is not a
single-query optimization and has no `read_group` parameter. `id` is always
present. When `fields == ['id']` it short-circuits without reading columns.

### read()

Signature in 13.0 (`odoo/models.py:2933`):

```python
def read(self, fields=None, load='_classic_read')
```

```python
data = records.read(['name', 'state', 'date_order'])   # list of dicts
data = records.read()                                  # all fields
```

`read()` is a low-level/RPC method. In Python code prefer direct attribute
access (`record.name`), which uses and fills the record cache. Unknown field
names raise `ValueError`; field access rights are checked per field.

---

## read_group Aggregation

Odoo 13.0 has one public aggregation method:

```python
def read_group(self, domain, fields, groupby, offset=0, limit=None,
               orderby=False, lazy=True)
```

There is no public `_read_group()`: `read_group()` wraps the private
`_read_group_raw()` and formats date labels. Odoo 17's `_read_group(...)` does
not exist here.

### Field Specification

| Form | Meaning |
|------|---------|
| `'amount_total'` | Field with its default aggregation |
| `'amount_total:sum'` | Field with an explicit aggregate |
| `'total:sum(amount_total)'` | Aggregate aliased to a result key |

Valid aggregates in 13.0 (`VALID_AGGREGATE_FUNCTIONS`, `odoo/models.py:247`):
`array_agg`, `count`, `count_distinct`, `bool_and`, `bool_or`, `max`, `min`,
`avg`, `sum`.

```python
groups = self.read_group(
    domain=[('state', '=', 'draft')], fields=['__count'], groupby=['partner_id'],
)

groups = self.read_group(
    domain=[('state', '=', 'done')],
    fields=['total:sum(amount_total)', 'lines:count(id)'], groupby=['partner_id'],
)
```

`count` counts non-null field values; `__count` counts rows.

### Groupby, Granularities, and lazy

A groupby entry is a field name or `'field:granularity'`; supported
granularities are `day`, `week`, `month`, `quarter`, and `year`, only for
Date/Datetime fields.

```python
groups = self.read_group(
    domain=[('date_order', '>=', '2024-01-01')],
    fields=['amount_total:sum'], groupby=['date_order:month'],
)
for group in groups:                  # drill down with __domain
    orders = self.search(group['__domain'])

# lazy=False applies all groupbys in one call; lazy=True (default) applies
# only the first and stores the rest in __context for the web client.
data = self.read_group(
    domain=[], fields=['amount_total:sum'],
    groupby=['partner_id', 'date_order:month'], lazy=False,
)
```

Date values in the result are formatted labels, not date objects. Each result
dict also carries `<groupby_field>_count` (rows in the group), `__domain`, and
`__context`. Many2one group values are `(id, display_name)` tuples. `orderby`
overrides natural group ordering; the 13.0 docstring notes it is supported only
for Many2one fields.

### group_expand

A field may name a method returning the full group list, so empty groups appear:

```python
state = fields.Selection(
    [('draft', 'Draft'), ('confirmed', 'Confirmed'), ('done', 'Done')],
    group_expand='_read_group_expand_states',
)

@api.model
def _read_group_expand_states(self, states, domain, order):
    return [key for key, _label in type(self).state.selection]
```

Groupby and aggregated fields must be stored columns; otherwise `read_group`
raises `UserError` (`database-persisted fields` / `Cannot aggregate field`).
Read access and record rules apply.

---

## CRUD Operations

### create()

Signature in 13.0 (`odoo/models.py:3726`):

```python
@api.model_create_multi
@api.returns('self', lambda value: value.id)
def create(self, vals_list)
```

It accepts a dict or a list of dicts:

```python
partner = self.env['res.partner'].create({'name': 'Azure Interior'})
partners = self.env['res.partner'].create([
    {'name': 'First'},
    {'name': 'Second'},
])
```

A dict is treated as `[dict]` and returns a one-record recordset; a list returns
all created records. Override it with `@api.model_create_multi` and iterate
`vals_list`:

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            vals.setdefault('state', 'draft')
        orders = super().create(vals_list)
        orders._after_create()
        return orders
```

Without the decorator an override receives whatever the caller passed (dict or
list), which breaks batch callers. `create()` also fills missing values from
`default_get()` and runs `_check_company()` when `_check_company_auto` is true.

### write() and unlink()

```python
def write(self, vals)     # odoo/models.py:3449
def unlink(self)          # odoo/models.py:3358
```

```python
records.write({'state': 'done'})
records.write({'state': 'done', 'date_done': fields.Datetime.now()})
self.env['sale.order'].browse(ids).unlink()
```

Both return `True` and raise `AccessError` without the matching rights. One SQL
statement is batched for the whole recordset; a per-record loop issues N
statements and N recomputations.

13.0 has no `@api.ondelete` decorator; block deletion by overriding `unlink()`:

```python
from odoo.exceptions import UserError

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def unlink(self):
        if any(order.state != 'draft' for order in self):
            raise UserError("Only draft orders can be deleted.")
        return super().unlink()
```

### copy() and default_get()

```python
def copy(self, default=None)          # odoo/models.py:4683
def default_get(self, fields_list)    # odoo/models.py:1216
```

```python
new_order = order.copy(default={'name': '%s (copy)' % order.name})

defaults = self.env['sale.order'].default_get(['partner_id', 'state'])
# {'partner_id': 7, 'state': 'draft'}
```

`copy()` calls `ensure_one()`, so it fails on a multi-record set. It builds
values with `copy_data()` (skipping `copy=False` fields), creates the record,
then copies translations. `default` overrides original values.

`default_get()` resolves context keys named `default_<field>`, then `ir.default`
values, then field defaults. This is why `with_context(default_state='draft')`
affects forms and `create()`.

### x2many Commands

Odoo 13.0 has no `Command` helper class; use the classic tuples:

| Tuple | Meaning |
|-------|---------|
| `(0, 0, vals)` | Create a related record |
| `(1, id, vals)` | Update a related record |
| `(2, id, 0)` | Delete the related record from the database |
| `(3, id, 0)` | Remove the relation without deleting the record |
| `(4, id, 0)` | Link an existing record |
| `(5, 0, 0)` | Remove all relations |
| `(6, 0, ids)` | Replace the relation with this list of ids |

```python
order.write({
    'order_line': [
        (0, 0, {'product_id': 2, 'product_uom_qty': 3}),
        (1, existing_line.id, {'product_uom_qty': 5}),
        (2, obsolete_line.id, 0),
    ],
    'tag_ids': [(6, 0, [1, 2, 3])],
})
```

---

## Display Name Methods

### name_get()

```python
def name_get(self):    # returns [(id, display_name), ...]
```

The default returns `_rec_name` converted with `convert_to_display_name`. If
`_rec_name` is unset or not a field, it returns `"model.name,id"`.

```python
class ResPartner(models.Model):
    _inherit = 'res.partner'

    def name_get(self):
        return [
            (partner.id, '%s [%s]' % (partner.name, partner.ref or '-'))
            for partner in self
        ]
```

`display_name` is computed by calling `name_get()`, so overriding `name_get()`
also changes `display_name`.

### name_search() and _name_search()

```python
def name_search(self, name='', args=None, operator='ilike', limit=100)   # models.py:1723
def _name_search(self, name='', args=None, operator='ilike', limit=100,
                 name_get_uid=None)                                      # models.py:1750
```

`name_search()` returns `(id, display_name)` pairs for Many2one widgets and
`name_create()`; it delegates to `_name_search()`, which must return
`(id, name)` pairs (core uses `lazy_name_get()`).

```python
from odoo.models import lazy_name_get
class ResPartner(models.Model):
    _inherit = 'res.partner'

    @api.model
    def _name_search(self, name='', args=None, operator='ilike', limit=100,
                     name_get_uid=None):
        args = list(args or [])
        if name:
            args += ['|', ('name', operator, name), ('ref', operator, name)]
        ids = self._search(args, limit=limit, access_rights_uid=name_get_uid)
        return lazy_name_get(self.browse(ids).with_user(name_get_uid))
```

`name_search()` alone cannot search several fields in 13.0; `_name_search()` is
the extension point. `name_create(name)` creates a record using only
`_rec_name` and returns its `name_get()` pair; it warns and returns `False`
when the model has no `_rec_name`.

---

## Domain Syntax

### Operators

Valid operators in 13.0 (`TERM_OPERATORS`, `odoo/osv/expression.py:145`):

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | equals | `[('state', '=', 'draft')]` |
| `!=` | not equals | `[('state', '!=', 'draft')]` |
| `>` `>=` `<` `<=` | comparison | `[('amount', '>=', 100)]` |
| `=?` | equals, or always true when value is `False`/`None` | `[('partner_id', '=?', partner)]` |
| `in` / `not in` | membership in a list | `[('id', 'in', [1, 2, 3])]` |
| `like` | matches pattern; `%` and `_` are wildcards | `[('name', 'like', 'test')]` |
| `ilike` | case-insensitive `like` | `[('name', 'ilike', 'TEST')]` |
| `not like` / `not ilike` | negated patterns | `[('name', 'not ilike', 'test')]` |
| `=like` | `like` without implicit wildcards | `[('code', '=like', 'A_1%')]` |
| `=ilike` | case-insensitive `=like` | `[('code', '=ilike', 'a_1%')]` |
| `child_of` | record is a descendant of the value | `[('categ_id', 'child_of', 4)]` |
| `parent_of` | record is an ancestor of the value | `[('categ_id', 'parent_of', 4)]` |

There is no `any` / `not any` operator in 13.0 (added in later versions). For
"at least one related record matches", use a dotted path or search the comodel:

```python
domain = [('order_line.product_id', '=', product_id)]   # dotted traversal

domain = [('id', 'in', self.env['sale.order.line'].search([
    ('product_id', '=', product_id),
]).ids)]                                                # explicit form
```

`like` / `ilike` wrap the value in `%...%`; `=like` / `=ilike` use it as-is, so
include wildcards yourself. `=?` compiles to `TRUE` when the right-hand value
is `False` or `None`, and behaves like `=` otherwise.

### Logical Operators

Domains use prefix notation: `&` and `|` consume the next two terms, `!` the
next one. Consecutive terms are implicitly ANDed.

```python
domain = [('state', '=', 'draft'), ('date_order', '>=', '2024-01-01')]

domain = ['|', ('state', '=', 'draft'), ('state', '=', 'confirmed')]

domain = ['!', ('state', '=', 'draft')]

domain = [
    '&',
    '|', ('state', '=', 'draft'), ('state', '=', 'confirmed'),
    '|', ('date_order', '>=', '2024-01-01'), ('date_order', '=', False),
]
```

### Relational Traversal and Dates

```python
domain = [('partner_id.country_id.code', '=', 'US')]
domain = [('order_line.product_id.categ_id', '=', 1)]   # any related record

from odoo import fields
domain = [('date_order', '>=', fields.Date.today())]
domain = [('create_date', '>=', fields.Datetime.now())]
```

Prefer `('id', 'in', ids)` over repeated `|` terms; an empty `in` list is
optimized to a false domain without querying. Server-format strings
(`DEFAULT_SERVER_DATE_FORMAT` / `DEFAULT_SERVER_DATETIME_FORMAT`) are the
portable date values.

---

## Environment Context

### with_context()

```python
def with_context(self, *args, **kwargs)     # odoo/models.py:5200
```

Returns a new recordset sharing the same prefetch object; the original is not
modified.

```python
records = records.with_context(active_test=False)

records = records.with_context({}, key='value')   # replace context, then add

names = records.with_context(lang='fr_FR').mapped('name')
records = records.with_context(allowed_company_ids=[1, 2])
```

`self.env.context` is the plain dictionary; `self._context` is a
backward-compatible property returning the same dictionary.

### sudo(), with_user(), with_env()

```python
record.sudo().write({'note': 'System note'})      # bypass rights and rules
record.with_user(user).action_confirm()           # run as another user
record.with_env(other_env)                        # attach a different environment
```

In 13.0, `sudo(user)` still works but logs a deprecation warning and delegates
to `with_user(user)`. `with_user()` changes the user, `sudo()` only bypasses
security. `with_env()` does not reuse the current cache, so later field
accesses may query again; it keeps the same prefetch ids.

### Environment Properties

```python
self.env.user       # res.users record of the current user
self.env.company    # first allowed_company_ids, else user.company_id
self.env.companies  # allowed companies, else all user companies
self.env.lang       # current language code, or None
self.env.context    # context dict
self.env.ref('base.main_company')     # resolve an XML id
self.env.is_superuser() / is_admin() / is_system()
```

`env.company` / `env.companies` read `allowed_company_ids`. In sudo mode no
sanity check is applied; otherwise an unauthorized company id raises
`AccessError`.

### Changing Company

Odoo 13.0 has no `with_company()`. Change the allowed companies in the context:

```python
record.with_context(
    allowed_company_ids=self.env['res.company'].browse(company_id).ids,
).read(['property_account_receivable_id'])
```

`with_company()` and `env.company_id` are later-version APIs and are not
available in 13.0.

## Recordset Utility Methods

```python
def mapped(self, func)                     # models.py:5311
def filtered(self, func)                   # models.py:5363
def filtered_domain(self, domain)          # models.py:5383
def sorted(self, key=None, reverse=False)  # models.py:5484
```

### mapped() and filtered()

`func` is a callable or a dot-separated field path. `mapped()` returns a list,
or a recordset when values are records (duplicates removed, order arbitrary).
`filtered()` keeps records for which the predicate or path is truthy, works in
memory, and preserves order.

```python
names = records.mapped('name')
banks = records.mapped('partner_id.bank_ids')
doubled = records.mapped(lambda r: r.amount_total * 2)

done_orders = orders.filtered(lambda o: o.state == 'done')
companies = partners.filtered('is_company')
```

On an empty recordset with a callable, `mapped()` calls `func(self)` with the
empty recordset; a lambda reading a scalar field then raises
`Expected singleton`.

### filtered_domain()

Evaluates a domain in Python against records in memory, preserving order.

```python
done = orders.filtered_domain([('state', '=', 'done')])

urgent = orders.filtered_domain([
    '&', ('state', '=', 'draft'),
    '|', ('priority', '=', '2'), ('date_order', '<', fields.Date.today()),
])
```

`child_of` / `parent_of` cannot be evaluated in memory; those operators fall
back to a `search()`.

### sorted()

```python
by_name = records.sorted('name')
by_amount = records.sorted(key=lambda r: r.amount_total, reverse=True)
```

`key` may be a field name, a callable, or `None`. With `None`, 13.0 re-runs a
search on the current ids to apply `_order`: it issues a query and drops
unsaved records, so prefer an explicit key for in-memory sorting.

---

## Iteration Patterns and Prefetch

The prefetch set is `_prefetch_ids`; slicing keeps the parent set, while
`browse()` on a single id has a one-id prefetch set. `with_prefetch(ids)` sets
an explicit set when a computed field returns records read together later.

### N+1 Queries and Batch Writes

```python
# BAD: one query per order
for order in orders:
    payments = self.env['account.payment'].search([('order_id', '=', order.id)])

# GOOD: one query for all payments
payments = self.env['account.payment'].search([('order_id', 'in', orders.ids)])
payments_by_order = {}
for payment in payments:
    payments_by_order.setdefault(payment.order_id.id, []).append(payment)

# BAD: N UPDATE statements and N recomputations
for record in records:
    record.write({'state': 'done'})

# GOOD: one UPDATE for the whole set
records.write({'state': 'done'})
```

When each record needs a different value, write per group rather than per record.

---

## Common Pitfalls

### Singleton Errors and Empty Recordsets

```python
total = records.amount_total                   # BAD: Expected singleton
total = sum(records.mapped('amount_total'))    # GOOD
record = records.ensure_one()                 # assert exactly one

empty = self.env['res.partner'].browse()
empty.name          # False
empty.child_ids     # empty recordset
empty.mapped(lambda r: r.name)   # ValueError: Expected singleton
```

Relational fields are the exception: One2many / Many2many access on a
multi-record set returns the union of related records. Test `if not records`
before accessing fields that assume one record.

### browse() and Loop Discipline

`browse()` does not validate ids: filter with `.exists()` before use. Every
`search()` is a round trip and flushes pending computations, so hoist searches
out of loops and use `in` domains.

```python
record = self.env['res.partner'].browse(maybe_missing_id).exists()
if not record:
    raise UserError("Partner no longer exists.")

record.name = 'New name'             # in-memory assignment, flushed later
records.write({'name': 'New name'})  # explicit batch update, no singleton

self.env.context['active_test'] = False   # BAD: mutates the environment
records = self.with_context(active_test=False).search([])   # GOOD
```

### sudo() Is Not a User Switch

`sudo()` keeps `env.uid` and bypasses access rights and record rules. Use
`with_user(user)` when code must run as that user. `sudo()` also does not add
the record's company to the allowed companies; company-dependent fields still
follow the context.

### Other Pitfalls

- `read_group` on computed non-stored fields raises `UserError`; store the
  field or aggregate a stored base field.
- Overriding `create(self, vals)` without `@api.model_create_multi` breaks
  callers that pass a list of dicts.
- An invalid field in `_order` raises `ValueError` on the first search, not at
  registry load.

---

## Base Code Reference

Verified against:

- `odoo/models.py` - model attributes,
  `search` / `search_count` / `search_read`, `read_group` / `_read_group_raw`,
  `create` / `write` / `unlink` / `copy`, `name_get` / `name_search` /
  `_name_search`, `browse` / `exists` / `ensure_one`, `mapped` / `filtered` /
  `filtered_domain` / `sorted` / `with_prefetch`, `with_context` / `sudo` /
  `with_user` / `with_env`, `_check_company`, `_check_company_auto`.
- `odoo/api.py` - `Environment`, `env.company`,
  `env.companies`, `env.lang`, `env.ref`, `env.is_superuser` / `is_admin` /
  `is_system`, and the `model`, `model_create_single`, `model_create_multi`,
  `returns`, `depends`, `depends_context`, `constrains`, `onchange` decorators.
- `odoo/fields.py` - field descriptor `__get__`
  (empty and singleton behaviour), `group_expand`, relational field access.
- `odoo/osv/expression.py` - `TERM_OPERATORS`,
  `=?`, `like` / `ilike`, `child_of` / `parent_of`, and the absence of
  `any` / `not any`.
- `odoo/exceptions.py` - `UserError`,
  `AccessError`, `MissingError`, `ValidationError`.

Odoo 13.0 does not provide `_read_group()`, `search_fetch()`, `with_company()`,
`Command`, `_rec_names_search`, `@api.ondelete`, `@api.multi`, or `@api.one`.
Use the 13.0 forms documented above.
