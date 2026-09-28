# Final review — technology matrix, production practices, and official documentation map

[← Course home](../README.md) · [Capstone PRD](12-capstone-prd.md) · [Sequence diagrams](13-sequence-diagrams.md)

## Final technology decision matrix (snapshot: 2026-09-28)

| Need | Selected technology / verified version | Why | Alternative / when it fits |
|---|---|---|---|
| Runtime | Node.js 24.21.0 Active LTS | Production-supported baseline; current Node docs; Express 5 support | Node 26.10.0 Current for testing new runtime features, not primary production baseline |
| Web framework | Express 5.2.1 stable | Explicit middleware/router composition; requested subject | Fastify for schema-centric high-performance conventions; NestJS for convention-heavy architecture/DI |
| TypeScript / Node typings | TypeScript 6.0.3 course baseline; `@types/node` 24.19.0 and `@types/express` 5.0.6; 7.0.2 latest stable | `typescript-eslint` 8.71.0 declares TS `<6.1`; choose working typed lint compatibility | Move to TS 7 when compiler/lint/formatter ecosystem supports it; TS 7 is not ignored, it is not the course's pinned production combination yet |
| Package manager | pnpm 12.6.0 stable | Reproducible lockfile and workspace behavior | npm 12.1.0 is a good simpler default; keep one manager and exact lockfile |
| Request/runtime validation | Zod 4.6.5 | Type inference and parsing in one schema; stable v4 APIs | Valibot for modular size; Ajv for JSON Schema and contract-first validation |
| SQL database | PostgreSQL 18.6 stable current minor | Durable relational source of truth; mature transactions, constraints, indexes | PostgreSQL 17.11 if a provider has not certified 18; PostgreSQL 19 Beta 4 is prerelease in this snapshot |
| ORM | Prisma CLI + `@prisma/client` + `@prisma/adapter-pg` 7.10.0 | Typed generated client, migrations, readable relations; stable supported line | Prisma 8 is RC in this snapshot (an unpinned CLI selects RC; v8 package/API shape differs); Drizzle ORM 0.45.3/Kit 0.31.11 for SQL-shaped TS control |
| Redis | Redis Open Source 8.10 release line; node-redis package `redis` 6.2.1 | Redis official docs recommend node-redis for Node | PostgreSQL if persistence/relational queries/traffic requirements suffice; ioredis for existing users/migration cases |
| PostgreSQL driver | `pg` 8.23.0 when using direct SQL / driver adapter | Parameterized SQL and pool control; avoid unused direct dependency | Prisma adapter or no direct driver use |
| Queue | BullMQ 6.3.9 | Queue/worker, retries, delayed/concurrent jobs; Redis backend chosen, node-redis adapter requires `redis` v5+ | BullMQ's optional PostgreSQL backend or managed queue if reducing infrastructure justifies fit; verify backend tradeoffs |
| Password hash | `argon2` 0.45.1 (Argon2id) | Maintained implementation; memory-hard password hashing | bcrypt for compatibility or environment constraints; never plaintext/fast SHA-256 |
| JWT implementation | `jose` 6.2.12 (optional module) | JOSE standards-oriented implementation; claims/key verification APIs | Sessions are the course default for first-party browser app; use JWT only with explicit need |
| Session middleware | `express-session` 1.19.0, `connect-pg-simple` 10.0.0 store option | Server-side revocation and shared session storage | Redis session store if low latency/shared Redis ops are already justified |
| Security headers / CORS | Helmet 8.3.0; `cors` 2.8.6 optional | Helmet headers; CORS is configured browser policy only | Gateway headers / explicit app policy; never confuse CORS with authz |
| HTTP client | Node global `fetch` | Built in, AbortSignal, Web-compatible APIs | Axios when interceptors/adapters offer meaningful value |
| OpenTelemetry | API 1.9.1 + Node SDK/OTLP HTTP trace and metric exporters 0.222.0 + SDK metrics 2.11.0 + auto-instrumentations 0.80.0 | Initialize before application imports; package lines version independently and follow dependency/peer ranges | Vendor-specific SDK if vendor lock-in is an explicit decision |
| Property-based tests (optional) | `fast-check` 4.10.2 | Generate invariant/parser cases where valuable | Deterministic examples/fuzzing if property-based complexity is not justified |
| Logs | Pino 10.3.1 | Structured JSON and redaction | Platform structured logger when field/redaction/correlation requirements fit |
| Test base | Built-in `node:test` + `node:assert/strict` | No dependency; services/policies and simple API tests | Vitest 5.0.2 for runner ergonomics; requires Node >=22.12 and Vite >=6.4 |
| API testing | Supertest 7.3.0 | In-process HTTP tests for Express | Native `fetch` with test listener for full socket behavior |
| Infrastructure tests | `@testcontainers/postgresql` / `@testcontainers/redis` 12.2.0 | Real DB/cache semantics isolated per suite | Managed shared CI test services if container runtime unavailable |
| API docs | OpenAPI 3.2.0 spec + `swagger-ui-express` 5.0.1 | Versioned language-neutral contract and UI | OpenAPI 3.1.2 if generator/validator support for 3.2 is insufficient |
| Types from API contract | `openapi-typescript` 7.13.0 optional | Generates TypeScript consumer types | Handwritten types only for tiny/closed systems; generated types do not runtime-validate |
| WebSockets | `ws` 8.22.0 optional | Small maintained Node WebSocket implementation | Socket.IO for rooms/reconnect/fallback features with their added semantics |
| Lint/format | ESLint 10.11.0 + typescript-eslint 8.71.0 + Prettier 3.9.9 | Peer-compatible with TypeScript 6.0.3; lint and format concerns separated | Biome 2.5.14 is compact, but its docs list TS 5.9 rather than TS 7 |
| Uploads | Provider SDK selected when storage target is chosen; Multer 2.4.0 only for intentionally API-proxied multipart | Direct-to-object-storage avoids API memory/network pressure | Multipart parser such as Multer 2.4.0 for small controlled uploads that must pass through API |
| Deployment | OCI Docker image + managed container platform/VM, Compose locally | Reproducible process roles and horizontal replica scaling | Serverless or Kubernetes where workload/team operational model supports them |

