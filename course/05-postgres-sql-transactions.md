# Module 5 — SQL, PostgreSQL, ORM tradeoffs, and concurrency

[← Module 4](04-types-validation-architecture.md) · [Course home](../README.md) · Next: [Authentication and security →](06-auth-security.md)

**Database snapshot:** PostgreSQL 18.6 is the newest stable minor in the current major line on 2026-09-28. PostgreSQL 19 remains Beta 4 and is not the production course baseline. PostgreSQL recommends running the current minor for a supported major. **ORM snapshot:** Prisma ORM 7.10.0 is the supported stable line; Prisma ORM 8 is a release candidate in this snapshot. Prisma ORM 7 requires a driver adapter for direct connections; use its v7 documentation, not the newest `/orm` docs by default.

## Concept — relational data is a contract

PostgreSQL persists rows in tables and enforces rules using data types, primary/foreign keys, unique constraints, check constraints, and transactions. Application-level checks are helpful but cannot protect a race if two processes act at once. Database constraints are the final safety net for invariants that must always hold.

### SQL foundations

```sql
CREATE TABLE organizations (
  id uuid PRIMARY KEY,
  name text NOT NULL CHECK (length(trim(name)) BETWEEN 1 AND 160),
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE products (
  id uuid PRIMARY KEY,
  organization_id uuid NOT NULL REFERENCES organizations(id),
  sku text NOT NULL,
  name text NOT NULL,
  price_cents integer NOT NULL CHECK (price_cents >= 0),
  available_stock integer NOT NULL CHECK (available_stock >= 0),
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, sku)
);

CREATE INDEX products_org_created_idx
  ON products (organization_id, created_at DESC, id DESC);
```

- A **primary key** uniquely identifies a row. A **foreign key** ensures the referenced row exists. **Constraints** express invariants close to durable data.
- **Joins** combine related rows. **Aggregates** (`COUNT`, `SUM`) summarize rows; `GROUP BY` forms groups; `HAVING` filters groups.
- A **CTE** (`WITH`) names a query component; it can improve clarity, but is not automatically faster.
- **Window functions** compute values over related rows while preserving row-level output (e.g., rank orders by time per organization).
- Normalize first to avoid update anomalies. Denormalize for measured read patterns with a consistency plan.
- JSONB is useful for flexible/provider metadata, not a substitute for columns, foreign keys, and constraints on core business entities.

## Mental model — a transaction is a protected business decision

ACID means atomicity (all or none), consistency (constraints/invariants remain true), isolation (concurrent work is controlled), durability (committed result survives according to database guarantees). A transaction is not just “group queries for convenience”; it defines what must commit together and the concurrency behavior under load.

### Create-order transaction

```mermaid
sequenceDiagram
  participant API as Express / Service
  participant DB as PostgreSQL
  API->>DB: BEGIN
  API->>DB: Reserve stock with conditional UPDATE
  DB-->>API: Updated row(s) or 0 rows
  API->>DB: INSERT order + order items + payment intent + outbox event
  DB-->>API: IDs / constraints validated
  API->>DB: COMMIT
  DB-->>API: Commit acknowledged
  API->>API: Return 201; outbox worker sends async confirmation
  Note over API,DB: Any failed step before COMMIT rolls back the business write set
```

Do not call a slow external payment/email service while holding a database transaction open. Persist the intent/outbox record transactionally, then do network work after commit and reconcile failures/idempotently.

## SQL before ORM

Practice these in `psql` before Prisma hides the syntax:

```sql
-- Tenant-scoped product read; tenant scope is part of the predicate.
SELECT id, sku, name, price_cents, available_stock
FROM products
WHERE organization_id = $1 AND id = $2;

-- Join order lines to catalog labels.
SELECT o.id, o.status, i.product_id, i.quantity, i.unit_price_cents
FROM orders AS o
JOIN order_items AS i ON i.order_id = o.id
WHERE o.organization_id = $1 AND o.id = $2;

-- Aggregate revenue by organization; WHERE filters input, HAVING filters groups.
SELECT organization_id, sum(total_cents) AS revenue_cents
FROM orders
WHERE created_at >= $1 AND status = 'paid'
GROUP BY organization_id
HAVING sum(total_cents) > 0;
```

