# Module 3 — Express 5: middleware, routing, and HTTP boundaries

[← Module 2](02-http-node-http.md) · [Course home](../README.md) · Next: [Types, validation and architecture →](04-types-validation-architecture.md)

**Baseline:** Express 5.2.1 stable snapshot, TypeScript on Node 24 LTS. Use Express 5 documentation and migration guide—not Express 4 examples as the primary pattern.

## Concept — an ordered control-flow framework

Express does not magically “call a controller.” A request enters an ordered stack. Each middleware can enrich request context, send a response, end the request, pass control with `next()`, or pass an error with `next(error)`. A route handler that sends a response must not also continue to another response-producing handler.

```text
HTTP Request
    ↓
app middleware (request ID → security → parsing → logs)
    ↓
mounted router middleware (auth → tenant → validation)
    ↓
route handler(s) → controller → application service
   ↙         ↓                 ↘
respond   next()             throw/rejected Promise
                                 ↓
                         error-handling middleware
                                 ↓
                             response
```

Middleware is control flow. Ordering changes behavior and security. A 404 handler belongs after routes; error middleware belongs after routes and must have the error signature.

## Express application lifecycle

`express()` creates an application function/configuration. `app.use()` mounts middleware; `app.METHOD()` registers route handlers (`app.get`, `app.post`, `app.put`, `app.patch`, `app.delete`, `app.options`, `app.head`); `express.Router()` composes a sub-router. An explicit `HEAD` route takes precedence; absent one, Express can use a matching `GET` handler for `HEAD`. `app.listen()` creates/listens on a Node HTTP server (it is a convenience wrapper). In Express 5, its callback receives a listen error as an argument; handle/report it rather than assuming bind failures will be thrown from the callback path. For graceful shutdown, keep the returned server handle—`const server = app.listen(port, "0.0.0.0")`—and call `server.close()` after marking readiness false and draining work.

```ts
// src/app.ts — deliberately small composition root
import express from "express";
import type { ErrorRequestHandler, RequestHandler } from "express";

const app = express();
app.disable("x-powered-by");

app.use(express.json({ limit: "100kb", strict: true }));
app.use(express.urlencoded({ extended: false, limit: "50kb", parameterLimit: 100 }));

const health: RequestHandler = (_req, res) => {
  res.status(200).json({ status: "ok" });
};
app.get("/health", health);

app.use((_req, res) => {
  res.status(404).json({ error: { code: "NOT_FOUND", message: "Route not found" } });
});

const errors: ErrorRequestHandler = (err, req, res, next) => {
  // logger.error({ err }, "request failed"); // wire a redacted structured logger here later
  if (res.headersSent) return next(err);
  res.status(500).json({ error: { code: "INTERNAL_ERROR", message: "Unexpected server error" } });
};
app.use(errors);

export { app };
```

The logger call is illustrative; wire the structured logger in a later module. Never return exception messages/stacks in production.

## Middleware types and composition

- **Application middleware:** `app.use(fn)` for request IDs, Helmet, parsers, logging, shared auth.
- **Router middleware:** `router.use(fn)` scoped to a router/mount prefix.
- **Built-in middleware:** `express.json`, `express.urlencoded`, `express.static`.
- **Third-party middleware:** Helmet, CORS configuration, session store, rate limiter—select and configure intentionally.
- **Error middleware:** `(err, req, res, next)`; place last. If headers are already sent, delegate to Express/default handling rather than writing a second response.

```ts
import { Router } from "express";
import type { RequestHandler } from "express";

// In the integrated app, module 7 declares this property via Express type augmentation.
declare global {
  namespace Express {
    interface Request { user?: { id: string } }
  }
}

const requireAuthentication: RequestHandler = (req, res, next) => {
  if (!req.user) {
    res.status(401).json({ error: { code: "UNAUTHENTICATED" } });
    return;
  }
  next();
};

const products = Router();
products.use(requireAuthentication); // all routes below this point require auth
products.get("/", listProducts);
products.post("/", createProduct);

app.use("/api/v1/products", products);
```

