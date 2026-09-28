# 00 — Stack, decisions, and complete roadmap

[← Course home](../README.md)

## Documentation-first policy

This course snapshot is dated **2026-09-28**. Official manuals define API semantics; official release pages define lifecycle/status. The npm registry reports published package versions but does not prove a package is the right choice. Before adopting an unfamiliar dependency, verify its official docs, release policy, Node engine range, security posture, peer dependencies, maintenance activity, and migration guide. Pin it in the lockfile and revisit it at planned upgrade intervals.

### Verified reference set

| Area | Snapshot / stability and compatibility | Official reference |
|---|---|---|
| Node release lifecycle | 26.10.0 Current; 24.21.0 latest LTS / Active; 22.x Maintenance | [Node releases](https://nodejs.org/en/about/previous-releases), [Node release schedule](https://github.com/nodejs/release) |
| Node APIs | Use the Node 24 API documentation for baseline; identify newer APIs separately | [Node 24 API docs](https://nodejs.org/docs/latest-v24.x/api/), [globals](https://nodejs.org/api/globals.html), [HTTP](https://nodejs.org/api/http.html), [streams](https://nodejs.org/api/stream.html), [test](https://nodejs.org/api/test.html) |
| Express | 5.2.1 stable; 5.x is the course line | [Express docs](https://expressjs.com/), [Express 5 migration](https://expressjs.com/en/guide/migrating-5/), [security](https://expressjs.com/en/advanced/best-practice-security.html), [performance/reliability](https://expressjs.com/en/advanced/best-practice-performance/) |
| TypeScript | 7.0.2 latest stable; **6.0.3 is course baseline**; `@types/node` 24.19.0 / `@types/express` 5.0.6; `typescript-eslint` 8.71.0 declares TypeScript `>=4.8.4 <6.1.0`; Biome 2.5.14 docs list TypeScript 5.9. Choose TS 6.0.3 for a compatible ecosystem, then revisit TS 7 when the toolchain supports it. | [Handbook](https://www.typescriptlang.org/docs/), [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html), [TypeScript 7.0.2 release](https://github.com/microsoft/TypeScript/releases/tag/v7.0.2), [current TypeScript download](https://www.typescriptlang.org/download/), [Node module guidance](https://www.typescriptlang.org/docs/handbook/modules/theory.html) |
| Schema validation | Zod 4.6.5 stable snapshot | [Zod docs](https://zod.dev/), [Zod v4 release/migration notes](https://zod.dev/v4) |
| PostgreSQL | 18.6 is the current stable minor; 19 Beta 4 is prerelease | [Version policy/current minor](https://www.postgresql.org/support/versioning/), [PostgreSQL 18 docs](https://www.postgresql.org/docs/18/), [18.6 release](https://www.postgresql.org/about/news/postgresql-186-1711-1615-1519-1424-and-19-beta-3-released-3365/), [19 beta notice](https://www.postgresql.org/about/news/postgresql-19-beta-4-released-3386/) |
| ORM | Prisma CLI / Client / PostgreSQL adapter 7.10.0 stable line; Prisma ORM 8 is RC (CLI 8.0.0-rc.17 and `@prisma/orm-postgres` 8.0.0-rc.13 on 2026-09-28), expected GA in October | [Prisma release status](https://www.prisma.io/docs/orm/release-status), [Prisma ORM 7 docs](https://www.prisma.io/docs/orm/v7), [Prisma transactions](https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions) |
| Redis server/client | Redis Open Source 8.10 release line (announced 2026-09-14); node-redis package `redis` 6.2.1 snapshot; Redis recommends node-redis over legacy ioredis for new work | [Redis Node client](https://redis.io/docs/latest/develop/clients/nodejs/), [ioredis migration](https://redis.io/docs/latest/develop/clients/nodejs/migration/) |
| Queue | BullMQ 6.3.9 snapshot; v6 node-redis adapter supports `redis` v5+; verify backend/migration details when introducing it | [BullMQ docs](https://docs.bullmq.io/), [BullMQ API](https://docs.bullmq.io/api), [changelog](https://docs.bullmq.io/changelog) |
| Helmet | 8.3.0 snapshot | [Helmet docs](https://helmetjs.github.io/), [Express security guidance](https://expressjs.com/en/advanced/best-practice-security.html) |
| Pino | 10.3.1 snapshot | [Pino documentation](https://getpino.io/#/) |
| OpenTelemetry Node | API 1.9.1; Node SDK and OTLP HTTP exporters 0.222.0; SDK metrics 2.11.0; auto-instrumentations Node 0.80.0 snapshot; Versions are independent; align through official peer/dependency metadata and initialize before app imports | [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/), [Node setup](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/) |
| Property testing | `fast-check` 4.10.2 optional; Use for invariants/parsers only when generated cases add value | [fast-check docs](https://fast-check.dev/docs/introduction/) |
| Local environment-file helper | `dotenv` 18.0.4 optional for `prisma.config.ts`; production config comes from the platform | [dotenv documentation](https://github.com/motdotla/dotenv) |
| PostgreSQL driver | `pg` 8.23.0 (when using raw driver / Prisma adapter) | [node-postgres docs](https://node-postgres.com/) |
| Express / Node types | `@types/express` 5.0.6; `@types/node` 24.19.0 | [Express types](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/express), [Node types](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/node) |
| Authentication helpers | `argon2` 0.45.1; `jose` 6.2.12 for optional JWT; `express-session` 1.19.0; `connect-pg-simple` 10.0.0 for PostgreSQL-backed session option; Maintained password hashing / JOSE implementation; opaque sessions remain application/session-store policy | [argon2](https://github.com/ranisalt/node-argon2), [jose](https://github.com/panva/jose), [express-session](https://github.com/expressjs/session), [connect-pg-simple](https://github.com/voxpelli/node-connect-pg-simple) |
| CORS middleware | `cors` 2.8.6 (optional); Browser CORS headers only; never authorization | [Express CORS guide](https://expressjs.com/en/resources/middleware/cors.html) |
| OpenAPI UI | `swagger-ui-express` 5.0.1; Serves interactive docs from a reviewed OpenAPI document; Express 5 peer range verified | [Swagger UI Express](https://github.com/scottie1984/swagger-ui-express), [OpenAPI 3.2](https://swagger.io/specification/v3.2/) |
| API type generation | `openapi-typescript` 7.13.0 (optional); Generate consumer types from contract; generation does not validate runtime input | [openapi-typescript](https://openapi-ts.dev/) |
| Lint/format | ESLint 10.11.0 + typescript-eslint 8.71.0 + Prettier 3.9.9; Biome 2.5.14 alternative; typescript-eslint peer range is TS `<6.1.0`; Biome docs list TS 5.9; choose the compatible TS 6.0.3 baseline | [typescript-eslint](https://typescript-eslint.io/), [ESLint](https://eslint.org/docs/latest/), [Prettier](https://prettier.io/docs/), [Biome TypeScript support](https://biomejs.dev/internals/language-support/) |
| Tests | Built-in `node:test` is baseline; Vitest 5.0.2 is optional (Node >=22.12, Vite >=6.4); Supertest 7.3.0; Testcontainers PostgreSQL/Redis modules 12.2.0 | [Node test runner](https://nodejs.org/api/test.html), [Vitest guide](https://vitest.dev/guide/), [Supertest](https://github.com/ladjs/supertest), [Testcontainers Node](https://node.testcontainers.org/) |
| OpenAPI | Specification 3.2.0 current snapshot; tooling compatibility can lag | [OpenAPI 3.2](https://swagger.io/specification/v3.2/), [Swagger UI](https://swagger.io/tools/swagger-ui/) |
| Observability | OpenTelemetry JS SDK and instrumentation; initialize before application modules | [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/), [Node guide](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/) |
| WebSocket | `ws` 8.22.0 snapshot | [ws repository / API](https://github.com/websockets/ws), [Node HTTP upgrade](https://nodejs.org/api/http.html#event-upgrade) |
| Package manager | pnpm 12.6.0 stable snapshot | [pnpm 12 release](https://pnpm.io/blog/releases/12.0), [pnpm docs](https://pnpm.io/) |
| Containers | Docker Engine + Compose (select supported release from Docker's current release channel when implementing) | [Docker docs](https://docs.docker.com/), [Compose](https://docs.docker.com/compose/) |

For an exact package version, the repository recorded a registry check with commands equivalent to:

```sh
node --version
pnpm --version
pnpm view express version
pnpm view prisma@7.10.0 version
pnpm view @prisma/client@7.10.0 version
pnpm view @prisma/adapter-pg@7.10.0 version
pnpm view zod version
```

A registry's `latest` tag can be a prerelease or incompatible with an existing project. On 2026-09-28, an unpinned Prisma CLI install selects the Prisma 8 RC; Prisma 8 also uses different packages and has documented gaps. The course therefore pins Prisma CLI, `@prisma/client`, and `@prisma/adapter-pg` to stable 7.10.0 and uses the v7 docs. The OpenTelemetry snapshot is informational: add telemetry packages only after selecting an exporter/backend, and keep the SDK/auto-instrumentation peer graph compatible.


### Stability and support labels used in this course

| Label | Meaning here | Course policy |
|---|---|---|
| Stable / Active LTS | Supported production baseline in the project's current release channel | Use after compatibility/security review; pin supported patches |
| Maintenance LTS | Supported line focused on maintenance/security | Compatibility target for existing systems, not the preferred new baseline |
| Current | New Node major before LTS | Test compatibility and learn new APIs; not default production runtime |
| Experimental / Beta / Release Candidate | API/product can change; not general production baseline | Explain only as a current ecosystem signal; do not build capstone on it |
| Deprecated | Maintainer advises migration; API may remain temporarily | Identify old usage and teach replacement |
| Removed | API no longer exists in selected major | Never use in new code; document migration |

**Snapshot examples:** Node 24 is Active LTS; Node 22 is Maintenance LTS; Node 26 is Current. Express 5 is the stable course framework line; Express 4 is in its announced maintenance period. PostgreSQL 18.6 and Prisma ORM 7 are stable; PostgreSQL 19 Beta 4 and Prisma ORM 8 RC are not the stable production baseline. For individual Node APIs use their `Stability` label in the Node 24 docs; avoid deprecated APIs and never mistake a newer Node 26 API for a Node 24 capability.

## Technology decision matrix

| Problem | Selected path | Alternative | Decision trigger |
|---|---|---|---|
| HTTP framework | Express 5 | Fastify / NestJS | Choose Fastify for schema-first hooks/serialization or Nest for enforced modular conventions and DI. Express is intentionally flexible. |
| Request validation | Zod at boundaries | Valibot / Ajv | Use Ajv when JSON Schema is the source contract or validation throughput is a measured bottleneck. |
| Persistence | PostgreSQL + Prisma 7 | Drizzle / `pg` SQL | Prefer Drizzle for SQL-shaped control; raw `pg` for explicit SQL and minimal abstraction. Never let ORM familiarity substitute for SQL. |
| Session/auth | Opaque server-side sessions in secure cookies as the first implementation | Signed JWT access/refresh tokens | JWT when independent verifiers or cross-service token use justifies revocation/key/rotation complexity. |
| Cache | Cache-aside Redis only for measured repeated reads | HTTP/reverse-proxy cache; PostgreSQL | Prefer HTTP cache for public cacheable representations; avoid caching volatile authorization-sensitive reads without a clear invalidation policy. |
| Jobs | BullMQ for Redis-backed jobs | PostgreSQL outbox + worker, managed queue | Use a simpler DB job table when operational footprint outweighs advanced scheduling/throughput. |
| Logging | Pino structured JSON | Platform logger | Pick whichever provides redaction, correlation and stable structured fields. |
| Tests | `node:test`, real PostgreSQL integration tests, Supertest/native fetch | Vitest, Testcontainers | Add runner convenience only if it materially improves workflow. Use Testcontainers where CI can run a container engine. |
| Contract | OpenAPI 3.2 (fall back to 3.1 if ecosystem support requires) | Code-first route metadata | Contract-first for external/parallel teams; code-first when schemas are a single source of truth and generation remains reliable. |
| File storage | S3-compatible object storage + short-lived signed upload/download URLs | Local disk for local development | Never rely on container filesystem for durable user uploads. |
| Deployment | One container per API or worker process, managed platform | VM/systemd, serverless, orchestrator | Use orchestration only when operations and service scale need it. |

## Course roadmap and learning outcomes

### Part I — Runtime and protocol foundations
1. **Node runtime:** V8, native bindings, libuv, event loop, thread pool, async model, buffers, streams, worker threads and child processes. Build an execution-order model and measure event-loop blocking.
2. **HTTP/networking:** DNS, TCP concepts, TLS, HTTP messages, methods, headers, cookies, status codes, content negotiation, caching and request lifecycle. Build a raw `node:http` API before Express.
3. **Express 5:** Application lifecycle, router stack, middleware order, body parsers, route syntax, request/response APIs, async error propagation, trust proxy and Express 4 migration.

### Part II — Application boundaries and data
4. **TypeScript at runtime boundaries:** strict Node types, ESM, `unknown`, discriminated errors, Express augmentation, Zod parsing of body/query/params/headers/env/provider responses, config validation and safe errors.
5. **API design and architecture:** REST resources, status choices, pagination, filtering/sorting/search, idempotency, versioning, controllers/services/repositories, feature modules, clean architecture/DDD concepts with restraint.
6. **SQL and PostgreSQL:** schema/constraints, joins/aggregations/CTEs/window-function concepts, indexes, plans, connection pools, migrations, isolation, locking, `EXPLAIN`, JSONB and full-text search concepts.
7. **ORM and transactions:** Prisma 7 setup/relations/CRUD/migrations/raw SQL, ORM-vs-SQL tradeoffs, N+1, order transaction, concurrency race conditions, idempotency and outbox.

### Part III — Identity, security and tenant boundaries
8. **Security foundations:** OWASP risk patterns, TLS, headers/Helmet, input validation, SQL/command injection, SSRF, XSS, CSRF, CORS, cookies, proxy trust, rate limiting, brute-force defenses and safe uploads.
9. **Authentication:** password hashing (Argon2id/bcrypt library selection after checking maintained docs), registration/login/logout/current-user/password reset/email verification, secure cookies, sessions, JWT claims, refresh rotation/reuse detection, revocation and key rotation.
10. **Authorization and SaaS tenancy:** RBAC, permissions, policies, role/ownership checks at route + service + database boundary, organization membership/invitations, tenant isolation, audit logs, organization/user quotas.

### Part IV — Distributed work and I/O
11. **Redis and cache:** key design, TTL, cache-aside, invalidation, session/rate-limit options, pub/sub/streams concepts, Redis availability/degradation, data ownership and stale data.
12. **Queues/reliability:** separate API and worker processes, BullMQ, retries/backoff/jitter, job idempotency, dead-letter/poison-job handling, timeouts, circuit breakers, outbox, delivery semantics and duplicate webhook/payment protection.
13. **Streams and integrations:** Node/Web Streams, Buffer, `pipeline`, backpressure, streaming exports/uploads, native `fetch` + AbortController, file validation/object storage, transactional email, webhook HMAC/replay control, SSE/WebSockets/reconnection/scaling.

### Part V — Verification and operations
14. **Testing:** unit tests for business rules; integration tests with real PostgreSQL/Redis; API/e2e flows, Testcontainers, fixtures, security tests, contract checks, property tests for invariants, external-service mocks without mocking the database everywhere.
15. **Contracts and observability:** OpenAPI, validation-contract parity, generated clients tradeoffs, request/correlation IDs, Pino redaction, metrics/traces/spans, OpenTelemetry propagation, safe error mapping and audit events.
16. **Performance and operations:** p50/p95/p99, load tests (k6/autocannon after checking current releases), event-loop utilization, query plans, connection pools, payload and logging cost, memory/resource leaks, graceful degradation, health/readiness/liveness and graceful shutdown.
17. **Delivery and deployment:** pnpm/lockfiles, dependency audits/SBOM, Docker/Compose, CI checks, migrations/zero-downtime changes, secrets, environments, reverse proxy/TLS, scaling, restart policy, process model, backups and runbooks.
18. **Capstone review:** product requirements, domain model, API contract, schema, threat model, architecture, tests, load plan, deployment/runbook. Submit your proposed design first; then receive a multi-discipline review.

## Capstone phases (incremental build plan)

1. Raw Node HTTP server; 2. Express setup; 3. routing; 4. middleware; 5. runtime validation; 6. error handling; 7. module boundaries; 8. PostgreSQL; 9. authentication; 10. authorization; 11. products; 12. orders; 13. transactions/concurrency; 14. Redis; 15. cache policy; 16. queue; 17. worker; 18. webhooks; 19. uploads; 20. SSE/realtime; 21. tests; 22. OpenAPI; 23. security hardening; 24. observability; 25. performance/load test; 26. Docker; 27. deployment/CI; 28. architecture review.

The complete product brief and learner design worksheet are in [Module 12](12-capstone-prd.md).

## Recurring review checklists

### API design
URL and method? Safe/idempotent? Request/response schemas? Status and error contract? Authentication? Authorization and tenant scope? Retry/idempotency? Pagination? Cache policy? Deprecation plan?

### Database
Constraints? Index? Transaction boundary? Lock/race? Tenant predicate? N+1 risk? Query plan and expected result size? Pool capacity? Migration rollback/forward plan?

### Security
Can an attacker read another tenant's data, forge/replay a token, bypass ownership, brute-force login, exploit reset/verification, upload hostile bytes, SSRF an internal service, inject SQL/shell, trigger unbounded work, or exhaust a pool/worker?

### Performance and operations
Does this block the event loop? Does it fan out queries? Can it stream? Does it need a queue? What timeout/retry budget applies? What happens if PostgreSQL, Redis, queue, object storage or email is down? Can the service drain on SIGTERM? What metric/log/trace diagnoses failure?
