# Architecture reference — required sequence diagrams

These diagrams are learning reference diagrams for the capstone. They show one reasonable flow, not a claim that every product must use exactly this design. Always revisit atomicity, retries, authorization and failure behavior for your system.

## HTTP request / lifecycle

```mermaid
sequenceDiagram
  participant Client
  participant DNS
  participant Proxy as Reverse Proxy / Load Balancer
  participant Node as Node HTTP
  participant Express
  participant Service
  participant Repo as Repository
  participant DB as PostgreSQL / Redis
  Client->>DNS: Resolve hostname
  DNS-->>Client: Address
  Client->>Proxy: HTTPS request (TCP/TLS or QUIC/TLS)
  Proxy->>Proxy: Validate limits / select healthy API
  Proxy->>Node: Forward trusted request
  Node->>Express: IncomingMessage + ServerResponse
  Express->>Express: Request ID → security → auth → validation → router
  Express->>Service: Typed input + principal + tenant context
  Service->>Repo: Business operation
  Repo->>DB: Scoped parameterized query / transaction
  DB-->>Repo: Result
  Repo-->>Service: Domain result
  Service-->>Express: Result / typed error
  Express-->>Node: Status + headers + representation
  Node-->>Proxy: HTTP response
  Proxy-->>Client: HTTPS response
```

## Express middleware control flow

```mermaid
sequenceDiagram
  participant R as Request
  participant M1 as Request ID middleware
  participant M2 as Security / body parser
  participant M3 as Auth / validation
  participant Router
  participant Handler
  participant Error as Error middleware
  participant Response
  R->>M1: next()
  M1->>M2: next()
  M2->>M3: next() or short-circuit response
  M3->>Router: next() or next(error)
  Router->>Handler: route match
  Handler-->>Response: send once
  Note over Handler,Error: throw / rejected Promise is forwarded by Express 5
  Handler->>Error: error flow
  Error-->>Response: safe mapped error
```

## Login

```mermaid
sequenceDiagram
  participant Browser
  participant Express
  participant Validation
  participant Auth as Auth Service
  participant DB as User / Session DB
  Browser->>Express: POST login credentials
  Express->>Validation: Validate email/password and size
  Validation-->>Express: Typed credentials
  Express->>Auth: Authenticate
  Auth->>DB: Read normalized user + password hash
  DB-->>Auth: User record
  Auth->>Auth: Verify Argon2id, apply throttle
  Auth->>DB: Rotate old ID, persist new session
  DB-->>Auth: Session created
  Auth-->>Express: Principal + opaque session ID
  Express-->>Browser: 200 + Secure HttpOnly SameSite cookie
```

## Refresh token rotation (JWT alternative)

```mermaid
sequenceDiagram
  participant Browser
  participant Express
  participant Auth as Auth Service
  participant DB as Refresh Token Store
  Browser->>Express: POST refresh + cookie + CSRF context
  Express->>Auth: Verify request boundary
  Auth->>DB: Lookup hash, family, expiry and status
  DB-->>Auth: Active token or rejected
  Auth->>DB: Atomically consume old token and create replacement
  DB-->>Auth: Rotation committed
  Auth-->>Express: New short access state + refresh token
  Express-->>Browser: Set replacement HttpOnly cookies
  Note over Auth,DB: Reuse of a consumed refresh token revokes token family
```

## Logout and revocation

```mermaid
sequenceDiagram
  participant Browser
  participant Express
  participant Auth
  participant Store as Session / Token Store
  Browser->>Express: POST logout + cookie + CSRF proof
  Express->>Auth: Resolve current identity
  Auth->>Store: Revoke session or refresh-token family
  Store-->>Auth: Revoked
  Auth-->>Express: Complete
  Express-->>Browser: 204 + clear cookie with matching options
```

## Authorization