Values must be parameters. Table/column identifiers and `ORDER BY` fragments generally cannot be treated as ordinary bind values—map validated enum choices to known SQL fragments; never concatenate an untrusted string.

## ORM selection: Prisma vs Drizzle

| Prisma ORM | Drizzle ORM |
|---|---|
| Declarative schema and generated typed client; readable CRUD and relations; migration workflow; approachable for application teams | SQL-shaped TypeScript query builder; lightweight mapping; explicit SQL control and close relational mental model |
| Strong fit when productivity/consistent models matter and the team accepts generated client/runtime conventions | Strong fit when engineers want query construction to look and behave close to SQL and need tight control |
| Prisma 7 direct database setup requires a driver adapter; v7 docs and migration guide matter | Drizzle ecosystem/API release state must be checked; v1 was beta in part of the 2026 docs, so pin a stable supported release rather than blindly copy “latest” |
| Raw SQL remains available and sometimes best | SQL expressions remain explicit; types still cannot make an unsafe query safe |

This course uses **Prisma 7.10.0** as the primary ORM and continues to teach SQL. Prisma 8 is an RC in this snapshot and is not the stable baseline. If Prisma setup has changed by the time you take this course, follow the official release-status page first.

### Reproducible Prisma 7 setup (September 2026 snapshot)

**Important:** on 2026-09-28, an unpinned `prisma` CLI install selects Prisma 8 RC. Prisma 8 is a separate migration path, with differently versioned packages and documented API gaps. Use the v7 docs and explicit package versions for this course; do not copy `npm install prisma` or `npx prisma` from a current homepage and accidentally mix majors.

```sh
pnpm add -D prisma@7.10.0
pnpm add @prisma/client@7.10.0 @prisma/adapter-pg@7.10.0 pg@8.23.0 dotenv@18.0.4
```

Prisma 7 moves the datasource URL to `prisma.config.ts` and requires a PostgreSQL driver adapter for direct connections. Keep the URL out of source control; use `.env` locally and platform-injected secrets in deployed environments.

```ts
// prisma.config.ts — local CLI configuration; production secrets are injected by deployment
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: env("DATABASE_URL") },
});
```

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

Run the locally pinned CLI (`pnpm exec prisma ...`) for every command, including migration generation/deploy. For example, `pnpm exec prisma generate` and `pnpm exec prisma migrate dev --name init`; production uses the reviewed `pnpm exec prisma migrate deploy` step. Never use an unpinned `npx prisma` in CI. Recheck official Prisma release status at upgrade time.


### Prisma model excerpt

```prisma
model Organization {
  id        String    @id @default(uuid()) @db.Uuid
  name      String
  products  Product[]
  orders    Order[]
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(6)

  @@map("organizations")
}

model Product {
  id             String   @id @default(uuid()) @db.Uuid
  organizationId String   @map("organization_id") @db.Uuid
  sku            String
  name           String
  priceCents     Int      @map("price_cents")
  availableStock Int      @default(0) @map("available_stock")
  createdAt      DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  organization   Organization @relation(fields: [organizationId], references: [id])

  @@unique([organizationId, sku])
  @@index([organizationId, createdAt, id])
  @@map("products")
}
```

A real complete schema also includes orders/items, explicit referential actions and timestamp columns; verify generated migration SQL and match table/column names. A Prisma schema is not a substitute for checking the resulting DDL.

Prisma 7 PostgreSQL direct connections use an adapter (and `prisma` CLI and generated client versions must be kept aligned). Conceptually:

```ts
// Bootstrap once per process. Exact generated-client path follows generator output.
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "./generated/prisma/client.js";

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });
export const prisma = new PrismaClient({ adapter });
```

Do not leave the non-null assertion as real configuration validation; Module 4 parses config before constructing the client. Configure connection pool capacity deliberately, close the client at shutdown, and budget all API/worker replicas against the database's connection limit.

## Transactions and race conditions

