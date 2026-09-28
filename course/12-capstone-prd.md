# Module 12 — Capstone PRD and design submission

[← Module 11](11-performance-deployment.md) · [Course home](../README.md) · [Sequence diagram reference](13-sequence-diagrams.md)

> **Design before implementation.** Do not start by generating routes or ORM models. First submit your proposed API, database schema, modules, authentication, authorization, caching, queue and deployment architecture using the worksheet at the end. Then request a review. I will review it as a senior backend architect, security engineer, database engineer, performance engineer and DevOps engineer. The capstone implementation is deliberately not supplied here.

## Product requirements document

### 1. Product summary

Build a **Multi-Tenant SaaS Commerce & Operations API** for organizations that manage a product catalog, team membership and customer orders. The API is a modular monolith with separately runnable API and worker process roles. Organizations are tenant boundaries. A single platform administrator may perform audited system-level management, but tenant access must never be inferred from a client-supplied ID alone.

### 2. Business goals

- Allow a person to register, verify their email, authenticate, manage a profile and recover an account safely.
- Allow a user to create an organization, invite members and assign constrained roles.
- Allow authorized organization members/admins to manage products and images.
- Allow customers to create and track orders without overselling inventory or charging twice on retries.
- Accept payment-provider webhooks safely under duplicate delivery and replay.
- Deliver email, report generation, file scanning and notifications asynchronously.
- Provide an auditable, measurable, documented and deployable API that works across multiple Node instances.

### 3. Users and roles

- **Unauthenticated visitor:** register/login, request password reset, verify email, read only explicitly public catalog endpoints if the product policy enables them.
- **User/customer:** manage own profile/settings; create orders; read/cancel only their own orders where business status allows.
- **Organization member:** actions granted by membership/permissions; tenant data only in active organizations.
- **Organization owner:** manage org settings, invite members, assign allowed roles, billing settings; cannot silently remove the final owner.
- **Organization admin:** configured subset of product/order/member management; cannot exceed owner or platform-admin privilege.
- **Platform admin:** audited global user/product/order/analytics actions; privileged routes require strong authentication and explicit audit event.

### 4. Functional requirements

#### Authentication and account lifecycle

- Registration with normalized unique email and Argon2id password hash.
- Email verification with one-time, expiring token; unverified state handled deliberately.
- Login with generic credential errors, brute-force/risk limits, session rotation and safe cookie policy.
- Logout revokes server-side session and clears cookie.
- Current-user endpoint returns minimal public profile/permissions, not hashes/session secrets.
- Password reset request gives non-enumerating response; reset token is one-time, purpose-bound, expiring and stored hashed; successful reset revokes relevant sessions.
- Session listing/revocation for security-sensitive account management.
- Optional refresh-token flow is a design extension; if used, rotate tokens and detect reuse.

#### Users and organizations

- User profile/settings read/update with field allowlist.
- Create organization; list/get/update organization only if membership/permission allows.
- Invitation create/accept/resend/revoke with expiring single-use token, constrained role assignment and audit history.
- Membership list/change/remove; protect last owner; tenant ID is part of every operation.

#### Products and files

- CRUD products with unique SKU per organization, cents-based price, active state and non-negative inventory.
- Search, category/price filtering, allowlisted sorting and stable cursor pagination.
- Product images use private object storage and an upload lifecycle: pending → uploaded/quarantined → scanned/safe or rejected. Tenant authorizes upload, read and deletion.
- Public product visibility/cache policy must be explicit; private tenant catalog responses may not leak through shared caches.

#### Orders and payments

- Create order with validated item quantities; snapshot price and product identity; reserve stock atomically.
- Use database constraints and transactions; do not call a slow payment/email service while holding an open transaction.
- Require idempotency key for retry-sensitive create/payment actions; same key + same request returns stable result, same key + different request conflicts.
- List/detail scoped by tenant and ownership; cancellation follows a state machine and restores/reserves stock consistently.
- Payment intent is created or requested asynchronously according to provider behavior; webhook is signature verified, deduplicated by provider event ID, transactionally applied and queued for notifications.
- Admin status changes audited and constrained to valid transitions.

#### Admin, analytics and notifications

- Platform admin list/search user, role changes, delete/deactivate user, manage products/orders and view aggregates.
- Audit every security/admin-sensitive action (actor, action, resource, tenant, request ID, timestamp, result).
- Admin notifications can use SSE or WebSocket; choose after assessing one-way vs bidirectional need, auth, reconnect and multi-instance delivery.
- Dashboard aggregates may be delayed/cached with `updatedAt`; checkout and inventory decisions may not trust stale analytics cache.

#### Jobs and integrations

