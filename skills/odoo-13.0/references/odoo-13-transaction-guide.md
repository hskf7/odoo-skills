---
name: odoo-13-transaction
description: Complete reference for Odoo 13 database transactions, cursor ownership, savepoints, flush and cache behavior, psycopg2 errors, retries, and row locking.
globs: "**/*.{py,xml}"
topics:
  - Cursor and request transaction lifecycle
  - commit() and rollback() ownership
  - Savepoints with flush=True and flush=False
  - flush(), invalidate_cache(), and compute recomputation timing
  - psycopg2 IntegrityError, UniqueViolation, SerializationFailure, DeadlockDetected
  - Catching database errors without poisoning the transaction
  - Retry patterns and RPC concurrency retries
  - SELECT FOR UPDATE and FOR NO KEY UPDATE locking
  - Long transactions and cron worker behavior
  - Odoo 13.0 API differences from newer versions
when_to_use:
  - Handling duplicate key or constraint errors
  - Isolating failures with savepoints
  - Understanding when ORM writes reach PostgreSQL
  - Recovering from an aborted transaction
  - Preventing or retrying serialization and deadlock failures
  - Writing resilient cron jobs and batch imports
  - Reviewing code that calls commit() or rollback() directly
---

# Odoo 13 Transaction Guide

Complete reference for database transactions, psycopg2 errors, savepoints, flushing, and commits in Odoo 13.0.

## Table of Contents