### The last item race

**BAD: read, then update in separate unconstrained steps.** Two requests may both observe stock `1` and create an order.

**GOOD: conditional update is atomic.**

```sql
UPDATE products
SET available_stock = available_stock - $3
WHERE organization_id = $1
  AND id = $2
  AND available_stock >= $3
RETURNING id, available_stock;
```

If no row is returned, stock is unavailable or the product is outside this tenant. Within the same DB transaction, insert the order and item rows. `CHECK (available_stock >= 0)` protects against a bug that bypasses the conditional update. The unique constraint on `(organization_id, sku)` prevents concurrent duplicate SKU creation.

For update collisions, use optimistic concurrency (`version` column / conditional `WHERE version = $old`) or row locking (`SELECT ... FOR UPDATE`) where a transaction genuinely needs exclusive inspection. Locks increase contention and can deadlock; keep transactions short, access rows in a consistent order, and retry only recognized serialization/deadlock failures with a bounded retry budget.

### Prisma interactive transaction sketch

```ts
const order = await prisma.$transaction(async (tx) => {
  const reserved = await tx.$queryRaw<Array<{ id: string; available_stock: number }>>`
    UPDATE products
       SET available_stock = available_stock - ${input.quantity}
     WHERE organization_id = ${organizationId}::uuid
       AND id = ${input.productId}::uuid
       AND available_stock >= ${input.quantity}
     RETURNING id, available_stock
  `;

  if (reserved.length !== 1) throw new ConflictError("Insufficient inventory");

  const created = await tx.order.create({
    data: {
      organizationId,
      customerId: userId,
      status: "pending",
      items: { create: [{ productId: input.productId, quantity: input.quantity, unitPriceCents }] },
      totalCents: unitPriceCents * input.quantity,
    },
  });

  await tx.paymentIntent.create({ data: { orderId: created.id, status: "requires_action" } });
  await tx.outboxEvent.create({ data: { type: "order.created", aggregateId: created.id } });
  return created;
}, { timeout: 5_000 });
```

This is a teaching sketch: validate bounded integer quantities; compute money with integer minor units/appropriate numeric handling; verify organization-scoped product lookup and price snapshot; match actual model names; configure a transaction timeout; and test actual PostgreSQL behavior. For raw SQL, Prisma's tagged template parameters are values, but dynamic SQL identifiers still need allowlisted mapping. Do not use unsafe raw query methods with interpolated input.

## N+1 queries and query shape

**BAD:** get 100 users, then issue one order query per user: `1 + N = 101` round trips. This increases DB and network latency and can exhaust the connection pool.

**Better:** one join/aggregate query or ORM relation query, depending projection and cardinality. Select only necessary columns; paginate; avoid returning huge nested objects.

```ts
// One relation query, but inspect SQL and choose relation loading behavior deliberately.
const users = await prisma.user.findMany({
  where: { organizationId },
  select: {
    id: true,
    displayName: true,
    orders: { select: { id: true, totalCents: true, createdAt: true }, take: 5 },
  },
  take: 50,
});
```

An ORM relation query can still be expensive. Detect N+1 using query logs in development, DB telemetry, query count assertions for important use cases, and traces showing repeated spans. Eager loading everything can create a different problem: large joins/cardinality explosion.

## Indexes, pagination, and query plans

An index speeds selected lookups/orderings at the cost of storage and write work. Build indexes from observed access patterns, often with tenant ID first for a tenant-scoped SaaS query. Verify the planner with `EXPLAIN`; use `EXPLAIN (ANALYZE, BUFFERS)` carefully because it executes the query. Avoid using `ANALYZE` on expensive writes in production without understanding consequences.

Offset pagination (`OFFSET 100000`) may scan/discard many rows and can shift during concurrent writes. Cursor pagination uses a stable ordered key:

```sql
SELECT id, name, created_at
FROM products
WHERE organization_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT $4;
```

A cursor must be opaque/validated and the sort must be deterministic (tie-break with unique ID). A page number is simpler for small result sets; a cursor is better for deep, changing feeds.

