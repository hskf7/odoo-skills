---
name: odoo-13-field
description: Complete reference for Odoo 13 field types, parameters, and when to use each. Use this guide when defining model fields, choosing field types, or configuring field parameters.
globs: "**/models/**/*.py"
topics:
  - Simple fields (Boolean, Integer, Float, Monetary, Char, Text, Html, Date, Datetime, Binary, Image, Selection, Reference)
  - Relational fields (Many2one, Many2oneReference, One2many, Many2many)
  - The special Id field
  - Fields that do not exist in 13.0 (Json, Properties)
  - Computed fields (compute, store, inverse, search)
  - Related fields
  - Field parameters (index, default, copy, store, groups, company_dependent, tracking, group_operator)
when_to_use:
  - Defining new model fields
  - Choosing appropriate field types
  - Configuring computed fields
  - Setting up relational fields
  - Optimizing field parameters
---

# Odoo 13 Field Guide

Complete reference for Odoo 13 field types, parameters, and when to use each.

This guide documents only what exists in Odoo 13.0. It is verified against the
13.0 source tree (`odoo/fields.py`, `odoo/models.py`, and the supporting tools).
Anything introduced in 14.0 or later is either omitted or explicitly marked
"not available in 13.0".

## Table of Contents

1. [Simple Fields](#simple-fields)
2. [Relational Fields](#relational-fields)
3. [The Id Field](#the-id-field)
4. [Fields That Do Not Exist in 13.0](#fields-that-do-not-exist-in-130)
5. [Computed Fields](#computed-fields)
6. [Related Fields](#related-fields)
7. [Field Parameters](#field-parameters)
8. [Indexes](#indexes)
9. [Defaults](#defaults)
10. [Field Type Selection Guide](#field-type-selection-guide)
11. [Base Code Reference](#base-code-reference)

---

## Simple Fields

### Boolean

```python
active = fields.Boolean(default=True)
is_company = fields.Boolean(string='Is a Company')
```

Stored as a PostgreSQL `bool`. The null value is `False`. `convert_to_column`
coerces any assigned value with `bool(value)`.

### Integer

```python
sequence = fields.Integer(default=10)
priority = fields.Integer(default=0)
```

Stored as `int4`. The null value is `0`. Values above `2**31 - 1` are returned
as floats through pure XML-RPC because the XML-RPC protocol cannot represent
them as integers. `group_operator` defaults to `'sum'` on `Integer`.

For decimal values, use `Float`; for currency amounts, use `Monetary`.

### Float

```python
# Fixed precision (total digits, decimal places)
weight = fields.Float(string='Weight', digits=(16, 3))

# Named precision resolved from decimal.precision records
price = fields.Float(string='Unit Price', digits='Product Price')
```

The `digits` argument is either a `(total, decimal)` tuple or a string naming a
`decimal.precision` record. A string is resolved at runtime via
`env['decimal.precision'].precision_get(name)` and returns `(16, precision)`.
When `digits` is `None`, the column type is `float8` (double precision); when
`digits` is set, the column type is `numeric`.

**Use the named precision form for domain-specific numbers** (prices,
quantities) so users can tune precision at `Settings > Technical > Decimal
Accuracy` without a code change. The `decimal_precision` addon is deprecated in
13.0: pass the precision name string directly instead of importing it.

Float rounding and comparison helpers are **not** methods on `fields.Float` in
13.0. They live in `odoo.tools.float_utils`:

```python
from odoo.tools.float_utils import float_round, float_is_zero, float_compare

rounded = float_round(value, precision_rounding=0.01)

if float_is_zero(qty, precision_rounding=self.uom_id.rounding):
    raise UserError(_("Quantity cannot be zero."))

cmp = float_compare(a, b, precision_rounding=0.01)
# cmp < 0  -> a < b
# cmp == 0 -> equal within precision
# cmp > 0  -> a > b
```

Each helper accepts either `precision_digits` (a pair) or `precision_rounding`
(a rounding unit such as `0.01`). Use them instead of raw `==` / `<` to avoid
floating-point drift.

### Monetary

```python
currency_id = fields.Many2one('res.currency', required=True)

amount = fields.Monetary(
    string='Amount',
    currency_field='currency_id',
)
```

`Monetary` is a `Float` expressed in a currency. The precision and symbol come
from the `Many2one` field named by `currency_field`, whose comodel must be
`res.currency`. `currency_field` is optional; if omitted the ORM picks
`currency_id` (or `x_currency_id`) when such a field exists, and asserts
otherwise. The column type is always `numeric`.

The value is rounded with the currency's rounding factor on write and cache
conversion. Never reuse one monetary column across mixed currencies in a
recordset: that is functional nonsense and can raise
`ValueError: Got multiple currencies ...`.

### Char - Short Text

```python
name = fields.Char(
    string='Name',
    required=True,
    size=128,
    translate=True,
)

code = fields.Char(string='Code', index=True, copy=False)
```

Use for short free-form strings (names, references, codes). Stored in a
`varchar` column whose width is derived from `size` (`pg_varchar(size)`); values
are also truncated to `size` on write. `trim` (default `True`) states whether
the web client trims the value; the trim is applied by the client, not the ORM.

### Text - Long Text

```python
description = fields.Text(string='Description', translate=True)
notes = fields.Text(string='Notes')
```

Use for multi-line plain text with no practical length limit. Backed by a
PostgreSQL `text` column.

### Html - Rich Text

```python
content = fields.Html(
    string='Content',
    translate=True,
    sanitize=True,
    sanitize_tags=True,
    sanitize_attributes=True,
    sanitize_style=False,
    strip_style=False,
    strip_classes=False,
    sanitize_form=True,
)
```

Use for HTML fragments (email bodies, CMS content). By default the sanitizer
removes `<script>`, event handlers, and similar vectors. Available sanitizer
parameters (all on the `Html` field in 13.0):

| Parameter | Default | Effect |
|-----------|---------|--------|
| `sanitize` | `True` | Run the sanitizer at all |
| `sanitize_tags` | `True` | Whitelist accepted tags |
| `sanitize_attributes` | `True` | Whitelist accepted attributes |
| `sanitize_style` | `False` | Sanitize inline styles |
| `strip_style` | `False` | Remove style attributes entirely |
| `strip_classes` | `False` | Remove class attributes |
| `sanitize_form` | `True` | Sanitize `<form>` elements |

When `translate=True` and `sanitize=True`, the field automatically switches to
`html_translate`, which translates each content block rather than the raw HTML.

### Date

```python
date_order = fields.Date(
    string='Order Date',
    default=fields.Date.context_today,
    copy=False,
)
```

Stored as a PostgreSQL `date` (no time, no timezone). Helper API on
`fields.Date`:

```python
fields.Date.today()                    # datetime.date.today() (server TZ)
fields.Date.context_today(record)      # today in record.env.user's TZ
fields.Date.to_date('2020-01-15')      # str -> date
fields.Date.to_string(fields.Date.today())  # date -> 'YYYY-MM-DD'
```

`fields.Date.from_string` still exists as a deprecated alias of `to_date` and
is slated for removal; prefer `to_date`.

Period helpers are static methods backed by `odoo.tools.date_utils`:

```python
fields.Date.start_of(fields.Date.today(), 'month')
fields.Date.end_of(fields.Date.today(), 'quarter')
fields.Date.add(fields.Date.today(), months=1)
fields.Date.subtract(fields.Date.today(), weeks=2)
# granularities: year, quarter, month, week, day, hour
```

### Datetime

```python
date_deadline = fields.Datetime(
    string='Deadline',
    default=fields.Datetime.now,
)
```

Stored as `timestamp` (naive UTC). Displayed in the user's timezone. Helper API
on `fields.Datetime`:

```python
fields.Datetime.now()                 # utcnow (microseconds stripped)
fields.Datetime.today()               # today at 00:00:00 UTC
fields.Datetime.to_datetime('2020-01-15 14:30:00')
fields.Datetime.to_string(fields.Datetime.now())
fields.Datetime.context_timestamp(record, dt)  # naive UTC -> user TZ aware
fields.Datetime.start_of(fields.Datetime.now(), 'day')
fields.Datetime.add(fields.Datetime.now(), hours=1)
```

`context_timestamp` is **not** meant for default values; use `now`. Datetime
fields are converted to the user's timezone automatically on display.
`fields.Datetime.from_string` is a deprecated alias of `to_datetime`.

### Binary

```python
file_data = fields.Binary(
    string='File',
    attachment=True,
)
filename = fields.Char(string='Filename')
```

With `attachment=True` (the default), content is stored as an `ir.attachment`
record and the model table holds no column at all. With `attachment=False` the
content is stored in a `bytea` column. The field depends on the `bin_size`
context key: when `bin_size=True` is set, reading the field returns a
human-readable size string (via `odoo.tools.human_size`) instead of the bytes.

```python
self.env['ir.attachment'].with_context(bin_size=True).search([...])
# field value is returned as a size such as "1.23 Kb", not raw bytes
```

`Binary` is not prefetched by default (`prefetch=False`).

### Image

```python
image_1920 = fields.Image(
    string='Image',
    max_width=1920,
    max_height=1920,
    verify_resolution=True,
)

image_512 = fields.Image(
    related='image_1920',
    max_width=512,
    max_height=512,
    store=True,
)
```

`Image` extends `Binary` and resizes on `create` / `write` while keeping aspect
ratio. Parameters: `max_width` (default `0`, no limit), `max_height` (default
`0`), `verify_resolution` (default `True`, rejects images over the maximum
resolution of `45e6` pixels, about 45 Mpx). If both limits are `0` and
`verify_resolution` is `False`, no processing happens; use plain `Binary`
instead.

SVG uploads are restricted to system users by `Binary` (and therefore `Image`).

### Selection

```python
state = fields.Selection(
    [
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
        ('done', 'Done'),
        ('cancel', 'Cancelled'),
    ],
    string='Status',
    default='draft',
    required=True,
)

# Dynamic selection from a method (name or callable)
kind = fields.Selection(selection='_selection_kind', string='Kind')

@api.model
def _selection_kind(self):
    return [(k, k.title()) for k in self._get_kinds()]
```

Selection values are stored as their **keys** in a `varchar` column. Never
rename an existing key without a migration script, because stored rows keep the
old key. Extend a selection in a child module with `selection_add`:

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    state = fields.Selection(
        selection_add=[('locked', 'Locked')],
    )
```

`selection_add` entries are inserted in an order consistent with the overridden
selection. `selection` is mandatory except for `related` or extended fields.
Selection has no `ondelete` parameter in 13.0 (that was added in a later
version).

### Reference

```python
ref = fields.Reference(
    selection='_selection_target_model',
    string='Referenced Record',
)

@api.model
def _selection_target_model(self):
    return [(m.model, m.name) for m in self.env['ir.model'].search([])]
```

Stored as a `'model,id'` string in a `varchar` column. It is a pseudo-relational
field: there is no foreign key in the database, so it is flexible but poor for
indexes and joins. Prefer `Many2oneReference` when you already store the model
name in a separate `Char` field.

---

## Relational Fields

All relational fields share `domain` (candidate filtering on the client),
`context` (client-side context), and `check_company` (mark the field for
verification in `_check_company`). `check_company` also derives a default
company domain when none is given. To actually enforce the check, set
`_check_company_auto = True` on the model.

### Many2one

```python
partner_id = fields.Many2one(
    comodel_name='res.partner',
    string='Customer',
    required=True,
    ondelete='restrict',
    domain="[('customer_rank', '>', 0)]",
    context={'default_customer_rank': 1},
    index=True,
)

# Default callable -> current user's partner
partner_id = fields.Many2one(
    'res.partner',
    default=lambda self: self.env.user.partner_id,
)

# Delegation inheritance (_inherits pattern)
partner_id = fields.Many2one(
    'res.partner',
    required=True,
    ondelete='cascade',
    delegate=True,
)
```

`ondelete` values:

| Value | Behaviour on referenced-record deletion |
|-------|----------------------------------------|
| `'set null'` | Set the FK column to `NULL` |
| `'cascade'` | Delete this record as well |
| `'restrict'` | Refuse to delete the referenced record |

When `ondelete` is omitted the ORM picks a sensible default: for a required
field it is `'restrict'`; otherwise `'set null'`. A `Many2one` from a
`TransientModel` to a regular model defaults to `'cascade'` when required.
`ondelete='set null'` combined with `required=True` is a programming error.

`delegate=True` (or a field name that appears in `_inherits`) makes the target
model's fields accessible on the current model. `auto_join` (default `False`)
generates SQL JOINs when searching through the field.

### Many2oneReference

```python
res_model = fields.Char(string='Resource Model', index=True)
res_id = fields.Many2oneReference(
    string='Resource ID',
    model_field='res_model',
    index=True,
)
```

A pseudo-relational field (no FK in the database). It stores an integer id and
reads the target model name from a separate `Char` field named by
`model_field`. This is faster than `Reference` for lookups and joins, and is
the pattern used by `ir.attachment` and `mail.followers`.

### One2many

```python
line_ids = fields.One2many(
    comodel_name='sale.order.line',
    inverse_name='order_id',
    string='Order Lines',
    domain=[('state', '!=', 'cancel')],
)
```

`One2many` is virtual: no column lives on the parent table. The value is the
recordset of child records whose `inverse_name` `Many2one` points back at the
parent. `comodel_name` and `inverse_name` are mandatory (except for `related`
or extended fields). The child model must declare the reverse `Many2one` with
the name passed as `inverse_name`.

`copy=False` by default: duplicating the parent does **not** duplicate the
child records unless you set `copy=True`. `limit` optionally caps the number of
lines read. `auto_join` (default `False`) generates SQL JOINs on search.

### Many2many

```python
tag_ids = fields.Many2many(
    comodel_name='res.partner.category',
    relation='sale_order_tag_rel',
    column1='order_id',
    column2='tag_id',
    string='Tags',
)

category_ids = fields.Many2many('res.partner.category', string='Categories')
```

Stored in a dedicated relation table with two FK columns. `relation`,
`column1`, and `column2` are optional: when omitted, names are auto-generated
from the model and comodel table names (`<table1>_<table2>_rel`, `<table>_id`).
Pass them explicitly whenever two `Many2many` fields on the same model target
the same comodel, otherwise the implicit relation names collide and the ORM
raises a `TypeError`.

`ondelete` on a `Many2many` may be `'cascade'` (default) or `'restrict'`, and
governs the comodel-side foreign key on `column2`. `limit` optionally caps the
number of related records read. `copy=True` by default, so copying a record
copies the relation.

### x2many write commands

`One2many` and `Many2many` fields accept tuples when written. There is **no**
`Command` class in 13.0 (that is 14.0+):

```python
record.write({'line_ids': [
    (0, 0, {'name': 'New line'}),    # create
    (1, line_id, {'qty': 2}),        # update existing
    (2, line_id, 0),                 # delete from DB
    (3, line_id, 0),                 # unlink (clear FK)
    (4, line_id, 0),                 # link existing
    (5, 0, 0),                       # clear all
    (6, 0, [id1, id2]),              # replace with the given ids
]})
```

---

## The Id Field

`id` is an automatic pseudo-field on every model. It is backed by the `Id`
field class (`fields.py`), is stored, read-only, and cannot be assigned:

```python
record.id          # int id, or False for an empty recordset
record.ids         # list of ids
```

`Id.type` is `'integer'` and its column is `int4`. The `__set__` descriptor
raises `TypeError("field 'id' cannot be assigned")`, so never treat `id` like a
writable field.

---

## Fields That Do Not Exist in 13.0

### Json

**Not available in 13.0.** There is no `fields.Json` and no `jsonb` field type.
If you need unstructured data, store serialized text in a `Text`/`Char` field,
or model the data with real fields.

### Properties

**Not available in 13.0.** The `Properties` / `PropertiesDefinition` pair was
introduced much later (17.0). Ad-hoc user-defined sub-fields are not supported
by the 13.0 ORM.

---

## Computed Fields

### Non-stored Compute

```python
subtotal = fields.Monetary(compute='_compute_subtotal', currency_field='currency_id')

@api.depends('price_unit', 'quantity')
def _compute_subtotal(self):
    for line in self:
        line.subtotal = line.price_unit * line.quantity
```

A compute method **must** assign a value to every record in `self`, including
when `self` is empty; otherwise the field later raises `CacheMiss`. Decorate the
method with `@api.depends` so the ORM knows when to invalidate the value.

### Stored Compute

```python
amount_total = fields.Monetary(
    compute='_compute_amount_total',
    store=True,
    currency_field='currency_id',
)

@api.depends('line_ids.price_subtotal')
def _compute_amount_total(self):
    for order in self:
        order.amount_total = sum(order.line_ids.mapped('price_subtotal'))
```

`store=True` persists the value so it becomes searchable and groupable, and
avoids recomputation on every read. By default, computed fields are not stored,
not copied, read-only unless an `inverse` is given, and recomputed as superuser
only when stored (`compute_sudo` defaults to `store`).

### Inverse (write back)

```python
full_name = fields.Char(compute='_compute_full_name', inverse='_inverse_full_name', store=True)

@api.depends('first_name', 'last_name')
def _compute_full_name(self):
    for rec in self:
        rec.full_name = "{} {}".format(rec.first_name or '', rec.last_name or '').strip()

def _inverse_full_name(self):
    for rec in self:
        parts = (rec.full_name or '').split(' ', 1)
        rec.first_name = parts[0]
        rec.last_name = parts[1] if len(parts) > 1 else ''
```

`inverse` gives a computed field write support. The inverse method must **not**
be decorated with `@api.depends`: it is triggered on write, not recompute.
Giving an `inverse` flips the field's default `readonly` from `True` to `False`.

### search (for non-stored computes)

```python
display_ref = fields.Char(compute='_compute_display_ref', search='_search_display_ref')

def _search_display_ref(self, operator, value):
    return ['|', ('name', operator, value), ('ref', operator, value)]
```

`search=` maps domain clauses on a non-stored computed field onto real fields.
The search method returns a domain (a list) and is called as
`search(records, operator, value)`.

### compute_sudo

```python
amount = fields.Float(compute='_compute_amount', store=True, compute_sudo=True)
```

When `compute_sudo=True` the compute method runs as superuser so it can read
fields the current user cannot. Defaults: `True` for stored computes, `False`
for non-stored.

### Recursive compute dependencies

The ORM detects self-referential dependency graphs automatically during field
setup (`resolve_depends` sets `recursive = True` when a field depends on itself
through a relation). There is **no** `recursive=` field parameter in 13.0:

```python
total = fields.Float(compute='_compute_total', store=True)

@api.depends('child_ids.total', 'own_amount')
def _compute_total(self):
    for rec in self:
        rec.total = rec.own_amount + sum(rec.child_ids.mapped('total'))
```

There is likewise **no** `precompute` parameter in 13.0 (that arrived in 14.0);
stored computed fields are flushed with an extra `UPDATE` after `INSERT`.

---

## Related Fields

### Basic Related

```python
partner_name = fields.Char(related='partner_id.name', readonly=True)

partner_country_id = fields.Many2one(
    related='partner_id.country_id',
    store=True,
    index=True,
)
```

Related fields are implemented as computes internally. They are read-only by
default; add `readonly=False` to forward writes to the target field. They are
not stored and not copied by default; set `store=True` to make them searchable
and indexable.

### Multi-level Related

```python
country_code = fields.Char(
    related='partner_id.country_id.code',
    string='Country Code',
    readonly=True,
)
```

### Company-dependent Related

```python
company_id = fields.Many2one('res.company', default=lambda self: self.env.company)
company_currency_id = fields.Many2one(related='company_id.currency_id', readonly=True)
```

---

## Field Parameters

### Common Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `string` | str | capitalized field name | Label shown in the UI |
| `help` | str | `None` | Tooltip |
| `required` | bool | `False` | Validation: value must be set |
| `readonly` | bool | `False` (`True` for computed/related) | UI only; code writes still work on stored fields |
| `index` | bool | `False` | Create a B-tree database index (no index types in 13.0) |
| `default` | value or callable | `None` | Static value or `lambda self: ...` |
| `copy` | bool | `True` scalars; `False` one2many/computed/related | Included by `copy()` |
| `store` | bool | `True` scalars; `False` computed/related/company_dependent | Persist to DB |
| `groups` | str | `None` | CSV of group xml ids that may read/write this field |
| `states` | dict | `None` | Map state values to `readonly`/`required`/`invisible` |
| `company_dependent` | bool | `False` | Per-company value stored in `ir.property` |
| `compute` | str | `None` | Name of the compute method |
| `inverse` | str | `None` | Name of the inverse method |
| `search` | str | `None` | Name of the search method for non-stored computes |
| `compute_sudo` | bool | `True` stored computes, else `False` | Recompute as superuser |
| `related` | str | `None` | Dotted path replacing the field's storage |
| `translate` | bool or callable | `False` | Per-language storage (string fields only) |
| `group_operator` | str | type-dependent | Aggregator used by `read_group()` |
| `group_expand` | str | `None` | Method expanding empty `read_group()` groups |
| `tracking` | bool or int | not set | With `mail.thread`: log field changes in the chatter |
| `change_default` | bool | `False` | Trigger a "user-onchange" on the field |
| `deprecated` | str | `None` | Deprecation message, if any |

`oldname` is a deprecated field attribute in 13.0
(`DEPRECATED_ATTRS` in `fields.py:36`): it logs a "use an upgrade script
instead" message. Do not use it; write a migration script instead. The
parameters `select` and `digits_compute` were renamed to `index` and `digits`
(`RENAMED_ATTRS` in `fields.py:35`) and log a warning when used.

### group_operator

`group_operator=` controls how `read_group()` aggregates a field in list and
pivot views. Valid values come from `VALID_AGGREGATE_FUNCTIONS`
(`odoo/models.py:247`):

`array_agg`, `count`, `count_distinct`, `bool_and`, `bool_or`, `max`, `min`,
`avg`, `sum`.

```python
amount = fields.Float(string='Amount', group_operator='sum')   # default for numeric

# Disable aggregation (useful for id-like numeric fields)
some_id = fields.Integer(string='ID', group_operator=None)

# Boolean aggregation
is_paid = fields.Boolean(group_operator='bool_and')
```

`group_operator` defaults to `'sum'` on `Integer`, `Float`, and `Monetary`, and
to `None` on other fields.

### tracking (requires mail.thread)

```python
class SaleOrder(models.Model):
    _name = 'sale.order'
    _inherit = ['mail.thread']

    state = fields.Selection([...], tracking=True)
    amount_total = fields.Monetary(tracking=20)     # priority, lower = shown first
```

`tracking` is contributed by the `mail` addon and only takes effect on models
that inherit `mail.thread`. `tracking=True` (equivalent to priority `100`) or an
integer priority (lower shown first) logs field changes in the chatter. In 13.0
`track_visibility` is still read by `mail.thread` but is deprecated: prefer
`tracking`.

### company_dependent

```python
property_payment_term_id = fields.Many2one(
    'account.payment.term',
    company_dependent=True,
    string='Default Payment Terms',
)
```

Company-dependent fields are not stored on the model's own table; their values
are kept as `ir.property` records keyed by company. The ORM wires them up
internally: `store=False`, `compute_sudo=False`, `copy=False`, with a default
`compute`/`inverse`/`search` and a `force_company` context dependency. Writing
to the field updates the property for the current company; reading resolves to
the current company's value.

### Field-specific Parameters

| Field | Parameters |
|-------|-----------|
| `Char` | `size`, `trim` |
| `Float` | `digits` (tuple or precision name string) |
| `Monetary` | `currency_field` |
| `Html` | `sanitize`, `sanitize_tags`, `sanitize_attributes`, `sanitize_style`, `strip_style`, `strip_classes`, `sanitize_form` |
| `Binary` | `attachment` |
| `Image` | `max_width`, `max_height`, `verify_resolution` |
| `Selection` | `selection`, `selection_add` |
| `Reference` | `selection` |
| `Many2one` | `comodel_name`, `ondelete`, `auto_join`, `delegate`, `domain`, `context`, `check_company` |
| `Many2oneReference` | `model_field` |
| `One2many` | `comodel_name`, `inverse_name`, `domain`, `context`, `auto_join`, `limit` |
| `Many2many` | `comodel_name`, `relation`, `column1`, `column2`, `domain`, `context`, `ondelete`, `limit`, `check_company` |

---

## Indexes

In 13.0 the `index` field parameter is a **boolean**. It is the only way to
declare an index in model code; there are no string index types such as
`'trigram'`, `'btree_not_null'`, or `'btree'` (those were added in later
versions), and there is no `models.Index(...)` helper.

```python
code = fields.Char(index=True)
category_id = fields.Many2one(index=True)
parent_path = fields.Char(index=True)   # required by _parent_store
```

`index=True` creates a standard B-tree index on the column. It has no effect on
non-stored computed or virtual fields. For composite or partial indexes that the
ORM cannot express, add them in SQL within a post-init hook or a migration
script.

---

## Defaults

```python
# Static value
active = fields.Boolean(default=True)
priority = fields.Integer(default=0)

# Callable (evaluated per record on create)
date = fields.Date(default=fields.Date.context_today)
datetime = fields.Datetime(default=fields.Datetime.now)

# Lambda reading env
user_id = fields.Many2one('res.users', default=lambda self: self.env.user)
company_id = fields.Many2one('res.company', default=lambda self: self.env.company)

# Method referenced by name (use @api.model on the method)
ref = fields.Char(default='_default_ref')

@api.model
def _default_ref(self):
    return self.env['ir.sequence'].next_by_code('my.model') or '/'
```

The callable signature is `default(self)` where `self` is an empty recordset of
the model. It must **not** write to the database: `create()` is not reentrant
from within a default. Defaults supplied in an action's context (for example
`default_partner_id`) override the field's own `default=`.

---

## Field Type Selection Guide

| Requirement | Use |
|-------------|-----|
| Short text (name, code) | `Char` |
| Long plain text | `Text` |
| Rich text (HTML) | `Html` |
| Yes / No | `Boolean` |
| Whole number | `Integer` |
| Decimal (non-currency) | `Float` with `digits='Named Precision'` |
| Money | `Monetary` (+ `currency_field`) |
| Date only | `Date` |
| Date + time | `Datetime` |
| File attachment | `Binary(attachment=True)` |
| Image (with resize) | `Image` |
| Fixed list of choices | `Selection` |
| Dynamic reference (arbitrary model) | `Reference` |
| Dynamic reference (model stored separately) | `Many2oneReference` |
| Link to one record | `Many2one` |
| Link to many records (inverse side) | `One2many` (with `inverse_name`) |
| Many-to-many relation | `Many2many` |
| Record primary key | `id` (the `Id` field) |
| Derived from other fields | `compute=...` |
| Projection of a related record's field | `related='path.to.field'` |
| Per-company value | `company_dependent=True` |
| Unstructured JSON / ad-hoc sub-fields | Not available in 13.0 |

---

## Base Code Reference

Verify every parameter and type described here against:

- `odoo/fields.py` - all field classes
  (`Field`, `Boolean`, `Integer`, `Float`, `Monetary`, `_String`, `Char`,
  `Text`, `Html`, `Date`, `Datetime`, `Binary`, `Image`, `Selection`,
  `Reference`, `_Relational`, `Many2one`, `Many2oneReference`,
  `_RelationalMulti`, `One2many`, `Many2many`, `Id`); `RENAMED_ATTRS` and
  `DEPRECATED_ATTRS` (lines 35-36); `_slots` and `__init__` for each class.
- `odoo/models.py` - `VALID_AGGREGATE_FUNCTIONS`
  (line 247), `read_group(domain, fields, groupby, offset=0, limit=None,
  orderby=False, lazy=True)` (line 2159), `_check_company` (line 3187),
  `_check_company_auto` (line 347).
- `odoo/tools/float_utils.py` - `float_round`,
  `float_is_zero`, `float_compare`, `float_repr`.
- `odoo/tools/date_utils.py` - `start_of`,
  `end_of`, `add`, `subtract` used by `fields.Date` / `fields.Datetime`.
- `odoo/tools/sql.py` - `pg_varchar`.
- `odoo/addons/base/models/decimal_precision.py` -
  `precision_get` (line 30) used by `Float.get_digits`.
- `addons/mail/models/mail_thread.py` -
  `tracking` and `track_visibility` handling (line 549).
