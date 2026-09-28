# Module 4 — TypeScript, validation, errors, and application boundaries

[← Module 3](03-express5.md) · [Course home](../README.md) · Next: [PostgreSQL and transactions →](05-postgres-sql-transactions.md)

**Snapshot:** TypeScript 6.0.3 is the course baseline; TypeScript 7.0.2 is the latest stable release ([official release](https://github.com/microsoft/TypeScript/releases/tag/v7.0.2)), but the current typed ESLint parser declares support only for TypeScript `<6.1.0`, so the course does not pretend that its lint toolchain supports TS 7. Revisit this as the ecosystem catches up. Zod 4.6.5 promotes formats such as email to top-level schemas (`z.email()`); older tutorials may show chain methods such as `z.string().email()`, which are deprecated in Zod 4. Verify this API at the time of installation.

## Concept — static types are not runtime validation

TypeScript checks code during development/build and erases types in emitted JavaScript. The Internet still sends bytes. `req.body as CreateOrderInput` is an assertion, not a check. Start untrusted input as `unknown`, parse it at the boundary, and pass a typed value into application logic.

```text
HTTP bytes → JSON parse → unknown value → schema validation → typed input
                                                         ↓
                                       controller → service → repository
```

Validate body, query, params, relevant headers, environment configuration, webhook payloads, and responses from external APIs. Validation answers “is this shape/value acceptable?” Authorization answers “may this identity perform this operation?” They are separate.

## TypeScript for a Node service

Recommended baseline: ESM (`"type": "module"` in `package.json`), `NodeNext` module resolution, strict type-checking, explicit output directory. Build with `tsc`; use a dev runner only for development and still run `tsc --noEmit` in CI. Node's native TypeScript type stripping has limits and does not type-check; this course does not rely on it as the build pipeline.

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true,
    "verbatimModuleSyntax": true,
    "types": ["node"],
    "sourceMap": true
  },
  "include": ["src/**/*.ts"]
}
```

Use `unknown` for caught errors and external values; narrow via `instanceof`, guards, or schemas. Use `never` to prove exhaustive unions. Prefer discriminated unions for state transitions; do not overuse `any`, non-null `!`, or broad casts to silence a problem.

## Runtime validation with Zod

```ts
import { z } from "zod";

export const createProductSchema = z.strictObject({
  name: z.string().trim().min(1).max(120),
  priceCents: z.number().int().nonnegative().max(100_000_000),
  sku: z.string().trim().min(1).max(64),
  categoryId: z.uuid(),
});

export type CreateProductInput = z.infer<typeof createProductSchema>;

export const listProductsQuerySchema = z.strictObject({
  search: z.string().trim().max(100).optional(),
  category: z.uuid().optional(),
  priceMin: z.coerce.number().int().nonnegative().optional(),
  priceMax: z.coerce.number().int().nonnegative().optional(),
  sort: z.enum(["createdAt", "name", "priceCents"]).default("createdAt"),
  order: z.enum(["asc", "desc"]).default("desc"),
  limit: z.coerce.number().int().min(1).max(100).default(25),
});
```

Be intentional about coercion: `z.coerce.number()` can turn `""` into `0`; reject or preprocess empty strings if that is not valid API behavior. Strict objects reject unexpected fields; permissive objects may strip or retain them depending schema. Document the choice. Keep validation errors useful but do not echo secrets or enormous user input.

### A typed Express boundary

```ts
import type { RequestHandler } from "express";
import { z } from "zod";
import { createProductSchema } from "./product.schemas.js";
import { productService } from "./product.service.js";

export const createProduct: RequestHandler = async (req, res) => {
  const parsed = createProductSchema.safeParse(req.body as unknown);
  if (!parsed.success) {
    throw new ValidationError(parsed.error.issues.map((issue) => ({
      path: issue.path.join("."),
      message: issue.message,
    })));
  }

  const product = await productService.create(parsed.data);
  res.status(201).location(`/api/v1/products/${product.id}`).json({ data: product });
};
```

`req.body` is often typed as `any` by framework type declarations; the `as unknown` step deliberately drops that unsafe assumption before parsing. The parsed schema result—not the original body—is the only value passed to the service.

For bigger route sets, create reusable `validateBody`, `validateQuery`, and `validateParams` middleware that store parsed values in typed `res.locals` or a typed request context. Keep the generic machinery modest: a schema per request boundary is more valuable than a clever validation framework no one can debug.

## Error architecture

Errors should communicate a safe, expected failure category to the HTTP boundary without leaking internals. Separate expected operational outcomes (validation, unauthenticated, forbidden, missing resource, conflict, dependency timeout) from programmer defects (broken invariant, null dereference, impossible state). Both are logged, but only known errors get a specific client response.

```ts
export type ErrorDetail = { path?: string; message: string };

export class AppError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly statusCode: number,
    public readonly details?: ErrorDetail[],
    options?: ErrorOptions,
  ) {
    super(message, options);
    this.name = new.target.name;
  }
}

