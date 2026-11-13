---
name: odoo-13-testing
description: Complete reference for Odoo 13 module testing - TransactionCase, SingleTransactionCase, SavepointCase, HttpCase, Form/O2MForm/O2MProxy/M2MProxy, @tagged/@users/@warmup, new_test_user, unittest.mock patching, assertQueryCount, browser_js/phantom_js and web_tour tours, plus --test-enable/--test-tags commands.
globs: "**/tests/**/*.py"
topics:
  - Test case types (TransactionCase, SingleTransactionCase, SavepointCase, HttpCase)
  - Test decorators (tagged, users, warmup, at_install, post_install)
  - Form testing (Form, O2MForm, O2MProxy, M2MProxy)
  - Browser testing via ChromeBrowser, browser_js, phantom_js and tours
  - Mocking and patching with unittest.mock
  - Query-count assertions
  - Tag selector syntax and test commands
when_to_use:
  - Writing module tests
  - Adding regression coverage
  - Testing UI flows or JavaScript via tours
  - Mocking external services
  - Asserting query counts to catch N+1 regressions
  - Running a single test class or method
---

# Odoo 13 Testing Guide

Comprehensive guide to testing Odoo 13 modules: base test classes, decorators, mocking, form helpers, browser testing and the commands that run them. Every API in this guide exists in the 13.0 source tree; APIs added in later versions are called out explicitly as unavailable.

## Table of Contents

