---
name: odoo-13-performance
description: Complete reference for Odoo 13 performance work, covering the prefetch cache, N+1 prevention, batch operations, read_group aggregation, stored computes, indexing, flush and cache control, cron batching, and query-count profiling.
globs: "**/*.{py,xml}"
topics:
  - Prefetch mechanism and the prefetch cache in Odoo 13
  - N+1 query prevention patterns
  - Batch create, write, and unlink behavior
  - Field selection with search_read, read, and bin_size
  - Aggregation with read_group (no _read_group in 13.0)
  - Stored computed fields and dependency declarations
  - Index strategy, _order, auto_join, and SQL constraints
  - Flush and cache invalidation in 13.0
  - Cron batching and long-running jobs
  - Profiling with --dev=all, debug_sql, and odoo.tools.profiler
  - Query-count assertions in tests
when_to_use:
  - Optimizing slow Odoo 13 code
  - Preventing N+1 queries
  - Writing batch create/write/unlink operations
  - Optimizing computed fields
  - Using read_group for aggregations
  - Choosing indexes, _order, and auto_join
  - Profiling and locking performance with tests
---

# Odoo 13 Performance Guide

Complete reference for writing performant Odoo 13 code. Every API named here was
checked against the 13.0 source tree; anything introduced after 13.0 is called out
explicitly as unavailable.

## Table of Contents