- Queue jobs for verification/reset/order email, image scanning/resizing, reports, cleanup and webhook retry/reconciliation.
- API accepts durable work without blocking on external email/storage/provider latency.
- Workers validate job payload/version, reload authoritative state, include tenant scope, are idempotent, have timeouts/retry budgets and emit trace/log context.
- External HTTP calls use timeout/AbortSignal, response validation, SSRF controls when URL is not fixed, bounded retries and circuit-breaker/degraded behavior.

### 5. API contract

Prefix all product APIs with `/api/v1`. Use JSON except file bytes and event streams. Document authentication, authorization, request/response schemas, errors, pagination and idempotency in OpenAPI. Avoid returning ORM rows directly. One acceptable envelope is `{ "data": ..., "meta": ... }`; simple endpoints may return a resource directly if the contract remains consistent. Errors use a stable shape such as `{ "error": { "code": "...", "message": "...", "details": [...] } }`; production messages never contain stack/SQL/secrets.

| Method | Path | Purpose / access expectation |
|---|---|---|
| POST | `/api/v1/auth/register` | Public; validate and enqueue verification |
| POST | `/api/v1/auth/login` | Public but risk/rate limited; session rotation |
| POST | `/api/v1/auth/refresh` | Optional token architecture; cookie + CSRF/reuse control |
| POST | `/api/v1/auth/logout` | Authenticated/CSRF protected; revoke session |
| GET | `/api/v1/auth/me` | Authenticated current principal |
| POST | `/api/v1/auth/forgot-password` | Public; generic response + rate limit |
| POST | `/api/v1/auth/reset-password` | One-time token; revoke sessions |
| GET | `/api/v1/users/me` | Authenticated own profile |
| PATCH | `/api/v1/users/me` | Authenticated allowlisted profile update |
| POST | `/api/v1/organizations` | Authenticated; creates owner membership atomically |
| GET | `/api/v1/organizations/:id` | Member with read permission |
| PATCH | `/api/v1/organizations/:id` | Owner/admin policy |
| DELETE | `/api/v1/organizations/:id` | Owner-only lifecycle/deactivation policy; consider async deletion |
| GET | `/api/v1/organizations/:id/members` | Authorized member manager/list policy |
| POST | `/api/v1/organizations/:id/members` | Owner/admin invitation with role limit |
| PATCH | `/api/v1/organizations/:id/members/:memberId` | Authorized role/status change; audit + last-owner guard |
| DELETE | `/api/v1/organizations/:id/members/:memberId` | Authorized removal; cannot strand organization |
| GET | `/api/v1/products` | Public/authorized catalog policy; filters/cursor/cache |
| GET | `/api/v1/products/:id` | Visibility and tenant policy |
| POST | `/api/v1/products` | Product-write permission; validated price/stock |
| PATCH | `/api/v1/products/:id` | Tenant scoped, field allowlist, optimistic concurrency as needed |
| DELETE | `/api/v1/products/:id` | Soft-delete/archive policy; historical order snapshots remain |
| POST | `/api/v1/files/upload-intents` | Auth + tenant/purpose/size checks; returns short-lived signed URL |
| POST | `/api/v1/orders` | Auth + idempotency key; inventory/order transaction |
| GET | `/api/v1/orders` | Owner/member policy; cursor pagination |
| GET | `/api/v1/orders/:id` | Tenant + ownership permission at query boundary |
| POST | `/api/v1/orders/:id/cancel` | Idempotent state transition; inventory/payment handling |
| POST | `/api/v1/payments` | Idempotency key; payment intent lifecycle |
| GET | `/api/v1/admin/users` | Platform admin; audited and rate limited |
| PATCH | `/api/v1/admin/users/:id/role` | Platform admin; audit/step-up policy |
| DELETE | `/api/v1/admin/users/:id` | Platform admin; deactivation/retention policy |
| POST | `/api/v1/webhooks/payment` | Provider signature over raw bytes; idempotent durable accept |
| GET | `/api/v1/notifications/stream` | Optional SSE; user-scoped authorization and reconnect policy |
| GET | `/health` | Basic operator health; avoid sensitive details |
| GET | `/readiness` | Routing eligibility; returns 503 when not ready |
| GET | `/liveness` | Process liveness; shallow and low cost |

#### HTTP policy baseline

- `200` read/update response; `201` resource created + `Location`; `202` durable async work accepted; `204` no content.
- `400` malformed syntax; `401` missing/invalid auth; `403` forbidden; `404` absent/concealed; `409` conflict/idempotency; `415` content type; `422` semantic validation (if selected); `429` rate/quota; `500` unexpected; `503/504` dependency/timeout according to boundary.
- Use `ETag`/conditional requests only where representation and authorization-aware cache semantics are sound. Do not cache sensitive session/auth responses.
- Pagination has a bounded page size; products/orders use stable cursor ordering for large feeds. Search/sort/filter values are schema-validated and SQL sorting is allowlisted.

