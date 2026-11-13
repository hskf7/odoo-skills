---
name: odoo-13-javascript
description: Complete reference for Odoo 13 legacy JavaScript: AMD modules, web.Widget lifecycle, field widgets, views, client actions, services, QWeb templates, assets, debugging, and QUnit tests.
globs: "**/static/src/**/*.{js,xml,scss}"
topics:
  - AMD modules with odoo.define and require
  - web.Class, web.mixins, and inheritance
  - web.Widget lifecycle and DOM helpers
  - Field widgets and web.field_registry
  - Views, controllers, and web.view_registry
  - Client actions and web.core.action_registry
  - Services, web.AbstractService, and service mixins
  - web.rpc, web.session, web.dom, web.utils
  - QWeb JavaScript templates in static/src/xml
  - Asset bundles declared in XML
  - Debugging and QUnit testing
when_to_use:
  - Writing or extending legacy Odoo 13 web client JavaScript
  - Creating custom field widgets or view widgets
  - Registering client actions or services
  - Adding QWeb templates and asset files to a module
  - Debugging or testing frontend behavior in Odoo 13
---

# Odoo 13 JavaScript Guide

Complete reference for the Odoo 13 legacy web client. Odoo 13 has **no OWL**:
the frontend is built on jQuery, Underscore, QWeb, and the `web.Widget` class
hierarchy. All code is loaded through the AMD-like `odoo.define` / `require`
system. Anything introduced in Odoo 14.0 or later is called out explicitly as
unavailable.

## Table of Contents