1. [Prefetch Mechanism](#prefetch-mechanism)
2. [N+1 Query Prevention](#n1-query-prevention)
3. [Batch Operations](#batch-operations)
4. [Field Selection Optimization](#field-selection-optimization)
5. [Aggregation with read_group](#aggregation-with-read_group)
6. [Compute Field Optimization](#compute-field-optimization)
7. [Index Strategy, _order, and auto_join](#index-strategy-_order-and-auto_join)
8. [SQL Constraints](#sql-constraints)
9. [SQL Optimization](#sql-optimization)
10. [Flush and Cache Control](#flush-and-cache-control)
11. [Clean Code Patterns](#clean-code-performance-patterns)
12. [Cron Batching](#cron-batching)
13. [Profiling and Query Count](#profiling-and-query-count)
14. [Performance Checklist](#performance-checklist)
15. [Common Anti-Patterns](#common-anti-patterns)
16. [Base Code Reference](#base-code-reference)

---

## Prefetch Mechanism

Odoo 13 loads a field for a whole prefetch group at once. The group is the
recordset's `_prefetch_ids`, not just the records you are iterating.

```python
# Constants from odoo/models.py
PREFETCH_MAX = 1000          # maximum number of prefetched records
IN_MAX = 1000                # maximum size of an IN (...) condition
```

The relevant pieces live on `BaseModel`:

- `_prefetch_ids` - the ids that share a prefetch group.
- `with_prefetch(prefetch_ids=None)` - returns a recordset with an explicit group.
- `_in_cache_without(field, limit=PREFETCH_MAX)` - ids still missing from cache.
- `_fetch_field(field)` - reads the field for the whole group.

```python
# GOOD: automatic prefetch
orders = self.search([('state', '=', 'done')])   # 1 query for orders
for order in orders:
    print(order.name)              # 1 query for all names
    print(order.partner_id.name)   # 1 query for all partners
# Roughly 3 queries regardless of recordset size.
```

`_fetch_field` also prefetches companion fields when the context allows it, so
reading one field can populate several cached fields. That is why
`with_context(prefetch_fields=False)` is the escape hatch when you want exactly
one column.

Every recordset carries `_prefetch_ids`; accessing a field fetches it for every
id in that group that is not cached yet. When a recordset is larger than
`PREFETCH_MAX`, iteration splits it into chunks of at most `IN_MAX` ids through
`cr.split_for_in_conditions()`.

```python
# Break a recordset out of its group: each access only touches self.
big_attachments = attachments.with_prefetch()

# Force a shared group between two unrelated recordsets.
a = self.browse(ids_a).with_prefetch(all_ids)
b = self.browse(ids_b).with_prefetch(all_ids)

# Disable companion-field prefetch when reading one column.
records.with_context(prefetch_fields=False).read(['name'])
```

### sudo() and the cache

In 13.0, `sudo()` returns a recordset attached to a new environment. The
docstring is explicit: the new environment does not benefit from the current
environment's data cache, although it keeps the same prefetch object. Calling
`sudo()` inside a loop therefore re-fetches from the database every iteration;
call it once on the recordset (`for order in orders.sudo(): ...`).

---

## N+1 Query Prevention

### search inside a loop

```python
# BAD: 1 + N queries
for order in orders:
    payments = self.env['payment.transaction'].search([
        ('order_id', '=', order.id)
    ])
    order.payment_count = len(payments)

# GOOD: one grouped query
groups = self.env['payment.transaction'].read_group(
    [('order_id', 'in', orders.ids)],
    fields=['order_id'],
    groupby=['order_id'],
)
counts = {g['order_id'][0]: g['order_id_count'] for g in groups}
for order in orders:
    order.payment_count = counts.get(order.id, 0)
```

In 13.0 the count key is `<groupby>_count`, so grouping by `order_id` produces
`order_id_count`.

### One2many traversal

```python
# GOOD: mapped() plus automatic prefetch
orders = self.search([('state', '=', 'done')])
for order in orders:
    for line in order.line_ids:
        print(line.product_id.name)
# About 3 queries: orders, lines, products.

# BETTER when only a few scalar fields are needed
lines_data = orders.mapped('line_ids').read(['product_id', 'quantity'])
```

### Computed field with a related access

```python
# BAD: missing dependency, one query per record
@api.depends('partner_id')
def _compute_partner_email(self):
    for order in self:
        order.partner_email = order.partner_id.email

# GOOD: dotted dependency, one batched fetch
@api.depends('partner_id', 'partner_id.email')
def _compute_partner_email(self):
    for order in self:
        order.partner_email = order.partner_id.email
```

### Conditional computation

Accessing a field on a single record forces a fetch. Use `filtered()` instead:
it reuses prefetched data (`orders.filtered(lambda o: o.partner_id.customer_rank > 0)`).

### Existence checks

```python
# BAD: fetches every matching row
if self.search([('state', '=', 'done')]):
    ...

# GOOD: stop at the first row
if self.search([('state', '=', 'done')], limit=1):
    ...

# GOOD: ask the database for the count
count = self.search_count([('state', '=', 'done')])
```

`search_count(args)` in 13.0 takes only a domain. It calls `search(args,
count=True)`, and `_search` ignores order, limit, and offset when counting.
`exists()` runs a single `SELECT id ... WHERE id IN %s` for real ids and returns
the subset that still exists.

---

## Batch Operations

### Batch create

Odoo 13 supports `create()` with a list of dicts. The signature is
`create(self, vals_list)` and a plain dict is still accepted for backward
compatibility.

```python
# GOOD: one create call, one compute round
records = self.create([
    {'name': 'Record %s' % i, 'state': 'draft'}
    for i in range(100)
])

# BAD: 100 create calls and 100 compute rounds
for i in range(100):
    self.create({'name': 'Record %s' % i})
```

Important 13.0 detail: `_create()` inserts rows **one by one**. The source
comment explains that batch INSERT was deliberately not introduced because
records do not all specify the same columns and the gains were low. Batch create
in 13.0 saves on defaults, inverse processing, and compute rounds, but it does
not collapse the INSERT statements. Do not claim `INSERT_BATCH_SIZE` behavior
for 13.0; that constant does not exist here.

### Batch write and unlink

```python
# GOOD: one UPDATE per distinct value set
self.search([('state', '=', 'draft')]).write({'state': 'cancel'})

# BAD: one UPDATE per record
for order in self.search([('state', '=', 'draft')]):
    order.write({'state': 'cancel'})

# GOOD: one DELETE
self.search([('state', '=', 'cancel')]).unlink()
```

`_write()` builds one `UPDATE ... WHERE id IN %s` and splits the id list with
`cr.split_for_in_conditions()` when it exceeds `IN_MAX`.

### Chunked processing for large datasets

`split_every(n, iterable, piece_maker=tuple)` lives in `odoo/tools/misc.py`.
`cr.split_for_in_conditions(ids, size=None)` is the cursor-level wrapper. Use
them to process large id lists in bounded chunks, committing between chunks in
cron jobs only.

### One2many and many2many writes

Writing a large x2many through `(0, 0, vals)` commands is O(N) in queries. The
core `test_performance` module asserts this explicitly: creating 11 lines costs
15 queries, while linking existing records with `(4, id)` or `(6, 0, ids)` is
O(1). Prefer linking existing records over re-creating them.

---

## Field Selection Optimization

### search_read when dicts are enough

```python
# GOOD
data = self.search_read(
    [('state', '=', 'done')],
    ['name', 'amount_total', 'date'],
)
# [{'id': 1, 'name': ..., 'amount_total': ..., 'date': ...}, ...]

# SLOWER: search plus read on a recordset
records = self.search([('state', '=', 'done')])
data = records.read(['name', 'amount_total', 'date'])
```

`search_read(domain=None, fields=None, offset=0, limit=None, order=None)` is
defined in `odoo/models.py`. It short-circuits when `fields == ['id']`.

### read(), load, and bin_size

`read(fields, load='_classic_read')` decorates many2one values with `name_get`;
`load=None` skips that decoration. For binary fields,
`with_context(bin_size=True).read(['datas'])` returns the file size instead of
the base64 payload; the field checks `bin_size` and `bin_size_<field_name>` in
the context (`odoo/fields.py`).

### active_test and tracking_disable

```python
# Include archived records.
all_records = self.with_context(active_test=False).search([])

# Skip chatter tracking on bulk updates.
records.with_context(tracking_disable=True).write({'state': 'done'})
```

`_where_calc()` adds `('active', '=', 1)` unless the domain already mentions
`active` or the context sets `active_test=False`. `mail.thread` checks
`tracking_disable` in both `create` and `write`.

---

## Aggregation with read_group

In 13.0 the aggregation API is `read_group(domain, fields, groupby, offset=0,
limit=None, orderby=False, lazy=True)`. There is no `_read_group()` method in
13.0; the internal implementation is `_read_group_raw()`, which returns the same
list-of-dicts shape after post-processing.

```python
# GOOD: one grouped query
results = self.read_group(
    domain=[('state', '=', 'done')],
    fields=['amount_total:sum'],
    groupby=['partner_id'],
    lazy=True,
)
# [{
#     'partner_id': (42, 'ACME'),
#     'partner_id_count': 5,
#     'amount_total': 1234.0,
#     '__domain': [...],
#     '__context': {...},
# }, ...]

for row in results:
    partner_id, partner_name = row['partner_id'] or (False, '')
    print('%s: %s (%s orders)' % (
        partner_name, row['amount_total'], row['partner_id_count']))
```

### Field specification syntax

`fields` accepts three forms:

- `'field'` - use the field's default `group_operator`.
- `'field:agg'` - aggregate with a PostgreSQL function.
- `'name:agg(field)'` - aggregate `field` and return it as `name`.

```python
results = self.read_group(
    [('state', '=', 'done')],
    ['amount_total:sum', 'id:count'],
    ['partner_id'],
)
```

`count_distinct` is supported in addition to the PostgreSQL aggregate
functions. Groupby fields must be stored database columns; computed non-stored
fields raise a `UserError`.

### Multiple groupbys and lazy mode

With `lazy=True` (the default) the result is grouped by the first field and the
remaining groupbys are put in `__context`. With `lazy=False` you get one dict
per combination of all groupby fields.

Each row carries `__domain` (the domain that reproduces the group), `__context`
(the remaining groupbys when `lazy=True`), and `<groupby>_count` (the number of
records in the group). There is no `__range` key in 13.0. Date groupbys return a
`(range, label)` tuple in the groupby field itself.

### read_group versus per-record loops

```python
# BAD: one query per record
for partner in partners:
    partner.order_count = self.search_count([('partner_id', '=', partner.id)])

# GOOD: one query for all partners
groups = self.read_group(
    [('partner_id', 'in', partners.ids)],
    ['partner_id'],
    ['partner_id'],
)
counts = {g['partner_id'][0]: g['partner_id_count'] for g in groups}
for partner in partners:
    partner.order_count = counts.get(partner.id, 0)
```

Use `read_group` whenever the answer is a count, sum, average, minimum, or
maximum per group. It also flushes the fields it touches through
`_flush_search()` before running the SQL.

---

## Compute Field Optimization

### Store expensive computations

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    amount_total = fields.Monetary(
        string='Total',
        compute='_compute_amount_total',
        store=True,
        compute_sudo=True,
    )

    @api.depends('line_ids.price_subtotal')
    def _compute_amount_total(self):
        for order in self:
            order.amount_total = sum(order.line_ids.mapped('price_subtotal'))
```

Storing lets you search, group, and sort on the field. It also makes the value
available to `read_group` and to SQL reports.

### Declare every dependency, including dotted paths

```python
# BAD: partner_id changes are not seen, or the value is fetched per record.
@api.depends('partner_id')
def _compute_partner_country(self):
    for order in self:
        order.partner_country_id = order.partner_id.country_id

# GOOD
@api.depends('partner_id', 'partner_id.country_id')
def _compute_partner_country(self):
    for order in self:
        order.partner_country_id = order.partner_id.country_id
```

### Avoid recursive dependencies

If A depends on B and B depends on A, the recompute never settles. Make both
fields depend on a common base field instead: `_compute_tax` depends on
`amount`, and `_compute_total` depends on `amount` and `tax`.

### Batch the compute body

```python
# BAD: one search per record
@api.depends('order_id')
def _compute_order_total(self):
    for line in self:
        line.order_total = self.search_count([('order_id', '=', line.order_id.id)])

# GOOD: one grouped query for the whole recordset
@api.depends('order_id')
def _compute_order_total(self):
    groups = self.env['sale.order.line'].read_group(
        [('order_id', 'in', self.order_id.ids)],
        ['order_id', 'price_subtotal:sum'],
        ['order_id'],
    )
    totals = {g['order_id'][0]: g['price_subtotal'] for g in groups}
    for line in self:
        line.order_total = totals.get(line.order_id.id, 0.0)
```

### precompute is not available in 13.0

Odoo 17's `precompute=True` field attribute does not exist in 13.0. A stored
computed field is computed after the INSERT in 13.0, so a create that needs the
value immediately costs an extra write. Do not present `precompute` as valid
13.0 API.

---

## Index Strategy, _order, and auto_join

### index=True

```python
reference = fields.Char(index=True)
external_id = fields.Char(index=True)
```

In 13.0 `index` is a boolean. The `btree_not_null` value is a later addition and
is not valid here. `Field.update_db_index()` creates a plain B-tree index named
`<table>_<column>_index` when `index=True`.

Index fields that are used in frequent `search()` domains, used in `_order`,
used as foreign keys that are filtered on, or used in `read_group` groupbys. Do
not index low-cardinality booleans or fields that are written constantly; each
index adds write cost.

### _order

`_order` is a class attribute, defaulting to `'id'`:

```python
class SaleOrder(models.Model):
    _name = 'sale.order'
    _order = 'date_order desc, id desc'
```

Rules for `_order`:

- Every field in `_order` must be stored.
- A many2one in `_order` triggers a JOIN through `_generate_m2o_order_by()`.
- A non-stored many2one is ignored with a debug log.
- Sorting on an unindexed column forces a sort of the whole result set.

```python
# GOOD: index the column you order by.
date_order = fields.Datetime(index=True)

# BAD: order by a computed non-stored field.
_order = 'amount_total desc'   # amount_total must be store=True
```

For large sets, let PostgreSQL sort with an index instead of sorting in memory:
`self.search([], order='amount_total desc', limit=5)`. `sorted(key=None)` with
no key re-searches by id, which is rarely what you want on a large recordset.

### auto_join on relational fields

13.0 has no model-level `_auto_join` attribute. The supported mechanism is the
field-level `auto_join=True` flag on `Many2one` and `One2many` fields, which
makes domain evaluation generate a JOIN instead of a subquery:

```python
partner_id = fields.Many2one('res.partner', auto_join=True)
line_ids = fields.One2many('sale.order.line', 'order_id', auto_join=True)
```

Use it when a domain repeatedly traverses the same relation and the subquery
plan is slow. `auto_join` is only supported on stored many2one and one2many
fields; other field types raise `NotImplementedError` in
`odoo/osv/expression.py`. Because a JOIN can duplicate rows, `_search()`
uniquifies the resulting ids.

---

## SQL Constraints

`_sql_constraints` is a list of `(name, sql_definition, message)` tuples:

```python
class SaleOrder(models.Model):
    _name = 'sale.order'

    _sql_constraints = [
        ('name_uniq', 'unique(name)', 'Order name must be unique!'),
        ('amount_positive', 'CHECK (amount_total >= 0)',
         'Amount total must be positive!'),
    ]
```

Notes for 13.0:

- The constraint name is prefixed with the table name at creation time.
- `unique(...)` and `CHECK (...)` are the common forms.
- A `foreign key` definition is added post-init.
- Constraints are enforced by PostgreSQL, so they cost nothing at read time and
  protect data written by SQL or imports.

Prefer a SQL constraint over a Python `@api.constrains` when the rule can be
expressed in SQL: the database enforces it for every writer, including direct
SQL and concurrent transactions.

---

## SQL Optimization

Use SQL for complex aggregations with joins across many tables, bulk data
migration scripts, heavy reports, and read-only analytics that would otherwise
cost thousands of ORM queries. Avoid SQL for writes: you lose computes, cache
invalidation, tracking, and access rules.

```python
def get_statistics(self):
    self.env.cr.execute("""
        SELECT state,
               COUNT(*)          AS count,
               SUM(amount_total) AS total
          FROM sale_order
         WHERE create_date >= %s
      GROUP BY state
    """, (fields.Date.today(),))
    return self.env.cr.dictfetchall()
```

Always use `%s` placeholders and pass a tuple. Never format user input into SQL.

`odoo/tools/sql.py` in 13.0 contains schema helpers such as `create_index`,
`table_exists`, and `constraint_definition`. It does **not** contain the
composable `SQL` class from Odoo 17, and `env.execute_query()` does not exist.
Build queries as strings with parameters.

Flush before reading through SQL:

```python
self.env['sale.order'].flush(['state', 'amount_total'])
self.env.cr.execute(
    "SELECT state, SUM(amount_total) FROM sale_order GROUP BY state")
```

`flush(fnames=None, records=None)` is defined on `BaseModel`. Without arguments
it flushes every pending computation and update.

Never SQL-write without a reason: it bypasses the cache, computes, tracking,
and access rules. Use `self.browse(ids).write({'state': 'done'})` instead.

---

## Flush and Cache Control

Odoo 13 defers writes: ORM updates accumulate in the environment cache and are
flushed lazily.

```python
# Flush all pending updates for a model.
self.env['sale.order'].flush()

# Flush only a few fields.
self.env['sale.order'].flush(['state', 'amount_total'])

# Flush specific records.
self.env['sale.order'].flush(['state'], records)

# Invalidate the whole cache.
self.env.cache.invalidate()

# Invalidate selected fields for selected ids.
self.env['sale.order'].invalidate_cache(['state'], ids)

# Invalidate everything for the model.
self.env['sale.order'].invalidate_cache()
```

`invalidate_cache(fnames=None, ids=None)` is defined on `BaseModel` and
delegates to `env.cache.invalidate(spec)`.

`Environment.flush_all()` and `Environment.invalidate_all()` do not exist in
13.0. Use the per-model methods above, or `self.env.cache.invalidate()` for the
whole cache. Do not present the 17.0 helpers as valid 13.0 API.

Flush manually before `cr.execute()` on columns you just wrote, before a stored
procedure or trigger that reads those columns, and before a method that must see
a value another model just computed.

```python
orders.write({'state': 'done'})
self.env['sale.order'].flush(['state'])
self.env.cr.execute(
    "SELECT state, SUM(amount_total) FROM sale_order GROUP BY state")
```

Do not invalidate after a write: it throws away the cache the write just
populated and forces a re-fetch. Trust the cache and read the field directly.

---

## Clean Code Performance Patterns

```python
# GOOD: field access through mapped()
partner_ids = orders.mapped('partner_id.id')
countries = orders.mapped('partner_id.country_id')   # dotted paths work

# GOOD: mapped() of a many2many returns a deduplicated recordset
all_tags = orders.mapped('tag_ids')

# GOOD: filter before operating
done = orders.filtered(lambda o: o.state == 'done')
done.action_invoice_create()

# GOOD: domain evaluation against an in-memory recordset
done = orders.filtered_domain([('state', '=', 'done')])

# GOOD: sort with a key
top = orders.sorted(key='amount_total', reverse=True)[:5]

# GOOD: display_name is computed once per recordset and cached
names = orders.mapped('display_name')
```

`mapped()` with a string walks the dotted path and unions recordsets, removing
duplicates. `filtered()` accepts a callable or a dotted field path.

Avoid recomputation in loops:

```python
# BAD: each write triggers a recompute pass.
for order in orders:
    order.write({'state': 'done'})

# GOOD: one batch write, one recompute pass.
orders.write({'state': 'done'})
```

Prefer `display_name` over `name_get()` in loops: `display_name` is an
automatic computed field that is cached and prefetched, while `name_get()` is
called per record.

---

## Cron Batching

`ir.cron._process_jobs()` selects every due job in one query, then locks and
runs each one. There is no per-run batch limit in the scheduler itself, so a cron
method that processes "everything" will hold one transaction for the whole run.

```python
# odoo/addons/base/models/ir_cron.py
cr.execute("""SELECT * FROM ir_cron
              WHERE numbercall != 0
                  AND active AND nextcall <= (now() at time zone 'UTC')
              ORDER BY priority""")
```

Batch inside the cron method:

```python
def _cron_process_pending(self):
    domain = [('state', '=', 'pending')]
    while True:
        records = self.search(domain, limit=500)
        if not records:
            break
        records._process_batch()
        # Commit per chunk so locks are released and progress is durable.
        self.env.cr.commit()
```

Guidelines:

- Use `search(..., limit=N)` and loop until empty.
- Commit between chunks; this is safe in cron because the job owns its cursor.
- Never commit inside a web request or a test.
- Keep `numbercall` and `doall` in mind: a negative `numbercall` means unlimited
  runs, and `doall` replays missed occurrences.

An unbounded `self.search([('state', '=', 'pending')])` can load millions of
records into memory; always pass `limit=500` or similar.

---

## Profiling and Query Count

### Counting queries in tests

`assertQueryCount` is defined on `BaseCase` in `odoo/tests/common.py`:

```python
def assertQueryCount(self, default=0, flush=True, **counters):
```

It counts `self.cr.sql_log_count` and only asserts while `self.warm` is true.
Use the `@warmup` decorator to run the test once for cache warm-up and once for
the real assertion.

```python
from odoo.tests.common import TransactionCase, users, warmup


class TestPerf(TransactionCase):

    @users('__system__', 'demo')
    @warmup
    def test_loop_is_constant_in_queries(self):
        orders = self.env['sale.order'].search([], limit=20)

        with self.assertQueryCount(__system__=3, demo=3):
            for order in orders:
                order.partner_id.name
```

Replace the expected counts with the baseline you measure on your own model;
the point of the test is that the count stays constant as the recordset grows.

Named counters are matched against `self.env.user.login`, which is why
`@users('__system__', 'demo')` is used together with
`assertQueryCount(__system__=3, demo=3)`.

The core `test_performance` module is the reference for expected counts: it
asserts 3 queries for the first traversal of `partner_id.country_id.name` and 0
for the second, cached traversal.

### Profiling with odoo.tools.profiler

```python
from odoo.tools.profiler import profile


class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    @profile
    def create(self, vals):
        return super().create(vals)

    @profile(whitelist=['sale.order', 'ir.model.data'])
    def action_quotation_send(self):
        ...
```

`profile(method=None, whitelist=None, blacklist=(None,), files=None,
minimum_time=0, minimum_queries=0)` logs the call tree, query counts, and
timings. The profiler itself adds overhead, so use it to locate hotspots, not
to benchmark.

### Developer mode and SQL logging

```bash
# Reload Python code on change, enable QWeb and Werkzeug debuggers.
odoo-bin --dev=all
# Accepted --dev options:
# all, [pudb|wdb|ipdb|pdb], reload, qweb, werkzeug, xml

# Log every SQL statement at DEBUG.
odoo-bin --log-level=debug_sql

# Shortcut for --log-handler=odoo.sql_db:DEBUG.
odoo-bin --log-sql
```

`debug_sql` is one of the accepted `--log-level` choices in
`odoo/tools/config.py`, and `--log-sql` is an explicit shortcut for the
`odoo.sql_db` logger. Use them to spot repeated identical queries, which are
the signature of an N+1 problem.

Reading the SQL log:

- Repeated `SELECT ... WHERE id = %s` with different ids: missing prefetch or a
  per-record loop.
- Repeated `SELECT ... WHERE <fk> = %s`: a search inside a loop.
- `UPDATE` per record: a loop around `write()`.

---

## Performance Checklist

- [ ] No `search()` inside loops
- [ ] Use `mapped()` and `filtered()` instead of Python loops for field access
- [ ] Use `search_read()` when only dicts are needed
- [ ] Use `read_group()` for aggregations; there is no `_read_group()` in 13.0
- [ ] Store expensive computed fields with `store=True`
- [ ] Declare dotted dependencies in `@api.depends`
- [ ] Use `with_context(bin_size=True)` for binary fields
- [ ] Use `with_context(tracking_disable=True)` for bulk writes on mail.thread
- [ ] Batch `create`, `write`, and `unlink` on recordsets
- [ ] Remember that 13.0 `create()` still inserts one row per INSERT
- [ ] Add `index=True` on frequently filtered and ordered fields
- [ ] Keep `_order` on stored, indexed columns
- [ ] Use field-level `auto_join=True` only where a JOIN beats a subquery
- [ ] Prefer SQL constraints for database-enforceable rules
- [ ] Call `sudo()` once on the recordset, not inside loops
- [ ] Flush before `cr.execute()` on pending fields
- [ ] Do not SQL-write; let the ORM do it
- [ ] Batch cron work with `search(limit=N)` and commit per chunk
- [ ] Add `assertQueryCount` tests to lock performance

---

## Common Anti-Patterns

### Unbounded search

An unbounded `self.search([('state', '=', 'draft')])` can load millions of
records; pass `limit=100` or similar.

### Per-record x2many creation

```python
# BAD: O(N) queries
for value in values:
    order.write({'line_ids': [(0, 0, {'value': value})]})

# GOOD: one write with all commands
order.write({'line_ids': [(0, 0, {'value': value}) for value in values]})
```

Even in one write, `(0, 0, ...)` commands still cost one INSERT per line in
13.0. When the lines already exist, link them with `(4, id)` or `(6, 0, ids)`
instead.

---

## Base Code Reference

Verified against the Odoo 13.0 source tree:

- `odoo/models.py` - `BaseModel`, `PREFETCH_MAX`, `_prefetch_ids`,
  `with_prefetch`, `_fetch_field`, `_read`, `_create`, `_write`, `unlink`,
  `search`, `search_count`, `search_read`, `read_group`, `_read_group_raw`,
  `exists`, `mapped`, `filtered`, `filtered_domain`, `sorted`, `flush`,
  `invalidate_cache`, `_order`, `_sql_constraints`, `_auto_init`
- `odoo/fields.py` - `index`, `auto_join`, `prefetch`, `group_operator`,
  `update_db_index`, binary `bin_size` handling
- `odoo/api.py` - `Environment`, `Cache.invalidate`, `@api.depends`,
  `@api.model_create_multi`
- `odoo/osv/expression.py` - domain-to-SQL translation and `auto_join` joins
- `odoo/sql_db.py` - `IN_MAX`, `split_for_in_conditions`, `sql_log_count`
- `odoo/tools/misc.py` - `split_every`
- `odoo/tools/sql.py` - schema helpers (`create_index`, `table_exists`,
  `constraint_definition`); no composable `SQL` class in 13.0
- `odoo/tools/profiler.py` - `profile` decorator
- `odoo/tools/config.py` - `--dev`, `--log-level` (including `debug_sql`),
  `--log-sql`
- `odoo/tests/common.py` - `BaseCase.assertQueryCount`, `warmup`, `users`
- `odoo/addons/base/models/ir_cron.py` - `_process_jobs`, `_process_job`,
  `numbercall`, `doall`
- `odoo/addons/test_performance/tests/test_performance.py` - reference query
  counts for read, write, create, x2many, and `read_group`
- `addons/mail/models/mail_thread.py` - `tracking_disable` context key
