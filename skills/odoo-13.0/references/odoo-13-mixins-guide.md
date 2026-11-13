---
name: odoo-13-mixins
description: Complete reference for Odoo 13 mixins. Covers mail.thread, mail.activity.mixin, mail.alias.mixin, utm.mixin, portal.mixin, image.mixin, website.published.mixin, website.published.multi.mixin, website.multi.mixin, website.seo.metadata and rating.mixin.
globs: "**/models/**/*.py"
topics:
  - mail.thread (messaging, chatter, followers, tracking)
  - mail.activity.mixin (activities)
  - mail.alias.mixin (email aliases)
  - utm.mixin (campaign tracking)
  - portal.mixin (customer portal access)
  - image.mixin (image fields and resized variants)
  - website.published.mixin and website.published.multi.mixin (publishing)
  - website.multi.mixin (multi-website records)
  - website.seo.metadata (SEO metadata)
  - rating.mixin (customer ratings)
when_to_use:
  - Adding messaging or chatter to a model
  - Setting up email aliases for incoming mail
  - Adding activities to records
  - Tracking marketing campaigns
  - Exposing records to the customer portal
  - Adding images to records
  - Publishing records on the website
  - Implementing customer ratings
---

# Odoo 13 Mixins Guide

Complete reference for Odoo 13 mixins: messaging, activities, email aliases, tracking, portal access, images, website publishing, SEO and ratings.

This guide documents only what exists in Odoo 13.0. It is verified against the 13.0 source tree. Anything introduced in a later version is omitted or marked "not available in 13.0".

## Table of Contents