### 6. Data requirements and invariants

The learner must propose a normalized relational schema with at least:

- `users`, password credential/verification state, `sessions` or refresh-token family records;
- `organizations`, `memberships`, invitations, roles/permissions or documented role-policy mapping;
- `products`, categories/images, organization-scoped SKU unique constraint;
- `orders`, `order_items` with immutable price/description snapshots, payments/payment intents, provider webhook event IDs;
- `idempotency_keys` scoped by tenant/user/operation and canonical request hash;
- `outbox_events`, audit events, file metadata/scan state;
- necessary foreign keys, unique/check constraints, indexes, retention/deletion policies.

Invariants: stock cannot be negative; an order item quantity is positive; order total is non-negative and matches validated snapshots; an order cannot cross organizations; a user cannot read another tenant's order; one webhook event applies its transition at most once; one idempotency key cannot represent two different requests; only eligible orders cancel; invitation/reset/verification tokens are one-time and expire.

### 7. Security and privacy requirements

- HTTPS at edge; restricted `trust proxy`; Helmet/security headers; explicit CORS origin/method/header allowlist; CSRF protection for cookie-authenticated state-changing browser requests.
- Runtime schemas for body/query/params/headers/env/provider/webhook/job payloads; body/file/query/page limits.
- Session/JWT lifecycle, password hashing, reset/email token, revocation and audit policies documented.
- Authorization on route, service and data boundary; tenant predicates; least-privilege DB/service credentials.
- SQL parameterization; allowlisted dynamic sort; SSRF controls; command invocation policy; private uploads, scanning and signed URL limits.
- Route-specific distributed rate limits for login/reset/admin/payment and org quotas; protect against brute force and expensive-query abuse.
- Redacted structured logs; no credentials, tokens, cookies, payment secrets or unnecessary PII in logs/traces.
- Dependency/lockfile scanning, secret scanning, SBOM concept, vulnerability update response and safe error responses.

### 8. Non-functional requirements (initial targets to challenge)

Treat these as proposed SLOs to revise after load testing, not universal guarantees:

- For the agreed representative catalog workload, p95 API latency under 300 ms and p99 under 800 ms excluding provider/large export work; report both end-to-end and dependency time.
- Order creation acknowledges only after durable transaction/idempotency state commits; external email is not on the request critical path.
- Every route has explicit timeout/body limits; overload is bounded and returns useful 429/503 rather than exhausting memory/connections.
- API is horizontally scalable with no required process-local session/job state.
- Recovery, DB backups, restore test, Redis/queue persistence expectations and data retention are documented.
- Error rate, queue age, p95/p99, event-loop delay, pool saturation and webhook/job retries are observable and alertable.
- Accessibility/localization/UI concerns are outside this API PRD except where they affect contract behavior.

### 9. Caching and consistency requirements

Candidate cache: cache-aside product catalog reads keyed by organization, visibility/representation version and query/filter. TTL and invalidation on product/category/image update/delete. Define stale window and Redis-outage behavior. User permissions, inventory/order state and payment state are not served from stale product cache. Explain browser/CDN/reverse-proxy vs Redis ownership. Never use cache as authorization authority.

### 10. Queue and delivery requirements

Select Redis/BullMQ, PostgreSQL queue, or managed queue based on operational needs. Specify API/worker roles, outbox or failure handling for DB→queue dual write, idempotency key, job payload version, attempt count/backoff/jitter/dead-letter/replay, per-tenant fairness/quotas, timeout, shutdown, queue-depth alarms and operator replay tooling. Explain what the user sees if email or image processing is delayed.

### 11. Testing requirements

- Unit tests: validation, auth policy, order state transitions, idempotency conflict behavior.
- Integration: real PostgreSQL migrations, constraints, transactions, N+1/query plans where feasible; Redis/cache/rate/queue integration where behavior relies on it.
- API/e2e: happy paths, duplicate order/payment/webhook, forbidden cross-tenant access, reset/verification, logout/revocation, admin action.
- Security regression: invalid session/JWT, CSRF, invalid CORS origin, body/file limit, SQL/path/SSRF input, rate limit, tenant isolation and token reuse.
- Contract: OpenAPI paths/schemas/statuses/auth match handlers; generated client drift considered.
- Property/invariant tests: stock never negative, tenant boundary cannot cross, idempotency stable, one-time token cannot be reused.
- Mock external provider boundaries; do not mock away all DB behavior.

