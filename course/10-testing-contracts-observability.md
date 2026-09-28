# Module 10 — Tests, OpenAPI contracts, structured logs, metrics, and traces

[← Module 9](09-streams-integrations-realtime.md) · [Course home](../README.md) · Next: [Performance and deployment →](11-performance-deployment.md)

**Snapshot:** `node:test` and `node:assert/strict` are built into Node 24. Supertest 7.3.0; Testcontainers Node PostgreSQL and Redis modules 12.2.0 (module versions are independent); optional Vitest 5.0.2 requires Node >=22.12 and Vite >=6.4. OpenAPI 3.2.0 is current in this snapshot, but some generators/validators lag; use 3.1.2 if a required tool cannot safely handle 3.2.0.

## Testing pyramid

```text
                   E2E / acceptance
                Register → order → logout

              Integration tests
      Express + real PostgreSQL/Redis + migrations

                  Unit tests
          services, policies, parsers, invariants
```

Test at the cheapest layer that proves the behavior. Do not test Express itself or merely assert private implementation calls. Mocks are valuable for external APIs and failure simulation; fake database mocks cannot prove SQL, transaction isolation, indexes, migration validity, or ORM behavior.

## Built-in test runner and unit test example

```ts
// src/modules/orders/order-policy.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import { canCancelOrder } from "./order-policy.js";

test("customer can cancel own pending order in active tenant", () => {
  assert.equal(canCancelOrder(
    { userId: "u1", organizationId: "org1", role: "member", permissions: new Set() },
    { customerId: "u1", organizationId: "org1", status: "pending" },
  ), true);
});

test("customer cannot cancel another tenant order", () => {
  assert.equal(canCancelOrder(
    { userId: "u1", organizationId: "org1", role: "member", permissions: new Set() },
    { customerId: "u1", organizationId: "org2", status: "pending" },
  ), false);
});
```

Run tests using the supported Node line's test command (for example `node --test dist/**/*.test.js` after compile). Type-check separately. Vitest is a reasonable alternative if watch mode, ecosystem integrations or its runner behavior materially help; do not select it only because frontend projects use it.

## Express API tests

Supertest exercises the actual Express app in-process without requiring a public listener; native `fetch` against a started test server is useful for full HTTP behavior. Keep `app.ts` separate from `server.ts` to avoid listening on import.

```ts
import test from "node:test";
import assert from "node:assert/strict";
import request from "supertest";
import { app } from "../../app.js";

test("unknown product is a stable 404 contract", async () => {
  const response = await request(app).get("/api/v1/products/missing");
  assert.equal(response.status, 404);
  assert.equal(response.body.error.code, "NOT_FOUND");
});
```

The test above is only useful if it starts from a deterministic fixture/database or a documented stub. Avoid tests that pass because a local database happened to be empty.

## Integration tests with real services

Use isolated PostgreSQL (and Redis where relevant). Testcontainers starts disposable containers, but requires a container runtime and cleanup; CI should pin image tags/digests and use bounded startup timeouts. Apply all migrations from empty state, seed explicit fixtures, and verify constraints, uniqueness, rollback, tenant filtering and concurrency. Transaction rollback per test can be fast but may hide behavior that crosses transaction boundaries; choose intentionally.

```ts
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { Client } from "pg";

const postgres = await new PostgreSqlContainer("postgres:18.6").start();
const client = new Client({ connectionString: postgres.getConnectionUri() });
await client.connect();
try {
  const result = await client.query("SELECT 1 AS ok");
  assert.equal(result.rows[0].ok, 1);
} finally {
  await client.end();
  await postgres.stop();
}
```

In the real capstone, use a fresh DB per test worker/suite, apply actual migrations, and connect via the same ORM adapter path as application code. The example uses `pg` for a direct connection smoke test; install it only when the test/app needs it, and pin/check the driver version then. Never run tests against production data.

## End-to-end and security scenarios

A capstone scenario should verify: register → email verification → login → create organization → invite member → create product → create order → duplicate idempotency request → payment webhook repeated → user reads own order → user cannot read another tenant → logout → protected request denied → admin action audited.

Add security cases: no auth, invalid/expired session, malformed JWT, CSRF failure, invalid Origin, invalid JSON, oversized body/file, SQL-injection-shaped input, path traversal, SSRF allowlist rejection, rate-limit exhaustion, membership revoked mid-session, duplicate webhook, refresh-token reuse, concurrent stock purchase, DB/Redis/provider timeout. Test that outputs/logs do not contain secrets.

