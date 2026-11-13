---
name: odoo-13-decorator
description: Complete reference for Odoo 13 API decorators (@api.model, @api.model_create_multi, @api.model_create_single, @api.depends, @api.depends_context, @api.constrains, @api.onchange, @api.returns) and their proper usage patterns.
globs: "**/models/**/*.py"
topics:
  - api.model (model-level methods)
  - api.model_create_multi (batch create)
  - api.model_create_single (legacy single create)
  - api.depends (computed field dependencies)
  - api.depends_context (context-dependent computes)
  - api.constrains (data validation)
  - api.onchange (form UI updates)
  - api.returns (return type specification)
  - Absent decorators (multi, one, cr, model_cr)
  - Decorator combinations and decision tree
when_to_use:
  - Writing computed fields
  - Implementing data validation
  - Creating onchange handlers
  - Defining model-level methods
  - Batching create() overrides
  - Rewriting legacy @api.multi / @api.one code for 13.0
---

# Odoo 13 Decorator Guide

Complete reference for the Odoo 13 `@api` decorators, how they behave, and how to combine them.

This guide covers only the decorators that exist in `odoo/api.py` of the Odoo 13.0 source tree. Several decorators that appear in newer Odoo releases do **not** exist in 13.0 and are documented in the "Absent decorators" section so you can recognize and reject them during migration.

## Table of Contents

