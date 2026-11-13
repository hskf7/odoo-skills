---
name: odoo-13-controller
description: Complete reference for Odoo 13 HTTP controllers, routing, authentication types, CSRF, CORS, and request/response handling.
globs: "**/controllers/**/*.py"
topics:
  - Controller basics (class structure, request object)
  - route decorator (URL parameters, route options)
  - Authentication types (user, public, none)
  - Request/Response types (http, json)
  - CSRF and CORS handling
  - File uploads and downloads
  - Session handling
  - Website and portal routes
  - Testing routes with HttpCase
when_to_use:
  - Writing HTTP controllers
  - Creating JSON-RPC and REST-style endpoints
  - Building website pages
  - Handling webhooks and payment callbacks
  - Implementing file uploads/downloads
  - Testing controllers with HttpCase
---

# Odoo 13 Controller Guide

Complete reference for Odoo 13 HTTP controllers, routing, and request handling.

## Table of Contents

1. [Controller Basics](#controller-basics)
2. [@route Decorator](#route-decorator)
3. [Authentication Types](#authentication-types)
4. [Request/Response Types](#requestresponse-types)
5. [Request Object API](#request-object-api)
6. [JSON-RPC vs HTTP Responses](#json-rpc-vs-http-responses)
7. [Error Handling](#error-handling)
8. [File and Binary Responses](#file-and-binary-responses)
9. [Streaming Responses](#streaming-responses)
10. [CSRF Handling](#csrf-handling)
11. [CORS Handling](#cors-handling)
12. [Session Handling](#session-handling)
13. [Website and Portal Routes](#website-and-portal-routes)
14. [Testing Routes with HttpCase](#testing-routes-with-httpcase)
15. [Best Practices](#best-practices)

---

## Controller Basics

### Controller Class Structure

```python
from odoo import http
from odoo.http import request


class MyController(http.Controller):

    @http.route('/my/path', type='http', auth='user')
    def my_handler(self, **kwargs):
        return request.render('my_module.template', {
            'records': request.env['my.model'].search([]),
        })
```

**Key points**:

- Extend `http.Controller` (defined in `odoo/http.py`).
- Use `@http.route(...)` on every public handler method.
- Access `request` (the module-level proxy in `odoo.http`) for env, session, HTTP data.
- Return the right response shape for the route `type`.

### Controller Inheritance / Override

Controllers are extended by normal Python inheritance. When you override a
method, you MUST re-decorate it with `@http.route()`; arguments you omit are
inherited from the parent via the routing map merge in
`odoo.http._generate_routing_rules`.

```python
from odoo import http
from odoo.addons.web.controllers.main import Home


class MyHome(Home):

    @http.route()  # keep path, type, auth from parent
    def index(self, *args, **kw):
        return super(MyHome, self).index(*args, **kw)
```

### Module Layout

```
my_module/
|-- __init__.py
|-- __manifest__.py
`-- controllers/
    |-- __init__.py       # from . import main, api
    |-- main.py
    `-- api.py
```

Remember to `from . import controllers` in the root `__init__.py`.

---

## @route Decorator

### URL Parameters (Werkzeug Converters)

Odoo 13 uses the Werkzeug routing syntax. Built-in converters include `int`,
`float`, `path`, `string` (default), `any`, and `uuid`. Odoo also registers
custom converters in `odoo/addons/base/models/ir_http.py` (`_get_converters`
returns `{'model': ModelConverter, 'models': ModelsConverter, 'int':
SignedIntConverter}`).

```python
from odoo.http import request

# Integer path parameter
@http.route('/order/<int:order_id>', type='http', auth='user')
def order_view(self, order_id):
    order = request.env['sale.order'].browse(order_id)
    if not order.exists():
        return request.not_found()
    return request.render('sale.order_view', {'order': order})

# Model converter - resolves an ID (with slug support) to a recordset
@http.route('/order/<model("sale.order"):order>', type='http', auth='user')
def order_by_model(self, order):
    return request.render('sale.order_view', {'order': order})

# Wildcard / path parameter
@http.route('/download/<path:file_path>', type='http', auth='user')
def download_any(self, file_path):
    ...

# Query string and form parameters arrive as kwargs
@http.route('/search', type='http', auth='user')
def search_orders(self, **kwargs):
    domain = []
    if kwargs.get('name'):
        domain.append(('name', 'ilike', kwargs['name']))
    orders = request.env['sale.order'].search(domain)
    return request.render('sale.order_list', {'orders': orders})
```

### Route Options

The full set of options accepted by the `route` decorator in 13.0 (from the
docstring and `response_wrap.routing` handling in `odoo/http.py`):

| Option     | Values / meaning                                                       |
|------------|------------------------------------------------------------------------|
| `route`    | string or list of strings (the URL pattern(s))                         |
| `type`     | `'http'` or `'json'`                                                   |
| `auth`     | `'user'`, `'public'`, `'none'`                                         |
| `methods`  | sequence of HTTP verbs (`None` = all methods allowed)                  |
| `cors`     | `Access-Control-Allow-Origin` value (e.g. `'*'` or an origin)          |
| `csrf`     | bool; CSRF protection for unsafe methods (default `True`)              |
| `website`  | bool; route participates in website frontend handling                  |
| `sitemap`  | bool or callable; website sitemap enumeration                          |
| `save_session` | bool; persist session changes after the request (default `True`)   |
| `multilang`| bool; website language-prefix handling (default `type == 'http'`)      |

Notes on Odoo 13 specifics:

- `auth` accepts exactly `'user'`, `'public'`, `'none'`; there is no
  `auth='bearer'` (implement token auth manually).
- `methods=None` allows every HTTP verb.
- `csrf` defaults to `True`; `website=True` requires the `website` module;
  `save_session` is consumed in `Root.get_response`.

> Not available in 13.0: the `nolint` route keyword (used in later Odoo
> versions) does not exist in the 13.0 tree.

### Multiple Paths on One Handler

```python
@http.route(['/path1', '/path2'], type='http', auth='public')
def dual(self):
    return "Same handler for both paths"
```

You can also pass multiple route strings in one decorator as the list above
(the idiomatic 13.0 form, used by the multi-path image routes in
`addons/web/controllers/main.py`).

---

## Authentication Types

The three auth modes are implemented in `odoo/addons/base/models/ir_http.py`
(`_auth_method_user`, `_auth_method_public`, `_auth_method_none`, dispatched by
`_authenticate`).

### auth='user' (Default)

Requires an authenticated user. `_auth_method_user` sets
`request.uid = request.session.uid` and raises `SessionExpiredException` when
no user is logged in; unauthenticated requests are redirected to the login
page by `HttpRequest._handle_exception`.

```python
@http.route('/my/orders', type='http', auth='user')
def my_orders(self):
    orders = request.env['sale.order'].search([
        ('user_id', '=', request.env.user.id),
    ])
    return request.render('my_module.orders', {'orders': orders})
```

Behavior:

- `request.env.user` is the logged-in user.
- `request.env.uid` is that user's id.
- Normal ACL / record rules apply.

### auth='public'

Allows unauthenticated access. `_auth_method_public` sets `request.uid` to the
shared `base.public_user` record when no user is logged in, otherwise to the
logged-in user.

```python
@http.route('/shop/products', type='http', auth='public')
def shop_products(self):
    products = request.env['product.product'].search([
        ('website_published', '=', True),
    ])
    return request.render('my_module.shop', {'products': products})
```

Behavior:

- If the visitor is logged in, runs as that user; otherwise as the public user
  (`base.public_user`).
- ACL / record rules still apply; use `.sudo()` only when truly needed.

### auth='none'

No environment, no database authentication. `_auth_method_none` sets
`request.uid = None`.

```python
@http.route('/healthz', type='http', auth='none', csrf=False)
def healthcheck(self):
    return "OK"
```

Behavior:

- `request.env` is NOT available; accessing `request.cr` raises
  `RuntimeError('request not bound to a database')`.
- Route works even when no database is selected.
- Use for static endpoints, health checks, login screens, or manual token
  verification (see the external endpoint pattern below).

---

## Request/Response Types

### type='http' - HTML / Text / Binary

The handler's return value can be (from `HttpRequest.dispatch` and the
`route` decorator's `response_wrap`):

- A falsy value -> `HTTP 204` (No Content).
- A `str` or `bytes` -> wrapped in a `Response` (interpreted as HTML).
- A werkzeug `Response` / `BaseResponse` object -> returned as-is.
- A `werkzeug.exceptions.HTTPException` -> converted to a response.

```python
from odoo.http import request


# Render a QWeb template
@http.route('/page', type='http', auth='user')
def my_page(self):
    return request.render('my_module.template', {
        'records': request.env['my.model'].search([]),
    })

# Plain text
@http.route('/ping', type='http', auth='none')
def ping(self):
    return "PONG"

# Redirect - use werkzeug.utils.redirect (no request.redirect on WebRequest)
@http.route('/go', type='http', auth='user')
def go(self):
    from werkzeug.utils import redirect
    return redirect('/web', 303)
```

> Important 13.0 difference: `request.redirect(...)` is NOT a method on
> `WebRequest`/`HttpRequest`. It only exists on frontend requests as a lambda
> set by the website module (`request.redirect = lambda url, code=302:
> werkzeug.utils.redirect(url_for(url), code)` in
> `http_routing/models/ir_http.py::_dispatch`). In a plain (non-website)
> controller, use `werkzeug.utils.redirect(url, code)`. Module-level helpers
> `odoo.http.local_redirect(path, query=None, keep_hash=False, code=303)` and
> `odoo.http.redirect_with_hash(url, code=303)` also exist.

### 404 / not_found

```python
@http.route('/maybe/<int:rec_id>', type='http', auth='user')
def maybe(self, rec_id):
    rec = request.env['my.model'].browse(rec_id).exists()
    if not rec:
        return request.not_found()
    return request.render('my_module.detail', {'rec': rec})
```

`request.not_found(description=None)` is a shortcut on `HttpRequest` that
returns `werkzeug.exceptions.NotFound(description)`.

---

## Request Object API

`from odoo.http import request`

The proxy object resolves to the current `WebRequest` (an `HttpRequest` or
`JsonRequest`) via a werkzeug `LocalStack`.

### Environment

| Attribute | Meaning |
|---|---|
| `request.env` | `odoo.api.Environment` bound to the current user |
| `request.env.user` | `res.users` recordset (current user) |
| `request.env.company` | Current company (`res.company`) |
| `request.env.companies` | Allowed companies (`res.company` recordset) |
| `request.env.lang` | Language code of the user's context |
| `request.env.uid` | Integer user id (`SUPERUSER_ID` for superuser) |
| `request.env.context` | Context dict (`lang`, `tz`, `uid`, ...) |

`request.env` is a lazy property; it is only created when first accessed. In
13.0 the environment bound to the request is `odoo.api.Environment(self.cr,
self.uid, self.context)`. You can `.sudo()` it like any ORM env:

```python
partner = request.env['res.partner'].sudo().browse(partner_id)
```

### Cursor / Database / Registry

| Attribute | Meaning |
|---|---|
| `request.cr` | `sql_db.Cursor`; raises `RuntimeError` with `auth='none'` |
| `request.db` | Current database name, or `None` with `auth='none'` |
| `request.registry` | The registry for `request.db` (deprecated: prefer `env`) |
| `request.uid` | Current user id (or `None`); settable |
| `request.context` | `frozendict` of context values |

### HTTP Data

| Attribute | Meaning |
|---|---|
| `request.httprequest` | Underlying `werkzeug.wrappers.Request` |
| `request.httprequest.method` | `'GET'`, `'POST'`, ... |
| `request.httprequest.headers` | werkzeug `Headers` |
| `request.httprequest.files` | `FileStorage` dict (multipart uploads) |
| `request.httprequest.remote_addr` | Client IP address |
| `request.params` | Merged query string + form + files (`http` type) |
| `request.jsonrequest` | Parsed JSON body dict (`json` type only) |

For `type='http'`, `params` is an `OrderedDict` built from `args`, `form`, and
`files` in `HttpRequest.__init__`. For `type='json'`, `params` is the
`"params"` object from the JSON-RPC envelope.

### Session

| Attribute | Meaning |
|---|---|
| `request.session` | dict-like `OpenERPSession` object |
| `request.session.uid` | Logged-in user id (or `None`) |
| `request.session.db` | Current database name |
| `request.session['cart_id']` | Custom session entries (JSON-serialisable) |
| `request.session.get('foo')` | Safe read |

### Response Helpers (methods that exist in 13.0)

```python
request.csrf_token(time_limit=3600)              # Fresh CSRF token
request.render(template, qcontext=None, lazy=True, **kw)  # Lazy QWeb render
request.make_response(data, headers=None, cookies=None)   # Raw body + headers
request.not_found(description=None)              # werkzeug 404 exception
```

Module-level helpers in `odoo.http`:

```python
local_redirect('/web', query={'a': 1}, code=303)   # local redirect
redirect_with_hash('/web#section', code=303)        # redirect preserving fragment
send_file(fp, mimetype=..., as_attachment=True)     # file/streaming response
content_disposition(filename)                       # RFC 5987 attachment header
```

> Not available in 13.0: `request.make_json_response(...)` and
> `request.redirect_query(...)` do not exist on the request object in 13.0.
> Use `request.make_response` plus a JSON header, or
> `werkzeug.utils.redirect` directly.

---

## JSON-RPC vs HTTP Responses

### type='json' - JSON-RPC 2.0

In Odoo 13, `type='json'` expects a JSON-RPC 2.0 envelope in the POST body
(`{"jsonrpc": "2.0", "method": "call", "params": {...}, "id": 1}`).
`JsonRequest.__init__` parses the body into `self.jsonrequest`, then copies
`self.jsonrequest.get("params", {})` into `self.params` (a plain dict, not an
array). The handler receives those entries as keyword arguments, and its
return value becomes the JSON-RPC `result`.

`type='json'` behavior:

- `request.jsonrequest` holds the raw parsed JSON body; `request.params` holds
  the `"params"` dict.
- CSRF is not checked for JSON requests (the `HttpRequest.dispatch` CSRF logic
  does not run for `JsonRequest`).
- Exceptions are serialized as JSON-RPC errors by
  `JsonRequest._handle_exception` and `serialize_exception`.

```python
@http.route('/api/records', type='json', auth='user')
def get_records(self, domain=None, limit=80, offset=0):
    domain = domain or []
    records = request.env['my.model'].search_read(
        domain, ['id', 'name', 'state'],
        limit=limit, offset=offset,
    )
    return {'records': records, 'count': len(records)}
```

Calling from the legacy (non-OWL) web client:

```javascript
// Legacy AMD module (Odoo 13 has no OWL)
odoo.define('my_module.records', function (require) {
    "use strict";
    var rpc = require('web.rpc');

    return {
        load: function () {
            return rpc.query({
                route: '/api/records',
                params: { domain: [['state', '=', 'open']], limit: 20 },
            });
        },
    };
});
```

### Error mapping in serialize_exception

`odoo.http.serialize_exception(e)` maps Python exceptions to the
`exception_type` value returned to JSON clients (and also drives the HTTP
error page codes below):

| Python exception                | exception_type       |
|---------------------------------|----------------------|
| `odoo.exceptions.UserError`     | `user_error`         |
| `odoo.exceptions.Warning`       | `warning`            |
| `odoo.exceptions.RedirectWarning` | `warning`          |
| `odoo.exceptions.AccessError`   | `access_error`       |
| `odoo.exceptions.MissingError`  | `missing_error`      |
| `odoo.exceptions.AccessDenied`  | `access_denied`      |
| `odoo.exceptions.ValidationError` | `validation_error` |
| `odoo.exceptions.except_orm`    | `except_orm`         |

All of these live in `odoo/exceptions.py`; `except_orm` is the deprecated
base class of `UserError`, `AccessError`, `CacheMiss`, `MissingError`, and
`ValidationError`. `DeferredException` and `QWebException` also exist there.

---

## Error Handling

### JSON requests

You do not normally need try/except around `type='json'` handlers: the
framework catches the exception in `JsonRequest.dispatch` and serializes it as
a JSON-RPC error. `UserError` and `AccessError` surface to the browser as
user-friendly dialogs.

### HTTP requests and UserError -> 500

For `type='http'`, a `UserError` is not an HTTP response, so it propagates to
`ir.http._handle_exception` and (for frontend requests) renders an error page.
On the backend (non-website) it produces a generic server error page. Handle
expected business errors yourself:

```python
from odoo import http
from odoo.http import request
from odoo.exceptions import UserError, AccessError, ValidationError


class PageController(http.Controller):

    @http.route('/page', type='http', auth='user')
    def my_page(self):
        try:
            return request.render('my_module.template', {
                'data': self._get_data(),
            })
        except UserError as e:
            return request.render('my_module.error', {'message': str(e)})
        except (AccessError, ValidationError):
            return request.not_found()
```

- `AccessError` on a frontend request maps to HTTP `403` (see
  `http_routing/models/ir_http.py::_get_exception_code_values`, which sets
  `code = 403` for `exceptions.AccessError` inside `except_orm`).
- `UserError` maps to HTTP `400` (with a message) on frontend requests.
- `ValidationError` is an `except_orm` subclass; treat it like `UserError`.

---

## File and Binary Responses

### Small payloads with request.make_response

```python
@http.route('/download/report/<int:report_id>', type='http', auth='user')
def download_report(self, report_id):
    report = request.env['ir.actions.report'].browse(report_id)
    pdf, _ = report._render_qweb_pdf(report.report_name, [report_id])
    return request.make_response(pdf, headers=[
        ('Content-Type', 'application/pdf'),
        ('Content-Disposition', 'attachment; filename="%s.pdf"' % report.name),
        ('Content-Length', len(pdf)),
    ])
```

### The `http.send_file` helper (13.0 file/binary helper)

`odoo.http.send_file(filepath_or_fp, mimetype=None, as_attachment=False,
filename=None, mtime=None, add_etags=True, cache_timeout=STATIC_CACHE,
conditional=True)` is the recommended way to serve files or binary fields with
proper caching, ETag, and conditional GET support. It is a modified version of
Flask's `send_file`.

```python
import base64
import io
from odoo import http
from odoo.http import request, send_file


class DownloadController(http.Controller):

    @http.route('/download/attachment/<int:attachment_id>', type='http', auth='user')
    def download_attachment(self, attachment_id):
        attachment = request.env['ir.attachment'].browse(attachment_id).exists()
        if not attachment:
            return request.not_found()
        attachment.check('read')  # ACL
        fp = io.BytesIO(base64.b64decode(attachment.datas or ''))
        return send_file(
            fp, mimetype=attachment.mimetype, as_attachment=True,
            filename=attachment.name,
        )
```

`content_disposition(filename)` returns the RFC 5987 header value
(`attachment; filename*=UTF-8''...`) and is used throughout
`addons/web/controllers/main.py`.

> Not available in 13.0: the `odoo.http.Stream` helper (with
> `Stream.from_attachment`, `Stream.from_binary_field`, `Stream.from_path`)
> was introduced in a later version. In 13.0 use `http.send_file`.

### Binary fields and the ir.http model

The canonical binary-serving pattern delegates to `request.env['ir.http']`:

```python
status, headers, content = request.env['ir.http'].binary_content(
    model='my.model', id=rec_id, field='binary_field',
    download=True, mimetype='application/octet-stream',
)
return request.env['ir.http']._response_by_status(status, headers, content)
```

`binary_content(xmlid=None, model='ir.attachment', id=None, field='datas',
unique=False, filename=None, filename_field='name', download=False,
mimetype=None, default_mimetype='application/octet-stream',
access_token=None)` resolves binary/image fields, attachments, and
company logos with ACL and caching. `_response_by_status(status, headers,
content)` maps status `304` to a not-modified response, `301` to a redirect,
`200` to a response with the headers, and anything else to `request.not_found()`.

---

## Streaming Responses

Odoo 13 streams file contents through `http.send_file`, which uses the WSGI
server's `file_wrapper` when available and falls back to an iterator. Pass a
file object positioned at the start, not a fully materialized `bytes`:

```python
from odoo import http
from odoo.http import send_file


class StreamController(http.Controller):

    @http.route('/export/stream', type='http', auth='user')
    def stream_export(self):
        def generate():
            for chunk in self._iter_chunks():
                yield chunk
        return send_file(generate(), mimetype='text/csv', as_attachment=True,
                         filename='export.csv', add_etags=False)
```

For large multipart uploads, stream the incoming file with
`upload.stream` (a `werkzeug.datastructures.FileStorage`) instead of
`upload.read()`.

---

## CSRF Handling

### Default Behavior

- `type='http'` unsafe methods (POST/PUT/DELETE per RFC 7231) require a
  `csrf_token` form field; GET/HEAD/OPTIONS/TRACE and `type='json'` do not.
- The check runs in `HttpRequest.dispatch` via
  `request.endpoint.routing.get('csrf', True)` and `self.validate_csrf(token)`.

### Form with CSRF Token

```xml
<form action="/my/form/submit" method="POST">
    <input type="hidden" name="csrf_token" t-att-value="request.csrf_token()"/>
    <input type="text" name="email"/>
    <button type="submit">Send</button>
</form>
```

The token is generated by `WebRequest.csrf_token(time_limit=3600)`, which
HMAC-signs the session id plus an expiry using the `database.secret`
`ir.config_parameter`. In JavaScript the token is `require('web.core').csrf_token`.

```python
@http.route('/my/form/submit', type='http', auth='public', methods=['POST'])
def form_submit(self, email=None, **kw):
    # csrf_token already validated by the framework and popped from params
    request.env['newsletter.signup'].sudo().create({'email': email})
    return werkzeug.utils.redirect('/thanks', 303)
```

### Disabling CSRF (Webhooks / Server-to-Server)

Only disable CSRF for endpoints you protect another way (HMAC signature, IP
whitelist, bearer token):

```python
@http.route('/webhook/payment', type='http', auth='none', methods=['POST'], csrf=False)
def payment_webhook(self, **kwargs):
    signature = request.httprequest.headers.get('X-Hub-Signature')
    # Verify HMAC here
    ...
    return "OK"
```

---

## CORS Handling

Set the `cors` option on the route. Odoo 13 emits `Access-Control-Allow-Origin`
(from `Response.set_default`) and serves preflight OPTIONS with compatible
headers (from `HttpRequest.dispatch`). An OPTIONS request to a route with
`cors` returns a 200 with `Access-Control-Max-Age: 86400` and
`Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept`.
`Response.set_default` also sets `Access-Control-Allow-Methods` to `POST`
(json), `GET, POST` (http default), or the route's `methods`.

```python
@http.route('/api/v1/status', type='json', auth='none', cors='*', csrf=False)
def api_status(self):
    return {'status': 'ok'}

# Restrict to a specific origin
@http.route('/api/v1/orders', type='json', auth='user', cors='https://partner.example.com')
def api_orders(self, **kw):
    ...
```

For public APIs, combine `cors='*'` with `csrf=False`.

---

## Session Handling

`request.session` is an `OpenERPSession` (a `werkzeug.contrib.sessions.Session`
subclass) with these defaults: `db`, `uid`, `login`, `session_token`,
`context`, `debug`.

```python
# Read / write
request.session['cart_id'] = 42
request.session.pop('cart_id', None)

# Authentication
request.session.authenticate(db, login=login, password=password)
request.session.logout(keep_db=True)
request.session.check_security()
```

Key behaviors:

- Session data is persisted only if `httprequest.session.should_save` is true
  AND the route does not set `save_session=False` (persistence in
  `Root.get_response`); `save_session=False` skips saving.
- `OpenERPSession.authenticate` stores `db`, `uid`, `login`, `session_token`
  and rotates the session id.
- `save_action` / `get_action` store transient action objects; `save_request_data`
  / `load_request_data` persist POST form data and files across the login
  redirect flow.

> Not available in 13.0: `request.update_env(user=...)`. To switch the
> authenticated user assign `request.uid` directly (the `env` property
> rebuilds the environment lazily).

---

## Website and Portal Routes

### website=True routes

```python
from odoo import http
from odoo.http import request


class WebsiteCatalog(http.Controller):

    @http.route('/shop', type='http', auth='public', website=True, sitemap=True)
    def shop(self, **kw):
        products = request.env['product.template'].search([
            ('website_published', '=', True),
            ('sale_ok', '=', True),
        ])
        return request.render('my_module.shop', {'products': products})
```

On website routes, the website module's `ir.http._dispatch` override adds:

- `request.website` = the current `website` record.
- `request.lang` = the resolved frontend language record.
- `request.is_frontend` = bool flag.
- `request.redirect(url, code=302)` = lambda that applies `url_for` rewriting
  before redirecting (this is the only place `request.redirect` exists in 13.0).

### sitemap

`sitemap=True` enumerates the route in `website.enumerate_pages` (in
`addons/website/models/website.py`), which inspects
`rule.endpoint.routing['sitemap']`. A callable returning a list of `(name,
url)` mappings enumerates dynamic pages; `sitemap=False` excludes the route
and `True` includes it when it has no path converters:

```python
@http.route('/blog', type='http', auth='public', website=True, sitemap=True)
def blog(self, **kw):
    ...
```

### Portal routes

Portal pages are ordinary `website=True` controllers under `/my/...` with
`auth='user'` (or `auth='public'` for specific actions such as
`/portal/attachment/add`):

```python
from odoo import http


class PortalHome(http.Controller):

    @http.route(['/my', '/my/home'], type='http', auth='user', website=True)
    def home(self, **kw):
        return request.render('portal.portal_my_home', {})
```

`request.env.user.share` is `True` for portal users; use it to branch UI or
restrict access. Portal controllers in `addons/portal/controllers/portal.py`
use the `website=True` flag and redirect with `request.redirect(...)` (the
frontend helper).

---

## Testing Routes with HttpCase

`odoo.tests.common.HttpCase` extends `TransactionCase` and provides HTTP-level
test helpers. In 13.0 both `browser_js` and its legacy alias `phantom_js`
exist (`phantom_js = browser_js` in `tests/common.py`).

### Plain HTTP test with url_open

```python
from odoo.tests.common import HttpCase


class TestController(HttpCase):

    def test_public_route(self):
        response = self.url_open('/shop')
        self.assertEqual(response.status_code, 200)

    def test_auth_required(self):
        # Unauthenticated access to a auth='user' route redirects to login
        response = self.url_open('/my/orders')
        self.assertEqual(response.status_code, 303)
```

`url_open(url, data=None, files=None, timeout=10, headers=None)` returns a
werkzeug response object.

### Authenticated request

```python
def test_authenticated(self):
    self.authenticate('admin', 'admin')
    response = self.url_open('/my/orders')
    self.assertEqual(response.status_code, 200)
```

`authenticate(user, password)` logs the test user in and sets the session.

### JSON endpoint test

```python
import json
from odoo.tests.common import HttpCase


class TestJsonEndpoint(HttpCase):

    def test_json(self):
        response = self.url_open(
            '/api/records',
            data=json.dumps({'jsonrpc': '2.0', 'method': 'call',
                             'params': {'limit': 5}, 'id': 1}),
            headers={'Content-Type': 'application/json'},
        )
        self.assertEqual(response.status_code, 200)
        payload = response.json()
        self.assertIn('result', payload)
        self.assertIn('records', payload['result'])
```

### Browser (JavaScript) test

```python
def test_browser_js(self):
    self.browser_js(
        '/web#action=my_module.my_action',
        code="console.log('loaded'); return true;",
        ready="document.querySelector('.o_kanban_view')",
        login='admin',
        timeout=60,
    )
```

`browser_js(url_path, code, ready='', login=None, timeout=60, **kw)` runs the
JavaScript `code` in a headless Chrome and waits for `ready` to be truthy.
`phantom_js` is the deprecated alias kept for backward compatibility.

Other test helpers available in 13.0: `TransactionCase`, `SavepointCase`,
`SingleTransactionCase`, `Form`, `O2MForm`, `M2MProxy`, `@tagged`,
`@at_install`, `@post_install`, `@users`, and `assertQueryCount`.

---

## Best Practices

1. **Keep controllers thin.** Move business logic into models.
2. **Pick the right `type`.** `'http'` for pages/forms/downloads; `'json'` for
   RPC and machine APIs.
3. **Default to `auth='user'`.** Use `'public'` for unauthenticated features,
   `'none'` only to bypass the ORM entirely.
4. **Don't `sudo()` by reflex.** Prefer ACLs / record rules.
5. **Validate inputs** (`int(...)`, whitelists, length checks) and **check
   existence** with `rec.exists()`.
6. **Respect CSRF.** Disable it only when you authenticate another way (HMAC,
   API key, bearer token).
7. **Set `cors` explicitly** for cross-origin JSON APIs, paired with
   `csrf=False`.
8. **Use `send_file`** for downloads (caching, ETag, conditional GET for free).
9. **Raise werkzeug exceptions** (`NotFound`, `Forbidden`, `BadRequest`) rather
   than ad-hoc strings; let `type='json'` exceptions serialize as JSON-RPC
   errors.

---

## Base Code Reference

The APIs documented here are defined in the Odoo 13 source:

- `odoo/http.py` - `Controller`, `route`, `WebRequest`, `HttpRequest`,
  `JsonRequest`, `Response`, `request` proxy, `send_file`,
  `content_disposition`, `local_redirect`, `redirect_with_hash`,
  `serialize_exception`, `OpenERPSession`, `Root`, session dispatching.
- `odoo/addons/base/models/ir_http.py` - `IrHttp` auth methods
  (`_auth_method_user`, `_auth_method_public`, `_auth_method_none`),
  `_authenticate`, `_dispatch`, `_postprocess_args`, `ModelConverter`,
  `binary_content`, `_response_by_status`.
- `odoo/exceptions.py` - `except_orm`, `UserError`, `RedirectWarning`,
  `AccessDenied`, `AccessError`, `CacheMiss`, `MissingError`,
  `ValidationError`, `DeferredException`, `QWebException`.
- `addons/web/controllers/main.py` - reference controllers (`Home`, `Binary`,
  `Database`, image/file serving routes), CSRF and redirect usage examples.
- `addons/http_routing/models/ir_http.py` - frontend `ir.http` override that
  adds `request.website`, `request.lang`, and `request.redirect`.
- `addons/website/models/website.py` - `enumerate_pages` sitemap enumeration.
- `addons/website/models/ir_http.py` - website `_dispatch` override.
- `addons/portal/controllers/portal.py` - portal (`/my/...`) controller
  patterns.
- `odoo/tests/common.py` - `HttpCase`, `url_open`, `authenticate`,
  `browser_js`, `phantom_js`, `ChromeBrowser`.