`next()` advances to the next matching callback. `next("route")` skips remaining callbacks for the current route in supported route-handler configurations. Do not call `next()` after sending a response. A mount path affects how sub-router paths compose; the browser-visible URL is the mount prefix plus route path.

## Express 5 async behavior

Express 5 forwards a rejected Promise from a route handler or middleware to error handling. A synchronous `throw` in a handler is also caught. The error middleware still maps the failure to the correct safe response.

```ts
// Express 5: no catch(next) wrapper needed for ordinary rejected promises.
app.get("/api/v1/products/:id", async (req, res) => {
  const product = await productService.getById(req.params.id);
  if (!product) throw new NotFoundError("Product");
  res.json({ data: product });
});
```

| Older Express 4-era pattern | Express 5 pattern |
|---|---|
| `asyncHandler(fn)` wrapper catches `Promise.reject` then calls `next` | Return/await the Promise; rejection is automatically passed to error flow |
| `try { await x } catch (e) { next(e) }` repeated in every handler | Throw expected typed errors; centralized error middleware maps them |
| Ignore async errors from detached callbacks | Detached work is not returned to Express; catch it explicitly and route it to a job/error reporting boundary |

An async wrapper can remain useful in shared framework-neutral abstractions or legacy Express 4 code. It is redundant for normal Express 5 handlers and can obscure control flow. Express only observes the Promise returned by a handler; it cannot catch an error from a timer/event callback that the handler did not return.

## Routing: modern path syntax

Express 5 uses a newer `path-to-regexp` grammar. Do not copy Express 4 wildcard/optional syntax.

```ts
app.get("/users/:userId", getUser);
app.get("/files/*splat", (req, res) => {
  // Wildcard capture is an array of path segments in Express 5.
  res.json({ segments: req.params.splat });
});
app.get("/{*splat}", rootAndNestedFallback); // includes `/`
app.get("/reports{.:format}", reportHandler); // optional extension group
```

- Wildcards must be named: `/*splat`; use `/{*splat}` if the root path should match too.
- Optional segments use braces, e.g. `/reports{/:year}` rather than the old `?` form.
- RegExp-style character groups in string paths are no longer supported; use separate explicit routes or a regular-expression route when necessary.
- Express 5 updated to path-to-regexp 8.x; path parameters are decoded for use, so validate them and do not use raw values as filesystem paths or SQL identifiers.
- String-path `req.params` has a null prototype; use own-property-safe access or the known parameter name, not assumptions about inherited object methods.

## Request object: data comes from different places

| Property | Meaning / caution |
|---|---|
| `req.params` | Matched path parameters; strings or wildcard arrays; runtime validation required. |
| `req.query` | Parsed query object; attacker-controlled and parser behavior/config matters. Express 5 exposes it as a getter; do not assign a replacement. |
| `req.body` | Parsed body only when matching parser middleware ran; in Express 5 it is `undefined` when not parsed. Treat as unknown until validated. |
| `req.headers` / `req.get(name)` | Incoming headers; values may be absent and untrusted. |
| `req.cookies` | Exists only if a cookie parser/session middleware populated it; cookie input is not inherently authenticated. |
| `req.ip`, `req.protocol`, `req.hostname` | May be derived from forwarded headers when `trust proxy` is configured. Trust only actual proxy hops/networks. |
| `req.originalUrl`, `req.baseUrl`, `req.path`, `req.method` | Request target and routing context; avoid logging query secrets. |
| `req.user`, `req.auth`, `req.tenant`, `req.requestId` | Application-added properties require TypeScript declaration merging and disciplined middleware order. |

Never let client-provided `X-Forwarded-For`, `X-Forwarded-Proto`, or `Host` choose identity, tenant, callback URL, or security policy without a trusted boundary.

## Response object

