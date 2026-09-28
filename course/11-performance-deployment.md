# Module 11 — Performance, reliability, Docker, CI/CD, and production operations

[← Module 10](10-testing-contracts-observability.md) · [Course home](../README.md) · Next: [Capstone PRD and design review →](12-capstone-prd.md)

## Concept — production is a runtime contract

A production backend runs under resource, network and failure constraints. Performance is not only requests per second: use **throughput + latency distribution (p50/p95/p99) + error rate + saturation**. Measure the system's bottleneck: event loop/CPU, DB query/pool, Redis, external service, serialization, or network. Avoid optimizing a benchmark that does not resemble the product.

## Performance review workflow

1. Define a representative load and success/error objective.
2. Measure baseline p50/p95/p99, throughput, concurrency, resource use and errors.
3. Trace slow requests; separate app time, pool wait, SQL, provider and serialization.
4. Inspect query plan, query count, payload size, event-loop delay, logging and GC.
5. Change one bottleneck; repeat same test; retain correctness/security checks.
6. Test overload and graceful degradation, not only the happy path.

For `GET /products`, test cursor/index/Redis cache behavior and payload size. For `POST /orders`, test DB locking/transaction/pool behavior with concurrency and duplicate keys. Do not use an in-memory mock as a load benchmark for PostgreSQL.

### Event-loop blocking

**BAD:** synchronous large CSV transform / regex / `JSON.stringify` of huge response in a route. All callbacks in that isolate wait.

**Better options:** improve algorithm, stream/chunk data, move CPU transform to a bounded Worker Thread pool, or enqueue durable work to a worker process. Workers are not a replacement for async I/O and are not free—message cloning and queue coordination cost time/memory.

### Capacity and pool math

Estimate resources across all replicas. If PostgreSQL max connections = 200 and 10 containers each create 20 pool connections, the API alone may consume the full DB capacity; add workers, migrations and monitoring. Bound HTTP agent sockets, DB/Redis connections, open files, queue concurrency, in-memory cache and request body sizes. A pool's queue can hide overload until latency spikes.

Use `process.memoryUsage()`, heap snapshots, CPU profiles, `monitorEventLoopDelay()`, PostgreSQL `pg_stat_activity`, slow query telemetry, Redis latency/eviction metrics, queue age, and container CPU/memory. Memory leak suspects: retained request closures, global maps/caches, listeners, intervals, streams not closed, and unbounded job/result retention.

## Timeouts and retries as one budget

Define request deadline and downstream budgets so the server can still respond. Example: edge 8 s > API deadline 6 s > provider 2 s + DB 2 s + response margin. Avoid every layer using a longer timeout than its caller. Use AbortSignal for fetch; configure DB statement/transaction timeouts; configure proxy idle/read timeouts; queue jobs get an execution timeout and bounded attempts.

Retries require idempotency/safe semantics. Use exponential backoff + jitter, max attempts, max elapsed time, `Retry-After`, and retry budget. No retries for validation/auth errors. Circuit breakers fail fast during sustained dependency outages. Fallback must preserve meaning: stale product catalog may be acceptable; guessed checkout price is not.

## Health, readiness, startup, graceful shutdown

- **Liveness:** is the process alive and able to make progress? Keep shallow; do not restart because a shared DB has a transient outage.
- **Readiness:** should this instance receive new traffic? false during startup, drain, or required-dependency failure.
- **Health/diagnostics:** operator-oriented component status; protect detailed diagnostics from public access.

Startup order: validate config → initialize telemetry/logging → connect DB/Redis/queue clients → apply dependency checks/migrations as deployment step → listen → mark ready. Do not listen before required services are connected unless the design explicitly supports degraded startup.