1. [Test File Layout](#test-file-layout)
2. [Base Test Classes](#base-test-classes)
3. [Test Decorators](#test-decorators)
4. [Mocking and Patching](#mocking-and-patching)
5. [Form Testing](#form-testing)
6. [Browser Testing](#browser-testing)
7. [Setup and Teardown](#setup-and-teardown)
8. [Assert Helpers](#assert-helpers)
9. [Test Data Helpers](#test-data-helpers)
10. [Running Tests](#running-tests)
11. [Best Practices](#best-practices)

---

## Test File Layout

Odoo 13 imports the `tests` sub-package of each installed addon and collects every submodule whose name starts with `test_` (`odoo/modules/module.py`). Therefore `tests/__init__.py` must import every test module explicitly, and a helper module is only collected when a `test_*.py` module imports it. Test modules are imported, not executed, at discovery time.

```
my_module/
    __manifest__.py
    models/
        business_trip.py
    tests/
        __init__.py
        test_business_trip.py
        test_ui.py
```

```python
# tests/__init__.py
from . import test_business_trip
from . import test_ui
```

```python
# tests/test_business_trip.py
from odoo.tests.common import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestBusinessTrip(TransactionCase):
    def test_default_state(self):
        trip = self.env['business.trip'].create({'name': 'Kickoff'})
        self.assertEqual(trip.state, 'draft')
```

### Where the test infrastructure lives in 13.0

| File | Contents |
|------|----------|
| `odoo/tests/__init__.py` | `from .common import *` - the only test module in 13.0 |
| `odoo/tests/common.py` | `BaseCase`, `TransactionCase`, `SingleTransactionCase`, `SavepointCase`, `HttpCase`, `ChromeBrowser`, `Form`, `O2MForm`, `O2MProxy`, `M2MProxy`, `tagged`, `users`, `warmup`, `new_test_user`, `TagsSelector` |
| `odoo/modules/module.py` | Test discovery (`get_test_modules`, `run_unit_tests`) and `OdooTestRunner` |
| `odoo/tools/config.py` | CLI options `--test-enable`, `--test-tags`, `--test-file`, `--stop-after-init` |

There is no `odoo/tests/form.py`, `odoo/tests/case.py`, `odoo/tests/loader.py` or `odoo/tests/tag_selector.py` in 13.0. `Form` and the proxies are defined directly in `odoo/tests/common.py` and re-exported by `odoo.tests`. Both import styles work:

```python
from odoo.tests import TransactionCase, tagged, Form
from odoo.tests.common import TransactionCase, tagged, Form
```

---

## Base Test Classes

### Class Hierarchy

`BaseCase` is the root; `TransactionCase` adds per-method isolation, `HttpCase` adds HTTP and browser helpers, `SingleTransactionCase` shares one transaction per class, and `SavepointCase` adds a savepoint per method.

`BaseCase` is abstract: it expects `self.registry`, `self.cr` and `self.env` to be initialized by a subclass. It provides `ref`, `browse_ref`, `assertRaises`, `assertQueryCount`, `assertRecordValues`, `assertTreesEqual` and `assertItemsEqual`.

### TransactionCase

Each test method runs in its own transaction with its own cursor. The transaction is rolled back and the cursor closed after each method. This is the default choice for model and business-logic tests.

```python
from odoo.tests.common import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestBusinessTrip(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Alice'})
        cls.trip = cls.env['business.trip'].create({
            'name': 'Kickoff',
            'partner_id': cls.partner.id,
        })

    def test_01_default_state(self):
        self.assertEqual(self.trip.state, 'draft')

    def test_02_confirm(self):
        self.trip.action_confirm()
        self.assertEqual(self.trip.state, 'confirmed')

    def test_03_isolated(self):
        # test_02's write was rolled back with its transaction
        self.assertEqual(self.trip.state, 'draft')
```

`setUpClass` runs once and its data is visible to every method. Each method gets a fresh cursor and environment, so writes do not leak between methods. `TransactionCase.setUp` creates `self.env` with `odoo.SUPERUSER_ID`, registers cleanups for the registry, cursor and environment, and patches `res.partner._get_gravatar_image` to avoid network access.

`TransactionCase` adds two helpers that `BaseCase` does not have:

```python
def test_patch_method(self):
    self.patch(type(self.env['business.trip']), 'compute_amount', lambda self: 42.0)
    self.assertEqual(self.env['business.trip'].compute_amount(), 42.0)

def test_patch_order(self):
    self.patch_order('business.trip', 'name desc')
    self.assertEqual(self.env['business.trip']._order, 'name desc')
```

`patch(obj, key, val)` performs `setattr` and registers the restore with `addCleanup`. `patch_order(model, order)` patches the model's `_order`.

### SingleTransactionCase

All test methods share one transaction, which is rolled back at class teardown. Tests see each other's writes, so method order matters.

```python
from odoo.tests.common import SingleTransactionCase


class TestStatefulFlow(SingleTransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.order = cls.env['sale.order'].create({
            'partner_id': cls.env.ref('base.res_partner_1').id,
        })

    def test_01_confirm(self):
        self.order.action_confirm()
        self.assertEqual(self.order.state, 'sale')

    def test_02_invoice(self):
        # test_01's change is still visible
        self.assertEqual(self.order.state, 'sale')
```

Use it when the class needs one shared, mutable integration state and per-method isolation is not wanted. `SingleTransactionCase` does not provide `self.patch` or `self.patch_order`; those are defined on `TransactionCase` only.

### SavepointCase

`SavepointCase` is present in 13.0 and is the recommended base when `setUpClass` builds expensive data that every test should see but no test should mutate for the others. All methods share one transaction, and each method runs inside a savepoint that is rolled back afterwards.

```python
from odoo.tests.common import SavepointCase


class TestHeavySetup(SavepointCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Bob'})

    def test_01_rename(self):
        self.partner.name = 'Robert'
        self.assertEqual(self.partner.name, 'Robert')

    def test_02_original_name(self):
        # test_01 was rolled back to the class savepoint
        self.assertEqual(self.partner.name, 'Bob')
```

`SavepointCase` extends `SingleTransactionCase`, so it also has no `self.patch`. In 14.0 `SavepointCase` was deprecated in favour of `TransactionCase` savepoint semantics; that deprecation does not apply to 13.0, where `SavepointCase` is a normal, supported class.

### HttpCase

`HttpCase` extends `TransactionCase` with an HTTP session, `url_open`, `authenticate`, `logout`, and a headless Chrome driver used by `browser_js` and `start_tour`. Tag it `post_install` so the database and assets are fully set up.

```python
from odoo.tests.common import HttpCase, tagged


@tagged('post_install', '-at_install')
class TestUI(HttpCase):
    def test_admin_tour(self):
        self.start_tour('/web', 'my_module_tour', login='admin')
```

Class attributes:

| Attribute | Default | Description |
|-----------|---------|-------------|
| `registry_test_mode` | `True` | Enter registry test mode for the duration of the test |
| `browser` | `None` | Lazily created `ChromeBrowser` instance |
| `browser_size` | `'1366x768'` | Chrome window size, for example `'375x667'` for mobile |

`HttpCase` has no `touch_enabled` and no `allow_inherited_tests_method` attribute in 13.0; those were added later.

---

## Test Decorators

### `@tagged`

Tags are stored on the test class in `test_tags`. Every class defined under `odoo.addons.*` starts with `{'standard', 'at_install'}` plus its module name, assigned by `MetaCase`. Prefix a tag with `-` to remove it.

```python
from odoo.tests.common import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestAfterInstall(TransactionCase):
    ...


@tagged('-standard', 'nightly')
class TestNightly(TransactionCase):
    """Only runs when --test-tags=nightly is passed."""
```

Important 13.0 details:

- `@tagged` is a class decorator. Applying it to a method sets an attribute on the function, but `TagsSelector` reads `test_tags` from the test instance, so method-level tagging has no effect.
- Tags are not inherited: a subclass of a tagged class starts again from `{'standard', 'at_install'}` unless it is decorated itself.
- The module name is also matched as a tag for backward compatibility, but the documented form is `/module`.

| Tag | Meaning |
|-----|---------|
| `standard` | Default - included unless excluded |
| `at_install` | Run during module installation (default) |
| `post_install` | Run after the installation set completes |
| `-at_install` | Remove the default `at_install` tag |
| `-standard` | Skip unless explicitly requested |

### `@users`

Run a test method once for each listed login. The login must already exist. Inside the method, `self.env.user` and `self.uid` reflect the current user. The cache is invalidated between subtests.

```python
from odoo.tests.common import TransactionCase, users


class TestAccess(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.env['res.users'].create({
            'name': 'Alice',
            'login': 'alice',
            'password': 'alicexxx',
            'groups_id': [(6, 0, [cls.env.ref('base.group_user').id])],
        })

    @users('admin', 'alice')
    def test_can_read(self):
        self.assertTrue(self.env['business.trip'].search([]))
```

`@users` uses `self.subTest(login=login)` for each login, so a failure identifies the user. The special login `__system__` is used by core performance tests for the superuser.

### `@warmup`

Run the test twice: once to warm caches, then again for real. The warm-up run is rolled back with a savepoint. `self.warm` is `False` during warm-up and `True` during the real run. `assertQueryCount` only asserts when `self.warm` is true.

```python
from odoo.tests.common import TransactionCase, warmup


class TestPerf(TransactionCase):
    @warmup
    def test_read_list(self):
        with self.assertQueryCount(3):
            self.env['business.trip'].search([]).mapped('name')
```

### `@at_install` and `@post_install`

Both still exist in 13.0 but are deprecated since 12.0. They are thin wrappers around `tagged`:

```python
from odoo.tests.common import at_install, post_install

@at_install(False)
@post_install(True)
class TestLegacy(TransactionCase):
    ...
```

Prefer the equivalent `@tagged('post_install', '-at_install')`.

### `@mute_logger`

`mute_logger` lives in `odoo.tools` (re-exported from `odoo.tools.misc`). It works as a decorator or a context manager and silences the named loggers.

```python
from odoo.tools import mute_logger


class TestThings(TransactionCase):
    @mute_logger('odoo.sql_db', 'odoo.http')
    def test_creates_mail(self):
        ...
```

### Decorators that do not exist in 13.0

`@standalone` and `@no_retry` are not available in 13.0. They were added in later versions. Do not use them in 13.0 tests.

---

## Mocking and Patching

### `self.patch` on TransactionCase

`TransactionCase.patch(obj, key, val)` sets an attribute and registers its restoration with `addCleanup`, so the patch is unwound even if the test fails.

```python
def test_patch_method(self):
    def replacement(self):
        return 'mocked'

    self.patch(type(self.env['business.trip']), 'compute_amount', replacement)
    self.assertEqual(self.env['business.trip'].compute_amount(), 'mocked')
```

`self.patch` is defined on `TransactionCase` only. `SingleTransactionCase` and `SavepointCase` do not have it; use `unittest.mock.patch` with `addCleanup` there.

### `unittest.mock.patch`

For external services, patch the name where it is looked up. Register `patch.stopall` with `addCleanup` so every started patch is stopped at teardown.

```python
from unittest.mock import patch


def test_external_call(self):
    self.addCleanup(patch.stopall)
    mock_post = patch('requests.post').start()
    mock_post.return_value.status_code = 200
    mock_post.return_value.json.return_value = {'ok': True}

    result = self.env['my.model'].call_external()

    mock_post.assert_called_once()
    self.assertTrue(result)
```

A context manager is simpler when the patch is local to one block:

```python
from unittest.mock import patch, mock_open


def test_read_config(self):
    with patch('builtins.open', mock_open(read_data='KEY=VALUE')):
        data = self.env['my.model']._read_config('/etc/nope')
        self.assertEqual(data, {'KEY': 'VALUE'})
```

### Helpers that do not exist in 13.0

`self.startPatcher(patcher)` and `cls.classPatch(...)` are not available in 13.0. They were introduced in later versions. Use `patch(...).start()` plus `addCleanup(patch.stopall)`, or `self.patch` on a `TransactionCase`.

---

## Form Testing

`Form` simulates the client-side form view entirely server-side. It calls `default_get`, runs onchanges, evaluates `attrs`/modifiers, and manages x2many commands. It is defined in `odoo/tests/common.py` in 13.0.

### Create Mode

Pass an empty recordset. `save()` returns the created record.

```python
from odoo.tests.common import Form


def test_create_via_form(self):
    form = Form(self.env['business.trip'])
    form.name = 'Berlin Workshop'
    form.partner_id = self.partner
    trip = form.save()
    self.assertEqual(trip.name, 'Berlin Workshop')
```

### Edit Mode

Pass a singleton recordset. Using the form as a context manager saves it on exit.

```python
def test_edit_via_form(self):
    with Form(self.trip) as form:
        form.name = 'Updated'
    self.assertEqual(self.trip.name, 'Updated')
```

`Form.save()` returns the record. In 13.0 there is no `form.record` attribute; keep the return value of `save()` or the record you passed in.

### What the Form helper enforces

- Assigning a field triggers the relevant onchange methods.
- `readonly` fields cannot be assigned, and required fields are checked on save; both raise `AssertionError`.
- Invisible and `column_invisible` fields are skipped by the required check.
- One2many and Many2many fields cannot be assigned directly; use their proxies.
- Many2one fields must be assigned a singleton recordset of the right model.

### One2many with `O2MProxy`

```python
def test_one2many(self):
    with Form(self.env['business.trip']) as form:
        form.name = 'Trip A'

        with form.expense_ids.new() as line:
            line.description = 'Taxi'
            line.amount = 25.0

        with form.expense_ids.new() as line:
            line.description = 'Hotel'
            line.amount = 250.0

        with form.expense_ids.edit(0) as line:
            line.amount = 30.0

        form.expense_ids.remove(index=1)

    trip = form.save()
    self.assertEqual(len(trip.expense_ids), 1)
    self.assertEqual(trip.expense_ids.amount, 30.0)
```

`O2MProxy` supports only commands 0 (create), 1 (update) and 2 (delete). `new()` and `edit(index)` return an `O2MForm`; `remove(index)` removes by position. The proxy is not editable when the field is `readonly`.

### Many2many with `M2MProxy`

`M2MProxy` behaves as a sequence of recordsets and supports `add`, `remove` and `clear`.

```python
with Form(user) as form:
    form.groups_id.add(self.env.ref('base.group_user'))
    form.groups_id.remove(id=self.env.ref('base.group_portal').id)
    form.groups_id.clear()
```

`remove` takes exactly one of `id=` or `index=`. In 13.0 there is no `M2MProxy.set(...)` method; to replace the whole selection, call `clear()` and then `add()` for each record.

### Selecting a Specific View

`view` accepts a view id, an XML ID, or an `ir.ui.view` record. If omitted, the model's default form view is used.

```python
with Form(self.env['sale.order'], view='sale.view_order_form') as form:
    form.partner_id = self.partner
```

### `O2MForm`

`O2MForm` is the public class returned by `O2MProxy.new()` and `O2MProxy.edit(index)`. It is a `Form` subclass that knows its parent proxy and index, merges parent values into modifier evaluation, and writes its values back into the parent's x2many command list on save. You normally use it through the proxy rather than instantiating it directly.

---

## Browser Testing

### `url_open`

```python
def url_open(self, url, data=None, files=None, timeout=10, headers=None):
    ...
```

Issue an HTTP request against the test server. A relative path is prefixed with `http://<host>:<http_port>`. If `data` or `files` is given, the request is a POST; otherwise it is a GET. The method returns a `requests.Response`.

```python
def test_controller(self):
    response = self.url_open('/my_module/api/ping')
    self.assertEqual(response.status_code, 200)
```

13.0's `url_open` has no `method=` and no `allow_redirects=` parameter. Use `data=`/`files=` to select POST, and inspect `response.history` or `response.url` for redirects.

### `authenticate` and `logout`

```python
def test_as_admin(self):
    self.authenticate('admin', 'admin')
    response = self.url_open('/web/session/get_session_info')
    self.assertEqual(response.status_code, 200)
```

`authenticate(user, password)` authenticates against the registry, updates the test session and, if a browser is running, sets its `session_id` cookie. `authenticate(None, None)` deletes the session; in 13.0 that path also touches `self.browser`, so only use it after the browser has been started. `logout(keep_db=True)` logs the test session out and saves it.

### `browser_js`

```python
def browser_js(self, url_path, code, ready='', login=None, timeout=60, **kw):
    ...
```

Start headless Chrome, optionally log in, navigate to `url_path`, wait for `ready` to evaluate truthy, then evaluate `code`. The test succeeds when the page logs `test successful`; a JavaScript error or a timeout fails the test.

```python
def test_some_js(self):
    self.browser_js(
        url_path='/web',
        code="console.log('test successful');",
        ready="odoo.isReady === true",
        login='admin',
    )
```

Details:

- If `ready` is empty, it defaults to `document.readyState === 'complete'`.
- `login` is used as both login and password, matching the demo data convention.
- `browser_js` sets the `web.base.url` system parameter to the test server URL and flushes it before navigating.
- On failure, Chrome screenshots are written under the `--screenshots` directory; `--screencasts` records a screencast.

### `phantom_js` (legacy alias)

In 13.0, `phantom_js` is a direct alias of `browser_js`:

```python
phantom_js = browser_js
```

Both names are valid. New tests should use `browser_js`; `phantom_js` remains for older addons and for the QUnit runner in `addons/web/tests/test_js.py`.

### `start_tour`

```python
def start_tour(self, url_path, tour_name, step_delay=None, **kwargs):
    ...
```

A wrapper around `browser_js` that starts a registered tour. The default code is `odoo.startTour('<tour_name>')`; the default ready expression is `odoo.__DEBUG__.services['web_tour.tour'].tours.<tour_name>.ready`. After the tour, the environment cache is invalidated so the test sees database changes made by the browser.

```python
def test_checkout_tour(self):
    self.start_tour('/shop', 'checkout_tour', login='demo', timeout=120)
```

### Defining a tour

Tours are registered with the `web_tour.tour` service. The `test: true` flag makes the tour available to the test runner.

```javascript
// static/tests/tours/my_tour.js
odoo.define('my_module.tour', function (require) {
"use strict";

var tour = require('web_tour.tour');

tour.register('my_module_tour', {
    test: true,
    url: '/web',
}, [{
    trigger: '.o_app[data-menu-xmlid="my_module.menu_root"]',
    content: 'Open the application',
    position: 'bottom',
}, {
    trigger: 'button[name="action_confirm"]',
    content: 'Confirm the record',
    position: 'right',
}]);

});
```

Tour steps support `trigger`, `extra_trigger`, `content`, `position`, `run`, `edition` and the predefined steps in `tour.STEPS` such as `tour.STEPS.SHOW_APPS_MENU_ITEM`.

### Loading tour assets

Tour files are loaded through the `web.assets_tests` bundle. In 13.0 assets are declared in XML, not in the manifest:

```xml
<!-- views/assets.xml -->
<odoo>
    <template id="assets_tests" inherit_id="web.assets_tests">
        <xpath expr="." position="inside">
            <script type="text/javascript" src="/my_module/static/tests/tours/my_tour.js"></script>
        </xpath>
    </template>
</odoo>
```

Add `views/assets.xml` to the manifest's `data` list and depend on `web_tour` (and `web` for the client itself).

---

## Setup and Teardown

### `setUpClass(cls)`

Runs once before any test method. Build shared fixtures with `cls.env`.

```python
@classmethod
def setUpClass(cls):
    super().setUpClass()
    cls.manager = cls.env['res.users'].create({
        'name': 'Manager',
        'login': 'mgr',
        'password': 'mgrxxxxx',
        'groups_id': [(6, 0, [cls.env.ref('base.group_user').id])],
    })
    cls.trip = cls.env['business.trip'].create({'name': 'Demo'})
```

### `setUp(self)`

Runs before each test method. Register cleanup with `self.addCleanup`.

```python
def setUp(self):
    super().setUp()
    self.addCleanup(self._cleanup_files)
```

### `addCleanup` and `addClassCleanup`

Register a callable to run at teardown regardless of the test outcome.

```python
import shutil
import tempfile

temp = tempfile.mkdtemp()
self.addCleanup(shutil.rmtree, temp)
```

`addClassCleanup` is available on all supported Python versions. On Python older than 3.8, `TreeCase` provides a backport so the API is consistent.

`HttpCase` additionally enters registry test mode in `setUp` and leaves it in a cleanup, and creates a magic `session_id` that is rolled back with the test.

---

## Assert Helpers

### `assertQueryCount(default=0, flush=True, **counters)`

Context manager that asserts the number of SQL queries executed. It fails when the real count is higher than expected and logs an info message when it is lower, so you can tighten the assertion.

```python
@warmup
def test_read_cost(self):
    trips = self.env['business.trip'].search([])
    with self.assertQueryCount(3):
        trips.mapped('partner_id.name')
```

Named counters pair with `@users`:

```python
@users('admin', 'demo')
@warmup
def test_multi_user_cost(self):
    with self.assertQueryCount(admin=4, demo=6):
        self.env['business.trip'].search([]).read(['name', 'state'])
```

The counter is `self.cr.sql_log_count`. When `self.warm` is `False` (during `@warmup`), the assertion is skipped and the block only flushes.

### `assertRecordValues(records, expected_values)`

Compare a recordset to a list of expected dictionaries, element by element by index. The order of the expected list matters.

```python
self.assertRecordValues(trips, [
    {'name': 'Trip A', 'state': 'draft'},
    {'name': 'Trip B', 'state': 'confirmed'},
])
```

Comparison rules:

- Falsy values match each other (`False` matches `None`).
- Monetary fields are compared with the currency's rounding; float fields with `digits` are compared at that precision.
- One2many and Many2many fields are compared as sets of ids; an empty relation must be written as `[]`.
- Many2one fields are compared by id; any falsy expected value matches an empty relation.

### `assertRaises`

`BaseCase.assertRaises` wraps `unittest`'s and clears the environment when the exception is raised, so a failed transaction does not poison later assertions.

```python
from odoo.exceptions import AccessError, ValidationError


def test_access(self):
    with self.assertRaises(AccessError):
        self.env['business.trip'].with_user(self.portal_user).search([])


def test_validation(self):
    with self.assertRaises(ValidationError):
        self.env['business.trip'].create({'name': False})
```

The callable form is also supported: `self.assertRaises(ValidationError, self.trip.write, {'name': False})`.

### `assertTreesEqual`

`TreeCase` registers `assertTreesEqual` as the equality function for `lxml` elements and `html.HtmlElement`, so `assertEqual` on two parsed trees compares tag, attributes, text, tail and children recursively.

### Assert helpers that do not exist in 13.0

`assertQueries`, `assertXMLEqual`, `assertHTMLEqual` and `assertURLEqual` are not available in 13.0. Use `assertEqual` with `assertTreesEqual` for XML/HTML trees, and compare URLs with plain string assertions.

---

## Test Data Helpers

### `new_test_user`

```python
def new_test_user(env, login='', groups='base.group_user', context=None, **kwargs):
    ...
```

Create a user with sensible defaults. `groups` is a comma-separated list of XML IDs. The helper fills `groups_id`, a default `name` of `"login (groups)"`, and an `email` derived from the login. Extra keyword arguments are passed to `create`. For example, `new_test_user(self.env, login='trip_mgr', groups='base.group_user,my_module.group_trip_manager')`.

`login` and `groups` are required; an empty value raises `ValueError`.

### `ref` and `browse_ref`

`self.ref(xid)` returns the database id for a fully qualified XML ID. `self.browse_ref(xid)` returns the recordset. Both require the `module.identifier` form and raise `ValueError` when the XML ID is missing. `self.env.ref(xid)` is the environment-level equivalent and is the usual choice outside `BaseCase` helpers.

### Helpers that do not exist in 13.0

`RecordCapturer` and `loaded_demo_data` are not available in 13.0. To capture created records, search for them after the operation; to detect demo data, check the database or skip the test explicitly.

---

## Running Tests

### Basic commands

```bash
# Install the module and run its at_install tests
odoo-bin -d test_db -i my_module --test-enable --stop-after-init

# Update the module and re-run its tests
odoo-bin -d test_db -u my_module --test-enable --stop-after-init

# Run only post_install tests of my_module
odoo-bin -d test_db -u my_module --test-tags=post_install --stop-after-init
```

`--test-enable` sets the test tag filter to `+standard` when no `--test-tags` is given. `--test-tags` enables tests by itself, so `--test-enable` is optional when a tag selector is present.

### `--test-tags` syntax

The selector is a comma-separated list of specs with the form `[-][tag][/module][:class][.method]`.

| Spec | Meaning |
|------|---------|
| `post_install` | Include tests tagged `post_install` |
| `-slow` | Exclude tests tagged `slow` |
| `/my_module` | Include `standard` tests of `my_module` |
| `/my_module:TestBusinessTrip` | Include a class |
| `/my_module:TestBusinessTrip.test_02_confirm` | Include a method |
| `*` | Match all tags |
| `-standard,my_tag` | Exclude `standard`, include `my_tag` |

Examples:

```bash
# Only the post_install tests of the module
odoo-bin -d test_db -u my_module --test-tags=/my_module:post_install --stop-after-init

# One class
odoo-bin -d test_db -u my_module --test-tags=/my_module:TestBusinessTrip --stop-after-init

# One method
odoo-bin -d test_db -u my_module --test-tags=/my_module:TestBusinessTrip.test_02_confirm --stop-after-init

# Everything, including tests that removed the standard tag
odoo-bin -d test_db -u my_module --test-tags=* --stop-after-init
```

When an include spec has no tag, `standard` is implied. When only exclusions are given, `standard` is included automatically. The module name is still accepted as a tag for backward compatibility, but `/module` is the documented form.

### Logging

`--log-level=test` shows each test class and method as it runs; `--log-handler=odoo.tests.common:DEBUG` shows query counts and browser events. `--screenshots` and `--screencasts` keep artifacts when a tour fails.

---

## Best Practices

1. **Default to `TransactionCase`.** Use `SavepointCase` when `setUpClass` is expensive and tests must not see each other's writes, `SingleTransactionCase` only for genuinely stateful flows, and `HttpCase` only for HTTP and browser tests.

2. **Tag `HttpCase` with `post_install`.** Browser tests need installed assets and a fully initialized database:

   ```python
   @tagged('post_install', '-at_install')
   class TestUI(HttpCase):
       ...
   ```

3. **Keep `setUpClass` fast and deterministic.** Create the minimum fixtures there and do not depend on demo data unless the test checks for it.

4. **Use `Form` to exercise the real UI flow.** Onchanges, defaults and modifiers are tested for free; a raw `create()` can miss a broken onchange or a required-field regression.

5. **Pair `assertQueryCount` with `@warmup`.** Counts are unstable on cold caches; the warm-up run flushes and invalidates, then rolls back, so the real run reflects steady-state cost.

6. **Prefer `self.patch` on `TransactionCase`.** It auto-unwinds on cleanup. On `SavepointCase` and `SingleTransactionCase`, use `patch(...).start()` with `self.addCleanup(patch.stopall)`.

7. **Test error paths.** Use `assertRaises` with `AccessError` and `ValidationError` to cover permission and constraint failures.

8. **Minimal template.**

```python
from odoo.tests.common import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestBusinessTrip(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Alice'})
        cls.trip = cls.env['business.trip'].create({
            'name': 'Demo',
            'partner_id': cls.partner.id,
        })

    def test_01_confirm(self):
        self.trip.action_confirm()
        self.assertEqual(self.trip.state, 'confirmed')

    def test_02_cancel_requires_manager(self):
        from odoo.exceptions import AccessError
        with self.assertRaises(AccessError):
            self.trip.with_user(self.env.ref('base.user_demo')).action_cancel()
```

---

## Base Code Reference

The guide is based on the Odoo 13.0 source tree. Reference files:

| File | Contents |
|------|----------|
| `odoo/tests/__init__.py` | Re-exports everything from `odoo.tests.common` |
| `odoo/tests/common.py` | `BaseCase`, `TransactionCase`, `SingleTransactionCase`, `SavepointCase`, `HttpCase`, `ChromeBrowser`, `Form`, `O2MForm`, `O2MProxy`, `M2MProxy`, `tagged`, `users`, `warmup`, `new_test_user`, `TagsSelector` |
| `odoo/modules/module.py` | `get_test_modules`, `run_unit_tests`, `OdooTestRunner`, test discovery |
| `odoo/tools/config.py` | `--test-enable`, `--test-tags`, `--test-file`, `--stop-after-init`, `--screenshots`, `--screencasts`, `--log-level` |
| `odoo/tools/misc.py` | `mute_logger` |
| `odoo/addons/base/tests/test_basecase.py` | `ref`, `browse_ref`, transaction isolation examples |
| `odoo/addons/base/tests/test_tests_tags.py` | Default tags, tag inheritance and `TagsSelector` behavior |
| `odoo/addons/base/tests/test_form_create.py` | `Form` creation examples |
| `odoo/addons/test_testing_utilities/tests/test_form_impl.py` | `Form`, `O2MProxy`, `M2MProxy`, modifiers and onchange behavior |
| `odoo/addons/test_performance/tests/test_performance.py` | `@users`, `@warmup`, `assertQueryCount` |
| `odoo/addons/test_main_flows/tests/test_flow.py` | `HttpCase.start_tour` |
| `odoo/addons/test_main_flows/static/tests/tours/main_flow.js` | `tour.register` and `tour.STEPS` |
| `odoo/addons/test_main_flows/views/assets.xml` | `web.assets_tests` XML registration |

**For more Odoo 13 guides, see [SKILL.md](../SKILL.md)**