Use `res.status(201).json(...)` for a JSON response; `res.set()` / `res.type()` for explicit headers/types; `res.end()` for ending without a body. `res.send()` handles general payloads. `res.location()` sets `Location`; `res.redirect()` redirects browsers/clients. `res.cookie()` / `res.clearCookie()` require cookie options and must be used with secure settings. Clear cookies using the same name, path, and domain scope as the cookie being removed; Express 5 ignores `maxAge` and `expires` passed to `res.clearCookie()`. `res.download()` / `res.sendFile()` need a constrained root and authorization; never pass user-controlled absolute paths. For large data use a stream and handle client disconnect/stream errors rather than loading everything into memory.

### Express 4 → Express 5: important changes

| Area | Express 4 old behavior/pattern | Express 5 modern behavior |
|---|---|---|
| Async errors | Rejected handler Promises were not automatically forwarded by Express 4; wrappers/catch-next common | Rejected Promise from a handler/middleware is forwarded to error middleware |
| Route paths | Unnamed `*`, `?`, and embedded regexp-like syntax often appeared in tutorials | Wildcards need names; optional groups use `{}`; path-to-regexp v8 grammar; some special chars reserved |
| `req.body` | Commonly defaulted to `{}` without parsed body | Is `undefined` if no parser populated it |
| `req.query` parser | Extended parser was the common default | Default parser is `simple`; property is a getter, so configure `app.set("query parser", ...)` deliberately rather than assigning `req.query` |
| `app.param(fn)` | Callback-based signature appeared in older apps | Removed; use `app.param(name, callback)` or explicit validation middleware |
| Wildcard `req.params` | Often treated as a single string | Wildcard match is an array of path segments |
| Removed `req.param(name)` | Looked across route/query/body and obscured source | Removed; use `req.params`, `req.query`, or validated `req.body` explicitly |
| JSON helpers | `res.json(body, status)`, `res.send(body, status)` examples | Removed; use `res.status(status).json(body)` / `.send(body)` |
| Redirect | `res.redirect(url, status)` old argument order | `res.redirect(status, path)` or `res.status(status).redirect(path)`; magic `'back'` removed—resolve a validated Referrer or safe fallback |
| Deprecated/removed methods | `app.del`, `req.param`, old pluralization aliases, removed signatures | Use `app.delete`, explicit API methods and current signatures |
| URL-encoded parser | `extended` defaults often relied upon | Default is `false`; set deliberately and cap body size/parameter count/depth |
| Listen callback | Many examples assume bind errors throw elsewhere | The callback receives an error argument in Express 5; handle `EADDRINUSE` and startup failure explicitly |
| `res.clearCookie` | Clearing examples often set `maxAge` / `expires` | Those options are ignored by `clearCookie`; use matching cookie scope/security attributes, and test browser behavior |
| Static/send file options | `hidden`/`from` aliases shown in older examples | Use current `dotfiles` and `root` options; default dotfile handling is `ignore` |
| Promise behavior | Callback-focused assumptions | Promise-returning handlers supported; detached async work still needs explicit handling |

Run the official migration guide and codemods on a copy/branch, inspect the diff, then test routing and error semantics. A codemod is not a security review.

## Body parsing and content-type boundaries

Configure only parsers the API uses. `express.json({ limit: "100kb" })` is not a multipart upload parser. URL-encoded forms are distinct from JSON. `express.raw()` is useful when exact bytes are needed, e.g. signed webhooks; mount it only on those endpoints before JSON parsing. `express.text()` is for text payloads. Multipart upload requires a dedicated parser or direct-to-object-storage signed URL architecture. Limits protect memory and CPU; parser errors must map safely to client errors.

## Bad example → production pattern

**BAD:**

```ts
app.post("/orders", async (req, res) => {
  const product = await db.product.findUnique({ where: { id: req.body.productId } });
  // No schema check, authentication, tenant/ownership policy, stock transaction,
  // idempotency, error mapping, or body size limit.
  const order = await db.order.create({ data: { productId: product!.id } });
  res.send(order);
});
```