1. [mail.thread - Messaging](#mailthread---messaging)
2. [mail.activity.mixin - Activities](#mailactivitymixin---activities)
3. [mail.alias.mixin - Email Aliases](#mailaliasmixin---email-aliases)
4. [utm.mixin - Campaign Tracking](#utmmixin---campaign-tracking)
5. [portal.mixin - Customer Portal](#portalmixin---customer-portal)
6. [image.mixin - Images](#imagemixin---images)
7. [avatar.mixin - Not Available in 13.0](#avatarmixin---not-available-in-130)
8. [Website Mixins](#website-mixins)
9. [rating.mixin - Customer Ratings](#ratingmixin---customer-ratings)
10. [Mixin Composition](#mixin-composition)
11. [Quick Reference](#quick-reference)
12. [Base Code Reference](#base-code-reference)

---

## mail.thread - Messaging

`mail.thread` provides chatter, followers, messages, notifications and field tracking. It is an `AbstractModel` named `mail.thread`.

### Basic Integration

```python
from odoo import fields, models


class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread']
    _description = 'Business Trip'

    name = fields.Char(required=True, tracking=True)
    partner_id = fields.Many2one('res.partner', string='Responsible', tracking=True)
    state = fields.Selection(
        [('draft', 'Draft'), ('confirmed', 'Confirmed'), ('done', 'Done')],
        default='draft', tracking=True,
    )
```

`_inherit` accepts a list. A single string also works, but the list form is the convention when more than one mixin is involved.

### Form View Integration

Odoo 13.0 uses `div.oe_chatter`. The `<chatter>` shortcut tag does not exist.

```xml
<div class="oe_chatter">
    <field name="message_follower_ids" widget="mail_followers"/>
    <field name="activity_ids" widget="mail_activity"/>
    <field name="message_ids" widget="mail_thread"/>
</div>
```

Only include `activity_ids` when the model also inherits `mail.activity.mixin`. The `mail_followers` and `mail_thread` widgets are registered in 13.0.

### Fields Added by mail.thread

| Field | Type | Description |
|-------|------|-------------|
| `message_is_follower` | Boolean | Is the current user a follower? |
| `message_follower_ids` | One2many | Followers of the record |
| `message_partner_ids` | Many2many | Followers expressed as partners |
| `message_channel_ids` | Many2many | Channels following the record |
| `message_ids` | One2many | Posted messages |
| `message_unread` | Boolean | Are there unread messages? |
| `message_unread_counter` | Integer | Number of unread messages |
| `message_needaction` | Boolean | Does the record need action? |
| `message_needaction_counter` | Integer | Number of messages needing action |
| `message_has_error` | Boolean | Any failed notifications? |
| `message_has_error_counter` | Integer | Number of failed notifications |
| `message_attachment_count` | Integer | Attachment count |
| `message_main_attachment_id` | Many2one | Main attachment |

`message_attachment_count` and the activity fields are restricted to internal users (`groups="base.group_user"`).

### Field Tracking

Add `tracking=True` to log changes in the chatter. An integer sets the display order; `True` becomes the default sequence 100.

```python
class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread']

    name = fields.Char(tracking=True)
    state = fields.Selection([('draft', 'Draft'), ('done', 'Done')], tracking=10)
    partner_id = fields.Many2one('res.partner', tracking=20)
    amount = fields.Monetary(tracking=30)
```

`mail.thread` reads the `tracking` attribute from the field object. It is accepted as a field keyword even though `odoo/fields.py` does not list it as a formal parameter.

The legacy `track_visibility` attribute is still read in 13.0 but is deprecated. `tracking` takes precedence when both are set.

```python
# Legacy 13.0 style. Still works, but prefer tracking=True.
state = fields.Selection(
    [('draft', 'Draft'), ('done', 'Done')],
    track_visibility='onchange',
)
```

`track_visibility` was removed in 14.0. Do not use it in new 13.0 code.

### Controlling Messaging Access: `_mail_post_access`

The model attribute `_mail_post_access` controls which ACL operation is required to post. The default is `'write'`.

```python
class PublicThread(models.Model):
    _name = 'public.thread'
    _inherit = ['mail.thread']

    # Anyone with read access can post a message
    _mail_post_access = 'read'
```

Valid values are `'read'`, `'write'`, `'create'` and `'unlink'`. The value is read by `get_mail_message_access()` when `mail.message` checks access on the related document.

### Posting Messages

`message_post()` is keyword-only in 13.0 and returns the new `mail.message` ID (an integer, not a record).

```python
def notify_confirmation(self):
    self.ensure_one()
    message_id = self.message_post(
        body='<p>Trip has been confirmed.</p>',
        subject='Trip Confirmation',
        message_type='comment',
        subtype='mail.mt_comment',
        partner_ids=[self.partner_id.id],
        author_id=self.env.user.partner_id.id,
    )
    return message_id
```

| Argument | Description |
|----------|-------------|
| `body` | Raw HTML string; sanitized by the mail layer |
| `subject` | Optional subject |
| `message_type` | `'notification'` (default), `'comment'`, `'email'`; `'user_notification'` is reserved |
| `subtype` | XML id string, e.g. `'mail.mt_comment'` or `'mail.mt_note'` |
| `subtype_id` | Integer subtype id, alternative to `subtype` |
| `partner_ids` | Extra partner IDs to notify (list of integers) |
| `channel_ids` | Extra channel IDs to notify (list of integers) |
| `author_id` | Partner ID of the author |
| `email_from` | Author email, used when `author_id` is unknown |
| `parent_id` | Parent message for threading |
| `attachment_ids` | Existing `ir.attachment` IDs |
| `attachments` | `(name, content)` or `(name, content, info)` tuples; content is not base64 encoded |
| `add_sign` | Append the user signature (default `True`) |
| `record_name` | Override the record display name |

The 13.0 parameter is `subtype` (XML id string) or `subtype_id` (integer). The `subtype_xmlid` keyword does not exist in 13.0. `partner_ids` and `channel_ids` must be plain integer lists; recordset commands such as `(4, id, 0)` are rejected.

```python
def attach_report(self, pdf_content):
    self.message_post(
        body='<p>Report attached.</p>',
        attachments=[('report.pdf', pdf_content)],
    )
```

### Posting with a Template

In 13.0 the helper is `message_post_with_template()`. It takes a template ID, not an XML id string.

```python
def send_template(self):
    template = self.env.ref('my_module.email_template_trip_confirmation')
    self.message_post_with_template(template.id)
```

Signature:

```python
message_post_with_template(template_id, email_layout_xmlid=None,
                           auto_commit=False, **kwargs)
```

It creates a `mail.compose.message` composer, applies the template onchange, and calls `send_mail()`. `message_post_with_source()` is a later API and is not available in 13.0. `message_post_with_view(views_or_xmlid, **kwargs)` also exists in 13.0.

### Followers Management

```python
# Add partners as followers
record.message_subscribe(partner_ids=[partner1.id, partner2.id])

# Add with specific subtypes
record.message_subscribe(
    partner_ids=[partner.id],
    subtype_ids=[self.env.ref('mail.mt_comment').id],
)

# Remove
record.message_unsubscribe(partner_ids=[partner.id])
```

`message_subscribe(partner_ids=None, channel_ids=None, subtype_ids=None)` performs access checks and returns a boolean. `message_unsubscribe()` removes followers.

### Subtypes and `_track_subtype`

Subtypes classify notifications so users can filter what they receive. Define one with `model="mail.message.subtype"` and set `name`, `res_model`, `default` and `description`.

In 13.0 `_track_subtype()` must return a `mail.message.subtype` record or `False`.

```python
def _track_subtype(self, init_values):
    self.ensure_one()
    if 'state' in init_values and self.state == 'confirmed':
        return self.env.ref('my_module.mt_trip_confirmed')
    return super()._track_subtype(init_values)
```

`_track_template(changes)` returns a dict mapping field names to `(template, post_kwargs)` tuples for automatic template posting.

### Context Keys for Control

| Key | Effect |
|-----|--------|
| `mail_create_nosubscribe` | Do not subscribe the author on create or post |
| `mail_create_nolog` | Do not log the automatic "created" message |
| `mail_notrack` | Skip field tracking |
| `tracking_disable` | Disable all mail.thread features |
| `mail_post_autofollow` | Subscribe recipients of `message_post` |
| `mail_notify_force_send` | Send notifications directly instead of queuing |
| `mail_auto_delete` | Delete notification emails after sending (default `True`) |
| `mail_activity_automation_skip` | Skip automated activity generation and updates |

```python
self.env['business.trip'].with_context(tracking_disable=True).create({'name': 'Silent Trip'})
```

### Incoming Mail Hooks

`message_route()` decides which model and thread an incoming message belongs to. `message_new(msg_dict, custom_values=None)` creates a record; `message_update(msg_dict, update_vals=None)` updates one. Override them to map incoming email fields to your model.

---

## mail.activity.mixin - Activities

`mail.activity.mixin` adds an `activity_ids` one2many and helper fields and methods to schedule activities (calls, meetings, to-dos) on records.

### Integration

```python
class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _description = 'Business Trip'

    name = fields.Char(required=True)
```

The activity mixin does not inherit `mail.thread` itself. List both when you want chatter and activities.

### Form View Integration

```xml
<div class="oe_chatter">
    <field name="message_follower_ids" widget="mail_followers"/>
    <field name="activity_ids" widget="mail_activity"/>
    <field name="message_ids" widget="mail_thread"/>
</div>
```

For kanban views, use `widget="kanban_activity"` on `activity_ids`; the widget is registered in 13.0.

### Fields Added

| Field | Description |
|-------|-------------|
| `activity_ids` | One2many on `mail.activity` |
| `activity_state` | `'overdue'`, `'today'`, `'planned'` or `False` |
| `activity_user_id` | Responsible user of the next activity |
| `activity_type_id` | Activity type of the next activity |
| `activity_date_deadline` | Deadline of the next activity |
| `activity_summary` | Summary of the next activity |
| `activity_exception_decoration` | `'warning'`, `'danger'` or `False` |
| `activity_exception_icon` | Icon for an exception activity |

All activity fields are restricted to internal users (`groups="base.group_user"`). `my_activity_date_deadline` does not exist in 13.0; it was added in a later version.

### Scheduling Activities

```python
from datetime import timedelta

from odoo import fields


def plan_review(self):
    self.activity_schedule(
        'mail.mail_activity_data_todo',
        user_id=self.env.user.id,
        summary='Review trip',
        note='<p>Please review and approve.</p>',
        date_deadline=fields.Date.today() + timedelta(days=3),
    )


def mark_done(self):
    self.activity_feedback(
        ['mail.mail_activity_data_todo'],
        feedback='Reviewed, looks good',
    )
```

`date_deadline` is a `Date`, not a `Datetime`. Passing a datetime logs a warning in 13.0.

| Method | Description |
|--------|-------------|
| `activity_schedule(act_type_xmlid='', date_deadline=None, summary='', note='', **act_values)` | Create an activity on each record |
| `activity_schedule_with_view(act_type_xmlid='', date_deadline=None, summary='', views_or_xmlid='', render_context=None, **act_values)` | Create an activity with a rendered QWeb note |
| `activity_reschedule(act_type_xmlids, user_id=None, date_deadline=None, new_user_id=None)` | Move or reassign activities |
| `activity_feedback(act_type_xmlids, user_id=None, feedback=None)` | Mark activities as done with feedback |
| `activity_unlink(act_type_xmlids, user_id=None)` | Remove activities without feedback |

`activity_schedule()` returns the created `mail.activity` recordset. It defaults `date_deadline` to `fields.Date.context_today(self)` and defaults the user to the activity type's default user or the current user.

Built-in activity type XML ids in 13.0: `mail.mail_activity_data_email`, `mail.mail_activity_data_call`, `mail.mail_activity_data_meeting` and `mail.mail_activity_data_todo`.

`mail_activity_automation_skip` skips automated activity generation, updates and unlinks.

---

## mail.alias.mixin - Email Aliases

Aliases create records from incoming email. The mixin is defined in `addons/mail/models/mail_alias.py`.

### How the 13.0 Mixin Works

`mail.alias.mixin` uses `_inherits` to delegate to `mail.alias`:

```python
class AliasMixin(models.AbstractModel):
    _name = 'mail.alias.mixin'
    _inherits = {'mail.alias': 'alias_id'}

    alias_id = fields.Many2one(
        'mail.alias', string='Alias', ondelete='restrict', required=True,
    )
```

Every record has exactly one alias. The alias is created automatically when the record is created and deleted when the record is deleted.

There is no `mail.alias.mixin.optional` in 13.0. That variant was introduced in a later version. In 13.0, `mail.alias.mixin` always requires an alias.

### Integration

```python
class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread', 'mail.alias.mixin']
    _description = 'Business Trip'

    name = fields.Char(required=True)

    def get_alias_model_name(self, vals):
        """Model used when an incoming email creates a new record."""
        return 'business.expense'

    def get_alias_values(self):
        """Values written on the alias after it is created."""
        values = super().get_alias_values()
        values['alias_defaults'] = {'trip_id': self.id}
        values['alias_contact'] = 'followers'
        return values
```

`get_alias_model_name(vals)` returns the model name for records created from incoming emails. The default returns `None`, meaning the alias model is the model that owns the alias.

`get_alias_values()` returns a dict written on the alias after creation. The default is `{'alias_parent_thread_id': self.id}`. Always call `super()` and update the result so the parent thread id is preserved.

The 13.0 method names are `get_alias_model_name()` and `get_alias_values()`. `_alias_get_creation_values()` does not exist in 13.0.

### Alias Fields

Because of `_inherits`, these `mail.alias` fields are available on the inheriting model:

| Field | Description |
|-------|-------------|
| `alias_name` | Local part of the email, e.g. `jobs` for `jobs@example.com` |
| `alias_model_id` | Model used to create records from incoming mail |
| `alias_user_id` | Owner of records created from incoming mail |
| `alias_defaults` | Python dict literal of default values for created records |
| `alias_force_thread_id` | Force incoming mail into an existing thread |
| `alias_domain` | Computed domain from the `mail.catchall.domain` parameter |
| `alias_parent_model_id` | Parent model holding the alias |
| `alias_parent_thread_id` | Parent record holding the alias |
| `alias_contact` | `'everyone'`, `'partners'` or `'followers'` |

`alias_domain_id` and `alias_email` do not exist in 13.0. The domain is the computed `alias_domain` field, read from the `mail.catchall.domain` system parameter.

`alias_defaults` is a `Text` field containing a Python dictionary literal. It is evaluated with `safe_eval` and must be a literal dict, for example `{'trip_id': 12}`.

`alias_contact` values:

- `everyone`: anyone can post to the alias.
- `partners`: only authenticated partners can post.
- `followers`: only followers of the related record or members of following channels can post.

### Form View

Expose `alias_name`, `alias_domain` (readonly), `alias_contact` and `alias_defaults` in the form. `alias_name` is not required at the database level in 13.0. An empty alias name means the alias is inactive. Alias names are cleaned and made unique by `mail.alias.create()` and `mail.alias.write()`.

Override `_alias_check_contact()` or `_alias_check_contact_on_record()` to customize who may post through the alias.

---

## utm.mixin - Campaign Tracking

`utm.mixin` tracks marketing campaigns through URL parameters and cookies. It is defined in `addons/utm/models/utm_mixin.py`.

```python
class Lead(models.Model):
    _name = 'crm.lead'
    _inherit = ['utm.mixin']

    name = fields.Char()
```

### Fields Added

| Field | Type | Cookie |
|-------|------|--------|
| `campaign_id` | Many2one `utm.campaign` | `odoo_utm_campaign` |
| `source_id` | Many2one `utm.source` | `odoo_utm_source` |
| `medium_id` | Many2one `utm.medium` | `odoo_utm_medium` |

### How It Works in 13.0

1. A user opens a URL such as `https://myodoo.com/?utm_campaign=winter_sale&utm_source=google`.
2. `ir_http` stores the URL parameters in cookies.
3. `utm.mixin.default_get()` reads the cookies and populates the fields when a record is created, for example from a website form.

`default_get()` also creates the `utm.campaign`, `utm.source` or `utm.medium` record when the cookie contains a name that does not exist yet. If the target model has an `is_website` field, the created UTM record is marked as a website record.

UTM values are ignored for users in the `sales_team.group_sale_salesman` group unless the call runs as superuser.

### tracking_fields()

`tracking_fields()` returns the mapping between URL parameter, field name and cookie name:

```python
def tracking_fields(self):
    return [
        ('utm_campaign', 'campaign_id', 'odoo_utm_campaign'),
        ('utm_source', 'source_id', 'odoo_utm_source'),
        ('utm_medium', 'medium_id', 'odoo_utm_medium'),
    ]
```

In 13.0 this is a regular method, not an `@api.model` method. `default_get()` calls it as `self.env['utm.mixin'].tracking_fields()`. As a consequence, overriding `tracking_fields()` on the model that inherits `utm.mixin` does not change what `default_get()` reads. To add custom tracking in 13.0, override `default_get()` on your model or extend `utm.mixin` itself.

---

## portal.mixin - Customer Portal

`portal.mixin` exposes records to external partners through the customer portal with an access URL and a security token. It is defined in `addons/portal/models/portal_mixin.py`.

```python
class SaleOrder(models.Model):
    _name = 'sale.order'
    _inherit = ['portal.mixin', 'mail.thread']

    partner_id = fields.Many2one('res.partner', required=True)

    def _compute_access_url(self):
        super()._compute_access_url()
        for order in self:
            order.access_url = '/my/orders/%s' % order.id
```

### Fields Added

| Field | Description |
|-------|-------------|
| `access_url` | Portal URL of the record; computed, default `'#'` |
| `access_token` | Security token stored on the record; `copy=False` |
| `access_warning` | Warning message shown on the portal; computed empty by default |

`access_token` is a UUID generated by `_portal_ensure_token()`. It is not a signed token in 13.0.

### Methods

| Method | Description |
|--------|-------------|
| `_compute_access_url()` | Compute `access_url`; override in your model |
| `_portal_ensure_token()` | Generate and return the record access token |
| `_get_share_url(redirect=False, signup_partner=False, pid=None, share_token=True)` | Build the share URL with access parameters |
| `get_access_action(access_uid=None)` | Return the action that opens the record for a portal user |
| `action_share()` | Return the portal share wizard action |
| `get_portal_url(suffix=None, report_type=None, download=None, query_string=None, anchor=None)` | Build a portal URL with the access token |
| `_notify_get_groups(msg_vals=None)` | Add the portal customer notification group |

The 13.0 method is `get_access_action()`, without a leading underscore. `_get_access_action()` and `_get_access_warning()` do not exist in 13.0. To customize the warning, override `_compute_access_warning()`.

### Share Link and Portal URL

```python
def action_share(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_url',
        'url': self._get_share_url(),
        'target': 'new',
    }
```

`_get_share_url()` returns `access_url` with `model`, `res_id` and `access_token` query parameters. With `redirect=True` it returns a `/mail/view` URL instead. `get_portal_url()` appends the access token and optional report parameters to `access_url`.

`get_access_action(access_uid=None)` returns an `ir.actions.act_url` action for share users or when `force_website=True` is in the context. Otherwise it falls back to the standard form view action.

---

## image.mixin - Images

`image.mixin` stores an image and automatically resized variants. It is defined in `odoo/addons/base/models/image_mixin.py`.

```python
class MyProduct(models.Model):
    _name = 'my.product'
    _inherit = ['image.mixin']

    name = fields.Char()
```

### Fields Added

| Field | Max Size | Storage |
|-------|----------|---------|
| `image_1920` | 1920 px | Base image |
| `image_1024` | 1024 px | Related, stored |
| `image_512` | 512 px | Related, stored |
| `image_256` | 256 px | Related, stored |
| `image_128` | 128 px | Related, stored |

All fields are `fields.Image`, which extends `fields.Binary` with `max_width`, `max_height` and `verify_resolution` options. The resized variants are related to `image_1920` and stored for performance.

### Form View Usage and Conventions

```xml
<field name="image_1920" widget="image" options="{'preview_image': 'image_128'}"/>
```

The `image` widget and the `preview_image` option are available in 13.0.

- Use `image_1920` as the writable base field.
- Use `image_128` or `image_256` in list and kanban views to avoid loading large images.
- `res.partner` inherits `image.mixin` in 13.0 and uses `image_1920` for the contact picture.
- The mixin does not generate a placeholder image. If a record has no image, the field is empty.

---

## avatar.mixin - Not Available in 13.0

`avatar.mixin` does not exist in Odoo 13.0. It was introduced in Odoo 15.0. Do not inherit it in a 13.0 module.

In 13.0, use `image.mixin` for the image fields. If you need a generated fallback avatar, compute it yourself. A common 13.0 pattern is the one used by `res.partner`: inherit `image.mixin` and fill `image_1920` from an external source such as Gravatar when the field is empty.

```python
class Team(models.Model):
    _name = 'team'
    _inherit = ['image.mixin']

    name = fields.Char(required=True)
```

The later `avatar_1920` through `avatar_128` fields, `_avatar_name_field` and `_avatar_get_placeholder_path()` are not available in 13.0.

---

## Website Mixins

The website mixins are defined in `addons/website/models/mixins.py`.

### website.published.mixin

```python
class MyDocument(models.Model):
    _name = 'my.document'
    _inherit = ['website.published.mixin']

    name = fields.Char(required=True)

    def _compute_website_url(self):
        super()._compute_website_url()
        for document in self:
            document.website_url = '/my/documents/%s' % document.id
```

| Field | Description |
|-------|-------------|
| `website_published` | Boolean related to `is_published`, writable |
| `is_published` | Boolean, `copy=False`, default from `_default_is_published()` |
| `can_publish` | Boolean computed by `_compute_can_publish()`; default `True` |
| `website_url` | Char computed by `_compute_website_url()`; default `'#'` |

| Method | Description |
|--------|-------------|
| `_compute_website_url()` | Compute the public URL; override in your model |
| `_default_is_published()` | Default for `is_published`; returns `False` |
| `website_publish_button()` | Toggle `website_published` |
| `open_website_url()` | Return an `ir.actions.act_url` action for `website_url` |
| `create_and_get_website_url(**kwargs)` | Create a record and return its website URL |
| `_compute_can_publish()` | Compute publishing rights; default `True` |
| `_get_can_publish_error_message()` | Error message used when publishing is denied |

`create()` and `write()` raise `AccessError` when a record is published or unpublished and `can_publish` is `False`. Override `_compute_can_publish()` for custom rights management. `website_url` defaults to `'#'`; always override `_compute_website_url()` or the publish button opens `'#'`.

### website.published.multi.mixin

Combines `website.published.mixin` and `website.multi.mixin` for records published per website.

```python
class MyDocument(models.Model):
    _name = 'my.document'
    _inherit = ['website.published.multi.mixin']

    name = fields.Char(required=True)
```

In this mixin, `website_published` is computed, inversed and searchable:

- `_compute_website_published()` uses the `website_id` context key. A record is published on the current website when `is_published` is true and either `website_id` is empty or equals the current website.
- `_inverse_website_published()` writes `is_published`.
- `_search_website_published()` supports only `=` and `!=` with boolean values.

### website.multi.mixin

Adds a `website_id` many2one to restrict a record to one website.

```python
class MyDocument(models.Model):
    _name = 'my.document'
    _inherit = ['website.multi.mixin']
```

The mixin already defines `website_id` with `ondelete='restrict'`. Do not redeclare it unless you need different options. `can_access_from_current_website(website_id=False)` returns `True` when the record's website is empty or matches the current website.

### website.seo.metadata

```python
class MyDocument(models.Model):
    _name = 'my.document'
    _inherit = ['website.seo.metadata']

    name = fields.Char(required=True)
```

| Field | Description |
|-------|-------------|
| `is_seo_optimized` | Boolean computed from title, description and keywords |
| `website_meta_title` | Meta title, translatable |
| `website_meta_description` | Meta description, translatable |
| `website_meta_keywords` | Meta keywords, translatable |
| `website_meta_og_img` | OpenGraph image URL |

| Method | Description |
|--------|-------------|
| `_default_website_meta()` | Return default OpenGraph and Twitter metadata |
| `get_website_meta()` | Merge custom metadata over the defaults |

Override `_default_website_meta()` to change default values. Do not override `get_website_meta()`; it only merges user-provided values.

### Website Mixin Composition

Combine `website.published.multi.mixin` and `website.seo.metadata` on a website document, and override `_compute_website_url()` as shown above.

---

## rating.mixin - Customer Ratings

`rating.mixin` receives and aggregates customer ratings. It is defined in `addons/rating/models/rating_mixin.py`.

```python
class ProjectTask(models.Model):
    _name = 'project.task'
    _inherit = ['mail.thread', 'rating.mixin']
    _description = 'Task'

    name = fields.Char()
    user_id = fields.Many2one('res.users')
    partner_id = fields.Many2one('res.partner')
```

In 13.0, `rating.mixin` does not inherit `mail.thread`. Add `mail.thread` explicitly if you want rating feedback posted in the chatter. The mixin checks for `message_post` at runtime and posts only when the model provides it.

### Fields Added

| Field | Description |
|-------|-------------|
| `rating_ids` | One2many on `rating.rating` |
| `rating_last_value` | Last consumed rating value; stored compute |
| `rating_last_feedback` | Feedback of the last rating; related |
| `rating_last_image` | Image of the last rating; related |
| `rating_count` | Number of consumed ratings |
| `rating_avg` | Average rating |

`rating_avg_text` does not exist in 13.0. `rating_percentage_satisfaction` belongs to the separate `rating.parent.mixin`, not to `rating.mixin`.

### Rater, Rated Partner and Access Token

By default the mixin uses `partner_id` as the customer giving the rating and `user_id.partner_id` as the person being rated.

```python
def rating_get_partner_id(self):
    """Partner giving the rating (customer)."""
    return self.customer_id or super().rating_get_partner_id()


def rating_get_rated_partner_id(self):
    """Partner being rated (internal user)."""
    return self.assignee_id.partner_id or super().rating_get_rated_partner_id()
```

The 13.0 names are `rating_get_partner_id()` and `rating_get_rated_partner_id()`. `_rating_get_partner()` and `_rating_get_rated_partner()` do not exist in 13.0.

`rating_get_access_token(partner=None)` creates or returns the `rating.rating` record and its access token.

### Sending a Rating Request

```python
def send_rating_request(self):
    self.ensure_one()
    template = self.env.ref('my_module.rating_request_email')
    self.rating_send_request(template, force_send=True)
```

Signature:

```python
rating_send_request(template, lang=False, subtype_id=False,
                    force_send=True, composition_mode='comment',
                    notif_layout=None)
```

`template` is a `mail.template` record, not an XML id string. The method uses `message_post_with_template()` internally. The default subtype is `mail.mt_note`, so the request does not spam followers.

### Applying a Rating

```python
def apply_rating(self, rate, token=None, feedback=None):
    return self.rating_apply(
        rate,
        token=token,
        feedback=feedback,
        subtype='mail.mt_comment',
    )
```

`rating_apply(rate, token=None, feedback=None, subtype=None)` writes the rating, marks it consumed, and posts a message when the model has `message_post`. The `subtype` argument is an XML id string.

In 13.0 the rating scale is 0 to 10. The public controller accepts 1, 5 and 10 at `/rating/<token>/<rate>`. Build a link with `'/rating/%s/%s' % (self.rating_get_access_token(), value)`.

### Statistics Helpers

| Method | Description |
|--------|-------------|
| `rating_get_repartition(add_stats=False, domain=None)` | Count ratings per value |
| `rating_get_grades(domain=None)` | Count great, okay and bad ratings |
| `rating_get_stats(domain=None)` | Return average, total and repartition |
| `_rating_domain()` | Domain selecting consumed ratings for this record |
| `_rating_get_parent_field_name()` | Parent relation field; default `None` |

---

## Mixin Composition

Stack mixins in `_inherit`. Order is usually not significant for these mixins, but keep the list readable.

| Use case | `_inherit` list |
|----------|-----------------|
| Chatter and activities | `['mail.thread', 'mail.activity.mixin']` |
| Alias and activities | `['mail.thread', 'mail.activity.mixin', 'mail.alias.mixin']` |
| Portal document | `['portal.mixin', 'mail.thread', 'mail.activity.mixin']` |
| Rated CRM document | `['mail.thread', 'mail.activity.mixin', 'utm.mixin', 'rating.mixin']` |
| Website document | `['website.published.multi.mixin', 'website.seo.metadata', 'mail.thread']` |
| Image-bearing record | `['image.mixin', 'mail.thread']` |

`rating.mixin` does not bring `mail.thread` in 13.0, so list `mail.thread` explicitly.

```python
class BusinessTrip(models.Model):
    _name = 'business.trip'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _description = 'Business Trip'

    name = fields.Char(required=True, tracking=True)
    state = fields.Selection(
        [('draft', 'Draft'), ('confirmed', 'Confirmed'), ('done', 'Done')],
        default='draft', tracking=True,
    )
```

---

## Quick Reference

### Mixin Summary

| Mixin | Purpose | Key Fields |
|-------|---------|------------|
| `mail.thread` | Messaging, followers, tracking | `message_ids`, `message_follower_ids` |
| `mail.activity.mixin` | Activities | `activity_ids`, `activity_state` |
| `mail.alias.mixin` | Required email alias | `alias_id`, `alias_name`, `alias_contact` |
| `utm.mixin` | Marketing tracking | `campaign_id`, `source_id`, `medium_id` |
| `portal.mixin` | Portal URL and token | `access_url`, `access_token` |
| `image.mixin` | Image with resized variants | `image_1920` through `image_128` |
| `website.published.mixin` | Publishing fields | `is_published`, `website_published`, `website_url` |
| `website.published.multi.mixin` | Publishing per website | `website_published`, `website_id` |
| `website.multi.mixin` | Website restriction | `website_id` |
| `website.seo.metadata` | SEO metadata | `website_meta_title`, `website_meta_description` |
| `rating.mixin` | Customer ratings | `rating_ids`, `rating_avg` |

### 13.0 API Names That Differ from Later Versions

| Later API | 13.0 API |
|-----------|----------|
| `subtype_xmlid=` in `message_post` | `subtype=` (XML id string) or `subtype_id=` (integer) |
| `message_post_with_source()` | `message_post_with_template(template_id)` |
| `mail.alias.mixin.optional` | Not available; use `mail.alias.mixin` |
| `_alias_get_creation_values()` | `get_alias_values()` |
| `alias_domain_id`, `alias_email` | `alias_domain` (computed) |
| `my_activity_date_deadline` | Not available; use `activity_date_deadline` |
| `_get_access_action()` | `get_access_action()` |
| `_get_access_warning()` | `_compute_access_warning()` |
| `avatar.mixin` | Not available; use `image.mixin` |
| `rating_avg_text` | Not available |
| `_rating_get_partner()` | `rating_get_partner_id()` |
| `_rating_get_rated_partner()` | `rating_get_rated_partner_id()` |
| `rating_send_request(rating_template=...)` | `rating_send_request(template, ...)` |

### Common Pitfalls

1. `message_post()` is keyword-only and returns an integer message ID.
2. Use `subtype='mail.mt_comment'` or `subtype_id=<int>`. `subtype_xmlid` is not available in 13.0.
3. `message_post_with_template()` takes a template ID. Pass `self.env.ref('module.template').id`, not the XML id string.
4. `track_visibility` still works in 13.0 but is deprecated. Use `tracking=True` or an integer sequence.
5. `mail.alias.mixin` requires an alias for every record. There is no optional alias mixin in 13.0.
6. `alias_defaults` is a Python dict literal stored in a `Text` field and evaluated with `safe_eval`.
7. `utm.mixin.tracking_fields()` is not an `@api.model` method in 13.0. Overriding it on the inheriting model does not affect `default_get()`.
8. `portal.mixin` uses `get_access_action()`, not `_get_access_action()`. `access_token` is a UUID stored on the record.
9. `website_url` defaults to `'#'`. Override `_compute_website_url()` or the publish button opens an empty URL.
10. `rating.mixin` does not inherit `mail.thread` in 13.0. Add `mail.thread` explicitly if you want chatter feedback.
11. `rating_send_request()` takes a template record, not an XML id string.
12. `avatar.mixin` does not exist in 13.0. Use `image.mixin`.

---

## Base Code Reference

The guide is based on the Odoo 13.0 source tree. The local checkout used for this guide contains `addons/mail`, `addons/portal`, `addons/website`, `addons/web` and `odoo/addons/base`. The `utm` and `rating` addons were verified against the `odoo/odoo` 13.0 branch sources listed below.

| File | Contents |
|------|----------|
| `addons/mail/models/mail_thread.py` | `mail.thread`, `message_post`, `message_subscribe`, `_track_subtype`, `_mail_post_access` |
| `addons/mail/models/mail_activity.py` | `mail.activity.mixin`, `activity_schedule`, `activity_feedback` |
| `addons/mail/models/mail_alias.py` | `mail.alias`, `mail.alias.mixin`, `get_alias_values` |
| `addons/mail/models/mail_cc_mixin.py` | `mail.thread.cc` (CC management) |
| `addons/mail/models/mail_blacklist.py` | `mail.thread.blacklist` |
| `addons/portal/models/portal_mixin.py` | `portal.mixin`, `_get_share_url`, `get_access_action`, `get_portal_url` |
| `addons/website/models/mixins.py` | `website.seo.metadata`, `website.multi.mixin`, `website.published.mixin`, `website.published.multi.mixin` |
| `odoo/addons/base/models/image_mixin.py` | `image.mixin` and image variants |
| `odoo/addons/base/models/res_partner.py` | `res.partner` inheriting `image.mixin` |
| `addons/utm/models/utm_mixin.py` | `utm.mixin`, `default_get`, `tracking_fields` (13.0 branch) |
| `addons/rating/models/rating_mixin.py` | `rating.mixin`, `rating_send_request`, `rating_apply` (13.0 branch) |
| `addons/rating/models/rating.py` | `rating.rating` fields and access token (13.0 branch) |
| `addons/rating/controllers/main.py` | `/rating/<token>/<rate>` routes (13.0 branch) |

**For more Odoo 13 guides, see [SKILL.md](../SKILL.md)**