1. [Transaction States](#transaction-states)
2. [Cursor and Request Transaction](#cursor-and-request-transaction)
3. [commit() and rollback()](#commit-and-rollback)
4. [Savepoints](#savepoints)
5. [Flush and Cache Interaction](#flush-and-cache-interaction)
6. [Compute Recomputation Timing](#compute-recomputation-timing)
7. [psycopg2 Errors](#psycopg2-errors)
8. [Catching Database Errors Safely](#catching-database-errors-safely)
9. [Retry Patterns](#retry-patterns)
10. [Concurrent Updates and Row Locks](#concurrent-updates-and-row-locks)
11. [Long Transactions](#long-transactions)
12. [Cron and Worker Retry Behavior](#cron-and-worker-retry-behavior)
13. [Odoo 13.0 API Differences](#odoo-130-api-differences)
14. [Exception Classes](#exception-classes)
15. [Quick Reference](#quick-reference)
16. [Common Patterns](#common-patterns)
17. [Base Code Reference](#base-code-reference)

---

## Transaction States

Odoo 13.0 cursors are serialized by default. `Cursor.__init__` calls `autocommit(False)`, and `Cursor.autocommit(False)` selects `ISOLATION_LEVEL_REPEATABLE_READ` when `self._serialized` is true, otherwise `ISOLATION_LEVEL_READ_COMMITTED`:

```python
# odoo/sql_db.py
isolation_level = \
    ISOLATION_LEVEL_REPEATABLE_READ \
    if self._serialized \
    else ISOLATION_LEVEL_READ_COMMITTED
self._cnx.set_isolation_level(isolation_level)
```

`Connection.cursor(serialized=True)` is the default. `serialized_cursor` is only an alias for `cursor`.

Under REPEATABLE READ:

- The transaction sees a snapshot taken at its first query.
- A concurrent update to a row you also update raises a serialization failure (`40001`) instead of losing an update.
- Schema changes by other transactions are not visible mid-transaction.
- Long transactions keep an old snapshot and can block vacuum.

State flow:

```
Normal --[SQL error]--> ABORTED --[ROLLBACK]--> Normal
                           |
                           +--[any SQL]--> InFailedSqlTransaction (25P02)
                           +--[COMMIT]---> error: cannot commit aborted transaction
```

Once PostgreSQL aborts the transaction, every statement fails with `25P02` until you roll back the whole transaction or roll back to a savepoint created before the failure.

| Cursor attribute | Meaning |
|------------------|---------|
| `cr.dbname` | Database name |
| `cr._cnx` | Underlying psycopg2 connection |
| `cr._obj` | Underlying psycopg2 cursor |
| `cr.cache` | Cursor-lifetime cache, cleared on close |
| `cr.sql_log_count` | Number of queries executed |
| `cr.IN_MAX` | 1000, recommended maximum `IN` list size |
| `cr.closed` | Whether the cursor has been closed |

`Cursor.split_for_in_conditions(ids, size=None)` splits large `IN` lists:

```python
for sub_ids in self.env.cr.split_for_in_conditions(ids):
    self.env.cr.execute(
        'SELECT id FROM res_partner WHERE id IN %s', (tuple(sub_ids),))
```

---

## Cursor and Request Transaction

In an HTTP request, `request.cr` lazily creates a cursor from the registry:

```python
# odoo/http.py
@property
def cr(self):
    if not self.db:
        raise RuntimeError('request not bound to a database')
    if not self._cr:
        self._cr = self.registry.cursor()
    return self._cr
```

`request.env` is built from that cursor, the request uid, and the request context. RPC calls open the cursor with `with odoo.registry(db).cursor() as cr:` in `odoo/service/model.py`.

In tests, `registry.cursor()` returns a `TestCursor` when the registry is in test mode:

```python
# odoo/modules/registry.py
def cursor(self):
    if self.test_cr is not None:
        return TestCursor(self.test_cr, self.test_lock)
    return self._db.cursor()
```

`TestCursor` does not really commit. It keeps a savepoint at its last commit:

| TestCursor call | Actual SQL |
|-----------------|------------|
| construction | `SAVEPOINT test_cursor_N` |
| `commit()` | flush, then `SAVEPOINT test_cursor_N` |
| `rollback()` | `ROLLBACK TO SAVEPOINT test_cursor_N` |
| `close()` | `ROLLBACK TO SAVEPOINT test_cursor_N` |

This is why a test can call `cr.commit()` and still be rolled back by the test runner.

| Context | Who commits | Notes |
|---------|-------------|-------|
| HTTP request | `request.__exit__` | Only if no exception and `request._failed` is unset |
| JSON-RPC / XML-RPC | `Cursor.__exit__` in `service/model.py` | Commits on clean exit, closes always |
| Cron job | `ir_cron._process_job` | `job_cr.commit()` and `cron_cr.commit()` in `finally` |
| Test | Test runner | `TestCursor` simulates commit with a savepoint |
| Shell / script | You | Use `with registry.cursor() as cr:` or commit explicitly |

The HTTP path is explicit about failure:

```python
# odoo/http.py
def __exit__(self, exc_type, exc_value, traceback):
    _request_stack.pop()
    if self._cr:
        try:
            if exc_type is None and not self._failed:
                self._cr.commit()
                if self.registry:
                    self.registry.signal_changes()
            elif self.registry:
                self.registry.reset_changes()
        finally:
            self._cr.close()
```

`_handle_exception` sets `self._failed = exception` before re-raising, so a handled exception still prevents the commit.

`Cursor.__enter__` returns the cursor. `Cursor.__exit__` commits when no exception occurred and always closes:

```python
with self.env.registry.cursor() as cr:
    cr.execute("UPDATE res_partner SET active = true WHERE id = %s", (partner_id,))
# committed on clean exit, closed in all cases
```

`Cursor.close()` rolls back the underlying connection before returning it to the pool. Closing a cursor is a rollback, not a commit.

---

## commit() and rollback()

```python
# odoo/sql_db.py
def commit(self):
    flush_env(self)
    result = self._cnx.commit()
    for func in self._pop_event_handlers()['commit']:
        func()
    return result

def rollback(self):
    clear_env(self)
    result = self._cnx.rollback()
    for func in self._pop_event_handlers()['rollback']:
        func()
    return result
```

`flush_env(self)` flushes the environment attached to the cursor before committing. `clear_env(self)` clears the environment cache before rolling back. Both are module-level helpers in `odoo/sql_db.py`:

```python
def flush_env(cr, *, clear=True):
    env_to_flush = None
    for env in list(Environment.envs):
        if env.cr is cr and (isinstance(env.uid, int) or env.uid is None):
            env_to_flush = env
            if env.uid is not None:
                break
    if env_to_flush is not None:
        env_to_flush['base'].flush()
        if clear:
            env_to_flush.clear()

def clear_env(cr):
    for env in list(Environment.envs):
        if env.cr is cr:
            env.clear()
            break
```

`env['base'].flush()` with no field names flushes every pending computation and write across all models. There is no `env.flush_all()` in 13.0.

13.0 has `Cursor.after(event, func)` with two events, `'commit'` and `'rollback'`:

```python
self.env.cr.after('commit', lambda: _logger.info('committed'))
```

Handlers run after the database operation. The docstring warns that cursor operations inside a handler run in the next transaction, which may itself be rolled back. Use a dedicated cursor for database work in a handler. There are no `precommit`, `postcommit`, `prerollback`, or `postrollback` callback queues in 13.0.

Manual `commit()` is appropriate only for:

1. Long cron jobs or imports that must release locks and persist progress in chunks.
2. Migration or maintenance scripts outside a normal request.
3. A deliberately independent audit/log write on a separate cursor.

```python
def _cron_sync(self):
    for chunk in self._chunks():
        self._sync_chunk(chunk)
        self.env.cr.commit()
```

Do not commit in the middle of business logic:

```python
# BAD: a mid-business-logic commit makes earlier work impossible to roll back
def create_order(self, vals):
    order = self.create(vals)
    self.env.cr.commit()
    order.action_confirm()   # if this fails, the order is already durable
    self.env.cr.commit()
```

A commit inside a method also breaks the caller's ability to treat the operation as one unit.

```python
def batch_import(self, rows):
    try:
        for row in rows:
            self.create(row)
        self.env.cr.commit()
    except Exception:
        self.env.cr.rollback()
        _logger.exception("Batch import failed, rolled back")
        raise
```

After `rollback()`, the environment cache is cleared, so records read before the rollback must be re-read.

---

## Savepoints

13.0 does not define `Savepoint` or `_FlushingSavepoint` classes. `Cursor.savepoint` is a context manager method:

```python
# odoo/sql_db.py
@contextmanager
@check
def savepoint(self, flush=True):
    """context manager entering in a new savepoint"""
    name = uuid.uuid1().hex
    if flush:
        flush_env(self, clear=False)
    self.execute('SAVEPOINT "%s"' % name)
    try:
        yield
        if flush:
            flush_env(self, clear=False)
    except Exception:
        if flush:
            clear_env(self)
        self.execute('ROLLBACK TO SAVEPOINT "%s"' % name)
        raise
    else:
        self.execute('RELEASE SAVEPOINT "%s"' % name)
```

Consequences:

- The savepoint name is a fresh UUID, so nested savepoints do not collide.
- With `flush=True` (default), pending ORM writes are flushed before `SAVEPOINT` and again before `RELEASE`.
- On exception, the environment cache is cleared, the savepoint is rolled back, and the exception is re-raised.
- On success, the savepoint is released.
- The context manager does not swallow exceptions. Catch outside it to continue.

```python
with self.env.cr.savepoint():
    self.create({'name': 'something'})
    raise ValueError("boom")
# The savepoint was rolled back; the outer transaction is still usable.
```

Isolating a database error:

```python
import psycopg2
from psycopg2 import errorcodes

try:
    with self.env.cr.savepoint():
        self.create({'email': email})
except psycopg2.IntegrityError as exc:
    if exc.pgcode == errorcodes.UNIQUE_VIOLATION:
        _logger.info("Duplicate email %s, skipping", email)
    else:
        raise
# The outer transaction is still valid here.
```

Without the savepoint, the same `IntegrityError` aborts the whole transaction and the next query fails with `InFailedSqlTransaction`.

`flush=True` is the default and is what you want for ORM work: pending writes are flushed before the savepoint, flushed again before release, and the cache is cleared on failure. `flush=False` is for operations that should not touch the ORM cache: DDL, a lock attempt that should fail fast, or raw SQL that does not read ORM-managed fields.

```python
# odoo/tools/sql.py - schema helper
with cr.savepoint(flush=False):
    cr.execute('ALTER TABLE "{}" ALTER COLUMN "{}" TYPE {}'.format(
        tablename, columnname, columntype,
    ), log_exceptions=False)
```

```python
# Lock attempt without flushing pending ORM writes
with self.env.cr.savepoint(flush=False):
    self.env.cr.execute(
        'SELECT id FROM sale_order WHERE id IN %s FOR UPDATE NOWAIT',
        (tuple(order_ids),),
    )
```

With `flush=False`, a failed savepoint does not clear the cache. If the failed statement could have changed cached values, call `invalidate_cache()` yourself.

Nested savepoints compose correctly. Rolling back the inner one leaves the outer savepoint and the main transaction valid:

```python
with self.env.cr.savepoint():          # outer
    with self.env.cr.savepoint():      # inner
        raise ValueError("inner failed")
    # inner rolled back; outer and main transaction still valid
# outer released normally
```

Batch import with per-record isolation:

```python
import psycopg2
from odoo.exceptions import ValidationError

def batch_create_safe(self, rows):
    created, failed = [], []
    for data in rows:
        with self.env.cr.savepoint():
            try:
                created.append(self.create(data))
            except (psycopg2.Error, ValidationError) as exc:
                failed.append({'data': data, 'error': str(exc)})
    return created, failed
```

Each failing row rolls back only its own savepoint. The rest of the batch remains in the outer transaction and is committed by the request or cron job.

`Environment.clear_upon_failure()` restores the recomputation and write queues if the block fails. It is used with a savepoint when a failed ORM operation must not leave the environment half-written:

```python
# odoo/addons/base/wizard/base_partner_merge.py
try:
    with mute_logger('odoo.sql_db'), self._cr.savepoint(), self.env.clear_upon_failure():
        records.sudo().write({field_id: dst_partner.id})
        records.flush()
except psycopg2.Error:
    records.sudo().unlink()
```

Anti-patterns:

```python
# BAD: manual savepoint with a reused name
cr.execute('SAVEPOINT foo')
cr.execute('RELEASE SAVEPOINT foo')
cr.execute('SAVEPOINT foo')   # fragile; a missed release leaks the savepoint

# GOOD: the context manager generates a unique name
with self.env.cr.savepoint():
    ...

# BAD: catching a database error without a savepoint
try:
    self.create(vals)
except psycopg2.IntegrityError:
    pass
self.search([])   # InFailedSqlTransaction: the transaction is already aborted

# BAD: swallowing an error and continuing with stale cache
try:
    with self.env.cr.savepoint(flush=False):
        self.env.cr.execute("UPDATE res_partner SET name = 'x' WHERE id = %s", (pid,))
except Exception:
    pass
self.env['res.partner'].browse(pid).name   # may still be the old cached value
```

---

## Flush and Cache Interaction

There is no `env.flush_all()` and no `flush_model()` in 13.0. The available calls are:

| Call | Scope |
|------|-------|
| `recordset.flush()` | All pending computations and writes, all models |
| `recordset.flush(['field_a', 'field_b'])` | Given fields of this model |
| `recordset.flush(['field'], records=subset)` | Given fields on given records |
| `self.env['base'].flush()` | All pending computations and writes, all models |
| `flush_env(cr)` | Flush the environment attached to a cursor |

`Model.flush` is defined in `odoo/models.py`:

```python
@api.model
def flush(self, fnames=None, records=None):
    """ Process all the pending computations (on all models), and flush all
    the pending updates to the database. """
```

With `fnames=None`, it calls `self.recompute()` and then writes every entry in `self.env.all.towrite`. With field names, it recomputes those fields and flushes only matching pending writes, including related fields and one2many inverses.

ORM writes are not necessarily in the database yet. `write()` stores values in `env.all.towrite` and marks dependent computed fields in `env.all.tocompute`. A raw `cr.execute()` that reads those columns can see stale data unless you flush first:

```python
def get_totals(self):
    self.env['sale.order'].flush(['state', 'amount_total'])
    self.env.cr.execute("""
        SELECT state, SUM(amount_total)
          FROM sale_order
         WHERE state IN %s
         GROUP BY state
    """, (('done', 'cancel'),))
    return dict(self.env.cr.fetchall())
```

Flush the exact fields you are about to read. Flushing everything is correct but can trigger expensive recomputations.

`Model.invalidate_cache(fnames=None, ids=None)` drops cached values after the database changed behind the ORM's back:

```python
@api.model
def invalidate_cache(self, fnames=None, ids=None):
    """ Invalidate the record caches after some records have been modified.
        If both ``fnames`` and ``ids`` are ``None``, the whole cache is cleared.
    """
```

Typical use after raw SQL:

```python
self.env.cr.execute(
    "UPDATE sale_order SET state = 'done' WHERE id = %s", (order_id,))
self.env['sale.order'].invalidate_cache(['state'], [order_id])
```

`invalidate_cache()` does not flush. If you have pending ORM writes, flush them first or they may be lost or overwrite the raw SQL change later.

`Environment.clear()` clears all record caches and discards both the recomputation and write queues:

```python
def clear(self):
    """ Clear all record caches, and discard all fields to recompute.
        This may be useful when recovering from a failed ORM operation.
    """
    self.cache.invalidate()
    self.all.tocompute.clear()
    self.all.towrite.clear()
```

`Cursor.rollback()` calls `clear_env(cr)`, which calls `env.clear()`. `Cursor.commit()` calls `flush_env(cr)`, which flushes and then clears.

The following are newer-version APIs and are not available in Odoo 13.0: `env.flush_all()`, `env.invalidate_all()`, `model.flush_model(...)`, `model.invalidate_model(...)`, and `cr.flush()`. Use `model.flush()`, `env['base'].flush()`, `model.invalidate_cache()`, and `env.clear()` instead.

---

## Compute Recomputation Timing

`write()` does not immediately issue one `UPDATE` per field. It stores values through `field.write()` into `env.all.towrite`, then calls `self.modified(vals)` to mark dependent computed fields:

```python
# odoo/models.py - write()
for fname in sorted(vals, key=lambda x: self._fields[x].type == 'monetary'):
    if fname in bad_names:
        continue
    field = self._fields[fname]
    field.write(self, vals[fname])

# determine records depending on new values
self.modified(vals)
```

`modified()` walks the field trigger graph and adds stored computed fields to `env.all.tocompute`. Non-stored computed fields are invalidated in the cache instead.

A stored computed field is recomputed in one of two ways:

1. On read, if it is still in `env.all.tocompute`:

```python
# odoo/fields.py - Field.__get__
if self.compute and (record.id in env.all.tocompute.get(self, ())) \
        and not env.is_protected(self, record):
    self.compute_value(recs)
```

2. On flush, which calls `recompute()` and then writes pending values:

```python
# odoo/models.py - flush()
if fnames is None:
    self.recompute()
    while self.env.all.towrite:
        model_name, id_vals = self.env.all.towrite.popitem()
        process(self.env[model_name], id_vals)
```

`recompute()` processes `env.fields_to_compute()`, calls the compute method through `mapped(field.name)`, and removes the field from the queue.

Practical consequences:

- Reading a computed field can trigger a compute and additional queries.
- A raw SQL read of a computed column must be preceded by `flush()`.
- A `search()` on a computed stored field can trigger a flush and recompute.
- `modified()` is called after values are written to the cache, because a recompute can trigger a search that flushes and would otherwise compute too early.
- `env.norecompute()` exists but is deprecated in 13.0 and simply yields. Do not build new code around it.

`write()` uses `env.protecting(...)` to prevent fields being written from being recomputed while inverses run. This is internal, but it explains why a compute method may see a partially updated recordset during a write. Do not rely on compute methods being called exactly once per write.

---

## psycopg2 Errors

| SQLSTATE | psycopg2 class in 2.7 | Meaning |
|----------|-----------------------|---------|
| `23502` | `IntegrityError` | NOT NULL violation |
| `23505` | `IntegrityError` | UNIQUE violation |
| `23514` | `IntegrityError` | CHECK violation |
| `23503` | `IntegrityError` | FOREIGN KEY violation |
| `40001` | `OperationalError` | Serialization failure |
| `40P01` | `OperationalError` | Deadlock detected |
| `55P03` | `OperationalError` | Lock not available |
| `25P02` | `InternalError` | In-failed SQL transaction |

Odoo 13.0 pins `psycopg2==2.7.7` on Python below 3.8 and `psycopg2==2.8.3` on Windows or Python 3.8 and above. The `psycopg2.errors` module is not available in 2.7.7. For portable 13.0 code, catch `psycopg2.IntegrityError` or `psycopg2.OperationalError` and inspect `exc.pgcode` with `psycopg2.errorcodes`:

```python
import psycopg2
from psycopg2 import errorcodes

try:
    self.create({'email': email})
except psycopg2.IntegrityError as exc:
    if exc.pgcode == errorcodes.UNIQUE_VIOLATION:
        ...
    else:
        raise
```

If the deployment uses psycopg2 2.8 or newer, `from psycopg2 import errors` and `errors.UniqueViolation` also work, but do not assume it in a 13.0 addon.

A unique violation is an `IntegrityError` with `pgcode == '23505'`. It aborts the transaction unless it happened inside a savepoint.

```python
# BAD: no savepoint, transaction is aborted after the catch
try:
    self.create({'email': email})
except psycopg2.IntegrityError:
    existing = self.search([('email', '=', email)])   # InFailedSqlTransaction

# GOOD: isolate with a savepoint
try:
    with self.env.cr.savepoint():
        self.create({'email': email})
except psycopg2.IntegrityError as exc:
    if exc.pgcode != errorcodes.UNIQUE_VIOLATION:
        raise
# Transaction remains valid.

# BEST: check first when the key is known and races are acceptable
existing = self.search([('email', '=', email)], limit=1)
if not existing:
    self.create({'email': email})
```

Check-then-create still races under concurrency. Keep the savepoint around the create if a duplicate is possible.

`odoo/models.py` defines `PGERROR_TO_OE` for three codes:

```python
PGERROR_TO_OE = defaultdict(
    lambda: (lambda model, fvg, info, pgerror: {'message': tools.ustr(pgerror)}), {
    '23502': convert_pgerror_not_null,
    '23505': convert_pgerror_unique,
    '23514': convert_pgerror_constraint,
})
```

`convert_pgerror_unique` opens a new cursor because the original transaction is already aborted:

```python
def convert_pgerror_unique(model, fields, info, e):
    # new cursor since we're probably in an error handler in a blown
    # transaction which may not have been rollbacked/cleaned yet
    with closing(model.env.registry.cursor()) as cr_tmp:
        cr_tmp.execute("""
            SELECT conname, t.relname, ARRAY(...)
              FROM pg_constraint
              JOIN pg_class t ON t.oid = conrelid
             WHERE conname = %s
        """, [e.diag.constraint_name])
```

In 13.0 this mapping is used by `models.load()` for import error messages. It is not a general-purpose exception translator for business code.

`SerializationFailure` (`40001`) is raised under REPEATABLE READ when a concurrent transaction modified a row you are trying to update. The RPC layer retries it automatically; inside a cron job or long-lived script you must retry yourself. `DeadlockDetected` (`40P01`) is raised when PostgreSQL detects a lock cycle and kills one transaction; it is also retried by the RPC layer. `LockNotAvailable` (`55P03`) is raised by `FOR UPDATE NOWAIT` and `FOR NO KEY UPDATE NOWAIT` when another transaction holds the lock; it is an `OperationalError`, not an `IntegrityError`. `InFailedSqlTransaction` (`25P02`) is raised by any statement after the transaction has been aborted; the only recovery is `ROLLBACK` or `ROLLBACK TO SAVEPOINT`.

---

## Catching Database Errors Safely

Catch a database error only in one of these two situations:

1. The failing statement is inside `with self.env.cr.savepoint():`, so the savepoint can be rolled back.
2. You are about to roll back the entire transaction.

Anything else leaves the transaction aborted and the environment cache potentially inconsistent.

```python
import psycopg2
from psycopg2 import errorcodes

def _try_create(self, vals):
    try:
        with self.env.cr.savepoint():
            return self.create(vals)
    except psycopg2.IntegrityError as exc:
        if exc.pgcode == errorcodes.UNIQUE_VIOLATION:
            _logger.info("Duplicate record for %s", vals.get('name'))
            return self.browse()
        raise
```

The `raise` for unexpected codes is important. Swallowing every `IntegrityError` hides NOT NULL, CHECK, and FOREIGN KEY bugs.

If the error happened outside a savepoint, roll back before doing anything else:

```python
try:
    self._do_work()
except psycopg2.Error:
    self.env.cr.rollback()
    _logger.exception("Work failed, transaction rolled back")
    raise
```

After `rollback()`, the environment cache is cleared. Re-browse records before using them.

A new cursor sees only committed data, but it can write when the original transaction is aborted:

```python
from contextlib import closing
import odoo

def _log_failure(self, message):
    with closing(self.env.registry.cursor()) as cr:
        env = odoo.api.Environment(cr, self.env.uid, self.env.context)
        env['audit.log'].create({'message': message})
        cr.commit()
```

Use this only for audit or error logging. The new cursor cannot see uncommitted work from the original transaction.

```python
# BAD: the transaction is aborted, but the code keeps going
try:
    self.env.cr.execute("INSERT INTO res_partner (name) VALUES ('x')")
except psycopg2.Error:
    pass
self.env.cr.execute("SELECT 1")   # InFailedSqlTransaction
```

If you need to continue after a failure, wrap the failing operation in a savepoint before it runs.

---

## Retry Patterns

`odoo/service/model.py` wraps RPC execution with a retry loop:

```python
PG_CONCURRENCY_ERRORS_TO_RETRY = (
    errorcodes.LOCK_NOT_AVAILABLE,
    errorcodes.SERIALIZATION_FAILURE,
    errorcodes.DEADLOCK_DETECTED,
)
MAX_TRIES_ON_CONCURRENCY_FAILURE = 5
```

The `check` decorator catches `OperationalError` and `QWebException` whose cause is an `OperationalError`, and retries when `e.pgcode` is one of the three codes:

```python
wait_time = random.uniform(0.0, 2 ** tries)
tries += 1
_logger.info("%s, retry %d/%d in %.04f sec...",
             errorcodes.lookup(e.pgcode), tries,
             MAX_TRIES_ON_CONCURRENCY_FAILURE, wait_time)
time.sleep(wait_time)
```

The wait is random between 0 and `2 ** tries` seconds, with at most 5 attempts. The HTTP layer uses the same decorator and rolls back the request cursor before retrying:

```python
# odoo/http.py - _call_function
@service_model.check
def checked_call(___dbname, *a, **kw):
    nonlocal first_time
    if self._cr and not first_time:
        self._cr.rollback()
        self.env.clear()
    first_time = False
    result = self.endpoint(*a, **kw)
    ...
    if self._cr is not None:
        flush_env(self._cr)
    return result
```

The final `flush_env` is deliberate: it forces pending writes to hit the database inside the retry-protected call, so a serialization error is retried rather than raised later at commit time.

`execute_cr` also resets the environment before each call:

```python
def execute_cr(cr, uid, obj, method, *args, **kw):
    odoo.api.Environment.reset()  # clean cache etc if we retry the same transaction
    recs = odoo.api.Environment(cr, uid, {}).get(obj)
    ...
```

Manual retry is needed in cron jobs, long scripts, and code that catches the error before the RPC layer sees it. Always roll back before retrying:

```python
import time
import psycopg2
from psycopg2 import errorcodes

def _write_with_retry(self, values, retries=3):
    for attempt in range(retries):
        try:
            return self.write(values)
        except psycopg2.OperationalError as exc:
            if exc.pgcode not in (
                errorcodes.SERIALIZATION_FAILURE,
                errorcodes.DEADLOCK_DETECTED,
                errorcodes.LOCK_NOT_AVAILABLE,
            ):
                raise
            if attempt == retries - 1:
                raise
            self.env.cr.rollback()
            time.sleep(0.05 * (2 ** attempt))
```

Important details:

- `rollback()` is mandatory. Retrying on an aborted transaction fails immediately.
- Re-browse records after the rollback; the old recordset may point at a cleared cache.
- Retry only concurrency errors. Retrying an `IntegrityError` will fail the same way.
- Make the operation idempotent, or ensure the retry starts from a clean state.

If the operation is small and the outer transaction must survive, retry inside a savepoint:

```python
def _update_with_savepoint_retry(self, values, retries=3):
    for attempt in range(retries):
        try:
            with self.env.cr.savepoint():
                return self.write(values)
        except psycopg2.OperationalError as exc:
            if exc.pgcode != errorcodes.SERIALIZATION_FAILURE:
                raise
            if attempt == retries - 1:
                raise
```

A savepoint rollback keeps the outer transaction alive, but the outer transaction's snapshot is unchanged. Under REPEATABLE READ, retrying the same read-then-write inside the same transaction can fail again. For serious contention, roll back the whole transaction or use row locks.

A retry must not duplicate side effects. Before retrying, ask: was an email sent, a file written, or an external API called? Is the create protected by a unique key? Can the operation be expressed as an idempotent upsert? External side effects belong after the database work, or behind an outbox pattern, not in the middle of a retry loop.

---

## Concurrent Updates and Row Locks

`Model._check_concurrency()` uses the `__last_update` context key:

```python
CONCURRENCY_CHECK_FIELD = '__last_update'

def _check_concurrency(self):
    if not (self._log_access and self._context.get(self.CONCURRENCY_CHECK_FIELD)):
        return
    check_clause = "(id = %s AND %s < COALESCE(write_date, create_date, (now() at time zone 'UTC'))::timestamp)"
    ...
    raise ValidationError(_('A document was modified since you last viewed it (%s:%d)') % (...))
```

This is the web client's stale-form protection. It is not a substitute for row locks in concurrent batch code.

`FOR UPDATE NOWAIT` takes a row lock immediately and raises `LockNotAvailable` (`55P03`) instead of waiting. Odoo 13.0 uses it in several core places.

`ir_sequence._update_nogap`:

```python
# odoo/addons/base/models/ir_sequence.py
def _update_nogap(self, number_increment):
    number_next = self.number_next
    self._cr.execute(
        "SELECT number_next FROM %s WHERE id=%s FOR UPDATE NOWAIT" % (self._table, self.id)
    )
    self._cr.execute(
        "UPDATE %s SET number_next=number_next+%s WHERE id=%s " % (self._table, number_increment, self.id)
    )
    self.invalidate_cache(['number_next'], [self.id])
    return number_next
```

`ir_cron` uses a separate lock cursor to claim a job:

```python
# odoo/addons/base/models/ir_cron.py
lock_cr.execute("""SELECT *
                   FROM ir_cron
                   WHERE numbercall != 0
                      AND active
                      AND nextcall <= (now() at time zone 'UTC')
                      AND id=%s
                   FOR UPDATE NOWAIT""",
                (job['id'],), log_exceptions=False)
```

`ir_cron._try_lock` uses the same pattern for manual edits and rolls back on failure:

```python
def _try_lock(self):
    try:
        self._cr.execute(
            """SELECT id FROM "%s" WHERE id IN %%s FOR UPDATE NOWAIT""" % self._table,
            [tuple(self.ids)],
            log_exceptions=False,
        )
    except psycopg2.OperationalError:
        self._cr.rollback()
        raise UserError(_("Record cannot be modified right now: "
                          "This cron task is currently being executed and may not be modified "
                          "Please try again in a few minutes"))
```

`website_visitor` uses a weaker lock that does not block foreign-key checks:

```python
# addons/website/models/website_visitor.py
try:
    with self.env.cr.savepoint():
        query_lock = "SELECT * FROM website_visitor where id = %s FOR NO KEY UPDATE NOWAIT"
        self.env.cr.execute(query_lock, (self.id,), log_exceptions=False)
        ...
except Exception:
    pass
```

Use `FOR NO KEY UPDATE` when you update non-key columns and want to avoid blocking inserts that reference the row.

Locking pattern for batch work:

```python
import psycopg2
from psycopg2 import errorcodes

def _process_locked(self, record):
    try:
        with self.env.cr.savepoint(flush=False):
            self.env.cr.execute(
                'SELECT id FROM %s WHERE id = %%s FOR UPDATE NOWAIT' % self._table,
                (record.id,),
            )
    except psycopg2.OperationalError as exc:
        if exc.pgcode == errorcodes.LOCK_NOT_AVAILABLE:
            _logger.info("Record %s is locked, skipping", record.id)
            return False
        raise
    record._do_process()
    return True
```

`savepoint(flush=False)` avoids flushing unrelated pending writes just to attempt the lock. The savepoint is still needed: a failed `NOWAIT` aborts the transaction otherwise.

Plain `SELECT ... FOR UPDATE` waits for the other transaction. It is appropriate when waiting is acceptable and you want strict serialization:

```python
self.env.cr.execute(
    'SELECT id FROM account_move WHERE id = %s FOR UPDATE', (move_id,))
```

A blocking lock inside a long transaction can stall workers. Prefer `NOWAIT` with a skip or retry policy for background jobs.

To reduce conflicts: group identical updates into one `write()` instead of one write per record; lock rows in a consistent order to avoid deadlocks; keep transactions short and do not hold a lock across an external API call; use `FOR NO KEY UPDATE` when key columns are not changing; retry serialization failures with backoff, or skip locked work.

---

## Long Transactions

A long transaction under REPEATABLE READ holds an old snapshot, holds row locks that block other workers, prevents vacuum from cleaning dead rows, increases the chance of serialization failures and deadlocks, and keeps the connection checked out from the pool.

For imports and cron jobs, commit per chunk and re-open the environment state:

```python
def _cron_import(self, rows, chunk_size=500):
    for start in range(0, len(rows), chunk_size):
        chunk = rows[start:start + chunk_size]
        for data in chunk:
            with self.env.cr.savepoint():
                self.create(data)
        self.env.cr.commit()
```

After `commit()`, the environment cache is cleared. Do not keep recordsets across the commit and expect their cached values to remain.

A commit is a point of no return. If the next chunk fails, the earlier chunks stay committed. Design the job so partial progress is acceptable, or record progress in a way that can resume.

Do not open a cursor and then wait on a network call, a user action, or a long computation. The connection stays in `idle in transaction`, holding locks and blocking vacuum. Do the external work first, then open the transaction.

`db_maxconn` limits the connection pool. A leaked cursor or a long transaction reduces capacity for other workers. Always close cursors you open:

```python
from contextlib import closing

with closing(self.env.registry.cursor()) as cr:
    ...
```

---

## Cron and Worker Retry Behavior

`ir_cron._process_jobs` selects due jobs, then for each job opens a separate `lock_cr` and runs:

```sql
SELECT * FROM ir_cron
 WHERE numbercall != 0
   AND active
   AND nextcall <= (now() at time zone 'UTC')
   AND id = %s
 FOR UPDATE NOWAIT
```

If the lock is unavailable, the `OperationalError` with `pgcode == '55P03'` is caught and the job is skipped:

```python
except psycopg2.OperationalError as e:
    if e.pgcode == '55P03':
        _logger.debug('Another process/thread is already busy executing job `%s`, skipping it.', job['cron_name'])
        continue
    else:
        raise
```

This is how multiple workers avoid running the same cron job twice.

`_process_job` runs the job on `job_cr` and updates the schedule on `cron_cr`. Both are committed in `finally`:

```python
finally:
    job_cr.commit()
    cron_cr.commit()
```

The schedule update (`nextcall`, `numbercall`, `lastcall`) is therefore committed even when the job body raised, because `_callback` catches the exception and rolls back `job_cr` first.

`_callback` catches every exception, logs it, and calls `_handle_callback_exception`:

```python
def _handle_callback_exception(self, cron_name, server_action_id, job_id, job_exception):
    """ Method called when an exception is raised by a job.
        Simply logs the exception and rollback the transaction. """
    self._cr.rollback()
```

There is no automatic retry of a failed cron job in 13.0. The job is marked as run, `nextcall` advances, and the failure is only visible in the log. If you need retries, implement them inside the job or use a queue module.

Cron does not go through `service_model.check`, so serialization failures and deadlocks are not retried automatically. A cron job that updates contended rows should use `FOR UPDATE NOWAIT` and skip locked work, catch `OperationalError` with `40001` or `40P01` and retry with backoff, or be split into smaller jobs that touch fewer rows.

HTTP workers use `service_model.check`, so a request that hits a serialization failure is retried up to 5 times. The retry rolls back the request cursor and clears the environment. Any Python-side state built before the failure is lost; the endpoint runs again from the start. This means request handlers must be safe to run twice. Do not send emails or call external APIs before the database work is committed unless the side effect is idempotent.

---

## Odoo 13.0 API Differences

The following points are specific to 13.0 and are easy to get wrong when porting code from newer versions.

13.0 has no `env.flush_all()` and no `env.invalidate_all()`. Use:

```python
self.env['base'].flush()                 # flush everything
self.env['sale.order'].flush(['state'])  # flush selected fields
self.env['sale.order'].invalidate_cache(['state'], ids)
self.env.clear()                         # drop caches and pending queues
```

13.0 exposes `cr.savepoint(flush=True)` as a context manager. There is no `Savepoint` class, no `_FlushingSavepoint`, and no `cr.savepoint(flush=False).rollback()` API. Use the context manager and catch outside it.

`Cursor` has no `flush()` method in 13.0. Flushing is an ORM operation: `model.flush()` or `env['base'].flush()`. Use `model.flush(fnames)` and `model.invalidate_cache(fnames, ids)`; there is no `flush_model` or `invalidate_model`.

`env.company` and `env.companies` exist, but `with_company()` does not. Use `with_context(allowed_company_ids=[company.id])` or `with_context(force_company=company.id)` where the model supports it.

Relational fields use command triplets in 13.0:

```python
order.write({
    'order_line': [
        (0, 0, {'product_id': product.id, 'product_uom_qty': 1}),
        (1, line.id, {'product_uom_qty': 2}),
        (2, line.id, 0),
        (3, line.id, 0),
        (4, other_line.id, 0),
        (5, 0, 0),
        (6, 0, line_ids),
    ],
})
```

`odoo.fields.Command` is not available in 13.0.

`_cr`, `_uid`, and `_context` still exist as read-only properties:

```python
_cr = property(lambda self: self.env.cr)
_uid = property(lambda self: self.env.uid)
_context = property(lambda self: self.env.context)
```

New code should use `self.env.cr`, `self.env.uid`, and `self.env.context`. The properties are kept for compatibility with older addons.

13.0 pins psycopg2 2.7.7 on Python below 3.8. Do not assume `psycopg2.errors` exists. Catch `psycopg2.IntegrityError` and `psycopg2.OperationalError` and compare `exc.pgcode` with `psycopg2.errorcodes`.

`Environment.reset()` is a classmethod that clears the set of environments. It is used by the RPC layer before re-executing a call after a retry. Business code should not need it; use `env.clear()` to clear caches and pending queues.

---

## Exception Classes

Odoo exception types live in `odoo/exceptions.py`:

| Class | Inherits | Typical meaning |
|-------|----------|-----------------|
| `UserError` | `except_orm` | Generic user-facing error |
| `ValidationError` | `except_orm` | Python or SQL constraint violation |
| `AccessError` | `except_orm` | Access right or record rule denial |
| `AccessDenied` | `Exception` | Login or password error, no traceback |
| `MissingError` | `except_orm` | Record does not exist or was deleted |
| `RedirectWarning` | `Exception` | Error with a follow-up action |
| `CacheMiss` | `except_orm`, `KeyError` | Internal cache miss |
| `DeferredException` | `Exception` | Deferred report or database creation error |
| `QWebException` | `Exception` | QWeb rendering error |
| `except_orm` | `Exception` | Deprecated base class |

```python
from odoo.exceptions import UserError, ValidationError, AccessError, RedirectWarning

if not partner.email:
    raise UserError(_("Partner has no email"))

if amount < 0:
    raise ValidationError(_("Amount must be positive"))
```

`UserError` and `ValidationError` are business errors. They do not abort the transaction by themselves; the request or RPC layer rolls back when the exception propagates. If you catch them and continue, the transaction remains usable.

| Error | Layer | Transaction state |
|-------|-------|-------------------|
| `UserError` | Odoo | Still valid |
| `ValidationError` | Odoo | Still valid |
| `psycopg2.IntegrityError` | PostgreSQL | Aborted unless inside a savepoint |
| `psycopg2.OperationalError` | PostgreSQL | Aborted unless inside a savepoint |
| `InFailedSqlTransaction` | PostgreSQL | Already aborted |

Do not convert a database error into a `UserError` and continue without a savepoint. The transaction is still aborted.

---

## Quick Reference

| Code | Name | Typical fix |
|------|------|-------------|
| `23502` | NOT NULL violation | Provide the required value |
| `23505` | UNIQUE violation | Savepoint, check first, or upsert |
| `23514` | CHECK violation | Fix the value or the constraint |
| `23503` | FOREIGN KEY violation | Create the referenced record first |
| `40001` | Serialization failure | Retry with backoff |
| `40P01` | Deadlock detected | Retry with backoff, lock in order |
| `55P03` | Lock not available | Skip, retry later, or wait |
| `25P02` | In-failed SQL transaction | Roll back or use a savepoint |

Savepoint decision tree:

```
Need error isolation?
|- Single operation that may fail          -> with cr.savepoint(): ...
|- Batch with per-record failures          -> for row in rows: with cr.savepoint(): ...
|- Lock attempt that should fail fast      -> with cr.savepoint(flush=False): SELECT ... FOR UPDATE NOWAIT
|- DDL or schema change                    -> with cr.savepoint(flush=False): ...
|- Failed ORM operation must not pollute   -> with cr.savepoint(), env.clear_upon_failure(): ...
```

After catching a database error, ask:

- [ ] Was the failing statement inside `with cr.savepoint():`?
- [ ] If not, did I call `env.cr.rollback()` before continuing?
- [ ] Did I re-browse records after the rollback?
- [ ] Do I need `invalidate_cache()` because raw SQL changed cached values?
- [ ] Will the caller retry, or should I retry here?
- [ ] Is the operation idempotent if it runs twice?

Best practices:

1. Use `with self.env.cr.savepoint():` around anything that may raise a database error.
2. Never commit in the middle of business logic.
3. Prefer check-then-create for known unique keys, but keep a savepoint for races.
4. Use `FOR UPDATE NOWAIT` plus `savepoint(flush=False)` for contended background work.
5. Flush the fields you are about to read with raw SQL.
6. Use `invalidate_cache()` after raw SQL that changes ORM-managed columns.
7. Retry only concurrency errors, and always roll back first.
8. Keep transactions short; do not hold locks across external calls.
9. Use a separate cursor for audit logging that must survive a rollback.
10. Remember that cron does not retry automatically in 13.0.

---

## Common Patterns

Safe batch create:

```python
import psycopg2
from odoo.exceptions import ValidationError

def batch_create_safe(self, rows):
    created, failed = [], []
    for data in rows:
        with self.env.cr.savepoint():
            try:
                created.append(self.create(data))
            except (psycopg2.Error, ValidationError) as exc:
                failed.append({'data': data, 'error': str(exc)})
    return created, failed
```

Upsert by key:

```python
def upsert_by_key(self, key_field, key_value, values):
    existing = self.search([(key_field, '=', key_value)], limit=1)
    if existing:
        existing.write(values)
        return existing
    return self.create({key_field: key_value, **values})
```

For concurrent callers, wrap the create in a savepoint and re-search on a unique violation.

Lock and skip:

```python
import psycopg2
from psycopg2 import errorcodes

def process_with_lock(self, records):
    for record in records:
        try:
            with self.env.cr.savepoint(flush=False):
                self.env.cr.execute(
                    'SELECT id FROM %s WHERE id = %%s FOR UPDATE NOWAIT' % self._table,
                    (record.id,),
                )
        except psycopg2.OperationalError as exc:
            if exc.pgcode == errorcodes.LOCK_NOT_AVAILABLE:
                _logger.info("Record %s is locked, skipping", record.id)
                continue
            raise
        record._do_process()
```

Retry on serialization failure:

```python
import time
import psycopg2
from psycopg2 import errorcodes

def write_with_retry(self, vals, retries=3):
    for attempt in range(retries):
        try:
            return self.write(vals)
        except psycopg2.OperationalError as exc:
            if exc.pgcode not in (
                errorcodes.SERIALIZATION_FAILURE,
                errorcodes.DEADLOCK_DETECTED,
            ):
                raise
            if attempt == retries - 1:
                raise
            self.env.cr.rollback()
            time.sleep(0.05 * (2 ** attempt))
```

Audit log across rollback:

```python
from contextlib import closing
import odoo

def _audit(self, message):
    with closing(self.env.registry.cursor()) as cr:
        env = odoo.api.Environment(cr, self.env.uid, self.env.context)
        env['audit.log'].create({'message': message})
        cr.commit()
```

Flush before raw SQL:

```python
def get_totals(self):
    self.env['sale.order'].flush(['state', 'amount_total'])
    self.env.cr.execute("""
        SELECT state, SUM(amount_total)
          FROM sale_order
         WHERE state IN %s
         GROUP BY state
    """, (('done', 'cancel'),))
    return dict(self.env.cr.fetchall())
```

Raw SQL update with cache invalidation:

```python
def _mark_done(self, order_id):
    self.env['sale.order'].flush(['state'])
    self.env.cr.execute(
        "UPDATE sale_order SET state = 'done' WHERE id = %s", (order_id,))
    self.env['sale.order'].invalidate_cache(['state'], [order_id])
```

Chunked cron with commits:

```python
def _cron_process(self, records, chunk_size=200):
    for start in range(0, len(records), chunk_size):
        chunk = records[start:start + chunk_size]
        for record in chunk:
            with self.env.cr.savepoint():
                record._process()
        self.env.cr.commit()
```

After each commit, re-browse records if you need them again.

---

## Base Code Reference

- `odoo/sql_db.py` - `Cursor`, `TestCursor`, `Connection`, `ConnectionPool`, `flush_env`, `clear_env`, `Cursor.savepoint`, `Cursor.commit`, `Cursor.rollback`, `Cursor.after`, `Cursor.IN_MAX`, `Cursor.split_for_in_conditions`
- `odoo/api.py` - `Environment`, `Environment.clear`, `Environment.clear_upon_failure`, `Environment.reset`, `Environment.manage`, `Environment.protecting`, `Environment.norecompute`
- `odoo/models.py` - `Model.flush`, `Model.invalidate_cache`, `Model.recompute`, `Model.modified`, `Model._check_concurrency`, `Model.write`, `Model._write`, `Model.create`, `Model.load`, `PGERROR_TO_OE`, `convert_pgerror_unique`, `convert_pgerror_not_null`, `convert_pgerror_constraint`
- `odoo/fields.py` - `Field.__get__` read-time recomputation, `Field.write`, `Field.compute_value`
- `odoo/service/model.py` - `check` decorator, `PG_CONCURRENCY_ERRORS_TO_RETRY`, `MAX_TRIES_ON_CONCURRENCY_FAILURE`, `execute_cr`, `execute`
- `odoo/http.py` - `request.cr`, `request.env`, `request._failed`, `request.__exit__`, `request._handle_exception`, `request._call_function`
- `odoo/modules/registry.py` - `Registry.cursor`, `Registry.in_test_mode`, `Registry.enter_test_mode`, `Registry.leave_test_mode`
- `odoo/exceptions.py` - `UserError`, `ValidationError`, `AccessError`, `AccessDenied`, `MissingError`, `RedirectWarning`, `CacheMiss`, `DeferredException`, `QWebException`, `except_orm`
- `odoo/addons/base/models/ir_cron.py` - `_process_jobs`, `_process_job`, `_callback`, `_handle_callback_exception`, `_try_lock`, `FOR UPDATE NOWAIT`
- `odoo/addons/base/models/ir_sequence.py` - `_update_nogap`, `FOR UPDATE NOWAIT`
- `odoo/addons/base/wizard/base_partner_merge.py` - savepoint with `clear_upon_failure`
- `odoo/addons/base/models/ir_model.py` - savepoint around unlink and module data cleanup
- `odoo/tools/sql.py` - `savepoint(flush=False)` for DDL
- `addons/website/models/website_visitor.py` - `FOR NO KEY UPDATE NOWAIT`
- `odoo/addons/base/tests/test_db_cursor.py` - `TestCursor` commit and rollback behavior
- `odoo/addons/test_new_api/tests/test_new_fields.py` - savepoint with `assertRaises`
- PostgreSQL documentation: Transaction Isolation, Explicit Locking, Error Codes