```ts
import { createServer } from "node:http";
import { app } from "./app.js";

const server = createServer(app);
let ready = false;

async function start() {
  validateConfig();
  await prisma.$connect();
  await redis.connect();
  await worker.waitUntilReady();
  server.listen(config.port, "0.0.0.0", () => { ready = true; });
}

app.get("/liveness", (_req, res) => res.status(200).json({ status: "alive" }));
app.get("/readiness", (_req, res) => {
  res.status(ready ? 200 : 503).json({ status: ready ? "ready" : "not_ready" });
});

let shuttingDown = false;
async function shutdown(signal: string) {
  if (shuttingDown) return;
  shuttingDown = true;
  ready = false; // load balancer stops routing new requests
  logger.info({ signal }, "shutdown started");

  const force = setTimeout(() => {
    logger.error("shutdown deadline exceeded");
    process.exit(1);
  }, 25_000);
  force.unref();

  try {
    await new Promise<void>((resolve, reject) => {
      server.close((error) => error ? reject(error) : resolve());
    });
    await worker.close();       // stop claiming work; drain or let jobs retry by policy
    await emailQueue.close();
    await redis.quit();
    await prisma.$disconnect();
    clearTimeout(force);
    process.exitCode = 0;
  } catch (error) {
    logger.error({ error }, "graceful shutdown failed");
    process.exitCode = 1;
  }
}

process.once("SIGTERM", () => void shutdown("SIGTERM"));
process.once("SIGINT", () => void shutdown("SIGINT"));
void start().catch((error) => {
  logger.fatal({ error }, "startup failed");
  process.exitCode = 1;
});
```

The API and worker may run as separate process roles; do not start an email worker in every API process unless that is explicitly intended. The snippet's methods are illustrative and need matching current client lifecycle APIs. Ensure shutdown deadline, websocket closes, in-flight request policy, queue lock extension, and readiness behavior are tested. Avoid `process.exit()` during normal shutdown before buffered logs/cleanup complete.

## Docker and local development

One container per process role; run as non-root; use a small maintained base; copy only built artifacts; pin dependency lockfile; configure health checks and resource limits in deployment. Do not bake secrets into image layers. Local Compose should use named volumes and local-only credentials.

```dockerfile
# Dockerfile (illustrative multi-stage build; pin image digest in a real release)
FROM node:24-bookworm-slim AS build
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY tsconfig.json ./
COPY src ./src
RUN pnpm build

FROM node:24-bookworm-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app
RUN corepack enable && chown node:node /app
COPY --from=build --chown=node:node /app/package.json /app/pnpm-lock.yaml ./
RUN pnpm install --prod --frozen-lockfile
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "--enable-source-maps", "dist/server.js"]
```

Review the exact Node image tag/digest and Corepack/package-manager behavior before using. For pnpm 12, pin `packageManager` and use the official pnpm 12 installation guidance. A production image should not require TypeScript/dev dependencies at runtime if compilation is a build stage.

```yaml
# compose.yaml — local development sketch (not production secrets/config)
services:
  postgres:
    image: postgres:18.6
    environment:
      POSTGRES_DB: course
      POSTGRES_USER: course
      POSTGRES_PASSWORD: local-only
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U course -d course"]
      interval: 5s
      timeout: 3s
      retries: 10
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:8
    command: ["redis-server", "--appendonly", "yes"]
    volumes: [redisdata:/data]
  api:
    build: .
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://course:local-only@postgres:5432/course
      REDIS_URL: redis://redis:6379
    ports: ["3000:3000"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
  worker:
    build: .
    command: ["node", "--enable-source-maps", "dist/worker.js"]
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://course:local-only@postgres:5432/course
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
volumes:
  pgdata:
  redisdata:
```

Do not publish DB/Redis ports in production. On a Docker network, use service DNS (`postgres`, `redis`), not `localhost` from the API container. Compose `depends_on` does not replace runtime reconnect/failure handling.

## CI/CD pipeline and dependency hygiene