**Better boundary:** route wiring establishes middleware; schema turns `unknown` into typed input; controller translates HTTP; service owns business invariants; repository scopes tenant and uses transaction/constraints; global error middleware maps known errors. Modules 4–8 implement those parts.

## Common mistakes

- Middleware registered after the route it was supposed to protect.
- A parser mounted after a route reads `req.body`.
- Error middleware placed before routes or declared with only three parameters.
- Calling `next()` after `res.json()`.
- Catching Express 5 rejections redundantly but failing to catch detached callbacks.
- Treating `req.query`, `req.params`, cookie values, and forwarded headers as trusted types.
- Using wildcard/optional path grammar copied from Express 4.
- Trusting proxy headers from all clients with `app.set("trust proxy", true)`.
- Returning stack traces or internal database messages to clients.

## Security notes

Mount security headers early; apply body and parameter limits; validate route/query/header/body values; disable `x-powered-by`; use explicit CORS origins; authorize each resource including tenant scope; keep raw webhook verification separate; and treat `sendFile` paths, redirects, and forwarded headers as security-sensitive. CORS is not access control.

## Performance notes

Do not call synchronous APIs on a request path. Set a production `NODE_ENV`; put compression/caching at a reverse proxy when appropriate; avoid huge JSON serialization; use streams for large bodies; keep middleware lean; set bounded database pools; and measure route-level latency plus event-loop and DB time. Express itself rarely dominates a slow endpoint compared with an N+1 query or downstream timeout.

## Exercises

- **Beginner:** add a router mounted at `/api/v1`; explain the final URL for each route.
- **Intermediate:** add a named wildcard fallback and verify `/` versus `/a/b` params.
- **Production:** design parser configuration for JSON API, URL-encoded login form, and payment webhooks requiring raw bytes.
- **Debugging:** deliberately put auth middleware after a route, reproduce an authorization bypass, then fix and add a regression test.
- **Architecture:** decide which checks are app-wide, router-wide, route-specific, service-level, and database-level; explain why no single middleware can replace all layers.

## Build this yourself before looking at a solution

Create a versioned Express 5 router with `GET /products`, `GET /products/:id`, and `POST /products`. Add 404, async not-found, parser-size handling and a terminal error handler. Run tests for middleware order, invalid content type, malformed JSON, path matching and rejected Promise behavior. **Expected architecture:** app composition separate from server startup; routers are transport composition; service code does not import Express. **Review:** check every route has explicit auth/validation/error behavior.

## Official documentation

- [Express 5 API](https://expressjs.com/en/5x/api.html) · [Routing](https://expressjs.com/en/guide/routing.html) · [Using middleware](https://expressjs.com/en/guide/using-middleware.html)
- [Error handling](https://expressjs.com/en/guide/error-handling.html) · [Request API](https://expressjs.com/en/5x/api.html#req) · [Response API](https://expressjs.com/en/5x/api.html#res)
- [Express 4 → 5 migration](https://expressjs.com/en/guide/migrating-5/) · [Production security](https://expressjs.com/en/advanced/best-practice-security.html) · [Production performance/reliability](https://expressjs.com/en/advanced/best-practice-performance/)
- [Express 5.1 became the npm default / support policy announcement](https://expressjs.com/en/blog/2025-03-31-v5-1-latest-release/) · [Current Express releases](https://github.com/expressjs/express/releases)

## What I should know before continuing

You can diagram the middleware stack, state when `next()` versus `next(error)` is appropriate, explain Express 5 Promise rejection forwarding, use Express 5 path syntax, describe parser order and limits, enumerate trust-sensitive `req` properties, and distinguish framework HTTP concerns from service/business logic. Next: [Module 4 — TypeScript, runtime validation, errors, and architecture](04-types-validation-architecture.md).