### 12. Observability and operations

Every request has validated/generated request ID and trace context; logs have method, route template, status, duration, request ID and safe actor/tenant metadata. Metrics include latency histogram, throughput, error rate, event-loop, DB pool/query, Redis latency/evictions, queue depth/oldest age, worker failure and provider latency. Traces connect API → DB/cache/provider/queue → worker. Include startup config validation, readiness/liveness, SIGTERM/SIGINT drain, migration process, secret management, backup/restore, runbooks, alerts and feature-flag/kill-switch use.

### 13. Deployment target

```text
Internet → HTTPS reverse proxy/load balancer → Node API replicas
                                  │              ├→ PostgreSQL
                                  │              ├→ Redis/cache/rate limit/queue
                                  │              └→ private object storage
                                  └→ health-based traffic selection
Queue → worker replicas → PostgreSQL / email provider / object storage
```

Use a modular monolith as the primary capstone; API and worker are separate processes for a clear operational boundary, not separate domain microservices. Use Docker/Compose locally, CI checks and immutable image deployment to a chosen managed container service/VM/orchestrator. State how secrets, TLS, database migrations, logs/metrics, network isolation, autoscaling and rollback work.

### 14. Acceptance criteria

- No capstone route consumes unvalidated `req.body/query/params`.
- All tenant-owned reads/writes include validated organization scope and authorization.
- Concurrent purchase of last unit cannot produce negative inventory or two valid reservations.
- Same idempotency key and body yields stable result; same key/different body conflicts.
- Repeated provider webhook event does not apply payment side effect twice.
- Session logout/password reset revokes credentials; reset/email tokens are one-time.
- Async email/report work does not block order response; failed work is visible and retryable.
- File bytes remain private until validation/scanning passes.
- OpenAPI documents endpoint schemas/errors/auth; automated tests catch drift.
- Unit/integration/e2e/security tests pass in CI with deterministic fixtures.
- Logs redact secrets; request IDs and traces correlate API and worker operations.
- Health/readiness/shutdown and production deployment behavior are verified in staging.
- Performance/load test reports p50/p95/p99, throughput, errors and saturation for products/orders.

## Your design submission — return this before asking for code

Please make decisions; do not try to guess a single “correct” architecture. Include rationale, alternatives and failure behavior. You can submit a first draft with open questions.

### 1. API routes

List methods/paths, auth requirement, status codes, request/response schemas, errors, pagination, idempotency and cache policy for the core auth, organization, product, order, admin and webhook flows. Identify endpoint changes you would make and why.

### 2. Database schema

Draw or list tables, keys, foreign keys, constraints, indexes and critical transaction boundaries. Explain how stock, order totals, webhook event dedupe, invitations, refresh/session revocation and idempotency remain correct under concurrency.

### 3. Modules and dependency direction

Propose feature modules/shared infrastructure and show which layer owns HTTP, business rules, data access, queue publication and external provider calls. Explain any abstractions you intentionally omit.

### 4. Authentication

Choose cookie session, JWT access/refresh, or a hybrid; describe credential storage, cookie flags, CSRF/CORS assumptions, rotation, expiry, revocation, password reset, email verification, session storage and failure behavior.

### 5. Authorization

Define platform/org/user roles and permissions; show one ownership policy; explain tenant resolution, route/service/data checks, admin privilege and audit events.

### 6. Caching

Choose what to cache and where; give key examples, tenant scope, TTL, invalidation, stale-data tolerance, Redis outage and authorization consistency behavior.

### 7. Queue architecture

Choose BullMQ/Redis, PostgreSQL-backed queue, or managed queue; show API → durable event → worker, outbox decision, job idempotency, retries, timeout, shutdown and poison-job handling.

### 8. Deployment architecture

Draw proxy/LB, API/worker replicas, PostgreSQL, Redis, storage and email boundaries. Define Node version, container/process model, `trust proxy`, secrets, migrations, health/readiness, shutdown, backups, monitoring, CI/CD and scaling.

### Review request

After your response, ask: **“Review this design as a senior backend architect. First identify blocking correctness/security risks, then database/concurrency risks, then operability/performance tradeoffs. Ask clarifying questions before prescribing changes. Do not write the implementation yet.”**

## Final architecture review rubric

The review will assess correctness, security, maintainability, scalability, observability, performance, testing and deployment. It will check: data ownership/tenant leaks; duplicate payments/webhooks; last-stock races; pool exhaustion; cache invalidation; session/token theft/revocation; CSRF/CORS confusion; file/webhook trust boundaries; job retries and outbox gap; proxy trust; request deadlines; schema migration compatibility; observability of failure; and restore/runbook readiness.