Use property-based testing selectively with fast-check 4.10.2 (registry snapshot) after checking its current official docs/release compatibility. Good candidates: parsers, price/quantity invariants, pagination cursors, state transition rules, “stock never below zero,” tenant policy across generated IDs. Fuzzing is not a substitute for threat modeling.

## OpenAPI contract

OpenAPI describes the observable API contract; it does not enforce it. Use one source of truth or a check that detects drift. The capstone uses a reviewed YAML/JSON document, then validates examples and generates docs/client only when tooling supports the selected OAS version.

```yaml
openapi: 3.2.0
info:
  title: SaaS Commerce API
  version: 1.0.0
servers:
  - url: https://api.example.com
paths:
  /api/v1/products:
    get:
      summary: List products visible to the caller
      parameters:
        - in: query
          name: limit
          schema: { type: integer, minimum: 1, maximum: 100, default: 25 }
        - in: query
          name: cursor
          schema: { type: string }
      responses:
        "200":
          description: Product page
          content:
            application/json:
              schema:
                type: object
                required: [data, meta]
                properties:
                  data:
                    type: array
                    items: { $ref: "#/components/schemas/Product" }
                  meta: { $ref: "#/components/schemas/PageMeta" }
        "401": { $ref: "#/components/responses/Unauthenticated" }
        "403": { $ref: "#/components/responses/Forbidden" }
        "429": { $ref: "#/components/responses/RateLimited" }
components:
  securitySchemes:
    sessionCookie:
      type: apiKey
      in: cookie
      name: __Host-sid
  schemas:
    Product:
      type: object
      required: [id, name, priceCents]
      properties:
        id: { type: string, format: uuid }
        name: { type: string }
        priceCents: { type: integer, minimum: 0 }
    PageMeta:
      type: object
      properties:
        nextCursor: { type: [string, "null"] }
```

A complete operation documents request schema, response body, status codes/errors, auth mechanism, rate limits/idempotency where relevant, and tenant/role requirement. Do not document a 200 response if the handler returns 201. Runtime validation schemas and OpenAPI schemas can drift; compare them in tests or generate one from the other with tooling whose version supports the chosen dialect.

## Structured logging with request/correlation ID

Pino emits JSON fields instead of string concatenation. Include request ID, method, route template (not sensitive raw query), status, duration, deployment/service version, trace ID where available, and actor ID only if policy allows. Do not log passwords, bearer tokens, cookies, session IDs, reset links, full bodies, payment data, or unnecessary PII. Redact at logger boundary and avoid high-cardinality labels in metrics.

```ts
import pino from "pino";
import { randomUUID } from "node:crypto";
import type { RequestHandler } from "express";

export const logger = pino({
  level: config.logLevel,
  redact: {
    paths: ["req.headers.authorization", "req.headers.cookie", "password", "token", "refreshToken"],
    censor: "[REDACTED]",
  },
});

export const requestLog: RequestHandler = (req, res, next) => {
  const requestId = acceptTrustedRequestId(req.get("x-request-id")) ?? randomUUID();
  res.locals.requestId = requestId;
  res.setHeader("X-Request-Id", requestId);
  const startedAt = process.hrtime.bigint();
  res.on("finish", () => {
    const durationMs = Number(process.hrtime.bigint() - startedAt) / 1e6;
    logger.info({
      requestId,
      method: req.method,
      route: req.route?.path ?? "unmatched",
      statusCode: res.statusCode,
      durationMs: Math.round(durationMs * 100) / 100,
    }, "request completed");
  });
  next();
};
```

A client-supplied request ID can be malicious/oversized; validate its character set/length or generate a fresh one. Propagate context to services and queue jobs; use AsyncLocalStorage or explicit context when justified, not global mutable current-request state. Never use request/user IDs as Prometheus label values.

## Observability: logs, metrics, traces

- **Logs** explain an event with context. Structured fields allow search/correlation.
- **Metrics** show aggregate behavior: request count/rate, duration histogram (p50/p95/p99 from histograms), error rate, DB pool wait, DB query latency, Redis latency, queue depth/age, worker failures, event-loop delay and memory.
- **Traces** link spans across HTTP → service → SQL/Redis → external API → queue/worker, using trace context propagation. A trace is sampled and may be absent; logs should still carry request/trace IDs when available.