```mermaid
sequenceDiagram
  participant Client
  participant Express
  participant Session as Session / Token verifier
  participant Policy as Authorization policy
  participant Service
  participant DB as PostgreSQL
  Client->>Express: Request resource in requested tenant
  Express->>Session: Authenticate credential
  Session-->>Express: User principal
  Express->>Policy: Resolve active membership + permission
  Policy-->>Express: Scoped principal/decision
  Express->>Service: Use case + tenant + principal
  Service->>DB: Query WHERE resource_id AND tenant_id AND owner/policy scope
  DB-->>Service: Row / not found
  Service-->>Express: Result / denial
  Express-->>Client: Minimal representation or 403/404
```

## Database transaction (order + inventory + payment intent)

```mermaid
sequenceDiagram
  participant Service
  participant DB as PostgreSQL
  Service->>DB: BEGIN
  Service->>DB: Atomically reserve stock WHERE available >= quantity
  DB-->>Service: Reserved row / no row
  Service->>DB: Insert idempotency record + order + item snapshots
  Service->>DB: Insert payment intent + outbox event
  Service->>DB: COMMIT
  DB-->>Service: Committed durable result
  Note over Service,DB: Any failure before commit rolls back all local business writes
```

## Cache-aside read

```mermaid
sequenceDiagram
  participant Client
  participant API
  participant Redis
  participant DB as PostgreSQL
  Client->>API: GET tenant-scoped product
  API->>Redis: GET versioned tenant key
  alt hit
    Redis-->>API: Cached representation
  else miss
    Redis-->>API: nil
    API->>DB: SELECT scoped row
    DB-->>API: Current row
    API->>Redis: SET JSON with bounded TTL
  end
  API-->>Client: Response
```

## Cache invalidation after write

```mermaid
sequenceDiagram
  participant API
  participant DB as PostgreSQL
  participant Outbox
  participant Redis
  API->>DB: BEGIN, update product, insert cache-invalidation outbox event
  API->>DB: COMMIT
  DB-->>API: Success
  API-->>Client: Updated product
  Outbox->>Redis: Delete/invalidate versioned detail/list keys (retryable)
  Redis-->>Outbox: Acknowledged
  Note over DB,Redis: PostgreSQL commit and Redis delete are not one distributed transaction, TTL/versioning bounds stale data
```

## Queue + worker

```mermaid
sequenceDiagram
  participant API
  participant DB as PostgreSQL
  participant Relay as Outbox Relay
  participant Q as Queue
  participant W as Worker
  participant Email as Email Provider
  API->>DB: Commit order + outbox record
  Relay->>DB: Read undelivered outbox row
  Relay->>Q: Publish idempotent job with event ID
  Q->>W: Deliver job (possibly repeated)
  W->>DB: Reload minimal tenant-scoped order
  W->>Email: Send confirmation with provider idempotency key
  Email-->>W: Accepted / transient failure
  W-->>Q: Acknowledge or bounded retry
```

## Webhook verification and idempotency

```mermaid
sequenceDiagram
  participant Provider
  participant API as Express raw-body route
  participant Verify as Signature verifier
  participant DB as PostgreSQL
  participant Queue
  Provider->>API: POST signed raw bytes + event ID
  API->>Verify: Verify signature/timestamp before JSON parse
  Verify-->>API: Verified / reject
  API->>DB: Transaction: insert unique event ID + payment state + outbox
  alt new event
    DB-->>API: Commit
    API-->>Provider: 204 accepted
    Queue->>DB: Relay durable outbox event
  else duplicate event ID
    DB-->>API: Unique conflict / prior processed state
    API-->>Provider: 204 already accepted
  end
```

## Payment request idempotency

```mermaid
sequenceDiagram
  participant Client
  participant API
  participant DB as PostgreSQL
  participant Pay as Payment Provider
  Client->>API: POST /payments + Idempotency-Key K
  API->>DB: Begin, reserve unique (tenant,user,operation,K)
  alt first request
    API->>DB: Persist request hash + pending operation
    API->>Pay: Create intent using provider idempotency key
    Pay-->>API: Provider intent
    API->>DB: Persist result + operation state
    API->>DB: Commit
    API-->>Client: Result
  else same key and same request hash
    DB-->>API: Existing result / in-progress state
    API-->>Client: Replay stable result or 202 status
  else same key but different body
    DB-->>API: Request hash mismatch
    API-->>Client: 409 conflict
  end
```

