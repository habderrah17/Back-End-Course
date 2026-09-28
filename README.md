# Production Backend Engineering with Node.js, Express 5, and TypeScript

> A documentation-first, progressive course for designing, building, securing, testing, operating, and evolving backend systems—not just writing routes.
>
> **Course context and ecosystem snapshot:** 28 September 2026 (UTC). Release facts were checked against the official project pages and package registries on this date. Recheck version/status before starting a new project; patch releases continue to change.

## Start here

1. Read this overview and [the current stack / decision matrix](course/00-stack-and-roadmap.md).
2. Start with [Module 1: The Node.js runtime](course/01-node-runtime.md). It intentionally begins before Express.
3. Build each phase of the capstone incrementally. Do not skip the raw HTTP server or relational SQL chapters.
4. At the end of [the capstone PRD](course/12-capstone-prd.md), submit your design before asking for an implementation review. The course intentionally asks you to reason before showing a full solution.

## Course philosophy

A backend is a set of cooperating runtime, network, application, data, security, and operations boundaries. A professional engineer should be able to explain what each boundary guarantees, what it cannot guarantee, and how failures cross it. We will prefer stable platform APIs first, add dependencies only when they solve a real problem, and mark time-sensitive choices rather than pretending that package versions stay current forever.

For each concept, lessons answer: **what / why / where it runs / how / when to use it / when not to / tradeoffs / failure modes / debugging / security / scaling**. Every substantial module includes code, a deliberately bad pattern, a production-minded pattern, exercises at multiple levels, an architecture challenge, official references, and a readiness checklist.

## Backend engineering mindset

- **Correctness before cleverness:** define invariants, status codes, transaction boundaries, and retry behavior.
- **Trust boundaries are real:** TypeScript types disappear at runtime; every external input needs parsing and validation.
- **Identity is not permission:** authentication identifies a principal; authorization answers whether that principal may act on this resource in this tenant.
- **The database is part of the design:** constraints and transactions protect invariants even when application code races or retries.
- **Failures are normal:** dependencies time out, processes restart, messages duplicate, and clients disconnect.
- **Measure before optimizing:** latency percentiles, query plans, event-loop delay, pool saturation, and error rates beat intuition.
- **Scale state deliberately:** multiple API processes need shared sessions, rate-limit state, cache policy, and job ownership.
- **Keep boundaries proportional:** a modular monolith is often the best starting point; abstraction without a use case is a cost.

## Current release status and verified stack

### Node.js release status (checked 2026-09-28)

| Version | Status | Use in course | Why |
|---|---|---|---|
| Node.js 26.10.0 | Current; not LTS | Feature-awareness and selected API comparisons only | Current releases are useful for early testing, but the Node project recommends Active or Maintenance LTS for production. |
| **Node.js 24.21.0** | **Latest LTS; Active LTS** | **Production-oriented baseline** | Current LTS line and compatible with the selected stack. Pin the exact patch in CI/container images and update deliberately. |
| Node.js 22.x | Maintenance LTS | Compatibility/testing where required | Still LTS-supported, but in maintenance; install the current official patch when maintaining existing services. |
| Node.js 20 and older | EOL as of this snapshot | Not a course baseline | Do not start a new production service on an unsupported line. |

**Terms:** *Current* is the short, fast-moving phase before LTS; *LTS* means long-term support and includes Active and Maintenance phases; *Active LTS* receives broader ongoing fixes/features; *Maintenance LTS* focuses on critical fixes/security; *EOL* receives no normal support. Node's official release page is authoritative for exact dates and patch versions.

### Stack decision matrix

Exact versions below are a reproducible **snapshot**, not a promise that you should install these versions forever. Stable releases are preferred. A candidate tagged RC/beta/experimental is not used as a production default. Native Node APIs are dependencies too—in the sense of runtime contract—but need no npm package.