export class ValidationError extends AppError {
  constructor(details: ErrorDetail[]) { super("VALIDATION_ERROR", "Request validation failed", 422, details); }
}
export class UnauthorizedError extends AppError {
  constructor() { super("UNAUTHENTICATED", "Authentication required", 401); }
}
export class ForbiddenError extends AppError {
  constructor() { super("FORBIDDEN", "Operation is not allowed", 403); }
}
export class NotFoundError extends AppError {
  constructor(resource = "Resource") { super("NOT_FOUND", `${resource} not found`, 404); }
}
export class ConflictError extends AppError {
  constructor(message = "Resource conflicts with current state") { super("CONFLICT", message, 409); }
}
export class RateLimitError extends AppError {
  constructor() { super("RATE_LIMITED", "Too many requests", 429); }
}
```

Avoid blindly serializing `message` from low-level exceptions. An ORM error may reveal table names; a driver error may include host information; a fetch error may include internal URLs. In production, log a sanitized structured error with request ID and cause, then return a stable envelope.

```ts
import type { ErrorRequestHandler } from "express";

export const errorHandler: ErrorRequestHandler = (err: unknown, req, res, next) => {
  if (res.headersSent) return next(err);

  if (err instanceof AppError) {
    // logger.warn({ err, requestId: res.locals.requestId }, "request rejected");
    res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        ...(err.details ? { details: err.details } : {}),
      },
    });
    return;
  }

  // logger.error({ err, requestId: res.locals.requestId }, "unhandled request failure");
  res.status(500).json({
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred",
    },
  });
};
```

### HTTP error mapping policy

| Domain failure | HTTP response | Notes |
|---|---:|---|
| Invalid syntax/shape | 400 or 422 | Pick one validation policy and document it. Malformed JSON is commonly 400; semantically invalid field values often 422. |
| Invalid/missing credentials | 401 | Avoid confirming whether an account exists. |
| Authenticated but disallowed | 403 | A privacy policy may return 404 for concealed resources. |
| Resource absent | 404 | Keep tenant/resource scope in lookup. |
| Duplicate/idempotency/state conflict | 409 | Return a stable machine-readable code. |
| Unsupported media | 415 | Include accepted content types in API docs. |
| Rate/quota exceeded | 429 | Add `Retry-After` when meaningful. |
| Third-party unavailable | 502/503/504 | Distinguish invalid upstream response, unavailable dependency, and deadline exceeded. |
| Unexpected defect | 500 | Generic client message; detailed redacted server-side diagnostics. |

## Practical REST design, pagination, and versioning

Treat a URL as a resource identifier and the HTTP method as the operation. Prefer `/products/:id` over action-shaped `/getProduct`; use action subresources only when the business operation is genuinely a command (for example, `POST /orders/:id/cancel`). Keep nesting shallow and use explicit relationships when a child has a lifecycle of its own.

- **Filtering/search:** use validated query parameters (`category`, `priceMin`, `priceMax`, `search`). Bound search length, result count and query cost.
- **Sorting:** expose a finite enum; map values to known columns, never concatenate raw query values into SQL.
- **Pagination:** offset is easy for small/admin pages; cursor pagination is more stable/efficient for large changing result sets. Cursors need deterministic ordering and tenant scope.
- **Idempotency:** PUT/DELETE should be designed around idempotent intended effects; retry-sensitive POST operations use an idempotency key and stored result.
- **Response envelope:** `{ data, meta }` helps attach pagination/links consistently; direct resources are smaller for simple endpoints. Choose a consistent API-level policy, not a dogma. Never wrap error and success payloads inconsistently.
- **API versioning:** URL `/api/v1` is visible and simple to route/document. Header/media-type versioning keeps URLs stable but is less discoverable and adds caching/tooling complexity. Version only for breaking contract changes; compatible optional fields/additive endpoints usually do not require a new version. Publish deprecation window, migration guide, sunset date and telemetry before removal.

A version number is not permission to break contracts within that version. Backward compatibility includes status codes, field semantics, pagination defaults, error codes and auth behavior—not only JSON property names.

## Architecture — code belongs at the right boundary

```text
Express route → controller (HTTP mapping)
              → service (use case / business invariants)
              → repository interface (data access contract)
              → Prisma / PostgreSQL (infrastructure)
```

- **Controller:** read validated transport input, call one use case, choose HTTP status/headers/response. No giant SQL/business rules.
- **Service:** business policy and operation boundaries. Does not import Express types or write HTTP responses.
- **Repository:** persistence queries, tenant predicate, transaction integration and mapping between DB and application types.
- **Domain:** invariants/value objects/entities where they simplify business rules; not mandatory boilerplate for a CRUD prototype.

A tiny service may use route handlers and direct DB access. A medium application benefits from feature modules. A large codebase benefits from explicit boundaries and dependency inversion where independent logic or teams justify it. “Clean architecture” is about controlling dependencies, not creating a class for every function.

Feature-first layout:

```text
src/
  app.ts                 # express composition, no listen
  server.ts              # startup, connections, listen, shutdown
  config/                # startup validation
  shared/{errors,logger,db,redis,middleware}/
  modules/
    products/{product.routes.ts,product.schemas.ts,product.controller.ts,
              product.service.ts,product.repository.ts,product.test.ts}
    orders/...