**Version note:** numbers are the snapshot, not a permanent recommendation. Registry `latest` tags may point to an RC or a major version outside a dependency's peer range. Use official project release/migration guides, verify Node engine and peer ranges, pin Prisma 7 explicitly for this course snapshot, then commit the lockfile. Use the latest patch of the selected stable major; do not pin an old vulnerable patch merely because it appears here.

## Official documentation map

These are primary sources to revisit when a module begins or a dependency is upgraded.

### Runtime, language, and framework

- **Node.js releases and API docs:** [Release lifecycle](https://nodejs.org/en/about/previous-releases) · [Node 24 API reference](https://nodejs.org/docs/latest-v24.x/api/) · [Node 26 API reference](https://nodejs.org/docs/latest-v26.x/api/)
- **Node core APIs:** [HTTP](https://nodejs.org/docs/latest-v24.x/api/http.html) · [HTTPS](https://nodejs.org/docs/latest-v24.x/api/https.html) · [streams](https://nodejs.org/docs/latest-v24.x/api/stream.html) · [Web Streams](https://nodejs.org/docs/latest-v24.x/api/webstreams.html) · [Buffer](https://nodejs.org/docs/latest-v24.x/api/buffer.html) · [crypto](https://nodejs.org/docs/latest-v24.x/api/crypto.html) · [worker threads](https://nodejs.org/docs/latest-v24.x/api/worker_threads.html) · [child processes](https://nodejs.org/docs/latest-v24.x/api/child_process.html) · [test runner](https://nodejs.org/docs/latest-v24.x/api/test.html) · [process](https://nodejs.org/docs/latest-v24.x/api/process.html)
- **TypeScript:** [Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) · [TS 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html) · [TypeScript 7.0.2 release](https://github.com/microsoft/TypeScript/releases/tag/v7.0.2) · [current download](https://www.typescriptlang.org/download/) · [Node module theory](https://www.typescriptlang.org/docs/handbook/modules/theory.html)
- **Express:** [Express 5 API](https://expressjs.com/en/5x/api.html) · [Routing](https://expressjs.com/en/guide/routing.html) · [Middleware](https://expressjs.com/en/guide/using-middleware.html) · [Error handling](https://expressjs.com/en/guide/error-handling.html) · [Express 5 migration](https://expressjs.com/en/guide/migrating-5/) · [Security guidance](https://expressjs.com/en/advanced/best-practice-security.html) · [Production performance/reliability](https://expressjs.com/en/advanced/best-practice-performance/)

### Protocols, data and async work

- **HTTP/TLS/DNS:** [HTTP Semantics RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) · [HTTP/1.1 RFC 9112](https://www.rfc-editor.org/rfc/rfc9112) · [HTTP caching RFC 9111](https://www.rfc-editor.org/rfc/rfc9111) · [TLS 1.3 RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) · [DNS RFC 1034](https://www.rfc-editor.org/rfc/rfc1034)
- **PostgreSQL:** [Version policy](https://www.postgresql.org/support/versioning/) · [PostgreSQL 18 manual](https://www.postgresql.org/docs/18/) · [Indexes](https://www.postgresql.org/docs/18/indexes.html) · [Transactions](https://www.postgresql.org/docs/18/transaction-iso.html) · [EXPLAIN](https://www.postgresql.org/docs/18/using-explain.html)
- **ORM:** [Prisma release status](https://www.prisma.io/docs/orm/release-status) · [Prisma ORM 7 docs](https://www.prisma.io/docs/orm/v7) · [Prisma migrations](https://www.prisma.io/docs/orm/v7/prisma-migrate) · [Drizzle](https://orm.drizzle.team/docs/overview)
- **Redis:** [Redis 8.10 announcement](https://redis.io/blog/announcing-redis-810-compact-hash-jsonpath-extensions-performance-improvements-and-more/) · [Node client](https://redis.io/docs/latest/develop/clients/nodejs/) · [Production client usage](https://redis.io/docs/latest/develop/clients/nodejs/produsage/) · [Data structures](https://redis.io/docs/latest/develop/data-types/)
- **Queue:** [BullMQ guide](https://docs.bullmq.io/guide) · [Connections](https://docs.bullmq.io/guide/connections) · [Retries](https://docs.bullmq.io/guide/retrying-failing-jobs) · [Idempotent jobs](https://docs.bullmq.io/patterns/idempotent-jobs)
- **Object storage:** choose the selected provider's official docs; for Amazon S3 see [S3 user guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) and [presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)

### Security, validation, tests, contracts and operations

- **Validation:** [Zod](https://zod.dev/) · [Zod 4 migration/release notes](https://zod.dev/v4) · [Valibot](https://valibot.dev/) · [Ajv](https://ajv.js.org/)
- **Authentication/security:** [OWASP Top 10](https://owasp.org/www-project-top-ten/) · [ASVS](https://owasp.org/www-project-application-security-verification-standard/) · [Password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) · [Session management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) · [CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) · [JWT RFC](https://www.rfc-editor.org/rfc/rfc7519) · [OAuth security BCP](https://www.rfc-editor.org/rfc/rfc9700)
- **Helmet/auth libraries:** [Helmet](https://helmetjs.github.io/) · [Multer multipart parser](https://github.com/expressjs/multer) · [Argon2 Node](https://github.com/ranisalt/node-argon2) · [jose](https://github.com/panva/jose) · [express-session](https://github.com/expressjs/session)
- **Testing:** [Node test runner](https://nodejs.org/api/test.html) · [Vitest](https://vitest.dev/guide/) · [fast-check](https://fast-check.dev/docs/introduction/) · [Supertest](https://github.com/ladjs/supertest) · [Testcontainers Node](https://node.testcontainers.org/)
- **OpenAPI:** [OpenAPI 3.2.0 specification](https://swagger.io/specification/v3.2/) · [Swagger UI](https://swagger.io/tools/swagger-ui/) · [OpenAPI TypeScript](https://openapi-ts.dev/)
- **Logging/observability:** [Pino](https://getpino.io/#/) · [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/) · [Node instrumentation](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/)
- **Containers/delivery:** [Docker](https://docs.docker.com/) · [Compose](https://docs.docker.com/compose/) · [pnpm](https://pnpm.io/) · [k6](https://grafana.com/docs/k6/latest/)

## Senior final review checklist

### Senior Backend Engineer

- Are route/controller/service/repository boundaries proportional and clear?
- Are business state transitions and retry semantics explicit?
- Does a modular monolith preserve useful domain boundaries without premature microservices?
- Can a new engineer trace a request from route to transaction and understand the contract?

### Security Engineer

- Can one tenant enumerate/read/change another tenant's data through object IDs, cache keys, file IDs, websocket room IDs or queue payloads?
- Are authentication credentials rotated/revoked, cookies correctly scoped, CSRF addressed, CORS accurately described, and reset/verification flows abuse-resistant?
- Are SQL, SSRF, upload, path, request-body, brute-force, dependency and secret boundaries tested?
- Do logs/traces/errors leak session/token/password/payment or personal data?

### Database Engineer

- Do schema constraints encode critical invariants?
- Can concurrent last-stock purchases, duplicate webhook delivery and idempotency-key reuse produce an invalid result?
- Are transaction boundaries short and external I/O outside locks?
- Do query plans, indexes, pool budgets, pagination and migration strategy fit expected scale?

### Performance Engineer

- What evidence supports the bottleneck and each optimization?
- What are p50/p95/p99, throughput, error rate and saturation under representative load?
- Is the event loop blocked? Are payloads/queries/serialization/caches bounded?
- What degrades safely when a dependency is slow or unavailable?

### DevOps / SRE Engineer

- Are startup, readiness, liveness, graceful shutdown and automatic restart tested?
- Are secrets managed outside the image, permissions/network boundaries least-privilege, and image/dependencies reviewed?
- Are DB backup/restore, migrations, rollback/forward fix, alerts, dashboards, deployment health gates and incident runbooks ready?
- Can API and worker scale independently without losing sessions/jobs or duplicating side effects?

## Final production principles

Use a current supported Active LTS Node patch and supported Express 5 release; run with `NODE_ENV=production`; return rejected Promises to Express 5 error middleware; avoid synchronous request-path work; validate input; use Helmet and secure cookies; log structured redacted events; handle errors centrally; restart/health-check processes; run behind a trusted reverse proxy and load balancer; cache measured repeat work; size pools across replicas; stream large data; and use queue workers for slow/CPU-heavy durable work. Every statement is a design constraint to verify, not a checkbox that proves production readiness.

## Course completion

Completion means you can design, implement, secure, test, debug, measure, deploy, scale and explain the capstone. It does **not** mean every optional technology is installed, nor that a passing test suite proves a system secure. Your capstone design review is the final evidence of reasoning.