1. [Module System](#module-system)
2. [web.Class and Mixins](#webclass-and-mixins)
3. [web.Widget Lifecycle](#webwidget-lifecycle)
4. [Widget DOM and RPC Helpers](#widget-dom-and-rpc-helpers)
5. [Field Widgets](#field-widgets)
6. [Views and Controllers](#views-and-controllers)
7. [Client Actions](#client-actions)
8. [Services](#services)
9. [Core Modules](#core-modules)
10. [QWeb JavaScript Templates](#qweb-javascript-templates)
11. [Assets and Manifest](#assets-and-manifest)
12. [Debugging](#debugging)
13. [Testing with QUnit](#testing-with-qunit)
14. [Common Patterns and Pitfalls](#common-patterns-and-pitfalls)
15. [What Does Not Exist in 13.0](#what-does-not-exist-in-130)
16. [Base Code Reference](#base-code-reference)

---

## Module System

Every JavaScript file is an AMD module declared with `odoo.define`. The first
argument is the module name; the second is a factory that receives `require`.

```javascript
odoo.define('my_module.MyWidget', function (require) {
"use strict";
var Widget = require('web.Widget');
var MyWidget = Widget.extend({ template: 'my_module.MyWidget' });
return MyWidget;
});
```

- Prefix module names with the addon name to keep them unique.
- Dependencies use `require('web.Widget')`, not ES `import`.
- The factory must return the module value. Side-effect modules (for example,
  a field registration) may return nothing.
- `odoo.define` also accepts an explicit dependency array:
  `odoo.define('my_module.registry', ['web.field_registry'], function (require) {...})`.
- There is no `/** @odoo-module **/` marker and no `@web/...` import path in
  13.0; those belong to the 14.0+ module system.

Modules are evaluated as the `web.assets_common` and `web.assets_backend`
bundles load. Circular dependencies are why core uses registries instead of
direct imports.

---

## web.Class and Mixins

`web.Class` is Odoo's small inheritance helper. It is not an ES class.

```javascript
var Class = require('web.Class');
var Animal = Class.extend({
    init: function (name) { this.name = name; },
    speak: function () { return this.name + ' makes a sound'; },
});
var Dog = Animal.extend({
    speak: function () { return this._super() + ' (woof)'; },
});
```

- `init` is the constructor.
- `this._super(...)` calls the overridden method; `include(properties)` adds
  methods to an existing class prototype.

`web.mixins` exports three mixins:

| Mixin | Purpose |
| --- | --- |
| `ParentedMixin` | parent/children ownership and destruction |
| `EventDispatcherMixin` | `on`, `off`, `trigger`, `trigger_up` |
| `PropertiesMixin` | `get`/`set` attribute map with change events |

`PropertiesMixin` gives widgets `this.get('key')` and `this.set('key', value)`.
`EventDispatcherMixin` provides `this.on`, `this.off`, `this.trigger`, and
`this.trigger_up`. `trigger_up` bubbles an event to the parent widget, where it
is matched against `custom_events`.

---

## web.Widget Lifecycle

`web.Widget` is the base class for all visual components. It handles QWeb
rendering, DOM insertion, event delegation, and parent/child destruction.

```javascript
odoo.define('my_module.Counter', function (require) {
"use strict";

var Widget = require('web.Widget');

var Counter = Widget.extend({
    template: 'my_module.Counter',
    events: { 'click .o_counter_increment': '_onIncrement' },

    init: function (parent, initialValue) {
        this._super(parent);
        this.value = initialValue || 0;
    },
    willStart: function () {
        return this._super().then(function () { /* async work */ });
    },
    start: function () {
        return this._super();  // this.$el and this.$() are available
    },
    destroy: function () {
        this._super();
    },
    _onIncrement: function () {
        this.value += 1;
        this.$('.o_counter_value').text(this.value);
    },
});

return Counter;

});
```

### Lifecycle order

1. `init(parent, ...)` - constructor. Call `this._super(parent)` first. The
   parent is registered so that destroying the parent destroys this widget.
2. `willStart()` - asynchronous preparation. Return a Promise. The default
   implementation loads `xmlDependencies`, `jsLibs`, `cssLibs`, and
   `assetLibs`.
3. `renderElement()` - renders `this.template` with QWeb and sets
   `this.$el` / `this.el`.
4. `start()` - called after the widget is in the DOM. Bind events, make RPCs,
   return a Promise if needed.
5. `destroy()` - removes the element and destroys children.

`appendTo(target)` performs `willStart` -> render -> insert -> `start`:

```javascript
var widget = new Counter(this, 3);
widget.appendTo(this.$('.o_counter_container'));
```

Insertion helpers: `appendTo`, `prependTo`, `insertAfter`, `insertBefore`,
`replace`, and `attachTo` (use an existing element as `this.$el`).

Widget properties: `template`, `tagName`, `className`, `attributes`, `events`,
`custom_events`, `xmlDependencies`, `$el`, and `el`. `events` keys use jQuery
delegation syntax (`'click .o_button': '_onClick'`); `custom_events` maps
`trigger_up` names to handlers, which receive the Odoo event object with its
payload in `event.data`.

---

## Widget DOM and RPC Helpers

### this.$

`this.$(selector)` is `this.$el.find(selector)`. With no argument it returns
`this.$el`.

```javascript
this.$('.o_field').addClass('o_dirty');
```

### this._rpc

`this._rpc(params, options)` builds and executes an RPC. It comes from
`web.ServicesMixin`, which `web.Widget` inherits.

```javascript
this._rpc({
    model: 'res.partner',
    method: 'search_read',
    args: [[['is_company', '=', true]], ['name']],
    kwargs: { limit: 10 },
}).then(function (partners) {
    // ...
});

this._rpc({
    route: '/my_module/endpoint',
    params: { id: 42 },
}).then(function (result) {
    // ...
});
```

The returned promise has an `abort()` method. Prefer `this._rpc` over the
global `web.rpc` inside a widget: the ajax service ignores the result once the
calling widget is destroyed.

### this.do_action

`this.do_action(action, options)` asks the action manager to execute an action
and returns a Promise.

```javascript
this.do_action({
    type: 'ir.actions.act_window',
    res_model: 'res.partner',
    views: [[false, 'list'], [false, 'form']],
    domain: [['is_company', '=', true]],
    target: 'current',
});
```

### this.trigger_up

`this.trigger_up(name, info)` dispatches a custom event to ancestor widgets.
The nearest ancestor that declares the event in `custom_events` handles it.

```javascript
this.trigger_up('field_changed', {
    dataPointID: this.dataPointID,
    changes: changes,
});
```

### this.call

`this.call(serviceName, methodName, ...args)` invokes a registered service
method synchronously through the service provider.

```javascript
this.call('notification', 'notify', {
    title: 'Saved',
    message: 'The record was saved.',
    type: 'success',
});
```

Because the call travels through `trigger_up('call_service', ...)`, the widget
must be attached to a service provider (normally the web client).

`this.do_notify(title, message, sticky, className)` is a convenience wrapper
around the notification service.

---

## Field Widgets

Field widgets render a single field inside a view. They extend
`web.AbstractField`, which itself extends `web.Widget`.

```javascript
odoo.define('my_module.FieldColorSwatch', function (require) {
"use strict";

var AbstractField = require('web.AbstractField');
var field_registry = require('web.field_registry');

var FieldColorSwatch = AbstractField.extend({
    supportedFieldTypes: ['char'],
    _renderReadonly: function () {
        this.$el.text(this.value || '');
        this.$el.css('background-color', this.value || 'transparent');
    },
    _renderEdit: function () {
        this.$el.empty();
        this.$el.append($('<input>', { type: 'text', val: this.value || '' }));
    },
});

field_registry.add('color_swatch', FieldColorSwatch);

return FieldColorSwatch;

});
```

Constructor: `init(parent, name, record, options)`.

Properties: `this.name`, `this.record`, `this.field`, `this.value`,
`this.recordData`, `this.string`, `this.nodeOptions` / `this.attrs`,
`this.viewType`, `this.mode`, `this.dataPointID` / `this.res_id`, and
`this.model`.

Methods to override: `_render`, `_renderEdit`, `_renderReadonly`, `_setValue`,
`commitChanges`, `isValid`, and `_reset`.

`_setValue` triggers `field_changed` upward and returns a Promise. It does not
re-render the widget; the widget must update its own DOM.

```javascript
_onInput: function (event) {
    this._setValue($(event.currentTarget).val());
},
```

Register it and use it in a view:

```javascript
var field_registry = require('web.field_registry');
field_registry.add('color_swatch', FieldColorSwatch);
```

```xml
<field name="color" widget="color_swatch"/>
```

View-specific registrations use a `viewType.` prefix: `form.color_swatch`,
`list.color_swatch`, `kanban.color_swatch`. The registry is a `web.Registry`
instance with `add`, `get`, `contains`, `getAny`, `keys`, `values`, and
`onAdd`.

---

## Views and Controllers

A JavaScript view is a factory that creates a controller, a renderer, and a
model. The base class is `web.AbstractView`.

```javascript
odoo.define('my_module.MyView', function (require) {
"use strict";
var AbstractView = require('web.AbstractView');
var MyView = AbstractView.extend({
    display_name: 'My View',
    mobile_friendly: true,
    config: {
        Controller: require('my_module.MyController'),
        Model: require('my_module.MyModel'),
        Renderer: require('my_module.MyRenderer'),
        SearchPanel: require('web.SearchPanel'),
    },
});
return MyView;
});
```

```javascript
var view_registry = require('web.view_registry');
view_registry.add('my_view', MyView);
```

`web.view_registry` is its own module. In 13.0 it is **not** exposed as
`core.view_registry`; the registries on `web.core` are `action_registry`,
`crash_registry`, and `serviceRegistry`.

The built-in controllers extend `web.BasicController`:

| Module | Class | View |
| --- | --- | --- |
| `web.ListController` | `ListController` | list |
| `web.FormController` | `FormController` | form |
| `web.KanbanController` | `KanbanController` | kanban |

A controller coordinates the model and renderer and owns the control panel
buttons. Constructor: `init(parent, model, renderer, params)`.

```javascript
var ListController = require('web.ListController');

var MyListController = ListController.extend({
    buttons_template: 'MyListView.buttons',
    events: _.extend({}, ListController.prototype.events, {
        'click .o_my_button': '_onMyButton',
    }),

    _onMyButton: function () {
        this.do_action({
            type: 'ir.actions.act_window',
            res_model: 'res.partner',
            views: [[false, 'list']],
        });
    },
});
```

`start()` is called after the controller is inserted. Use `this.$buttons` for
the control panel button area when the controller provides one.

Renderers and models: `web.AbstractRenderer` owns the view body DOM,
`web.AbstractModel` loads data and exposes `get`, and `web.BasicModel` is the
model used by form, list, and kanban views. A custom view normally reuses
`web.BasicModel` and `web.BasicRenderer`.

---

## Client Actions

A client action is a full-page widget registered under a tag. It extends
`web.AbstractAction`, which mixes in `web.ActionMixin`.

```javascript
odoo.define('my_module.MyClientAction', function (require) {
"use strict";
var AbstractAction = require('web.AbstractAction');
var core = require('web.core');
var MyClientAction = AbstractAction.extend({
    contentTemplate: 'my_module.MyClientAction',
    start: function () {
        var self = this;
        return this._super().then(function () {
            self.$('.o_my_action_button').on('click', function () {
                self.do_notify('Hello', 'Client action clicked.');
            });
        });
    },
});
core.action_registry.add('my_client_action', MyClientAction);
return MyClientAction;
});
```

`web.ActionMixin` renders the `Action` template and inserts the
`contentTemplate` into its `.o_content` zone, so client actions should set
`contentTemplate`, not `template`. The mixin also provides `renderButtons`,
`updateControlPanel`, `getTitle`, `on_attach_callback`, `on_detach_callback`,
and `willRestore`.

```xml
<record id="action_my_client" model="ir.actions.client">
    <field name="name">My Client Action</field>
    <field name="tag">my_client_action</field>
</record>
```

Set `hasControlPanel: true` if the action needs a control panel.

---

## Services

Services are long-lived objects shared by the web client. They are registered
in `core.serviceRegistry` and instantiated by `web.ServiceProviderMixin`.

```javascript
odoo.define('my_module.MyService', function (require) {
"use strict";
var AbstractService = require('web.AbstractService');
var core = require('web.core');

var MyService = AbstractService.extend({
    dependencies: ['ajax'],
    start: function () { /* called once when deployed */ },
    doSomething: function (value) {
        return this._rpc({
            route: '/my_module/do_something',
            params: { value: value },
        });
    },
});

core.serviceRegistry.add('my_service', MyService);
return MyService;
});
```

```javascript
this.call('my_service', 'doSomething', 42).then(function (result) {
    // ...
});
```

- `dependencies` lists services that must be deployed first.
- `start()` is called once, after dependencies are available.
- Services inherit `web.ServicesMixin`, so `this._rpc`, `this.call`, and
  `this.do_action` are available.
- A service is not a widget: no `$el`, no template.
- `web.AbstractService` extends `web.Class` with `EventDispatcherMixin` and
  `ServicesMixin`.

Built-in services:

| Name | Module | Purpose |
| --- | --- | --- |
| `ajax` | `web.AjaxService` | RPC and asset loading |
| `crash_manager` | `web.CrashManager` | error dialogs |
| `notification` | `web.NotificationService` | toast notifications |
| `report` | `web.ReportService` | report downloads |
| `local_storage` | `web.LocalStorageService` | browser local storage |
| `session_storage` | `web.SessionStorageService` | browser session storage |

`web.ServiceProviderMixin` deploys services and handles `call_service`.
`web.ServicesMixin` adds the caller-side helpers (`call`, `_rpc`, `do_action`,
`do_notify`, `loadViews`, `loadFilters`).

---

## Core Modules

### web.core

In 13.0, `web.core` exports `qweb` (the shared `web.QWeb` instance), `Class`,
`bus` / `main_bus`, `_t` / `_lt`, `action_registry`, `crash_registry`,
`serviceRegistry`, and `csrf_token`.

```javascript
var core = require('web.core');
this.$el.text(core._t('Save'));
```

`core.qweb.render(templateName, context)` renders a QWeb template.

### web.rpc

`web.rpc` is the low-level RPC helper. Inside a widget, prefer `this._rpc`.

```javascript
var rpc = require('web.rpc');
rpc.query({
    model: 'res.partner',
    method: 'read',
    args: [[1, 2, 3], ['name']],
}).then(function (records) { /* ... */ });
```

`rpc.buildQuery(options)` returns `{route, params}` without sending the
request. Options include `model`, `method`, `args`, `kwargs`, `context`,
`domain`, `fields`, `groupBy`, `limit`, `offset`, `orderBy`, `route`, and
`params`.

### web.session

`web.session` is a singleton `web.Session` instance.

```javascript
var session = require('web.session');
session.uid;            // current user id
session.db;             // database name
session.user_context;   // user context object
session.user_has_group('base.group_system');
```

Useful methods: `session_is_valid`, `session_authenticate`, `session_logout`,
`user_has_group`, `get_cookie`, `set_cookie`, `load_modules`,
`load_translations`, `load_css`, `load_js`, `load_qweb`, `get_currency`,
`get_file`, `session_reload`, `rpc`, and `setCompanies`.

### web.dom

`web.dom` contains DOM utilities: `append`, `prepend`, `detach`, `cssFind`,
`getPosition`, `getSelectionRange`, `setSelectionRange`, `autoresize`,
`makeAsyncHandler`, `makeButtonHandler`, `renderButton`, `renderCheckbox`,
`initAutoMoreMenu`, and `destroyAutoMoreMenu`.

### web.utils
`web.utils` contains general helpers: `assert`, `generateID`, `get_cookie`,
`set_cookie`, `human_number`, `human_size`, `insert_thousand_seps`, `into`,
`is_email`, `lpad`, `rpad`, `round_decimals`, `round_precision`, `stableSort`,
`swap`, `toBoolElse`, `traverse`, `traversePath`, `unaccent`, `xml_to_json`,
`json_node_to_xml`, `xml_to_str`, `deepFreeze`, `closestNumber`,
`binaryToBinsize`, `confine`, `float_is_zero`, `is_bin_size`, `isValidAnchor`,
`intersperse`, and `assetsDomain`. For example, `utils.human_size(2048)`
returns `'2.00 Kb'` and `utils.generateID()` returns a unique string.

---

## QWeb JavaScript Templates

JavaScript QWeb templates live in `static/src/xml/*.xml`. The root element is
`<templates>`, not `<odoo>`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
    <t t-name="my_module.Counter">
        <div class="o_counter">
            <span class="o_counter_value"><t t-esc="widget.value"/></span>
            <button type="button" class="o_counter_increment">Increment</button>
        </div>
    </t>
</templates>
```

The template context for a widget render always contains `widget`, which is the
widget instance. That is why templates use `widget.value`, `widget.name`, and
so on.

| Directive | Meaning |
| --- | --- |
| `t-esc="expr"` | output HTML-escaped text |
| `t-raw="expr"` | output unescaped HTML |
| `t-if` / `t-elif` / `t-else` | conditionals |
| `t-foreach` / `t-as` | loops |
| `t-call="template.name"` | include another template |
| `t-set` / `t-value` | define a variable |
| `t-att-name` / `t-attf-name` | dynamic and interpolated attributes |

```xml
<t t-name="my_module.PartnerList">
    <ul>
        <li t-foreach="widget.partners" t-as="partner" t-att-data-id="partner.id">
            <strong t-if="partner.is_company" t-esc="partner.name"/>
            <t t-else=""><t t-esc="partner.name"/></t>
        </li>
    </ul>
</t>
```

Render from JavaScript:

```javascript
var core = require('web.core');
this.$el.html(core.qweb.render('my_module.PartnerList', { widget: this }));
```

A widget with a `template` renders automatically in `renderElement()`.
### Loading templates

Templates are loaded through the manifest `qweb` key:

```python
{
    'name': 'My Module',
    'qweb': [
        'static/src/xml/my_templates.xml',
    ],
}
```

A widget can also declare `xmlDependencies`:

```javascript
var MyWidget = Widget.extend({
    template: 'my_module.Counter',
    xmlDependencies: ['/my_module/static/src/xml/my_templates.xml'],
});
```

`willStart()` loads these files with `ajax.loadXML` into `core.qweb`.

Escaping rules:

- Use `t-esc` for user data. It escapes HTML.
- Use `t-raw` only for trusted, already-sanitized HTML.
- There is no `t-out` in 13.0 (14.0+).
- There is no `t-inherit` / `t-inherits` in 13.0 (17.0+).

---

## Assets and Manifest

Odoo 13 has **no `assets` key in the manifest** and no `ir.asset` model.
Assets are declared by inheriting the bundle templates in XML.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <template id="assets_backend" name="my_module assets"
              inherit_id="web.assets_backend">
        <xpath expr="." position="inside">
            <script type="text/javascript"
                    src="/my_module/static/src/js/my_widget.js"/>
            <link rel="stylesheet" type="text/scss"
                  href="/my_module/static/src/scss/my_widget.scss"/>
        </xpath>
    </template>
</odoo>
```

Frontend assets inherit `web.assets_frontend` the same way.

| Bundle | Used by |
| --- | --- |
| `web.assets_common` | backend and website |
| `web.assets_backend` | backend web client |
| `web.assets_frontend` | website and portal |
| `web.report_assets_common` | reports |
| `web.report_assets_pdf` | PDF reports |

These are XML template ids, not manifest keys; the `web` module defines them
in `addons/web/views/webclient_templates.xml`.

### Manifest example

```python
{
    'name': 'My Module',
    'version': '13.0.1.0.0',
    'depends': ['web'],
    'data': ['views/assets.xml', 'views/my_views.xml'],
    'qweb': ['static/src/xml/my_templates.xml'],
    'installable': True,
    'application': False,
    'license': 'LGPL-3',
}
```

- `qweb` is a list of XML files containing `<templates>`.
- `data` includes the asset inheritance file.
- There is no `assets` key. Adding one has no effect in 13.0.
- `__manifest__.py` is the modern filename; `__openerp__.py` is deprecated but
  still read.
---

## Debugging

Server-side developer mode:

```bash
odoo-bin --dev=all -c /path/to/odoo.conf
```

`--dev` accepts a comma-separated list: `all`, `pudb`, `wdb`, `ipdb`, `pdb`,
`reload`, `qweb`, `werkzeug`, `xml`. `--dev=all` enables Python auto-reload,
XML reload, and QWeb template reloading.
Browser debug modes:

| Parameter | Effect |
| --- | --- |
| `?debug=1` | developer mode |
| `?debug=assets` | load non-minified assets, bypass asset cache |
| `?debug=tests` | load the `web.assets_tests` bundle |
| `?debug=assets,tests` | both |

`?debug=assets` is checked by `config.isDebug('assets')` in the data manager
and action manager, which then skip their caches. The debug manager's "Load
non-minified assets" action sets exactly this parameter. `odoo.debug` is
truthy in debug mode, and `odoo.define` performs extra checks when it is set.

---

## Testing with QUnit

Odoo 13 uses QUnit 2.9.1. Tests live under `static/tests/` in each addon and
run in the browser.

```javascript
odoo.define('my_module.tests', function (require) {
"use strict";
var core = require('web.core');
var Widget = require('web.Widget');

QUnit.module('my_module', {}, function () {
    QUnit.test('renders the widget value', async function (assert) {
        assert.expect(1);
        core.qweb.add_template(
            '<templates><t t-name="my_module.test_template">' +
            '<p><t t-esc="widget.value"/></p></t></templates>'
        );
        var MyWidget = Widget.extend({ template: 'my_module.test_template' });
        var widget = new MyWidget();
        widget.value = 42;
        await widget.appendTo($('#qunit-fixture'));
        assert.strictEqual(widget.$el.text(), '42');
        widget.destroy();
    });
});
});
```

Test helpers from `web.test_utils`:

| Helper | Purpose |
| --- | --- |
| `createView(params)` / `createAsyncView(params)` | create a view backed by a mock server |
| `createModel(params)` / `createParent(params)` | create a model or parent widget |
| `createActionManager(params)` | create an action manager |
| `mock.addMockEnvironment(widget, params)` | attach a mock RPC environment |
| `mock.intercept(widget, callback)` / `mock.patch(...)` | intercept or patch RPCs |
| `dom.click(element)` / `dom.triggerEvent(element, name)` | simulate DOM events |
| `fields.editInput(element, value)` | set an input value |
| `form.clickSave(form)` / `kanban.reload(kanban)` | view helpers |

`web.MockServer` is in `static/tests/helpers/mock_server.js`; `createView` wires it up.

Running tests: open `/web/tests` in a browser, or add `?debug=tests` to load
`web.assets_tests` in the web client. Addon test files are added by inheriting
`web.qunit_suite`:

```xml
<template id="qunit_suite" inherit_id="web.qunit_suite">
    <xpath expr="//div[@id='qunit']" position="before">
        <script type="text/javascript"
                src="/my_module/static/tests/my_tests.js"/>
    </xpath>
</template>
```

Python-side browser tests use `HttpCase.browser_js()` or its legacy alias
`phantom_js()`:

```python
from odoo.tests.common import HttpCase, tagged

@tagged('post_install', '-at_install')
class TestMyModule(HttpCase):

    def test_my_widget(self):
        self.browser_js(
            '/web',
            "odoo.__DEBUG__.services['web.clickEverywhere'](1);",
            ready="odoo.isReady === true",
            login='admin',
            timeout=300,
        )
```

`browser_js` starts a headless Chrome session. Both names are present in 13.0.

---

## Common Patterns and Pitfalls

Always call `this._super` in `init`, `willStart`, `start`, and `destroy`.
Skipping it in `init` breaks parent registration and `PropertiesMixin`;
skipping it in `willStart` skips `xmlDependencies` loading.

Do not use `this.$el` before rendering: it is undefined until
`renderElement()` runs, so do DOM work in `start()`.

Use `trigger_up` for child-to-parent communication. A field widget calls
`this._setValue(...)`, which triggers `field_changed` upward.

`this._rpc` returns a promise with `abort()`; keep it and abort it in
`destroy()` when the widget may disappear mid-request.

Registry keys are global. Prefix custom keys with the addon name. Do not use
ES modules or OWL (`import` / `export`, `/** @odoo-module **/`, `@web/...`).

---

## What Does Not Exist in 13.0

| API | Status in 13.0 |
| --- | --- |
| OWL (`Component`, `useState`, `useService`, `@odoo/owl`) | not available |
| `/** @odoo-module **/` and `import` / `export` | not available |
| `registry.category(...)` | not available |
| `assets` key in the manifest | not available; use XML inheritance |
| `ir.asset` model | not available |
| `t-out` | not available; use `t-esc` or `t-raw` |
| `t-inherit` / `t-inherits` | not available |
| `with_company()` | not available; use `with_context` |
| `Command` class for x2many writes | not available; use `(0, 0, vals)` tuples |
| `_read_group` | not available; use `read_group` |
| `Json` / `Properties` fields | not available |
| `@api.multi`, `@api.one`, `@api.cr` | not available |

---

## Base Code Reference

Verified against the Odoo 13.0 source tree:

- `addons/web/static/src/js/boot.js` - `odoo.define` implementation
- `core/class.js`, `core/mixins.js`, `core/widget.js` - `web.Class`, mixins,
  `web.Widget` lifecycle, `renderElement`, `setElement`, `appendTo`, `$`
- `core/service_mixins.js`, `core/abstract_service.js` - `ServicesMixin`
  (`call`, `_rpc`, `do_action`, `do_notify`), `ServiceProviderMixin`,
  `web.AbstractService`
- `core/registry.js`, `core/rpc.js`, `core/session.js`, `core/dom.js`,
  `core/utils.js` - registries, `web.rpc`, `web.Session`, `web.dom`, `web.utils`
- `services/core.js`, `services/session.js`, `services/ajax_service.js` -
  `web.core` exports, `web.session` singleton, `web.AjaxService`
- `fields/abstract_field.js`, `fields/field_registry.js` - `web.AbstractField`,
  `web.field_registry`
- `views/view_registry.js`, `views/abstract_view.js` - `web.view_registry`,
  `web.AbstractView`
- `views/basic/basic_controller.js`, `views/list/list_controller.js`,
  `views/form/form_controller.js`, `views/kanban/kanban_controller.js` -
  `web.BasicController`, `web.ListController`, `web.FormController`,
  `web.KanbanController`
- `chrome/abstract_action.js`, `chrome/action_mixin.js` - `web.AbstractAction`,
  `web.ActionMixin`
- `static/src/xml/base.xml`, `static/src/xml/notification.xml` - `<templates>`,
  `t-name`, `t-esc`, `t-raw`, `t-if`, `t-foreach` examples
- `views/webclient_templates.xml` - `assets_common`, `assets_backend`,
  `assets_frontend`, `web.qunit_suite`, `assets_tests`
- `__manifest__.py` - `qweb` key example
- `addons/mail/views/mail_templates.xml` - `web.assets_backend` inheritance
- `addons/web/static/tests/` - QUnit test layout and helpers
- `odoo/tests/common.py`, `odoo/tools/config.py` - `browser_js` / `phantom_js`,
  `--dev` option