OpenTelemetry JS snapshot used in the course: API 1.9.1, Node SDK and OTLP HTTP trace/metric exporters 0.222.0, SDK metrics 2.11.0, auto-instrumentations 0.80.0; each package has its own version stream, so check the installed peer graph rather than assuming matching versions. OpenTelemetry JS supports Node instrumentation. Initialize SDK/instrumentation **before importing application modules**, or libraries may bind no-op providers. Start with HTTP and Express instrumentation, then DB/client instrumentation; verify instrumentation version compatibility and data minimization. Vendor-neutral APIs do not make telemetry free: control cardinality, sampling and sensitive attributes.

Pin this compatible snapshot rather than expecting every OpenTelemetry package to share one version number:

```sh
pnpm add @opentelemetry/api@1.9.1 @opentelemetry/sdk-node@0.222.0 \
  @opentelemetry/auto-instrumentations-node@0.80.0 \
  @opentelemetry/exporter-trace-otlp-http@0.222.0 \
  @opentelemetry/exporter-metrics-otlp-http@0.222.0 \
  @opentelemetry/sdk-metrics@2.11.0
```

```ts
// dist/instrumentation.js — load before server.js with `node --import`.
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(), // endpoint via OTEL_EXPORTER_OTLP_* env or explicit config
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter(),
    exportIntervalMillis: 10_000,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
// During shutdown: await sdk.shutdown() after request/worker telemetry is flushed.
```

```text
request span
 ├─ Express route span
 │   ├─ PostgreSQL query span
 │   ├─ Redis GET span
 │   └─ provider HTTP span
 └─ enqueue producer span ── trace context carried in job ── worker consumer span
```

## Testing external dependencies

Mock payment/email API at an HTTP boundary (MSW or a local mock server) to deterministically simulate timeouts, invalid JSON, retries and 429. Keep database real in integration tests. Avoid mocking `fetch` to such an extent that URL construction/headers/timeouts are never tested. Add contract tests against OpenAPI/provider schemas.

## Common mistakes

- Only unit tests, no real DB tests.
- E2E tests rely on stale shared state.
- Mocked Prisma tests validate call shape but not SQL constraints/transactions.
- OpenAPI exists but isn't checked against implementation.
- Logs include headers/body/token or raw SQL parameters.
- Request ID from arbitrary client is trusted unbounded.
- Metrics labels include user ID, order ID or full URL.
- OpenTelemetry initialized after Express/DB modules loaded.
- A health check calls every dependency at high frequency and causes a load spike.

## Exercises

- **Beginner:** test a pure authorization policy with `node:test` and strict assertions.
- **Intermediate:** test Express 404/validation/error mapping with Supertest and deterministic fixtures.
- **Production:** spin isolated PostgreSQL/Redis, migrate from zero, run duplicate order/webhook and cross-tenant integration cases.
- **Debugging:** intentionally cause N+1 or a provider timeout; trace one request and correlate structured logs without exposing secrets.
- **Architecture:** design an OpenAPI-first vs Zod/code-first contract workflow, including drift detection and version compatibility.

## Build this yourself before looking at a solution

Create the test pyramid and API contract for register/login/products/orders/webhooks. Include trace/log context, redaction policy, metric names/labels, and a deterministic DB fixture strategy. **Expected architecture:** tests verify public behavior and business invariants; real services where semantics matter; contracts are versioned. **Review:** ask whether each test catches a plausible production regression.

## Official documentation

- [Node test runner](https://nodejs.org/docs/latest-v24.x/api/test.html) · [Node assert](https://nodejs.org/docs/latest-v24.x/api/assert.html) · [Supertest](https://github.com/ladjs/supertest)
- [Testcontainers Node](https://node.testcontainers.org/) · [PostgreSQL module](https://node.testcontainers.org/modules/postgresql/) · [Redis module](https://node.testcontainers.org/modules/redis/)
- [OpenAPI 3.2 Specification](https://swagger.io/specification/v3.2/) · [Swagger UI](https://swagger.io/tools/swagger-ui/) · [OpenAPI Initiative](https://www.openapis.org/)
- [Pino](https://getpino.io/#/) · [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/) · [OpenTelemetry Node guide](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/)
- [Vitest 5 requirements](https://vitest.dev/guide/) · [Vitest migration](https://vitest.dev/guide/migration/)

## What I should know before continuing

You can place tests at appropriate pyramid layers, use real PostgreSQL for database semantics, define a contract and detect drift, create redacted request logs, select useful low-cardinality metrics, and trace a request across dependencies without relying on one opaque log message. Next: [Module 11 — production performance and deployment](11-performance-deployment.md).
