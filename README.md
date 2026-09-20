# http v2.0.0 — Express-style backend framework for Umbral

Build HTTP backends in Umbral: routers with chained routes, sub-router mounting,
typed controllers, middlewares, CORS, CSRF tokens, rate limiting, cookies and a
minimal reverse proxy. Outgoing calls to external APIs use the native `pulse`
client directly.

Requirements: Umbral `>=1.5.2` and `ump`.

## Installation

```bash
ump add http
```

## Quick start

```umbral
equip { Router, Request } origin 'http';

f: listUsers(req, res, next) {
    res.status(200, ["ok" => true]);
    r: (null);
}

c: router = n: Router();
router.route("/users").get(listUsers);

c: req = n: Request("/users", "GET");
c: result = awa: router.execute("/users", "GET", req);
tprint(result);
```

## Application server with Http

`Http` is the equivalent of `express()`: instead of calling a function,
instantiate it with `n: Http()` to reach the full configuration and the
server. Register routes exactly like on a `Router`, then `onReady()`:

```umbral
equip { Http } origin 'http';

f: root(req, res, next) {
    res.json(["estado" => "activo", "mensaje" => "API de Umbral"]);
    r: (null);
}

f: ready(port) {
    tprint("Servidor corriendo en el puerto &port");
}

c: aplicacion = n: Http();
c: puerto = 3000;

aplicacion.get("/", root);
awa: aplicacion.onReady(puerto, ready);
```

Minimal initial server, Express style with a callback:

```umbral
equip { Http } origin 'http';

c: http = n: Http();
c: puerto->Int = 3000;

f: ready(port) {
    tprint("Servidor corriendo en el puerto &port");
}

http.get("/", root);
awa: http.onReady(puerto, ready);
```

Three rules: every statement ends with `;`, callbacks are named functions
(Umbral has no anonymous functions, and they cannot capture outer variables,
so they receive what they need as arguments), and `onReady()` runs with `awa:`.

Notes:

- Handlers are named functions (Umbral has no anonymous arrow functions).
- `onReady()` serves on the configured host (`127.0.0.1` by default,
  change it with `app.set("host", "0.0.0.0")`, read it with
  `app.setting("host")`).
- `onReady(port, ready)` starts the server and fires `ready(port)` once
  listening, before accepting. There is a single merged method: no separate
  `listen`.
- `onReady()` blocks forever serving one connection at a time, so call it
  last with `awa:`. The callback is optional: `awa: app.onReady(3000)`.
- `Http` mirrors the `Router` API: `route()`, `get/post/put/delete/patch/
  options()`, `run()` (mounts, `use` equivalent), plus `execute()` for tests
  without sockets and `listRoutes()`.
- Bodies with `Content-Type: application/json` arrive parsed; otherwise raw
  text. Query strings land in `req.query`, `:param` segments in `req.params`.
- `set(name, value)` stores any setting (`get()` stays the HTTP verb, so the
  reader is called `setting(name)`).

## Router

One router per file keeps use cases modular: import `Router`, instantiate it,
register routes, mount it elsewhere.

```umbral
equip { Router } origin 'http';

c: router->Router = n: Router();

router.route("/login").post(employeeController);
router.run("/parameters", parametersRouter);
router.run("/api", auth, internalsRoutes);
```

### Chained routes

`route(path)` returns a `Route` bound to the router. Verbs register on the
owner and return the `Route` for chaining. Express order everywhere:
middlewares first, handler last.

```umbral
router.route("/users").get(listUsers);
router.route("/users").post(auth, createUser);
```

### Sub-routers with run()

`run()` is the equivalent of `use` in Express. Valid shapes:

```umbral
router.run(childRouter);
router.run("/prefix", childRouter);
router.run("/prefix", middleware, childRouter);
```

Rules: the child `Router` is always the last argument. A path, when present,
always comes first. Middlewares are only allowed after a path.

### Direct verbs

```umbral
router.get("/status", statusController);
router.post("/users", auth, createUser);
router.put("/users/:id", updateUser);
router.delete("/users/:id", removeUser);
router.patch("/users/:id", patchUser);
router.options("/users", optionsController);
```