1. [@api.model](#apimodel)
2. [@api.model_create_multi](#apimodel_create_multi)
3. [@api.model_create_single](#apimodel_create_single)
4. [@api.depends](#apidepends)
5. [@api.depends_context](#apidepends_context)
6. [@api.constrains](#apiconstrains)
7. [@api.onchange](#apionchange)
8. [@api.returns](#apireturns)
9. [Absent decorators](#absent-decorators)
10. [Create, write, and unlink overrides](#create-write-unlink-overrides)
11. [Combinations](#combinations)
12. [Decision Tree](#decision-tree)

---

## @api.model

Marks a method where `self` is a recordset but its contents are irrelevant -- only the model (class) matters. Use it for factories, class-level helpers, or methods callable as `self.env['my.model'].method(...)`.

```python
from odoo import api, fields, models

class SaleOrder(models.Model):
    _name = 'sale.order'

    @api.model
    def default_get(self, fields_list):
        vals = super().default_get(fields_list)
        vals['team_id'] = self.env.user.team_id.id
        return vals

    @api.model
    def _default_pricelist(self):
        return self.env['product.pricelist'].search([], limit=1)

    pricelist_id = fields.Many2one(
        'product.pricelist',
        default=_default_pricelist,
    )
```

Notes:

- Over RPC, `@api.model` methods receive `self` as an empty recordset of the model.
- In 13.0 the decorator is defined at `odoo/api.py:296`. It sets `method._api = 'model'`, which routes the call through `_call_kw_model` in the `call_kw` dispatcher (`odoo/api.py:355`).
- If the decorated method is literally named `create`, `@api.model` silently falls back to `@api.model_create_single` (see `odoo/api.py:301-304`). Do **not** rely on that fallback: when overriding `create()` you should explicitly use `@api.model_create_multi`.

---

## @api.model_create_multi

The modern decorator for overriding `create()`. It makes the method receive a **list** of dicts, even when the caller passes a single dict.

```python
from odoo import api, fields, models

class SaleOrder(models.Model):
    _name = 'sale.order'

    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            vals.setdefault('state', 'draft')
            if not vals.get('name'):
                vals['name'] = self.env['ir.sequence'].next_by_code('sale.order')
        orders = super().create(vals_list)
        orders._post_create_hook()
        return orders
```

The decorator adapts the callsite: callers may still pass a single `dict`, which the wrapper silently converts to `[dict]` before invoking the method (`odoo/api.py:335-340`, `_model_create_multi`). That means `create()` always receives a **list** inside the method body.

`@api.model_create_multi` **is available and used by core in 13.0**. The base `models.py` overrides `create()` with it (`odoo/models.py:3726`):

```python
@api.model_create_multi
@api.returns('self', lambda value: value.id)
def create(self, vals_list):
    ...
```

The `@api.returns('self', lambda value: value.id)` above is how base `create()` downgrades the returned recordset to an id for the legacy/traditional call style.

Unlike Odoo 14+, 13.0 does **not** log the "not overriding the create method in batch" warning, and there is no slow per-record fallback wrapper triggered by decorating `create()` with `@api.model`. You still get single-record semantics via `@api.model_create_single` (next section), but the canonical 13.0 pattern is `@api.model_create_multi`.

---

## @api.model_create_single

The legacy decorator for `create()` overrides. It makes the method receive a **single** dict; a caller may still pass a list of dicts, which the wrapper loops over one record at a time.

```python
from odoo import api, models

class SaleOrder(models.Model):
    _name = 'sale.order'

    @api.model_create_single
    def create(self, vals):
        vals.setdefault('state', 'draft')
        return super().create(vals)
```

Implementation detail (`odoo/api.py:314-331`):

```python
def _model_create_single(create, self, arg):
    # 'create' expects a dict and returns a record
    if isinstance(arg, Mapping):
        return create(self, arg)
    if len(arg) > 1:
        _create_logger.debug("%s.create() called with %d dicts", self, len(arg))
    return self.browse().concat(*(create(self, vals) for vals in arg))
```

Both `model_create_single` and `model_create_multi` set `wrapper._api = 'model_create'`, so both route through `_call_kw_model_create` in the dispatcher (`odoo/api.py:363`). The difference is only the shape of the value handed to your method body:

| Decorator | Your method receives | Caller may pass |
|-----------|----------------------|-----------------|
| `@api.model_create_multi` | `list` of dicts | single dict OR list |
| `@api.model_create_single` | single `dict` | single dict OR list (looped) |

Prefer `@api.model_create_multi` for new code; `@api.model_create_single` exists for backward compatibility and for the `@api.model` fallback on a method named `create`.

---

## @api.depends

Declares the fields whose changes should trigger recomputation of a computed field.

```python
class SaleOrder(models.Model):
    _name = 'sale.order'

    amount_untaxed = fields.Monetary()
    tax_total      = fields.Monetary()
    amount_total   = fields.Monetary(compute='_compute_amount_total', store=True)

    @api.depends('amount_untaxed', 'tax_total')
    def _compute_amount_total(self):
        for order in self:
            order.amount_total = order.amount_untaxed + order.tax_total
```

### Dotted paths (relational dependencies)

`@api.depends` accepts dot-separated field paths.

```python
@api.depends('partner_id.name', 'partner_id.email')
def _compute_partner_display(self):
    for order in self:
        order.partner_display = "%s <%s>" % (
            order.partner_id.name, order.partner_id.email,
        )
```

### One2many traversal

```python
@api.depends('order_line.price_subtotal')
def _compute_amount_untaxed(self):
    for order in self:
        order.amount_untaxed = sum(order.order_line.mapped('price_subtotal'))
```

### Dynamic dependencies

Pass a single callable; its dependencies are computed by calling it with the field's model (`odoo/api.py:199-202`).

```python
@api.depends(lambda self: (self._rec_name,) if self._rec_name else ())
def _compute_display_name(self):
    ...
```

### Rules and caveats

1. **Cannot depend on `id`.** The decorator raises `NotImplementedError("Compute method cannot depend on field 'id'.")` when any dependency path contains the `id` segment (`odoo/api.py:201-202`).
2. **List every field you actually read** -- missing dependencies cause stale cached values.
3. **Dotted paths are OK** in `@api.depends` (but **not** in `@api.constrains` or `@api.onchange`).
4. **Assign to every record in `self`** even when you only care about a subset, so the ORM does not re-call the method for the missed ones.
5. **Cycles** -- add `recursive=True` on the field when the dependency path loops back on itself (e.g. a `parent_id` tree). The `recursive` flag exists on the field in 13.0 (`odoo/fields.py:236`, set at `odoo/fields.py:703`).

### Stored vs non-stored computes

A computed field with no `store` is computed on demand and never written to the database. It cannot be used as a database sort, and it does not survive outside the current environment.

```python
# non-stored: recomputed every time it is read
amount_total = fields.Monetary(compute='_compute_amount_total')

# stored: computed then persisted; only recomputed when a dependency changes
amount_total = fields.Monetary(compute='_compute_amount_total', store=True)
```

Use `store=True` when the field is filtered, grouped, or searched on frequently, or when its value must be consistent in reports and SQL. In 13.0 a stored computed field is recomputed in superuser mode by default (`compute_sudo` defaults to `store`; see `odoo/fields.py:361`). Set `compute_sudo=False` on the field when the compute must run under the current user's access rights.

### Compute, inverse, and search methods

A computed field can name three cooperating methods via the field's `compute`, `inverse`, and `search` attributes (all three are field slots in `odoo/fields.py:237-240`):

```python
full_name = fields.Char(
    compute='_compute_full_name',
    inverse='_inverse_full_name',
    search='_search_full_name',
    store=True,
)

@api.depends('first_name', 'last_name')
def _compute_full_name(self):
    for r in self:
        r.full_name = ('%s %s' % (r.first_name or '', r.last_name or '')).strip()

def _inverse_full_name(self):
    for r in self:
        parts = (r.full_name or '').split(' ', 1)
        r.first_name = parts[0]
        r.last_name = parts[1] if len(parts) > 1 else ''

def _search_full_name(self, operator, value):
    return ['|', ('first_name', operator, value), ('last_name', operator, value)]
```

- `compute` runs when the field is read and not cached (or when a dependency changed and the field is stored).
- `inverse` is called when the field is written. Only computed fields with an `inverse` are writable: without it, the field defaults to `readonly=True` (`odoo/fields.py:363`).
- `search` lets the field be searched on even when it is not stored. It receives `(operator, value)` and must return a domain. A field with a `search` method is searchable without being stored.
- The `inverse` and `search` methods take the recordset `self` and need no `@api` decorator; the field setup resolves them by name (`odoo/fields.py:1154-1165`).

---

## @api.depends_context

Tells the ORM that a **non-stored** computed field's value depends on context keys. When one of those keys changes, Odoo invalidates the cached value and re-computes.

```python
class ProductProduct(models.Model):
    _inherit = 'product.product'

    price = fields.Float(compute='_compute_product_price')

    @api.depends_context('pricelist')
    def _compute_product_price(self):
        pricelist_id = self.env.context.get('pricelist')
        pricelist = self.env['product.pricelist'].browse(pricelist_id) if pricelist_id else self.env['product.pricelist'].get_default_pricelist()
        for product in self:
            product.price = pricelist.get_products_price(product).get(product.id, 0.0)
```

### Special context keys

In 13.0 the decorator's docstring lists exactly three keys with special support (`odoo/api.py:209-230`):

| Key | Meaning |
|-----|---------|
| `'force_company'` | Value in context, falling back to the current `env.company.id` |
| `'uid'` | `(env.uid, env.su)` tuple -- recomputed when the user or sudo flag changes |
| `'active_test'` | Value in `env.context`, falling back to the value in the field's `context` |

These are handled in `Field.cache_key` (`odoo/fields.py:654-680`). Other keys are used as-is and must be **hashable**: a `list` value is automatically converted to a `tuple`, and any other unhashable value raises `TypeError`.

Note that `company`, `lang`, and `bin_size` are **not** special-cased in the `@api.depends_context` docstring in 13.0. `bin_size` is wired directly by the `Binary` field via its `depends_context` slot (`odoo/fields.py:1904`), not by the decorator.

### Combining with @api.depends

```python
price = fields.Float(compute='_compute_price')

@api.depends('list_price')
@api.depends_context('pricelist')
def _compute_price(self):
    ...
```

Both decorators stack: the compute is re-run when a listed field changes or a listed context key changes. The `_depends_context` attribute set by the decorator is merged into the field's `depends_context` during setup (`odoo/fields.py:459-466`).

---

## @api.constrains

Validates data integrity on the listed fields. Raise `ValidationError` on failure.

```python
from odoo import api, fields, models
from odoo.exceptions import ValidationError
from odoo.tools.translate import _

class SaleOrder(models.Model):
    _name = 'sale.order'

    @api.constrains('date_order', 'validity_date')
    def _check_dates(self):
        for order in self:
            if order.validity_date and order.validity_date < order.date_order.date():
                raise ValidationError(_("Validity date cannot precede the order date."))

    @api.constrains('order_line')
    def _check_has_lines(self):
        for order in self:
            if not order.order_line:
                raise ValidationError(_("A confirmed order must have at least one line."))
```

### Rules (from `odoo/api.py:101-133`)

1. **Only simple field names.** Dotted paths (`'partner_id.customer'`) are silently ignored.
2. **Triggered only when listed fields are present in `create()` / `write()`.** The constraint method runs when at least one listed field name is in the written field set (`odoo/models.py:1207-1214`, `_validate_fields`). A field that is not in the view or not in the vals dict does not trigger the check.
3. **Raise `ValidationError`** (from `odoo.exceptions`, defined at `odoo/exceptions.py:116`). The error message is shown to the user verbatim, so write it in user-facing language. Using `UserError` for a constraint is possible but semantically wrong -- constraints are data-integrity checks, and `ValidationError` is what the ORM and UI expect.
4. A constraint can list several fields; it fires whenever **any** of them is in the write. It also warns at class setup if a listed name is not a field, or is not writeable (`odoo/models.py:672-683`).

### Ensuring a constraint always runs

For an absolute invariant (for example, "a confirmed order must have at least one line") that must hold even when the user does not touch `order_line`, override `create()` / `write()` and call the check explicitly:

```python
@api.model_create_multi
def create(self, vals_list):
    records = super().create(vals_list)
    records._check_has_lines()
    return records

def write(self, vals):
    result = super().write(vals)
    self._check_has_lines()
    return result
```

### Error message style

The text you pass to `ValidationError` is the full message the user sees. Do not rely on the ORM to prefix the field name for you. Keep it specific about what was wrong and what is expected:

```python
raise ValidationError(_(
    "The validity date (%s) cannot precede the order date (%s)."
    % (order.validity_date, order.date_order)
))
```

---

## @api.onchange

Runs on the **form view** when the user edits one of the listed fields. The method is invoked on a **pseudo-record**: a single record holding the unsaved form values. Field assignments on it are sent back to the client.

```python
class SaleOrderLine(models.Model):
    _name = 'sale.order.line'

    @api.onchange('product_id')
    def _onchange_product_id(self):
        if self.product_id:
            self.price_unit = self.product_id.list_price
            self.name       = self.product_id.display_name
            self.product_uom_id = self.product_id.uom_id
        else:
            self.price_unit = 0.0
            self.name = ''
```

### Warnings and notifications

Return a dict with a `warning` key (`odoo/api.py:143-147`). If `type` is `notification` the warning is a notification; otherwise it is a dialog (the default).

```python
@api.onchange('discount')
def _onchange_discount(self):
    if self.discount and self.discount > 50:
        return {
            'warning': {
                'title': _("High Discount"),
                'message': _("Discount above 50%% requires manager approval."),
                'type': 'notification',
            }
        }
```

### Dynamic domains

Onchange can return a `domain` dict (field name -> domain) to filter relational dropdowns:

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    partner_id = self.partner_id.id
    return {
        'domain': {
            'child_ids': [('partner_id', '=', partner_id)],
        },
    }
```

### Rules (from `odoo/api.py:134-183`)

1. **Only simple field names.** Dotted paths (`'partner_id.tz'`) are ignored.
2. **No CRUD on the pseudo-record.** Calling `create()`, `read()`, `write()`, or `unlink()` from inside an onchange is undefined behaviour -- the record may not exist in the database yet. Set fields directly (`self.field = value`) or use `self.update({...})`.
3. **A one2many / many2many field cannot modify itself via onchange** (webclient limitation, issue #2693).
4. Assign values directly or via `self.update({'field': value})`.

### Onchange vs compute

Onchange and compute solve different problems and do not replace each other:

| Aspect | `@api.onchange` | `@api.depends` compute |
|--------|-----------------|------------------------|
| Where it runs | Form view, client-side trigger | Server-side, any environment |
| When it runs | User edits the listed field | Field read, or dependency changed (stored) |
| `self` | A single pseudo-record with unsaved values | A real recordset |
| Effect | Assignments returned to the client only | Value cached (and stored if `store=True`) |
| Persisted? | No -- lost if the user cancels | Yes when stored |

Onchange does **not** run on server-side `create()` or `write()`. Any invariant that must hold everywhere must also be expressed as a `@api.depends` compute or a `@api.constrains` check.

---

## @api.returns

Declares the model of the recordset returned by a method. It adapts the method's result between the record style and the traditional (id-based) style.

```python
class Partner(models.Model):
    _name = 'res.partner'

    @api.returns('self')
    def copy(self, default=None):
        return super().copy(default)

    @api.returns('res.partner')
    def get_main_contact(self):
        return self.mapped('contact_ids')[:1]
```

Signature (`odoo/api.py:235-267`):

```python
def returns(model, downgrade=None, upgrade=None):
```

- `model` is a model name, or the string `'self'` for the current model.
- `downgrade` is a function converting the record-style value to a traditional-style output.
- `upgrade` is a function converting the traditional-style value to a record-style output.

When called from the record-style API, the result is a recordset. When called over the traditional/RPC API, Odoo downgrades the recordset to `id`, `ids`, or `False` (`odoo/api.py:270-281`).

The `@api.returns` declaration is **inherited automatically**: a method overriding a decorated method keeps the same `@api.returns` (`odoo/api.py:64` and `propagate` at `odoo/api.py:85-93`). This is why base `create()` still downgrades correctly in subclasses.

---

## Absent decorators

These decorators exist in other Odoo releases but are **not defined in 13.0's `odoo/api.py`**. Do not use them in 13.0 code; treat them as migration errors.

### @api.multi, @api.one, @api.cr, @api.model_cr

None of these legacy decorators are defined in `odoo/api.py`. In 13.0 every model method is a **record-style** method: `self` is a recordset and the cursor is reachable only through `self.env.cr`.

```python
# 13.0: correct record-style method
def action_confirm(self):
    for order in self:
        order.state = 'sale'
    return True
```

```python
# NOT valid in 13.0 -- these decorators do not exist
@api.multi
def action_confirm(self): ...      # ImportError

@api.one
def action_confirm(self): ...      # ImportError

@api.cr
def do_it(self, cr, uid, ...): ... # ImportError
```

If you are migrating old-API (pre-9.0) code, the migration is:

| Old style | 13.0 replacement |
|-----------|------------------|
| `@api.multi` | No decorator at all -- `self` is already a recordset |
| `@api.one` | No decorator -- loop over `self` yourself, or call `self.ensure_one()` |
| `@api.cr` / `@api.model_cr` | No decorator -- use `self.env.cr`, `self.env.uid`, `self.env.context` |

Note that the `Meta` metaclass still detects the string `'cr'` on a `_api` attribute and logs a deprecation warning (`odoo/api.py:54-61`), but no decorator in the 13.0 public API sets that attribute. It is a legacy hook only.

### @api.ondelete, @api.private, @api.autovacuum

- `@api.ondelete` -- **not available in 13.0** (added in Odoo 15). To reject a deletion, override `unlink()` and raise `UserError` (see next section). Be aware this can break module uninstallation, so guard carefully.
- `@api.private` -- **not available in 13.0**. The leading-underscore convention (`_compute_*`, `_prepare_*`, `_action_*`) is the only way to keep a method off the RPC surface; the RPC dispatcher refuses underscore-prefixed methods.
- `@api.autovacuum` -- **not available in 13.0**. Scheduled cleanup needs its own `ir.cron` record or an `ir.actions.server` action.

---

## Create, write, and unlink overrides

In 13.0 the CRUD methods are plain record-style methods; only `create()` carries an `@api` decorator in the base class.

### create()

Override with `@api.model_create_multi` and receive a list:

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if not vals.get('name'):
            vals['name'] = self.env['ir.sequence'].next_by_code('sale.order')
    return super().create(vals_list)
```

Return the recordset from `super().create(vals_list)`. If you must also run business rules that are missing from vals (the `@api.constrains` gap), call them on the created records:

```python
@api.model_create_multi
def create(self, vals_list):
    records = super().create(vals_list)
    records._check_has_lines()
    return records
```

### write()

`write(self, vals)` receives the recordset in `self` and a single `vals` dict. No decorator.

```python
def write(self, vals):
    result = super().write(vals)
    # run after the write so computed fields have settled
    self._check_dates()
    return result
```

The base method is defined at `odoo/models.py:3449`. Return the result of `super().write(vals)` (which is `True`).

### unlink()

`unlink(self)` receives the recordset in `self`. No decorator. Use it to reject deletion or to clean up related data first:

```python
from odoo.exceptions import UserError

def unlink(self):
    for order in self:
        if order.state not in ('draft', 'cancel'):
            raise UserError(_("Only draft or cancelled orders can be deleted."))
    return super().unlink()
```

The base method is defined at `odoo/models.py:3358`.

**Uninstall caveat:** because 13.0 has no `@api.ondelete(at_uninstall=False)`, a `unlink()` guard runs during module uninstallation too. If your check raises during uninstall, the transaction rolls back and leftover data stays in the database. Keep delete guards permissive enough for uninstall, or skip them when the module is being uninstalled.

### Updating one2many / many2many via commands

13.0 has no `Command` class. Relational writes use the tuple command syntax:

```python
# (0, 0, vals)        create and link a new record
# (1, id, vals)       update an existing linked record
# (2, id, 0)          unlink and delete
# (3, id, 0)          unlink only (detach, keep the record)
# (4, id, 0)          link an existing record
# (5, 0, 0)           unlink all
# (6, 0, ids)         replace all links with the given ids
```

Example inside a `write()` override:

```python
def write(self, vals):
    vals.setdefault('order_line', [(5, 0, 0)])
    return super().write(vals)
```

---

## Combinations

### Computed + stored + searchable + writable

```python
full_name = fields.Char(
    compute='_compute_full_name',
    inverse='_inverse_full_name',
    search='_search_full_name',
    store=True,
)

@api.depends('first_name', 'last_name')
def _compute_full_name(self):
    for r in self:
        r.full_name = ('%s %s' % (r.first_name or '', r.last_name or '')).strip()

def _inverse_full_name(self):
    for r in self:
        parts = (r.full_name or '').split(' ', 1)
        r.first_name = parts[0]
        r.last_name = parts[1] if len(parts) > 1 else ''

def _search_full_name(self, operator, value):
    return ['|', ('first_name', operator, value), ('last_name', operator, value)]
```

### @api.depends + @api.depends_context

```python
price = fields.Float(compute='_compute_price')

@api.depends('list_price')
@api.depends_context('pricelist', 'uid')
def _compute_price(self):
    ...
```

Both decorators stack: the compute re-runs when a listed field or a listed context key changes.

### @api.model with @api.constrains

`@api.constrains` is normally a record-level decorator, but it can be stacked with `@api.model` when the check body does not iterate (rarely useful):

```python
@api.model
@api.constrains('code')
def _check_code_format(self):
    for rec in self:
        if rec.code and not rec.code.isalnum():
            raise ValidationError(_("Code must be alphanumeric."))
```

### Batch create + explicit constraint

```python
@api.model_create_multi
def create(self, vals_list):
    records = super().create(vals_list)
    records._check_business_rules()    # force invariants missing from vals
    return records
```

---

## Decision Tree

```
Defining / overriding a FIELD
|-- Computed from other fields? ........... @api.depends
|   |-- Also needs context? ............... + @api.depends_context
|   |-- Should be searchable? ............. store=True OR search=...
|   `-- Should be writable? ............... inverse=...
|
Defining / overriding a METHOD
|-- Overriding create()? .................. @api.model_create_multi
|   `-- Legacy single-dict shape? ......... @api.model_create_single
|-- Model-level (self content irrelevant)?  @api.model
|-- Validates integrity? .................. @api.constrains (simple field names only)
|-- Form UI reaction? ..................... @api.onchange (no CRUD, no dotted paths)
|-- Returns a recordset via RPC? .......... @api.returns('model' or 'self')
`-- Regular record-level method ........... no decorator

Deletion guard / daily cleanup / RPC blocking
|-- Reject deletion? ...................... override unlink() (no @api.ondelete in 13.0)
|-- Block RPC on a public method? ......... rename to _name (no @api.private in 13.0)
`-- Daily cleanup job? .................... own ir.cron (no @api.autovacuum in 13.0)
```

---

## Quick Reference

| Decorator | Purpose | Dotted paths? | Notes |
|-----------|---------|---------------|-------|
| `@api.model` | Method where self content is irrelevant | -- | Also fallback-handles `create(single_dict)` via `model_create_single` |
| `@api.model_create_multi` | Batch-capable `create()` override | -- | Receives a list; preferred override pattern |
| `@api.model_create_single` | Legacy single-dict `create()` override | -- | Loops over a list one record at a time |
| `@api.depends(*fields)` | Compute dependencies | Yes | Cannot depend on `'id'` |
| `@api.depends_context(*keys)` | Context dependencies | -- | Special keys: `force_company`, `uid`, `active_test`; values must be hashable |
| `@api.constrains(*fields)` | Data validation | No | Only runs when listed fields are in vals |
| `@api.onchange(*fields)` | Form UI reaction | No | No CRUD on pseudo-record |
| `@api.returns(model, ...)` | Declare return-model for RPC | -- | Inherited by overrides automatically |

Absent in 13.0: `@api.multi`, `@api.one`, `@api.cr`, `@api.model_cr`, `@api.ondelete`, `@api.private`, `@api.autovacuum`.

---

## Base Code Reference

Verified against the Odoo 13.0 source tree:

- `odoo/api.py` -- all `@api` decorators: `constrains` (L101), `onchange` (L134), `depends` (L184), `depends_context` (L209), `returns` (L235), `model` (L296), `_model_create_single` (L314), `model_create_single` (L323), `_model_create_multi` (L335), `model_create_multi` (L342), `_call_kw_model` (L355), `_call_kw_model_create` (L363), `_call_kw_multi` (L372).
- `odoo/models.py` -- `unlink()` (L3358), `write()` (L3449), `create()` with `@api.model_create_multi` + `@api.returns('self', lambda value: value.id)` (L3726), `_validate_fields` constraint trigger (L1207), `_constraint_methods` setup (L670), `_onchange_methods` setup (L690).
- `odoo/fields.py` -- field slots `compute`/`inverse`/`search`/`store`/`recursive`/`depends_context` (L230-241), `compute_sudo` default derived from `store` (L361), `cache_key` special keys `force_company`/`uid`/`active_test` (L654-680), inverse/search resolution by name (L1154-1165), `Binary.depends_context` for `bin_size` (L1904).
- `odoo/exceptions.py` -- `UserError` (L33), `AccessError` (L80), `MissingError` (L104), `ValidationError` (L116).