```text
pull request / push
  → frozen install from lockfile
  → formatting + lint + TypeScript typecheck
  → unit tests
  → PostgreSQL/Redis integration tests
  → build + OpenAPI validation + migration check
  → dependency/security scan + secret scan + SBOM
  → build immutable image + provenance/signing policy
  → deploy staging → smoke/health checks → production rollout
  → monitor SLO/error budget; rollback/forward-fix decision
```

Use reviewed feature branches/PRs, lockfiles, semantic version ranges with reviewed updates, and migration compatibility. `npm audit`/equivalent identifies known advisories but does not prove safety. Add Dependabot/Renovate concepts, SBOM, code scanning, dependency allowlists/build script review, and secrets scanning. Pin GitHub Actions by full commit SHA in high-assurance CI. Do not ask a production install to silently fetch unreviewed versions.

Database deployments should use **expand → migrate → contract**: add compatible nullable/new structures; deploy code that can read old/new; backfill; switch behavior; later remove old fields. Use a forward repair plan for destructive schema/data migrations. Do not couple every API replica's startup to running migrations concurrently.

## Git workflow and feature flags

Work on a focused feature branch; keep a PR reviewable; add migration and contract notes with behavior changes; run CI before merge; do not edit production data/schema by hand. Migrations are deployable artifacts and need compatibility review. A rollback of application code is only safe if its schema and queued job payloads remain compatible.

Feature flags enable staged rollout, kill switches and per-tenant capabilities. Keep flag evaluation server-side for authorization-sensitive behavior; a UI-hidden button is not a security boundary. Define owner, default, expiry/removal issue, audit and emergency kill behavior. Prefer a small configuration-backed flag set before building a flag platform.

## Reverse proxy, load balancer, process management

A reverse proxy/load balancer commonly handles TLS termination, request/header/body limits, compression, static assets, caching, health routing, upstream timeouts, and load distribution. Express should generally run behind a trusted proxy in production. Set `trust proxy` to a precise hop count/network/config per hosting topology; never broadly trust arbitrary client forwarded headers.

Modern container orchestration typically runs one Node process per container and scales replicas. Node's `cluster` / PM2 can manage multiple processes on a VM, but does not replace shared sessions, queues, health management or orchestration. PM2 is common knowledge and may be appropriate on a VM; in managed/container platforms use their supervisor/restart policy and avoid double-supervising without a reason.

Stateless HTTP means a replica can serve a request without relying on its own memory as the only copy of session/job/cache state. Store durable/shared state in PostgreSQL, Redis, object storage or a queue. Sticky sessions can mask local state but are not a substitute for shared durable state.

## Load testing and degraded operation

Use k6 or autocannon after checking current official version/API. Establish a safe isolated environment and representative payload/tenant data. Record concurrency, throughput, p50/p95/p99, errors, saturation and DB/Redis/provider behavior. Requests/sec alone can hide a 3-second p99 or high error rate. Test ramp, steady state, spike and recovery. Don't load-test a third-party provider without permission.

Graceful degradation examples:

- analytics unavailable: serve checkout; omit analytics or return last successful snapshot with timestamp.
- email provider slow: commit business action, enqueue email, retry asynchronously.
- Redis cache unavailable: bypass only if DB capacity can absorb traffic; otherwise shed load/return 503.
- DB unavailable: fail readiness; bounded request failure, no unbounded retries.

## Architecture choices and when not to use Express

| Choice | Strength | Cost / good alternative when |
|---|---|---|
| Express | Small, flexible, broad ecosystem | Requires team conventions and assembled components; Fastify offers more schema/serialization conventions/performance focus; Nest offers stronger application structure/DI |
| Modular monolith | One deployable, clear module boundaries, simple transactions | Split when independent scaling/ownership/release/security boundaries justify network complexity |
| Microservices | Independent deploy/scale/ownership and failure boundaries | Add network retries, data consistency, observability and on-call complexity; do not split by table or fashion |
| Node API process | Excellent for I/O-heavy HTTP work | CPU-heavy tasks belong in efficient algorithms/workers/other runtimes |
| Serverless | Event-driven, scale-to-zero, managed operational model | Cold start, duration, socket/pool and runtime constraints require adaptation |

