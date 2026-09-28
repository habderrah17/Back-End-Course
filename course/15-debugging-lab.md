# Debugging lab — intentionally broken backend systems

[← Course home](../README.md) · [Capstone PRD](12-capstone-prd.md) · [Sequence diagrams](13-sequence-diagrams.md)

This lab is meant to be run against a disposable branch and local test infrastructure. Do not test brute force, resource exhaustion, webhook replay, SSRF or load behavior against production. For every incident use the same loop:

1. **Reproduce** with deterministic input/concurrency and write down the environment.
2. **Observe** status, latency percentiles, logs, traces, DB/Redis/queue metrics, resource use.
3. **Diagnose** the first broken invariant/boundary; do not patch the visible symptom only.
4. **Fix** the design and add a regression test/alert.
5. **Explain** the mental model and the deployment/scaling effect.

## Lab 1 — async error disappears (legacy Express 4 assumption)

**Broken:**

```ts
app.get("/broken", async (_req, _res) => {
  await Promise.reject(new Error("database offline"));
});
```

**Reproduce/observe:** run on Express 4 in a disposable migration exercise (or compare source/runtime behavior); request `/broken`. Express 4 can leave the rejection outside router error flow, causing an unhandled rejection/unstable process depending runtime policy. On Express 5, the returned rejected Promise is forwarded to error middleware.

**Fix:** Express 5 current supported release + terminal error middleware. Ensure asynchronous work is returned/awaited; detached event/timer callbacks still need explicit error handling. Test both a rejected Promise and a callback-based failure.

**Mental model:** a framework can observe the Promise returned by a handler, not arbitrary work detached from it.

## Lab 2 — authorization middleware is in the wrong order

**Broken:**

```ts
app.get("/api/v1/admin/users", listUsers);
app.use("/api/v1/admin", requirePlatformAdmin);
```

**Reproduce:** call the admin endpoint with no session; inspect the response. It succeeds because the middleware was registered later and the earlier route ended the response.

**Observe:** middleware trace/request logs show no auth decision for the route.

**Fix:** mount security/auth middleware before protected routes or attach it to a protected router; include a regression test for unauthenticated and non-admin access.

**Mental model:** Express is ordered control flow, not a declarative policy engine that scans all registrations globally.

## Lab 3 — authorization bypass / IDOR / cross-tenant order

**Broken:**

```ts
const order = await prisma.order.findUnique({ where: { id: orderId } });
res.json(order);
```

**Reproduce:** as organization A's member, request an order ID owned by organization B. Use fixtures with distinct tenants and assert the response.

**Observe:** row returned even though UI never linked it. IDs are identifiers, not credentials.

**Fix:** resolve authenticated tenant/membership; query with `id + organizationId + ownership/permission predicate`; return consistent 404/403 policy; test service directly and through API. Add audit for privileged access.

**Mental model:** authorization must be checked at the data boundary; controller-only checks are easy to omit.

## Lab 4 — N+1 query and connection-pool saturation

**Broken:** fetch all users, then query order totals in a loop.

```ts
const users = await db.user.findMany({ take: 100 });
for (const user of users) {
  user.orders = await db.order.findMany({ where: { customerId: user.id } });
}
```

**Reproduce:** seed 100+ users; call the endpoint; record query count and DB spans. Load test concurrent callers. Query count grows linearly with users.

**Observe:** p95/p99 rises, repeated similar SQL spans dominate traces, pool wait increases.

**Fix:** aggregate/join/projection or controlled relation query; paginate/select only needed data; verify SQL and `EXPLAIN`; add query-count/integration assertion.

**Mental model:** a loop over an async database call is serial network round trips unless explicitly batched; “ORM typed” does not mean efficient.

## Lab 5 — stale Redis product cache

**Broken:** cache product for a day and never invalidate after update.

**Reproduce:** GET product → PATCH product → GET again; compare DB and Redis value. Simulate cache delete failure after DB commit.

**Observe:** stale price/name, possibly incorrect tenant representation.

**Fix:** tenant/version-scoped keys, justified TTL, write/invalidation design, event/outbox for stronger reliability, cache outage policy, and a test proving update visibility within required freshness window.

**Mental model:** DB commit and cache mutation are not one atomic transaction; TTL is an upper bound, not an invalidation guarantee.

## Lab 6 — two buyers purchase the last item

**Broken:**