Do not hold a DB transaction open around a slow provider call in a real implementation. The diagram is conceptual: model an explicit pending state and durable reconciliation/outbox workflow while preserving unique idempotency reservation and provider dedupe.

## File upload with signed URL

```mermaid
sequenceDiagram
  participant Browser
  participant API
  participant DB as PostgreSQL
  participant Store as Private Object Storage
  participant Worker as Scanner / Transform Worker
  Browser->>API: Request upload intent (tenant, purpose, size)
  API->>API: Authenticate + authorize + validate limits
  API->>DB: Create pending file metadata and opaque object key
  API-->>Browser: Short-lived signed URL and required headers
  Browser->>Store: Upload directly to quarantine object
  Store-->>Browser: Upload acknowledgement
  Browser->>API: Confirm upload
  API->>DB: Verify object metadata / queue scan event
  Worker->>Store: Fetch bytes and inspect signature / scan
  Worker->>DB: Mark safe or reject
```

## WebSocket connection

```mermaid
sequenceDiagram
  participant Browser
  participant Proxy
  participant API as Node HTTP + ws
  participant Auth as Session / Origin checks
  participant Bus as Redis Pub/Sub / event history
  Browser->>Proxy: HTTPS request with Upgrade headers
  Proxy->>API: Forward upgrade on long-lived connection
  API->>Auth: Validate Origin, session, limits and subscription policy
  Auth-->>API: Principal or reject upgrade
  API-->>Browser: 101 Switching Protocols
  Browser->>API: Typed subscription message
  API->>Auth: Authorize channel/resource
  Bus-->>API: Tenant/user notification
  API-->>Browser: Validated event frame
```

## Server-Sent Events

```mermaid
sequenceDiagram
  participant Browser
  participant API as Express SSE route
  participant Session as Session store
  participant Hub as Shared event source
  Browser->>API: GET event stream + cookie
  API->>Session: Authenticate and authorize subscriptions
  Session-->>API: Principal
  API-->>Browser: 200 text/event-stream + ready event
  Hub-->>API: Notification with event ID
  API-->>Browser: id / event / data frame
  Browser-->>API: Network close / reconnect with Last-Event-ID
  API->>Hub: Remove subscription and close resources
```

## Graceful shutdown

```mermaid
sequenceDiagram
  participant Orchestrator
  participant LB as Load Balancer
  participant API
  participant Queue
  participant DB as PostgreSQL / Redis
  Orchestrator->>API: SIGTERM
  API->>API: Mark readiness false
  API-->>LB: Readiness 503
  LB->>API: Stop routing new traffic
  API->>API: Stop accepting connections, drain in-flight HTTP
  API->>Queue: Stop claiming jobs, complete or release for retry
  API->>API: Close SSE/WebSocket connections by policy
  API->>DB: Close Redis/DB/queue connections
  DB-->>API: Closed
  API-->>Orchestrator: Exit before termination deadline
```

## Deployment topology

```mermaid
flowchart TB
  Internet --> TLS[HTTPS / TLS]
  TLS --> Proxy[Reverse Proxy / Load Balancer]
  Proxy --> API1[Node API replica 1]
  Proxy --> API2[Node API replica 2]
  API1 --> PG[(PostgreSQL: durable business truth)]
  API2 --> PG
  API1 --> Redis[(Redis: cache / rate limit / queue backend)]
  API2 --> Redis
  API1 --> Q[Queue]
  API2 --> Q
  Q --> Worker[Node worker process]
  Worker --> PG
  Worker --> Email[Email provider]
  Worker --> Storage[Private object storage]
  API1 --> Storage
  API2 --> Storage
```

## Shutdown and deployment review questions

Does the proxy's drain window exceed API/worker shutdown time? Can clients retry safely? Does a job lease recover after process kill? Are websockets reconnected with jitter? Can a worker publish the same outbox event twice? Are Redis/PostgreSQL shared state policies correct across replicas? Is every service observable and its secret/network scope minimized?