| Problem | Course choice / verified snapshot | Why | Alternative / when it fits |
|---|---|---|---|
| Runtime | Node.js 24.21.0 Active LTS | Production support window; native HTTP, streams, fetch, crypto, tests, workers | Node 26 Current for compatibility testing and evaluation of newer APIs, not the baseline |
| HTTP framework | Express 5.2.1 stable | Requested framework; mature ecosystem; explicit middleware and routing | Fastify for schema-centric/high-throughput needs; NestJS for stronger conventions/DI in a larger team |
| TypeScript / types | TypeScript 6.0.3 course baseline; `@types/node` 24.19.0, `@types/express` 5.0.6; TypeScript 7.0.2 is latest stable ([official release](https://github.com/microsoft/TypeScript/releases/tag/v7.0.2)) | TS 7 is newer, but `typescript-eslint` 8.71.0 declares TypeScript `<6.1.0`; use TS 6.0.3 for a supported typed-lint toolchain and evaluate TS 7 when the ecosystem catches up | JavaScript for tiny scripts; types still do not validate incoming data |
| Package manager | pnpm 12.6.0 stable | Reproducible lockfile, workspace support, explicit package-manager pin | npm is a strong default and broadly available; keep one package manager per repo |
| Runtime schema validation | Zod 4.6.5 stable | TypeScript inference, readable parsing and errors; use at every untrusted boundary | Valibot for smaller modular API; Ajv for JSON Schema/OpenAPI-first validation |
| Relational database | PostgreSQL 18.6 stable | Mature SQL, transactions, constraints, indexing, JSONB and strong tooling; current minor per PostgreSQL release policy | PostgreSQL 17.11 when hosted-provider compatibility lags; PostgreSQL 19 is beta in this snapshot, not a production course target |
| ORM | Prisma CLI, `@prisma/client`, and `@prisma/adapter-pg` 7.10.0 stable line | Readable typed client, migrations, broad docs and a practical teaching experience | Prisma ORM 8 is Release Candidate in this snapshot (its CLI and PostgreSQL client packages are separately versioned, and v8 has API gaps); unpinned `prisma` selects the RC. Explicitly pin stable Prisma 7.10.0. Drizzle when SQL-shaped TypeScript queries are preferred |
| Redis server/client | Redis Open Source 8.10 release line; `redis` (node-redis) client 6.2.1 | Redis 8.10 was announced 2026-09-14; Redis official docs recommend node-redis | ioredis for existing systems needing its behavior; Redis docs describe it as the older client to migrate from |
| Queue | BullMQ 6.3.9 | Mature job abstraction; retries, delayed jobs, concurrency; Redis backend in the capstone | PostgreSQL-backed queue when minimizing infra matters and throughput/feature needs fit; verify BullMQ v6 backend/API choices before implementation |
| HTTP client | Node global `fetch` | Native, promise-based, supports AbortSignal and streaming | Axios where its interceptors, adapters, or legacy compatibility justify another dependency |
| Security headers | Helmet 8.3.0 | Express-recommended security header middleware | Explicit headers for a narrowly controlled API; review CSP and deployment-specific settings |
| Logging | Pino 10.3.1 | Structured JSON, low overhead, transports can offload work | Platform logger if it preserves structured fields and redaction controls |
| Authentication | `argon2` 0.45.1; `express-session` 1.19.0 + `connect-pg-simple` 10.0.0 session-store option; `jose` 6.2.12 only for JWT extension | Argon2id; revocable server-side sessions are the first-party browser default | JWT only where its independent verification/use case warrants rotation and revocation complexity |
| SQL driver (when needed) | `pg` 8.23.0 | Parameterized SQL/connection pool; used by the Prisma PostgreSQL adapter | Avoid a direct driver if the app has no raw SQL/driver use case |
| Unit/integration test default | `node:test` + `node:assert/strict` (built into Node) | No test-framework dependency; sufficient for a backend baseline | Vitest 5.0.2 when its ecosystem/runner features help; current docs require Node >=22.12 and Vite >=6.4 |
| HTTP API test helper | Supertest 7.3.0 | Convenient in-process HTTP assertions for Express | Native `fetch` against a test server for end-to-end behavior |
| Real service integration tests | `@testcontainers/postgresql` and `@testcontainers/redis` 12.2.0 (pin each module) | Exercise real PostgreSQL/Redis rather than pretending mocks prove SQL semantics | Isolated shared test services when CI cannot run Docker |
| API contract / docs | OpenAPI 3.2.0 + `swagger-ui-express` 5.0.1; `openapi-typescript` 7.13.0 optional | Reviewed language-neutral contract; Express 5 peer range checked; generated types do not validate runtime input | OpenAPI 3.1.2 if toolchain support for 3.2 lags; code-first only with drift checks |
| Runtime telemetry | `@opentelemetry/api` 1.9.1 + Node SDK and OTLP HTTP trace/metric exporters 0.222.0 + `sdk-metrics` 2.11.0 + auto-instrumentations 0.80.0 snapshot | Vendor-neutral traces/metrics; package lines version independently and should be installed as a compatible set | Provider SDKs if you explicitly accept vendor coupling |
| Property-based tests (optional) | `fast-check` 4.10.2 | Generate inputs and test invariants; only add where property testing catches real bug classes | Fuzz/handwritten boundary cases for small functions |
| WebSockets | `ws` 8.22.0 | Maintained low-level WebSocket implementation, small abstraction | Socket.IO when its rooms, fallback, and reconnect semantics justify protocol/framework behavior |
| Formatting/lint | ESLint 10.11.0 + typescript-eslint 8.71.0 + TypeScript 6.0.3; Prettier 3.9.9 | The parser peer range supports TS `<6.1.0`; Biome 2.5.14 docs list TS 5.9. TypeScript 7 is latest but not yet inside those compatibility ranges. | Re-evaluate when tool support changes; do not silence peer constraints |
| File storage | Private S3-compatible object storage + short-lived signed URLs; choose provider SDK at implementation | Durable file bytes outside API container; direct upload reduces API bandwidth/heap pressure | Local disk for development or bounded multipart API upload for small controlled files |
| Deployment | OCI Docker image + managed container platform; Compose locally | Reproducible runtime, distinct API/worker processes | VM/systemd or serverless for appropriate workload/operating constraints |

**Version-verification discipline:** run `node --version`, `pnpm --version`, query the exact candidate with `pnpm view <package>@<chosen-version> version`, inspect official release/support docs and peer ranges, then commit the lockfile. A package's `latest` tag can be an RC; do not use a moving `latest` tag in production automation. Pin Node image digests and upgrade in reviewed changes. The unpinned Prisma CLI `latest` is an RC here, so explicitly select Prisma 7.10.0 when following this course; check Prisma release status before an upgrade.

## 1–21. Course overview

The detailed progression, exercises, references, stability labels, and dependency decision matrix live in [00 — Stack, decisions, and roadmap](course/00-stack-and-roadmap.md). This overview sets the mental frame before the lessons begin.

1. **Course philosophy:** build understanding at runtime, protocol, data, security, and operations boundaries before relying on framework convenience.
2. **Backend engineering mindset:** correctness, trust boundaries, failure handling, observability, performance evidence, and proportional architecture.
3. **Current stack and lifecycle:** September 2026 version snapshot; stable vs Current/LTS/RC/experimental/deprecated; reverify official docs before installation.
4. **Node runtime:** V8, event loop, libuv, OS I/O, worker threads, processes, cancellation, streams, backpressure, and resource limits.
5. **HTTP before frameworks:** DNS/TCP/TLS, HTTP semantics, headers, methods, status codes, caching, timeouts, idempotency, and a raw `node:http` server.
6. **Express 5:** ordered middleware, routers, request/response APIs, modern route syntax, async errors, safe error handling, and Express 4 migration distinctions.
7. **TypeScript and runtime boundaries:** ESM/NodeNext, strict typing, Zod parsing, safe configuration, error taxonomy, and type/runtime separation.
8. **API design:** resource-oriented routes, validation, stable error envelopes, cursor pagination, compatibility, versioning, idempotency, and OpenAPI.
9. **Application architecture:** feature-first modular monolith, controller/service/repository boundaries, dependency direction, and when not to add abstractions.
10. **SQL and PostgreSQL:** relational modeling, keys/constraints, joins/aggregates, indexes, query plans, pagination, migrations, and ORM tradeoffs.
11. **Transactions and concurrency:** isolation, row locks, conditional writes, inventory races, idempotency records, outbox, and safe retries.
12. **Authentication:** password hashing, opaque sessions, secure browser cookies, CSRF, OAuth/JWT tradeoffs, rotation, recovery, and revocation.
13. **Authorization and tenancy:** organization membership, RBAC/ownership policy, tenant-scoped repositories, isolation tests, and audit history.
14. **Redis and caching:** data structures, cache-aside, TTL/invalidation, rate limits, circuit breakers, backpressure, and an explicit outage policy.
15. **Queues and reliability:** BullMQ, producer/worker process roles, outbox relay, at-least-once delivery, idempotent effects, retries, and dead-letter operations.
16. **Streams and integrations:** outbound `fetch`, object storage, bounded upload/download, signed webhooks, email, and payment/provider boundaries.
17. **Realtime:** SSE vs WebSocket vs polling; connection lifecycle, authentication, tenant rooms, backpressure, and horizontal scaling.
18. **Testing and contracts:** `node:test`, API/integration/E2E layers, real PostgreSQL/Redis tests, generated cases, and contract drift detection.
19. **Observability:** redacted structured logs, low-cardinality metrics, OpenTelemetry traces, context propagation, dashboards, and alerts.
20. **Deployment and scaling:** Docker/Compose, CI/CD, health/readiness, graceful shutdown, process roles, pool budgets, load tests, rolling deploys, and recovery.
21. **Capstone and debugging:** design a multi-tenant Commerce & Operations API before coding; submit schema/routes/auth/cache/queue/deployment design for review, then use 18 broken-system labs to practice diagnosis and repair.


### Node runtime mental model

```text
JavaScript source ──> V8 executes JS on an event-loop thread
                            │
      ┌─────────────────────┼─────────────────────────┐
      │                     │                         │
 libuv event loop     OS async I/O             libuv worker pool
 (timers, poll,       (network readiness,      selected fs/crypto/DNS
 check, close)         sockets, etc.)           work; bounded resources
      │                     │                         │
      └────────────── callbacks/promises ──────────────┘
                            │
              Worker Threads for CPU parallelism
              (separate JS isolates; not a free DB pool)
```

Node is not accurately described as “one thread does everything.” A JavaScript callback usually runs to completion on one event-loop thread per isolate. The OS, libuv pool, native bindings, and worker threads perform other work. CPU-heavy JavaScript can still stall that isolate and delay unrelated requests.

### HTTP mental model

```text
Client creates request bytes
  → DNS resolves a destination
  → TCP connection (or QUIC for HTTP/3) + TLS where configured
  → proxy/load balancer may terminate TLS and forward HTTP
  → Node parses HTTP and exposes IncomingMessage / ServerResponse
  → app parses/validates, authorizes, performs work
  → response status + headers + body are serialized
  → proxy/network/client receive response
```

HTTP is a protocol, not an object database. Methods, status codes, headers, body framing, cache semantics, cookies, and intermediaries all shape behavior. A request may be retried or duplicated; design writes accordingly.

### Express mental model

```text
REQUEST → ordered middleware stack → router match → route handlers
        → response OR next(error) → error middleware → RESPONSE
```

Express is a middleware and routing framework over Node's HTTP server. Middleware is ordered control flow: it can add context, reject, respond, or transfer control. Express does not automatically provide schema validation, authentication, database transactions, or safe authorization policies.

### Database mental model

```text
HTTP controller → application service → repository/query boundary
       → bounded connection pool → PostgreSQL transaction/constraints/indexes
```

A TypeScript model is not a database constraint. Use foreign keys, unique/check constraints, transactions, isolation/locking where appropriate, and query plans. Treat the pool as a finite shared resource.

### Security mental model

```text
untrusted bytes → parse → validate → authenticate principal → resolve tenant
                → authorize action + resource ownership → execute invariant-preserving write
                → minimize response → audit/log without secrets
```

CORS is a browser response-access policy, not authorization. JWTs do not automatically make a system safer than server-side sessions. Tenant scoping belongs at the data boundary, not only in a controller `if` statement.

## Complete request lifecycle

```mermaid
sequenceDiagram
  participant C as Client
  participant D as DNS
  participant P as Reverse Proxy / Load Balancer
  participant N as Node HTTP Server
  participant E as Express Middleware / Router
  participant S as Controller / Service
  participant R as Repository
  participant DB as PostgreSQL / Redis
  C->>D: Resolve API hostname
  D-->>C: Address records
  C->>P: TCP + TLS; HTTP request
  P->>P: Limits, TLS policy, routing, optional request ID
  P->>N: Forward request (trusted proxy boundary)
  N->>E: IncomingMessage / ServerResponse
  E->>E: Request ID, headers, auth, validation, route
  E->>S: Typed application input + principal context
  S->>R: Business operation
  R->>DB: Parameterized query / transaction
  DB-->>R: Rows / commit result
  R-->>S: Domain result
  S-->>E: Result or expected error
  E-->>N: Status, headers, JSON/stream
  N-->>P: HTTP response
  P-->>C: Response
```

## Capstone architecture preview

```text
Internet / HTTPS
        │
Reverse proxy + load balancer (TLS, limits, health checks)
        │
 ┌──────┴───────────┐      ┌─────────────────────┐
 │ Node API replicas│─────▶│ PostgreSQL (source of│
 └──────┬───────────┘      │ truth, constraints)  │
        ├─────────────────▶└─────────────────────┘
        │
        ├──────────────▶ Redis (cache, rate limits, optional sessions)
        │
        └──────────────▶ Queue ──▶ Worker process ──▶ Email provider / object storage
```

### Authentication architecture (course default)

For a first-party browser app, the course compares server-side opaque sessions in secure cookies with access/refresh tokens. We implement cookie-based sessions first because revocation and logout are straightforward; then model a short-lived access token plus rotating refresh token and reuse detection as an alternate architecture. Cookies are `HttpOnly`, `Secure` in production, appropriately `SameSite`, narrowly scoped, and paired with a CSRF strategy when cross-site cookie use requires it.

```mermaid
sequenceDiagram
  participant B as Browser
  participant A as Express API
  participant V as Validation
  participant S as Auth Service
  participant DB as PostgreSQL / Session Store
  B->>A: POST /api/v1/auth/login + credentials
  A->>V: Parse and validate untrusted body
  V-->>A: Valid login input
  A->>S: Authenticate identity
  S->>DB: Find user; verify password hash; create session
  DB-->>S: User + opaque session record
  S-->>A: Session identifier (never expose its secret in logs)
  A-->>B: 200 + Set-Cookie (HttpOnly, Secure, SameSite)
```

### Database architecture

PostgreSQL owns durable business truth: users, organizations, memberships, products, inventory, orders, payment events, refresh/session records, and audit entries. Migrations are reviewed and deploy-compatible. ORM convenience does not replace SQL understanding, constraints, or `EXPLAIN`.

### Redis architecture

Redis is a bounded-lifetime/supporting store: cache-aside product reads, distributed rate-limit counters, queue backend, and optionally session data. PostgreSQL remains authoritative. Every cached value has a key namespace, TTL, tenant scope, invalidation story, and fallback behavior if Redis is unavailable.

### Queue architecture

```mermaid
sequenceDiagram
  participant C as Client
  participant API as Express API
  participant DB as PostgreSQL
  participant Q as Queue (Redis-backed)
  participant W as Worker
  participant M as Email / Object Storage
  C->>API: Place order
  API->>DB: Commit order + transactional outbox event
  DB-->>API: Commit
  API-->>C: 201 Created (do not wait for email)
  API->>Q: Publish outbox event (or outbox relay publishes)
  Q->>W: Deliver job (at least once)
  W->>M: Send confirmation; use idempotent provider key where available
  W-->>Q: Acknowledge success / retry bounded failure
```

### Deployment architecture

A stateless API tier runs one process per container by default; scale with replicas behind a load balancer. API and worker are separate process roles built from the same versioned application image where practical. PostgreSQL and Redis are managed or operated with explicit backups, limits, network isolation, credentials, metrics, and recovery plans. Object storage handles durable uploads. Logs go to stdout as structured JSON; traces/metrics go to an observability backend.

## Course files

| File | Focus |
|---|---|
| [00 — Stack and roadmap](course/00-stack-and-roadmap.md) | Documentation-first choices, release caveats, full course sequence |
| [01 — Node runtime](course/01-node-runtime.md) | V8, libuv, event loop, async, Node core, workers and processes |
| [02 — HTTP and raw Node server](course/02-http-node-http.md) | HTTP, TCP/TLS/DNS concepts, status codes, streams, raw API |
| [03 — Express 5](course/03-express5.md) | Express mental models, middleware, routing, request/response, migration |
| [04 — TypeScript, validation, errors, architecture](course/04-types-validation-architecture.md) | Runtime schemas, errors, layering, config, type safety |
| [05 — SQL, PostgreSQL, Prisma, transactions](course/05-postgres-sql-transactions.md) | Relational foundations, concurrency, migrations and performance |
| [06 — Authentication and security](course/06-auth-security.md) | Passwords, sessions, JWT tradeoffs, cookies, CSRF/CORS, threats |
| [07 — Authorization and multi-tenancy](course/07-authorization-multitenancy.md) | RBAC, ownership, tenant isolation, audit boundaries |
| [08 — Redis, caching, queues, reliability](course/08-redis-cache-queues.md) | Cache policy, BullMQ, workers, idempotency, retries and breakers |
| [09 — Streams, uploads, integrations, realtime](course/09-streams-integrations-realtime.md) | Buffers, streams, outbound HTTP, webhooks, SSE/WebSockets |
| [10 — Testing, contracts, observability](course/10-testing-contracts-observability.md) | Unit/integration/e2e, OpenAPI, logging, metrics/traces |
| [11 — Performance, Docker, CI/CD, production](course/11-performance-deployment.md) | Production operation, health/shutdown, scaling and deployment |
| [12 — Capstone PRD and design review](course/12-capstone-prd.md) | Multi-tenant SaaS commerce, phases, deliverables, learner-first design prompt |
| [13 — Sequence diagrams](course/13-sequence-diagrams.md) | HTTP, middleware, auth, DB, cache, queues, webhook, payment, uploads, realtime, shutdown, deployment |
| [14 — Final review and documentation map](course/14-final-review-and-documentation-map.md) | Final technology decisions, senior review rubrics, official reference index, production checklist |
| [15 — Debugging lab](course/15-debugging-lab.md) | Deliberately broken backend scenarios, reproduction, diagnosis, repair and mental models |

## How to progress

Do not treat the roadmap as a checklist of packages to install. Finish each module's exercises, write down tradeoffs, and update the running capstone. At checkpoints, test failure paths: repeated requests, invalid input, revoked identity, a lost dependency, a process restart, and concurrent writes. Keep architecture notes next to code so a future reviewer can understand not only what you chose but why.
 not only what you chose but why.
tart, and concurrent writes. Keep architecture notes next to code so a future reviewer can understand not only what you chose but why.
 not only what you chose but why.


## Begin Module 1

Start the actual course with [Module 1 — The Node.js runtime](course/01-node-runtime.md). Module 1 intentionally teaches the runtime before HTTP frameworks.