```

TypeScript dependency direction should point toward business/application contracts; Express and Prisma are outer details. Do not create an interface solely to mock a class in a unit test. Use an interface when it marks a real boundary or permits a meaningful alternative.

## Typed environment configuration

Fail before listening if required configuration is missing/invalid. Environment variables are strings; do not scatter `process.env.X!` across modules.

```ts
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "staging", "production"]),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.url(),
  REDIS_URL: z.url().optional(),
  CORS_ORIGINS: z.string().default(""),
  SESSION_SECRET: z.string().min(32),
});

const parsed = envSchema.safeParse(process.env);
if (!parsed.success) {
  console.error("Invalid runtime configuration", parsed.error.issues.map(({ path, message }) => ({ path, message })));
  process.exit(1);
}
export const config = parsed.data;
```

In production, secrets come from a platform secret store, not committed `.env` files. Avoid printing the entire environment on startup. Parse comma-separated origin lists explicitly and compare normalized origins—not substring matches.

## Type augmentation

Declaration merging can add `Request.user`, `Request.tenant`, and `Request.requestId`, but augmentation only tells the compiler what your middleware guarantees. It does not initialize the runtime property. Prefer an explicit `res.locals` context or a discriminated authenticated-request helper when that makes the nullable boundary visible. Keep augmentation declarations in an included `.d.ts`, and test middleware order.

## Bad → production example

**BAD:** a controller reads `req.body` with an assertion, applies business policy, queries Prisma, formats a response, sends an email, and catches every error as 400.

**WHY:** assertions are not validation; HTTP is coupled to domain behavior; expected and unexpected errors become indistinguishable; retries can duplicate side effects; unauthorized tenant queries can leak data.

**Production shape:** parser → Zod parse → auth and policy → controller → service → transaction/repository → outbox/queue → stable response. Modules 5–8 add persistence, identity, tenant scoping, and async work.

## Common mistakes

- `as MyType` on untrusted request values.
- Catching `unknown` as if it were always an `Error`.
- Catch-all `catch { res.status(400) }` which labels bugs as client mistakes.
- Returning database/provider error text directly.
- Reimplementing business validation in controller and service with drift.
- Building deep generic abstractions before a second use case exists.
- Using a validation library but validating only body, not query/params/config/provider results.
- Assuming augmentation populates a property at runtime.

## Security and performance notes

Validate and bound all external values; avoid detailed auth errors; sanitize error logs; keep secrets out of validation issue payloads. Validation itself costs CPU, so cap payload size and measure high-volume schemas. Do not apply `z.coerce` blindly. Parsed data should be minimal/whitelisted so unknown fields cannot overwrite privilege-bearing fields (`role`, `organizationId`, `price`).

## Exercises

- **Beginner:** define request schemas for registration, product filtering, and order creation; list rejected boundary cases.
- **Intermediate:** implement a service-layer `ConflictError` and test 409 mapping independently of Express.
- **Production:** define configuration schema for DB, Redis, session, CORS and object storage; ensure startup fails without a secret and does not log it.
- **Debugging:** a malformed query causes an ORM exception and 500. Reproduce, validate query parameters, and ensure malformed JSON and domain conflicts map differently.
- **Architecture:** sketch a minimal version with no repository interface, then identify the concrete pressure that would justify a repository/service boundary.

## Build this yourself before looking at a solution

Implement `POST /products` and `GET /products` as one feature module. Validate body/query, map known errors globally, and keep the service callable from a unit test without Express. Add an environment parser. **Expected architecture:** transport-specific controller, independent use case, persistence still replaceable. **Code review:** no unchecked `req.body`, no `any` escape hatch, no internal error leakage, and no unvalidated `sort` string reaching SQL.

## Official documentation

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) · [Current TypeScript release](https://www.typescriptlang.org/download/) · [Node module theory](https://www.typescriptlang.org/docs/handbook/modules/theory.html)
- [Zod](https://zod.dev/) · [Zod v4](https://zod.dev/v4) · [Express 5 TypeScript types are community-maintained `@types/express`](https://www.npmjs.com/package/@types/express)
- [Express error handling](https://expressjs.com/en/guide/error-handling.html) · [Node `process.env`](https://nodejs.org/docs/latest-v24.x/api/process.html#processenv)
- [Prisma Client error handling](https://www.prisma.io/docs/orm/v7/prisma-client/debugging-and-troubleshooting/handling-exceptions-and-errors)

## What I should know before continuing

You can explain type erasure; treat request/env/provider inputs as `unknown`; parse to inferred types; separate validation, authentication and authorization; map expected errors safely; articulate controller/service/repository responsibilities; and validate configuration before listening. Next: [Module 5 — SQL, PostgreSQL and transactions](05-postgres-sql-transactions.md).