## Migrations, isolation, connection pools

- Development: generate and inspect migration; apply to a disposable/local DB.
- CI: start isolated DB, apply all migrations from zero, then run tests.
- Production: use a reviewed deployment migration command (`prisma migrate deploy` for Prisma migration history), separate migration from API startup when coordination matters, and plan backward-compatible expand → migrate data → contract changes.
- Never let production startup silently reset a database. Destructive migration rollback may be impossible; create a forward-fix/backup plan.
- PostgreSQL default isolation is Read Committed. Serializable gives stronger guarantees but can abort transactions and requires safe retry logic. Learn the invariant, then choose isolation/locking; do not set the strongest level everywhere blindly.
- Pool sizes are finite. Pool wait time is user latency and may be a saturation signal. Size from DB capacity, replicas, workers, admin/monitoring, and connection proxies.

## Common mistakes

- Trusting ORM types instead of DB constraints.
- Writing unsafe SQL by concatenating sort/filter strings.
- Putting a network payment/email call inside an open transaction.
- Checking inventory with a read then doing an unconditional update.
- Adding indexes without measuring write/storage cost.
- Offset paging through huge tables.
- Returning whole ORM records, including internal/security fields.
- Ignoring ORM-generated SQL and assuming relation loading avoids N+1.
- Confusing test mocks with PostgreSQL integration tests.

## Security / performance review

At the repository boundary, every user/tenant-owned query includes the tenant key. Use parameterized values; allowlist sort columns; whitelist writable fields; use DB roles with least privilege; avoid leaking constraints/SQL errors; cap page size. Inspect query count, selected columns, index usage, rows scanned, lock wait, pool wait, and transaction duration.

## Exercises

- **Beginner:** write tables for Organization, User, Membership, Product, Order, OrderItem; identify primary/foreign/unique/check constraints.
- **Intermediate:** query products with category, price filters and deterministic cursor pagination; write a useful composite index and explain why.
- **Production:** implement order+items+payment intent+outbox in one transaction with inventory reservation and idempotency record.
- **Debugging:** reproduce a last-item race with parallel transactions; show why application-only `if (stock > 0)` fails and how the conditional update/constraint fixes it.
- **Architecture:** choose Prisma relation query, query builder, or SQL for a complex analytics endpoint. Define the point where abstraction is no longer helping.

## Build this yourself before looking at a solution

Use `psql` first: create constrained tables, join orders, aggregate revenue, inspect `EXPLAIN`, and write the inventory update. Then map a small subset with Prisma 7 and compare generated SQL. **Expected architecture:** PostgreSQL is the truth; Prisma handles routine access; SQL remains an explicit tool; transaction boundaries match business invariants. **Review:** exercise concurrent requests and inspect migrations before accepting them.

## Official documentation

- [PostgreSQL 18 manual](https://www.postgresql.org/docs/18/) · [SQL tutorial](https://www.postgresql.org/docs/18/tutorial.html) · [constraints](https://www.postgresql.org/docs/18/ddl-constraints.html)
- [Indexes](https://www.postgresql.org/docs/18/indexes.html) · [transactions/isolation](https://www.postgresql.org/docs/18/transaction-iso.html) · [explicit locking](https://www.postgresql.org/docs/18/explicit-locking.html) · [EXPLAIN](https://www.postgresql.org/docs/18/using-explain.html)
- [Version policy/current minor](https://www.postgresql.org/support/versioning/) · [PostgreSQL 18 release](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [Prisma ORM release status](https://www.prisma.io/docs/orm/release-status) · [Prisma 7 docs](https://www.prisma.io/docs/orm/v7) · [Prisma transactions](https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions)
- [Drizzle PostgreSQL overview](https://orm.drizzle.team/docs/get-started-postgresql)

## What I should know before continuing

You can write joins/aggregates and parameterized queries; explain constraints, indexes and a query plan; implement atomic inventory reservation; describe transaction isolation and retry tradeoffs; spot N+1 and pool saturation; and use Prisma without delegating all database thinking to it. Next: [Module 6 — Authentication and security](06-auth-security.md).