### Dispatch

```umbral
c: result = awa: router.execute("/users/42", "GET", req);
```

Local routes run first, then mounted routers by prefix. Path params use
`:param` segments. A missing match returns a `404` entry. Use `listRoutes()`
to inspect local routes.

## Controllers

Handlers run as `fn(request, res, next)` and may be sync or async. Extra
arguments are ignored, so one-argument handlers keep working. Use named
functions (Umbral has no anonymous functions).

```umbral
equip { Router, Request, Response, Next, QueryParams, HttpError } origin 'http';
equip { UsersService } origin './users.service.um';

c: usersService = n: UsersService();

asy f: findAll(req->Request, res->Response, next->Next) {
    c: filters->QueryParams = req.query;
    c: response = awa: usersService.findAll(filters);
    i: (response == null) {
        next.withError(n: HttpError(500, "fetch failed", null));
        r: (null);
    }
    res.status(200, response.json());
}

asy f: findById(req->Request, res->Response, next->Next) {
    c: id = req.getParam("id");
    c: response = awa: usersService.findById(id);
    i: (response == null) {
        res.status(404, ["error" => "User not found"]);
        r: (null);
    }
    res.status(200, response.json());
}
```

Notes:

- Import service classes and instantiate them locally. Importing a singleton
  instance does not carry its methods.
- `response.json()` works on entries and lists (native). Services should
  return plain data, not class instances.
- `tw:` inside `asy` functions does not propagate (the runtime returns null).
  Signal failures with return values plus `next.withError()`, or with `res`
  directly. Sync throws become `500` entries automatically.

## Request

Incoming message built per request. Filled by the router with `:param` values.

| Member | Description |
|--------|-------------|
| `url->Str` | Server path. |
| `method->Str` | HTTP verb in uppercase. |
| `headers` | Header pairs list. |
| `body` | Body data, or null. |
| `params->PathParams` | Route params. Annotates cleanly. |
| `query->QueryParams` | Query values. Annotates cleanly. |
| `setHeader(name, value)` | Appends a header pair, chainable. |
| `getHeader(name)` | Header value, or null. |
| `setParam(name, value)` / `getParam(name)` | Route params. Prefer `getParam("id")` over `req.params.id`: dictionaries are immutable, so dynamic key access is unavailable. |
| `setQuery(name, value)` / `getQuery(name)` | Query values. |
| `setBody(body)` / `getBody()` | Body access. |

```umbral
c: filters->QueryParams = req.query;
c: page = req.getQuery("page");
c: id = req.getParam("id");
```

## Response

Per-request response. Return `null` from the handler to send its entry, or
return a value to send that value instead.

| Member | Description |
|--------|-------------|
| `code->Int`, `message->Str`, `data`, `headers` | Mutable state. |
| `status(code)` / `status(code, body)` / `status(code, body, message)` | Sets status, chainable. |
| `json(data)` / `send(body)` | Sets the body, chainable. |
| `setHeader(name, value)` / `getHeader(name)` | Header pairs. |
| `toJson()` | Serializes the entry. |
| `ok/created/accepted/noContent/badRequest/unauthorized/forbidden/notFound/methodNotAllowed/conflict/internalServerError/notImplemented/serviceUnavailable/gatewayTimeout(data, message)` | Factory helpers returning a new Response. |

```umbral
res.status(200, ["users" => users]);
res.status(404, ["error" => "missing"]);
res.setHeader("Set-Cookie", "session=abc123");
```

## Next

Flow control received as `fn(request, res, next)`.

| Member | Description |
|--------|-------------|
| `continue()` | Marks the chain as continued. |
| `withError(error)` | Fails the dispatch with a 500 entry. |
| `wasCalled()` / `hasError()` / `getError()` | State readers. |
| `reset()` | Back to clean state. |

A middleware returning `false` blocks with `403`. Any other value continues.

## Middlewares

Zero-config, Express style. Register globally or per route:

```umbral
equip { logger, cors, securityHeaders } origin 'http';

router.run("/api", cors, apiRouter);
router.get("/admin", logger, adminController);
```

| Middleware | Description |
|------------|-------------|
| `logger(req, res, next)` | Logs timestamp, method and url. |
| `cors(req, res, next)` | Permissive CORS headers (open defaults). Preflight needs an explicit OPTIONS route or server level support. |
| `securityHeaders(req, res, next)` | Helmet style baseline headers. |

Per-route options are unavailable: Umbral has no closures, so a factory like
`cors(options)` cannot capture config. Custom middlewares are plain named
functions returning `true` (or `false` to block with `403`).

## Security helpers

Shared state lives in store instances owned by the user file: create one and
wire it explicitly. JWT stays in its own library, databases in theirs.

```umbral
equip { TokenStore, RateStore, issueToken, verifyToken, checkRate, getCookie } origin 'http';

c: tokens = n: TokenStore();
c: limit = n: RateStore(100, 60000);

f: showForm(req, res, next) {
    c: token = issueToken(tokens, 60000);
    res.setHeader("Set-Cookie", "csrf=&token");
    res.status(200, ["csrf" => token]);
    r: (null);
}

f: submitForm(req, res, next) {
    i: (verifyToken(tokens, req.getHeader("X-CSRF-Token")) != true) {
        res.status(403, ["error" => "bad csrf"]);
        r: (null);
    }
    res.status(200, ["ok" => true]);
    r: (null);
}

f: guarded(req, res, next) {
    i: (checkRate(limit, "127.0.0.1") != true) {
        res.status(429, ["error" => "slow down"]);
        r: (null);
    }
    res.status(200, ["ok" => true]);
    r: (null);
}
```

| Member | Description |
|--------|-------------|
| `TokenStore` / `issue(ttlMs)` / `consume(token)` | Opaque single-use tokens with expiry. CSRF grade (`Std.random()` entropy, not auth grade). |
| `RateStore(limit, windowMs)` / `hit(key)` | Fixed-window buckets. |
| `issueToken(store, ttlMs)` / `verifyToken(store, token)` | CSRF double-submit helpers. |
| `checkRate(store, key)` | True when allowed, false when over the limit. |
| `getCookie(req, name)` | Cookie value, or null. |
| `forward(target, req, headers)` | Minimal reverse proxy over native `pulse`. Forwards method, path and body; headers need an explicit literal entry. |

## Shared types

```umbral
equip { QueryParams, PathParams, HttpError } origin 'http';
```

`QueryParams` and `PathParams` wrap entries in a list, so typed annotations
validate (`c: filters->QueryParams = req.query;`). `HttpError(status, message,
details)` travels through `tw:` (sync) and `next.withError()`, serializes with
`toJson()`.

## External APIs and proxy

Outgoing calls use native `pulse` directly, or `forward()` to proxy:

```umbral
c: upstream = awa: pulse("https://api.example.com/users", "GET");
tprint(upstream.status);

c: proxied = awa: forward("http://127.0.0.1:9000", req, null);
```

## Docs

Every class, method, function and property carries umdocs:

```bash
umbral --doc src/Router.um
```

## Changelog

### v1.1.0

- Application server: `Http` with `get/post/put/delete/patch/options()`,
  `route()`, `run()`, settings and blocking `onReady(port, ready)` over
  native `Net` sockets (HTTP/1.1 parse and render included).
- Express style routing: `route()` chaining, `run()` sub-router mounting
  (`use` equivalent), verbs with Express order (middlewares first, handler last).
- Typed controllers as `fn(request, res, next)` with `Response`/`Next` per dispatch.
- Shared types: `QueryParams`, `PathParams`, `HttpError`.
- Middlewares: `logger`, `cors`, `securityHeaders`.
- Security helpers: `TokenStore`, `RateStore`, CSRF, rate limit, cookies, `forward` proxy.
- `Response` headers plus `status/json/send` chainable API.
- Removed the `fetch` style client: external calls use native `pulse`.
- Requires Umbral `>=1.5.2`.