Choose another framework/runtime for specialized edge environments, strict RPC contracts, strongly opinionated enterprise modularity, or a measured performance requirement that justifies migration and ecosystem costs. Express is not inherently wrong; the goal is choosing intentionally.

## Common mistakes

- `NODE_ENV` not set to production.
- Application listens before config/dependencies are ready.
- Readiness always returns 200 while DB is unavailable.
- Liveness performs a slow DB query and causes restart loops.
- SIGTERM kills an in-flight order/worker job with no recovery plan.
- DB pool budget ignores replicas/workers.
- Docker image runs as root or includes `.env`/source credentials.
- `localhost` used for another Compose service.
- Migrations run concurrently on every API replica.
- Retries extend beyond caller deadline.
- Load test reports only RPS.
- PM2/cluster added without shared state design.

## Production checklist

- Latest supported Active LTS Node patch; Express current supported 5.x.
- `NODE_ENV=production`; validated config/secrets; dependency/lockfile review.
- TLS at a configured proxy, exact `trust proxy`, Helmet/CORS/body/request limits.
- Async error handling and safe logs; no synchronous request-path operations.
- Compression/caching/log processing where appropriate at proxy or worker.
- DB indexes, connection pool budgets, cache policy, bounded outbound deadlines.
- Restart policy, health/readiness/liveness, graceful shutdown, load balancing.
- Backups/restore test, migration strategy, alerting/runbooks, staged rollout/rollback.

## Exercises

- **Beginner:** distinguish liveness/readiness and propose paths/status codes.
- **Intermediate:** write a Docker Compose dev stack and show service DNS/secret handling.
- **Production:** calculate DB/Redis connection and job concurrency budgets for 6 API replicas + 2 workers.
- **Debugging:** simulate SIGTERM during in-flight requests and a worker retry; prove no order loss/duplication.
- **Architecture:** draw the deployment topology, proxy trust chain, failure modes, data backups, and rollback strategy.

## Build this yourself before looking at a solution

Create an operational readiness package: Dockerfile, Compose, startup validation, health routes, signal shutdown, CI stages, migration plan, load-test plan, dashboards, alerts, and runbook for Redis/DB/provider outage. **Expected architecture:** separate API/worker roles; observable and drainable process lifecycle; durable DB truth. **Review:** threat-model secrets, network exposure, image supply chain, and data recovery.

## Official documentation

- [Express production performance/reliability](https://expressjs.com/en/advanced/best-practice-performance/) · [Express production security](https://expressjs.com/en/advanced/best-practice-security.html)
- [Docker docs](https://docs.docker.com/) · [Docker Compose](https://docs.docker.com/compose/) · [Node image](https://hub.docker.com/_/node)
- [Node process signals](https://nodejs.org/docs/latest-v24.x/api/process.html#signal-events) · [Node HTTP server](https://nodejs.org/docs/latest-v24.x/api/http.html#class-httpserver)
- [PostgreSQL migrations concepts](https://www.postgresql.org/docs/18/ddl-alter.html) · [Prisma deploy migrations](https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production)
- [k6 docs](https://grafana.com/docs/k6/latest/) · [autocannon](https://github.com/mcollina/autocannon) · [pnpm 12 release](https://pnpm.io/blog/releases/12.0)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## What I should know before continuing

You can quantify latency/saturation; trace bottlenecks; size pools across replicas; set bounded timeout/retry budgets; distinguish health states; drain requests/jobs on shutdown; deploy immutable containers behind a proxy; and explain when a modular monolith, Express, worker process or alternative runtime is appropriate. Next: [Module 12 — capstone PRD and your design submission](12-capstone-prd.md).