```ts
const product = await findProduct(id);
if (product.availableStock > 0) {
  await createOrder(product);
  await decrementStock(id);
}
```

**Reproduce:** start two transactions together with stock 1 and a barrier after both reads.

**Observe:** two successful orders or negative stock; a sequential happy-path test misses it.

**Fix:** one atomic conditional stock update in a DB transaction, nonnegative CHECK constraint, unique/idempotency protections, and a concurrency integration test that launches both requests.

**Mental model:** check-then-act is a race across transactions; only an atomic DB predicate/lock/constraint arbitrates concurrent writers.

## Lab 7 — duplicate payment or webhook

**Broken:** each webhook delivery blindly marks payment/creates fulfillment; payment endpoint retries create a second provider intent.

**Reproduce:** deliver same provider event ID three times; send same `Idempotency-Key` twice concurrently; kill worker after external side effect but before queue ack.

**Observe:** duplicate fulfillment/email/charge or inconsistent payment state.

**Fix:** provider signature over raw bytes; unique webhook event ID; transactional state transition/outbox; scoped idempotency record with request hash; provider idempotency key; consumer dedupe/reconciliation. Assert repeat delivery yields stable result.

**Mental model:** networks and queues offer at-least-once attempts, not end-to-end exactly-once effects.

## Lab 8 — refresh-token replay after rotation

**Broken:** refresh token remains valid after issuing a replacement.

**Reproduce:** refresh once, then replay the old cookie; attempt two parallel refresh requests.

**Observe:** old token obtains another access credential; attacker and user remain active; parallel requests can fork token families.

**Fix:** transactionally consume a single-use token hash and persist successor; detect reuse and revoke family; handle parallel request conflict predictably; test session/token revocation.

**Mental model:** token rotation is a state transition, not just minting a new string.

## Lab 9 — CORS used as authorization

**Broken:** server allows only frontend origin and assumes all other clients cannot call API.

**Reproduce:** use `curl` without an Origin header to call a protected route. CORS does not stop it.

**Observe:** response contains data despite browser policy; Postman/server client can also call it.

**Fix:** authenticate and authorize every route/service/resource; configure CORS separately to control browser JavaScript response access; add a direct non-browser unauthorized test.

**Mental model:** CORS is enforced by browsers for response access, not by the server as an access-control system.

## Lab 10 — wrong client IP behind proxy

**Broken:** trust all `X-Forwarded-For` values or trust no configured proxy while rate limiting `req.ip`.

**Reproduce:** send spoofed forwarded headers through a direct connection; compare logged `req.ip` through trusted load balancer and locally.

**Observe:** attacker can bypass IP quota or poison audit/location records; secure cookie/protocol behavior may differ.

**Fix:** configure exact proxy hops/network according to topology; strip/overwrite forwarded headers at edge; test spoofed header and direct-origin traffic.

**Mental model:** forwarded headers are ordinary untrusted input until a trusted proxy boundary authenticates/rewrites them.

## Lab 11 — timeout/retry storm

**Broken:** three layers each retry 5 times; timeout at client is shorter than server's downstream work; backoff has no jitter.

**Reproduce:** make a test provider return 503/timeout; raise concurrency; observe attempts per user operation.

**Observe:** one request fans into many provider calls, queue depth grows, service remains busy after client timed out, synchronized retry spikes.

**Fix:** propagate deadline/AbortSignal, bounded retry budget with jitter, idempotency, classify permanent errors, queue durable work, circuit break repeated outages.

**Mental model:** retries multiply load; a retry is a new attempt, not a free continuation.

## Lab 12 — event-loop blocking and memory retention

**Broken A:** `while (Date.now() < end) {}` inside a route. **Broken B:** global `Map` caches every request key forever. **Broken C:** attach an event listener/timer per request and never remove it.

**Reproduce:** compare `/health` latency during CPU work; send varied keys and inspect heap snapshots; count listeners over repeated connect/disconnect.

**Observe:** event-loop delay/CPU rises for A; heap retained size increases for B; listener warnings/memory growth for C.

**Fix:** better algorithm/worker/job process for CPU work; bounded TTL/size/eviction for cache; cleanup listeners/timers/streams and abort on disconnect. Add resource metrics and soak tests.

**Mental model:** one synchronous callback monopolizes its JS isolate; references/timers keep objects alive even when the request is done.

