---
name: odoo-13-actions
description: Complete reference for Odoo 13 actions (ir.actions.*), menus (ir.ui.menu), and scheduled jobs (ir.cron). Covers act_window, act_url, server, report, client actions, menu structure, cron fields, and action bindings.
globs: "**/*.{py,xml}"
topics:
  - Window actions (ir.actions.act_window) with view_mode tree,form
  - URL actions (ir.actions.act_url)
  - Server actions (ir.actions.server) states code/object_create/object_write/multi
  - Report actions (ir.actions.report)
  - Client actions (ir.actions.client)
  - Scheduled actions (ir.cron) - interval_type, numbercall, doall
  - Menus (ir.ui.menu) with menuitem shortcut
  - Action bindings (binding_model_id, binding_type, binding_view_types)
when_to_use:
  - Creating window actions and menus
  - Writing server actions / automations
  - Setting up scheduled jobs
  - Exposing actions / reports in the "Action" or "Print" menu
  - Building client (JS) actions
---

# Odoo 13 Actions Guide

Reference for Odoo 13 `ir.actions.*`, `ir.ui.menu`, and `ir.cron`.

## Table of Contents

1. [Action Basics](#action-basics)
2. [Window Actions](#window-actions)
3. [URL Actions](#url-actions)
4. [Server Actions](#server-actions)
5. [Report Actions](#report-actions)
6. [Client Actions](#client-actions)
7. [Scheduled Actions](#scheduled-actions)
8. [Menus](#menus)
9. [Action Bindings](#action-bindings)
10. [Automated Actions](#automated-actions)
11. [Contextual Actions](#contextual-actions)
12. [Returning Actions From Python](#returning-actions-from-python)
13. [Quick Reference](#quick-reference)
14. [Base Code Reference](#base-code-reference)

---

## Action Basics

An action tells the client what to do after a user interaction: open a view, run
server code, print a report, open a URL, refresh the interface, or close a dialog.

All action models inherit `ir.actions.actions` (`odoo/addons/base/models/ir_actions.py`):

| Field | Type | Notes |
|-------|------|-------|
| `name` | Char | Required label |
| `type` | Char | Model name, set by each concrete model |
| `xml_id` | Char | Computed external ID |
| `help` | Html | Optional translated description |
| `binding_model_id` | Many2one `ir.model` | Adds the action to the model's Action/Print menu |
| `binding_type` | Selection | `action` (default) or `report` |
| `binding_view_types` | Char | Default `list,form`; empty string means all views |

Concrete models: `ir.actions.act_window`, `ir.actions.act_url`,
`ir.actions.server`, `ir.actions.report`, `ir.actions.client`,
`ir.actions.act_window_close`, and `ir.cron` (scheduled server actions).

From Python an action can be returned as `False` (no follow-up action; the view
or dialog stays open), a client tag string, an XML ID, or a dict. To close a
dialog, return `{'type': 'ir.actions.act_window_close'}`.

Use `<record>` for every action type. Odoo 13 still accepts the legacy
`<act_window>` and `<report>` shortcuts, but `<record>` exposes every field.

```xml
<odoo>
    <record id="action_my_model" model="ir.actions.act_window">
        <field name="name">My Records</field>
        <field name="res_model">my.model</field>
        <field name="view_mode">tree,form</field>
    </record>
</odoo>
```

---

## Window Actions

`ir.actions.act_window` opens a model through one or more views.

### Key Fields

| Field | Type | Notes |
|-------|------|-------|
| `name` | Char | Required label |
| `res_model` | Char | Required and validated against the registry |
| `view_mode` | Char | Comma-separated types, no spaces; default `tree,form` |
| `view_id` | Many2one `ir.ui.view` | Reference view, placed first when its type is in `view_mode` |
| `view_ids` | One2many `ir.actions.act_window.view` | Ordered `(sequence, view_mode, view_id)` list |
| `views` | Binary (computed) | Ordered `(view_id, view_mode)` pairs; read-only |
| `search_view_id` | Many2one `ir.ui.view` | Specific search view |
| `res_id` | Integer | Open one record; intended for a `form`-only action |
| `domain` | Char | Python list expression |
| `context` | Char | Python dict expression; required, default `{}` |
| `target` | Selection | `current` (default), `new`, `inline`, `fullscreen`, `main` |
| `limit` | Integer | List pagination, default 80 |
| `groups_id` | Many2many `res.groups` | Restrict visibility |
| `usage` | Char | Filters menu and home actions |
| `filter` | Boolean | Legacy flag |
| `help` | Html | Empty-list help |
| `binding_model_id` | Many2one `ir.model` | Bind to the Action menu |
| `binding_type` | Selection | `action` by default |
| `binding_view_types` | Char | Default `list,form` |

`view_mode` uses `tree` in 13.0. `list` is not a 13.0 view type. `VIEW_TYPES`
in `ir_actions.py` is `tree`, `form`, `graph`, `pivot`, `calendar`, `gantt`,
`kanban`; the `web` addon adds `qweb` to `ir.actions.act_window.view`. The
`search` type is handled by `ir.ui.view`, not by `VIEW_TYPES`.

### Basic Example

```xml
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">My Records</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">tree,kanban,form</field>
    <field name="domain">[('active', '=', True)]</field>
    <field name="context">{'search_default_my_records': 1, 'default_user_id': uid}</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create your first record</p>
        <p>Start by giving it a name.</p>
    </field>
</record>
```

`uid` is available in the XML evaluation context. At runtime the client
evaluates `domain` and `context` with the current user and context.

### Specific Views and Ordering

```xml
<record id="action_my_model" model="ir.actions.act_window">
    <field name="name">Customers</field>
    <field name="res_model">res.partner</field>
    <field name="view_mode">tree,form</field>
    <field name="view_id" ref="view_partner_tree_custom"/>
    <field name="search_view_id" ref="view_partner_search_custom"/>
</record>
```

`view_id` is a reference view, not an exclusive choice. The computed `views`
field puts it first when its type appears in `view_mode`, then appends the
remaining modes. Use `ir.actions.act_window.view` when several views of the
same type exist or ordering must be explicit. A unique index enforces one view
per mode per action.

`ir.actions.act_window.view` also has a `multi` boolean. When set, the action is
not shown on the right toolbar of a form view.

### Target Values

Set `target` to `new` for a dialog, for example on a `my.config.wizard` action
with `view_mode` set to `form`.

| Value | Effect |
|-------|--------|
| `current` (default) | Replace the main content area; breadcrumb added |
| `new` | Open in a modal dialog |
| `inline` | Edit the form inline without a modal |
| `fullscreen` | Full-screen takeover |
| `main` | Replace main content and reset breadcrumbs |

### Reading From Python

`ir.actions.act_window.for_xml_id(module, xml_id)` calls `env.ref()` and returns
`record.read()[0]`. Odoo 13 does not provide `_for_xml_id`.

---

## URL Actions

`ir.actions.act_url` opens a URL.

| Field | Type | Notes |
|-------|------|-------|
| `url` | Text | Required |
| `target` | Selection | `new` (default) or `self` |

Odoo 13 has no `download` target. To force a download, point `url` at a
controller that returns the file with `Content-Disposition: attachment`.

```xml
<record id="action_open_docs" model="ir.actions.act_url">
    <field name="name">Documentation</field>
    <field name="url">https://www.odoo.com/documentation/13.0/</field>
    <field name="target">new</field>
</record>
```

From Python, return a dict with `'type': 'ir.actions.act_url'`, `'url'`, and
`'target'`.

---

## Server Actions

`ir.actions.server` runs server-side logic. Odoo 13 supports exactly four
`state` values:

| `state` | Purpose |
|---------|---------|
| `code` | Execute a Python snippet |
| `object_create` | Create a record of `crud_model_id` |
| `object_write` | Update the current record |
| `multi` | Run child server actions in sequence |

The field help mentions "Send Email", "Add Followers", and "Create Next
Activity", but those are not 13.0 `state` values. There is no `webhook` state.

### Common Fields

| Field | Purpose |
|-------|---------|
| `name` | Action label |
| `model_id` | Model the action is declared on; required |
| `model_name` | Related stored `model_id.model` |
| `state` | `code`, `object_create`, `object_write`, or `multi`; default `object_write` |
| `usage` | `ir_actions_server` (default) or `ir_cron` |
| `sequence` | Ordering for `multi`; default 5, lower first |
| `code` | Python source; visible only to `base.group_system` |
| `child_ids` | Many2many of child actions, sorted by `sequence` |
| `crud_model_id` | Target model for create/write when different from `model_id` |
| `link_field_id` | Field linking a created record back to the active record |
| `fields_lines` | One2many `ir.server.object.lines` value mapping |
| `groups_id` | Restrict who may run the action |
| `binding_model_id` | Bind to the Action menu |
| `binding_type` | `action` by default |

### `code` - Run Python

```xml
<record id="action_server_notify" model="ir.actions.server">
    <field name="name">Notify Owner</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">
if records:
    for record in records:
        record.message_post(body="Processed via server action")
action = {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'title': 'Done',
        'message': '%d record(s) processed' % len(records),
        'type': 'success',
    },
}
    </field>
</record>
```

Evaluation context from `IrActionsServer._get_eval_context()`:

| Name | Description |
|------|-------------|
| `env` | Current environment |
| `model` | `env[model_id.model]`, a void recordset |
| `record` | Active record when `active_model` matches and `active_id` is set; may be `None` |
| `records` | Active recordset when `active_model` matches and `active_ids` is set; may be `None` |
| `action` | Assign here to return a follow-up action |
| `log(message, level='info')` | Writes to `ir.logging` |
| `Warning` | `odoo.exceptions.Warning` |
| `uid`, `user` | Current user id and record |
| `time`, `datetime`, `dateutil`, `timezone`, `float_compare` | Python helpers |
| `b64encode`, `b64decode` | Base64 helpers |

`run_action_code_multi()` uses `safe_eval(..., mode="exec", nocopy=True)`.
Assigning to `action` makes `run()` return that value.

### `object_write` - Update the Active Record

Odoo 13 uses `fields_lines`, not the later top-level `update_path` / `value`
fields on `ir.actions.server` (not available in 13.0).

```xml
<record id="action_mark_done" model="ir.actions.server">
    <field name="name">Mark Done</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">object_write</field>
    <field name="fields_lines" eval="[(0, 0, {
        'col1': env['ir.model.fields']._get('my.model', 'state').id,
        'value': 'done',
        'evaluation_type': 'value',
    })]"/>
</record>
```

`run_action_object_write()` reads `active_id` from the context, evaluates each
line, and writes on that record. The generic `run()` loop re-invokes it for each
id in `active_ids`.

### `object_create` - Create a Record

```xml
<record id="action_spawn_task" model="ir.actions.server">
    <field name="name">Create Task</field>
    <field name="model_id" ref="model_res_partner"/>
    <field name="state">object_create</field>
    <field name="crud_model_id" ref="project.model_project_task"/>
    <field name="link_field_id" ref="project.field_project_task__partner_id"/>
    <field name="fields_lines" eval="[(0, 0, {
        'col1': env['ir.model.fields']._get('project.task', 'name').id,
        'value': 'Follow-up task',
        'evaluation_type': 'value',
    })]"/>
</record>
```

The record is created on `crud_model_id.model`. When `link_field_id` is set, a
one2many or many2many field receives `(4, new_id)`; any other field receives the
new id. Recursion in `child_ids` is rejected by a constraint.

### `multi` - Chain Actions

```xml
<record id="action_multi" model="ir.actions.server">
    <field name="name">Process and Notify</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">multi</field>
    <field name="child_ids" eval="[(6, 0, [
        ref('action_mark_done'),
        ref('action_server_notify'),
    ])]"/>
</record>
```

`run_action_multi()` iterates `child_ids.sorted()` and returns the last truthy
result.

`ir.server.object.lines` has `col1` (`ir.model.fields`), `value`, and
`evaluation_type`: `value` is a literal (integers are coerced for many2one and
integer fields), `reference` is a record id (many2many is rejected), and
`equation` is evaluated with `safe_eval`.

### Running a Server Action

```python
action = self.env.ref('my_module.action_server_notify')
action.with_context(active_model='my.model', active_ids=records.ids).run()
```

`run()` checks `groups_id` and raises `AccessError` when the user lacks the
group. It returns the last action produced by the state handler, or `False`.
Cron jobs call the same path through `ir.actions.server._run_actions(ids)`.

---

## Report Actions

`ir.actions.report` prints a QWeb template.

| Field | Notes |
|-------|-------|
| `name` | Report label and default download name |
| `model` | Required model name |
| `model_id` | Computed `ir.model` from `model` |
| `report_type` | `qweb-pdf` (default), `qweb-html`, `qweb-text` |
| `report_name` | Required external ID of the QWeb template |
| `report_file` | Base name for generated files |
| `print_report_name` | Python expression using `object` and `time` |
| `paperformat_id` | Defaults to the company paperformat |
| `attachment_use` | Reuse a saved attachment instead of regenerating |
| `attachment` | Python expression producing the attachment filename |
| `multi` | Hide from the form view's right toolbar when set |
| `groups_id` | Restrict by group |
| `binding_model_id` | Bind to the Print menu |
| `binding_type` | Defaults to `report` |

```xml
<record id="action_report_my_model" model="ir.actions.report">
    <field name="name">My Model Report</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">my_module.report_my_model_template</field>
    <field name="report_file">my_module.report_my_model</field>
    <field name="print_report_name">'Report - %s' % (object.name or '')</field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="paperformat_id" ref="base.paperformat_euro"/>
</record>
```

### `<report>` Shortcut

```xml
<report
    id="action_report_my_model"
    string="My Model Report"
    model="my.model"
    report_type="qweb-pdf"
    name="my_module.report_my_model_template"
    file="my_module.report_my_model"
    print_report_name="'Report - %s' % (object.name or '')"
    attachment_use="False"/>
```

The shortcut maps `string` to `name`, `name` to `report_name`, and `file` to
`report_file`. It also accepts `attachment`, `attachment_use`, `usage`,
`parser`, `print_report_name`, `auto`, `header`, `multi`, `groups`,
`paperformat`, and `menu`. Unless `menu="False"` is given, it calls
`create_action()`, setting `binding_model_id` to the report's model and
`binding_type` to `report`.

### Printing From Python

```python
report = self.env.ref('my_module.action_report_my_model')
return report.report_action(self)
```

`report_action(docids, data=None, config=True)` returns an
`ir.actions.report` dict with `context.active_ids` set. For an admin without a
company report layout, it may return the document layout configurator action
with `close_on_report_download` set.

Useful methods: `_get_report_from_name(report_name)`,
`render(res_ids, data=None)` (dispatches to `render_qweb_pdf`,
`render_qweb_html`, or `render_qweb_text`), `get_paperformat()`,
`create_action()`, and `unlink_action()`. `attachment` and `print_report_name`
are evaluated with `safe_eval` and the `object` and `time` variables.

---

## Client Actions

`ir.actions.client` runs a JavaScript action registered in
`core.action_registry`.

| Field | Description |
|-------|-------------|
| `tag` | Required registry key |
| `params` | Dictionary passed to the handler, stored as `params_store` |
| `target` | `current` (default), `new`, `fullscreen`, `main` |
| `res_model` | Optional model, mostly for needactions |
| `context` | Python dict expression, default `{}` |

Odoo 13 has no `inline` target for client actions and no `soft_reload` tag.

```xml
<record id="action_open_dashboard" model="ir.actions.client">
    <field name="name">Dashboard</field>
    <field name="tag">my_module.dashboard</field>
    <field name="params" eval="{'filter': 'my_open'}"/>
</record>
```

Register the handler with the legacy AMD API: define a widget extending
`web.AbstractAction` and add it with
`core.action_registry.add('my_module.dashboard', Dashboard)` inside an
`odoo.define` module. The action dict is available as `this.action`, including
`this.action.params`.

Built-in tags include `reload`, `reload_context`, `home`, `login`, `logout`,
`display_notification`, `report.client_action`, `change_password`, and
`apps`/`apps.updates`.

`display_notification` params include `title`, `subtitle`, `message`, `type`,
`sticky`, `className`, `onClose`, and `buttons`.

```python
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'title': 'Success',
        'message': 'Data imported',
        'type': 'success',
        'sticky': False,
    },
}
```

---

## Scheduled Actions

`ir.cron` schedules an `ir.actions.server`. It delegates to
`ir_actions_server_id` (`delegate=True`), so `name`, `model_id`, `state`, and
`code` are written directly on the cron record.

| Field | Description |
|-------|-------------|
| `cron_name` | Related stored `ir_actions_server_id.name`; `name` also works via delegation |
| `user_id` | User whose permissions drive execution; required, defaults to current user |
| `active` | `True` to enable |
| `interval_number` | Integer, default 1 |
| `interval_type` | `minutes`, `hours`, `days`, `weeks`, `months`; default `months` |
| `numbercall` | Remaining executions; default 1, negative means unlimited |
| `doall` | Catch up on missed occurrences after downtime |
| `nextcall` | Next planned execution; required, defaults to now |
| `lastcall` | Last successful execution; also exposed in the context as `lastcall` |
| `priority` | Integer, default 5; 0 is higher priority, 10 lower |
| `model_id` | Model the server action runs on |
| `state` | Server action state; crons default to `code` |
| `code` | Python snippet, same context as a `code` server action |

`create()` forces `usage='ir_cron'`. `default_get()` forces
`default_state='code'` unless a default state is supplied.

```xml
<odoo>
    <data noupdate="1">
        <record id="ir_cron_my_job" model="ir.cron">
            <field name="name">My Module: Daily Sync</field>
            <field name="model_id" ref="model_my_model"/>
            <field name="state">code</field>
            <field name="code">model._cron_daily_sync()</field>
            <field name="user_id" ref="base.user_root"/>
            <field name="interval_number">1</field>
            <field name="interval_type">days</field>
            <field name="numbercall">-1</field>
            <field name="doall" eval="False"/>
            <field name="active" eval="True"/>
            <field name="priority">10</field>
        </record>
    </data>
</odoo>
```

Wrap the record in `<data noupdate="1">` when users may adjust the schedule
without an upgrade overwriting it.

### `interval_type` Values

Defined in `odoo/addons/base/models/ir_cron.py`: `minutes`, `hours`, `days`,
`weeks`, `months`. `weeks` is `relativedelta(days=7 * interval)`; `months`
uses `relativedelta(months=interval)`, so month-end dates follow calendar
arithmetic.

### Scheduler Semantics

`_process_jobs()` selects:

```sql
numbercall != 0 AND active AND nextcall <= (now() at time zone 'UTC')
ORDER BY priority
```

For each due job, `_process_job()` loops while `nextcall < now` and
`numbercall` is non-zero. It decrements a positive `numbercall`, runs the
callback when `not ok or doall`, advances `nextcall` by the interval, and sets
`active=False` when `numbercall` reaches zero. A negative `numbercall` is never
decremented and means unlimited.

`_callback()` runs `ir.actions.server.browse(server_action_id).run()` and rolls
back the job transaction on exception.

### Writing Cron Code

```python
@api.model
def _cron_daily_sync(self):
    records = self.search([('sync_required', '=', True)], limit=500)
    records.action_sync()
```

Use `@api.model`, operate on explicit recordsets, and batch work. The cron runs
as `user_id`, usually `base.user_root` (OdooBot).

### Triggering a Cron Manually

Odoo 13 has no `_trigger()` method. Use `method_direct_trigger()`:

```python
cron = self.env.ref('my_module.ir_cron_my_job')
cron.method_direct_trigger()
```

It checks write access, runs the server action as `cron.user_id` with
`lastcall` in the context, and updates `lastcall`.

`write()` and `unlink()` call `_try_lock()` and raise `UserError` when the job
is executing. `try_write(values)` is the non-raising variant used by
`toggle(model, domain)`.

---

## Menus

Menus live in `ir.ui.menu`. The usual declaration is `<menuitem>`.

```xml
<!-- Top-level app menu -->
<menuitem id="menu_my_module_root"
          name="My Module"
          sequence="50"
          web_icon="my_module,static/description/icon.png"/>

<!-- First-level -->
<menuitem id="menu_my_records"
          name="Records"
          parent="menu_my_module_root"
          action="action_my_model"
          sequence="10"/>

<!-- Second-level -->
<menuitem id="menu_reporting"
          name="Reporting"
          parent="menu_my_module_root"
          sequence="90"/>
```

### Attributes

| Attribute | Purpose |
|-----------|---------|
| `id` | External ID |
| `name` | Visible label; falls back to the action name, then the XML id |
| `parent` | Parent menu external ID; omit for top level |
| `action` | Action external ID; a module prefix is added when missing |
| `sequence` | Integer, lower first; model default 10 |
| `groups` | Comma-separated group XML IDs; prefix `-` to remove a group |
| `web_icon` | `module,path/to/icon.png`; only applied to top-level menus |
| `active` | `True`/`False` |

The converter only stores `web_icon` when no `parent` is given.
`ir.ui.menu._compute_web_icon_data()` reads a two-part `module,path` value on
create and write and stores the image in `web_icon_data`.

`ir.ui.menu.action` is a Reference limited to `ir.actions.report`,
`ir.actions.act_window`, `ir.actions.act_url`, `ir.actions.server`, and
`ir.actions.client`.

### Visibility

`_visible_menu_ids()` filters menus in two steps:

1. Drop menus whose `groups_id` does not intersect the user's groups.
2. For menus with an action, require read access on the action's target model.
   The model getter covers `ir.actions.act_window` (`res_model`),
   `ir.actions.report` (`model`), and `ir.actions.server` (`model_id.model`).

Menus without an action are folders and appear when a visible child needs them.
`base.group_no_one` is excluded unless debug mode is active.

```xml
<menuitem id="menu_admin_tools"
          name="Admin Tools"
          parent="menu_my_module_root"
          action="action_admin_tools"
          groups="base.group_system,-base.group_portal"/>
```

---

## Action Bindings

Any `ir.actions.*` can be bound to a model, adding it to that model's Action or
Print menu.

| Field | Description |
|-------|-------------|
| `binding_model_id` | `ir.model` where the action is displayed |
| `binding_type` | `action` (default) or `report` |
| `binding_view_types` | Comma-separated view types; default `list,form` |

`ir.actions.actions.get_bindings(model_name)` returns a dict with `action` and
`report` keys. It discards actions whose `groups_id` does not intersect the
user's groups and actions the user cannot read.

`fields_view_get(toolbar=True)` maps `tree` to `list`, then keeps an action when
the current view type is in `binding_view_types.split(',')`. An empty
`binding_view_types` falls back to the current view type, so the action appears
in every view. A kanban view therefore only shows actions with an empty
`binding_view_types` or one that explicitly includes `kanban`.

### Server Action Binding

```xml
<record id="action_bulk_archive" model="ir.actions.server">
    <field name="name">Archive Selected</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">records.action_archive()</field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="binding_type">action</field>
    <field name="binding_view_types">list</field>
</record>
```

`ir.actions.server.create_action()` sets `binding_model_id` to `model_id` and
`binding_type` to `action`. `unlink_action()` clears the binding.

### Report Binding

```xml
<record id="action_report_invoice_custom" model="ir.actions.report">
    <field name="name">Custom Invoice</field>
    <field name="model">account.move</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">my_module.report_invoice_custom</field>
    <field name="binding_model_id" ref="account.model_account_move"/>
</record>
```

`ir.actions.report` defaults `binding_type` to `report`, so it appears under
Print. `create_action()` and `unlink_action()` manage the binding for the
`<report>` shortcut.

| `binding_view_types` | Where shown |
|----------------------|-------------|
| `list` | List view only |
| `form` | Form view only |
| `list,form` | Default, both |
| empty string | Every view, including kanban |

---

## Automated Actions

Automation Rules are provided by the separate `base_automation` addon. That
addon is not part of the base checkout used to verify this guide, so this
section does not assert its model or field names.

What is verifiable in 13.0 is the contract an automation executes: an
`ir.actions.server` record. The rule triggers it on a model event, and the
server action runs with `active_model` and `active_id`/`active_ids` in its
context, exactly like a manually invoked contextual action.

- Put reusable logic in an `ir.actions.server` with `state='code'`.
- Keep the action's `model_id` equal to the model the automation watches.
- Use `record` for single-record triggers and `records` for multi-record runs.
- Test the server action directly with `run()` before wiring it to a rule.
- Do not use 14.0+ fields such as `update_path`, `value`, or the `webhook`
  state; they do not exist in 13.0.

---

## Contextual Actions

A contextual action is bound to a model through `binding_model_id`. It appears
in the Action menu of list and form views, and in the Print menu when
`binding_type='report'`.

```xml
<record id="action_contextual_export" model="ir.actions.server">
    <field name="name">Export Selected</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="state">code</field>
    <field name="code">records.action_export()</field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="binding_view_types">list</field>
</record>
```

The action receives `active_model`, `active_id` (form view), and `active_ids`
(list view) through the context. For a `code` server action, `record` and
`records` are populated from these keys. For `object_write`, the handler uses
`active_id`; the generic `run()` loop re-invokes it for each id in `active_ids`.

```python
action = self.env.ref('my_module.action_contextual_export')
action.create_action()   # bind to action.model_id
action.unlink_action()   # remove the binding
```

---

## Returning Actions From Python

### Open a Form View

```python
def action_open_partner(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': 'Partner',
        'res_model': 'res.partner',
        'view_mode': 'form',
        'res_id': self.partner_id.id,
        'target': 'current',
    }
```

Add `domain` and `context` keys to the same dict to open a filtered list with
default values and search filters.

### Open an Existing Action by XML ID

```python
def action_wizard(self):
    action = self.env.ref('my_module.action_configure').read()[0]
    action['context'] = {'default_partner_id': self.id}
    return action
```

For window actions:

```python
action = self.env['ir.actions.act_window'].for_xml_id(
    'my_module', 'action_configure')
```

### Print a Report

```python
def action_print(self):
    report = self.env.ref('my_module.action_report_my_model')
    return report.report_action(self)
```

### Close a Dialog

```python
return {'type': 'ir.actions.act_window_close'}
```

### Reload or Notify

```python
return {'type': 'ir.actions.client', 'tag': 'reload'}

return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'title': 'Saved',
        'type': 'success',
    },
}
```

A method called by a `type="object"` button may return any action dict. The
`action_*` prefix is a convention, not a framework requirement. Returning
`False` leaves the view or dialog open; return `ir.actions.act_window_close` to
close a dialog.

---

## Quick Reference

| Type | Model | Use |
|------|-------|-----|
| Window | `ir.actions.act_window` | Open views for a model |
| URL | `ir.actions.act_url` | Open a URL |
| Server | `ir.actions.server` | Run Python, create, update, or chain actions |
| Report | `ir.actions.report` | QWeb PDF, HTML, or text |
| Client | `ir.actions.client` | Registered JavaScript action |
| Close | `ir.actions.act_window_close` | Close the current dialog |
| Cron | `ir.cron` | Schedule a server action |

| Target | Window actions | Client actions |
|--------|----------------|----------------|
| `current` | Yes (default) | Yes (default) |
| `new` | Yes | Yes |
| `inline` | Yes | No |
| `fullscreen` | Yes | Yes |
| `main` | Yes | Yes |

Menus reference actions by XML ID:

```xml
<menuitem id="menu_root" name="My Module" sequence="50"/>
<menuitem id="menu_records" parent="menu_root" action="a1" sequence="10"/>
```

### 13.0 Differences to Remember

| Topic | Odoo 13.0 |
|-------|-----------|
| List view type | `tree`; `list` is not valid |
| Window default `view_mode` | `tree,form` |
| `ir.actions.act_url` targets | `new`, `self` only |
| Server action states | `code`, `object_create`, `object_write`, `multi` |
| Server action write fields | `fields_lines`; no `update_path` / `value` |
| Server action webhook | Not available |
| Client action targets | No `inline`; no `soft_reload` |
| Cron manual trigger | `method_direct_trigger()`; no `_trigger()` |
| Window action helper | `for_xml_id()`; no `_for_xml_id()` |
| `binding_view_types` default | `list,form`; empty means all views |

---

## Base Code Reference

- `odoo/addons/base/models/ir_actions.py` - action models, `VIEW_TYPES`,
  `get_bindings()`, `for_xml_id()`, `run()`, and the evaluation context.
- `odoo/addons/base/models/ir_actions_report.py` - `ir.actions.report`,
  `report_type`, `report_action()`, `render()`, and paperformat handling.
- `odoo/addons/base/models/ir_cron.py` - delegated `ir_actions_server_id`,
  `interval_type`, `numbercall`, `doall`, `nextcall`, `priority`, and
  `method_direct_trigger()`.
- `odoo/addons/base/models/ir_ui_menu.py` - `ir.ui.menu` and visibility.
- `odoo/models.py` - `fields_view_get(toolbar=True)` binding filter.
- `odoo/tools/convert.py` - menu, report, and legacy act_window shortcuts.
- `addons/web/models/models.py` - `qweb` view mode extension.
- `addons/web/static/src/js/core/misc.js` - built-in client action tags.
- `addons/web/static/src/js/chrome/action_manager.js` - action dispatch.
- `odoo/addons/test_action_bindings/` - binding tests.