## Lab 13 — database connection exhaustion

**Broken:** every request constructs a new Prisma client/pool, or each of 12 replicas opens 30 connections without DB budget.

**Reproduce:** scale test replicas, inspect DB sessions and pool wait; terminate one instance.

**Observe:** connection cap reached, requests queued/time out, health probes may worsen outage.

**Fix:** one reusable client/pool per process, globally calculate max connections, account workers/migrations, use provider pooler where appropriate, set acquisition/query timeouts, shed load and alert on pool saturation.

**Mental model:** pools are finite process resources multiplied by replica count.

## Lab 14 — queue poison message and retry storm

**Broken:** malformed job retries forever every second; worker catches and logs but acknowledges success; payload contains whole private user record.

**Reproduce:** enqueue invalid schema version; run worker with retry config and inspect failed/active/pending counts.

**Observe:** queue age/depth grows, provider traffic spikes, PII sits in Redis, useful jobs starve.

**Fix:** validate versioned minimal payload; classify permanent vs transient error; bounded backoff/jitter; failed-job handling/replay policy; per-queue concurrency/rate limits; remove/expire records safely; alert on age and failures.

**Mental model:** persistent queue data outlives the API request and must be versioned, minimized, idempotent and operable.

## Lab 15 — oversized request / unsafe upload / path traversal

**Broken:** `express.json()` default/unbounded request, or `res.sendFile(req.query.path)`; file type trusted from extension.

**Reproduce:** send oversized payload, `../../etc/...`, double extension or mismatch between MIME and magic bytes against disposable app.

**Observe:** memory/CPU spike or unauthorized file read.

**Fix:** proxy + parser limits; runtime schema; fixed storage root/opaque object key; no user-controlled path; size/file-count cap; MIME + signature validation, quarantine/scan and tenant authorization.

**Mental model:** parsed strings and metadata are not filesystem-safe or proof of content.

## Lab 16 — health check causes restart loop

**Broken:** liveness endpoint queries DB/Redis, and orchestrator restarts process whenever a shared dependency has a transient outage.

**Reproduce:** stop local PostgreSQL while keeping process alive; probe liveness/readiness.

**Observe:** all replicas restart and reconnect simultaneously, compounding outage.

**Fix:** shallow liveness (process progress); readiness reports ability to accept traffic; bounded dependency checks; backoff/restart policy; circuit breaker and alert. Do not expose detailed health internals publicly.

**Mental model:** liveness decides whether a process should be killed; readiness decides whether it should receive traffic.

## Lab 17 — WebSocket/SSE resource leak and missing cross-node event

**Broken:** local in-process notification map never unsubscribes; one API replica publishes only to its own sockets; no origin/auth check on upgrade.

**Reproduce:** open/close connections repeatedly; connect clients to API replicas A/B; publish from A; attempt cross-tenant subscription.

**Observe:** connection count and heap climb; B never receives event; unauthorized origins/tenant channels may subscribe.

**Fix:** validate session/Origin/tenant at connection and subscription; enforce limits/heartbeat; unsubscribe on close; shared pub/sub or durable event cursor; stale-session revalidation/close policy.

**Mental model:** persistent connections are long-lived resources; sticky routing does not create shared event state.

## Lab 18 — path/query/API contract drift

**Broken:** Swagger says 201 but route returns 200; Express 4 wildcard syntax is copied into Express 5; raw `sort` is concatenated into SQL.

**Reproduce:** validate OpenAPI against sample requests, call paths with wildcard/root and `?sort=...` injection-shaped input.

**Observe:** client generator lies, valid route stops matching, or attacker controls query structure.

**Fix:** use Express 5 path grammar; enum-validate sort and map to SQL identifiers; validate OpenAPI/implementation in CI; add backward-compatible deprecation/migration check.

**Mental model:** the API contract is observable behavior, and route/query syntax is part of the attack surface.

## Team debugging exercise

Pick any three labs. Before reading the fix, write:

- reproduction command/test and exact preconditions;
- expected logs/metrics/traces and resource signals;
- root cause versus symptom;
- invariant the repair must preserve;
- one regression test and one production alert;
- failure behavior under horizontal scaling and dependency outage.

Then exchange reviews. A strong diagnosis explains **why** the failure occurred, proves the fix under concurrency/retry, and considers tenant security, observability and operations—not just the first passing happy-path request.
