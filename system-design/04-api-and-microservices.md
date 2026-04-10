# System Design — API Design & Microservices

## API Design

### REST Best Practices
```
Resources are nouns, not verbs:
  ✓ GET    /orders/{id}
  ✗ GET    /getOrder?id=123

Correct status codes:
  200 OK              — success with body
  201 Created         — POST/PUT created new resource
  204 No Content      — success, no body (DELETE)
  400 Bad Request     — client input error
  401 Unauthorized    — not authenticated
  403 Forbidden       — authenticated but no permission
  404 Not Found       — resource doesn't exist
  409 Conflict        — state conflict (duplicate, optimistic lock)
  422 Unprocessable   — validation error
  429 Too Many Requests
  500 Internal Error  — never leak stack traces

Versioning:
  /v1/orders          — URL versioning (most common, explicit)
  Accept: v=2         — header versioning (cleaner URLs)
  ?version=2          — query param (least preferred)
```

### REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP/1.1 | HTTP/1.1 or 2 | HTTP/2 |
| Data format | JSON | JSON | Protocol Buffers (binary) |
| Schema | OpenAPI (optional) | Strongly typed schema | .proto file (required) |
| Fetching | Fixed endpoints | Client specifies fields | Service-defined methods |
| Over/under-fetch | Common issue | Solved | N/A (RPC style) |
| Real-time | Polling/SSE/WS | Subscriptions (WS) | Bidirectional streaming |
| Performance | Medium | Medium | High (binary, multiplexing) |
| Best for | Public APIs, CRUD | BFF, mobile, flexible queries | Internal service-to-service |

**When to use GraphQL**: Multiple clients (web, mobile) needing different data shapes; BFF (Backend for Frontend) pattern; rapidly changing frontend requirements.

**When to use gRPC**: Internal microservice communication; high-throughput, low-latency; polyglot services with strict contracts.

---

## Microservices Architecture

### Decomposition Strategies

**By Business Capability**
- Align services to business functions (Order Service, Payment Service, Inventory Service)
- Most natural and durable decomposition
- Conway's Law: org structure mirrors system structure

**By Subdomain (Domain-Driven Design)**
- **Bounded Context**: explicit boundary where a domain model applies
- **Aggregate**: cluster of objects treated as a unit; root entity controls access
- Each bounded context → candidate microservice
- **Ubiquitous Language**: same terminology in code and business discussions

**Strangler Fig Pattern**
- Incrementally migrate monolith to microservices
- Route traffic to new service; retire monolith piece by piece
- Never "big bang" rewrite

### Service Communication

**Synchronous (HTTP/gRPC)**
```
Client → Service A → Service B → Service C
```
- Simple to reason about
- Tight temporal coupling: A waits for B waits for C
- Failure of C causes failure of A
- Latency = sum of all hops

**Asynchronous (Events/Messages)**
```
Service A → publishes event → Message Broker → Service B (async)
```
- Temporal decoupling: B can be down, A still works
- Better resilience
- Harder to trace, eventual consistency

**Best Practice**: Async for writes/mutations, sync for queries where immediate response needed

### API Gateway
```
Client
  ↓
API Gateway (auth, rate-limit, routing, SSL, logging)
  ↓         ↓         ↓
User Svc  Order Svc  Product Svc
```
- Single entry point for all clients
- Cross-cutting concerns: authentication, rate limiting, request transformation
- AWS: API Gateway, Kong, Nginx, Istio (service mesh)

### Backend for Frontend (BFF)
```
Web App    → Web BFF     → internal services
Mobile App → Mobile BFF → internal services
```
- Each client type gets its own aggregation layer
- Avoids one-size-fits-all API
- BFF team owned by frontend team

---

## Service Mesh

When you have many microservices, cross-cutting concerns (retries, circuit breaking, mTLS, observability) are repeated in every service.

**Service Mesh** (Istio, Linkerd, AWS App Mesh):
- Sidecar proxy (Envoy) injected alongside each service
- All traffic flows through sidecar
- Centralized control plane manages policy

Provides:
- **mTLS**: Mutual TLS between services automatically
- **Circuit breaking**: At proxy level, no code changes
- **Retries + timeouts**: Configurable per-route
- **Canary deployments**: Traffic splitting by percentage
- **Distributed tracing**: Automatic trace propagation
- **Metrics**: Request rate, error rate, latency per service

---

## Event-Driven Architecture Patterns

### Event Sourcing + CQRS Combined
```
Command (WriteOrder) 
  → Command Handler 
  → Validates 
  → Appends Event (OrderCreated) to Event Store
  → Event Bus
  → Projection updates Read Model

Query
  → Read Model (denormalized, optimized)
```

### Outbox Pattern
Problem: Write to DB and publish event — if publish fails, inconsistency.

```
Transaction:
  INSERT INTO orders(...)
  INSERT INTO outbox(event_type, payload, sent=false)

Separate process (polling or CDC):
  SELECT FROM outbox WHERE sent = false
  Publish to message broker
  UPDATE outbox SET sent = true
```
- Guarantees at-least-once event delivery
- Use with idempotent consumers

### Change Data Capture (CDC)
- Capture DB changes (inserts, updates, deletes) from WAL (Write-Ahead Log)
- Publish as events without changing application code
- Tools: Debezium (Kafka connector), AWS DMS
- Use cases: sync data between databases, populate search index, feed analytics

---

## Caching at the API Layer

### HTTP Caching
```
Cache-Control: max-age=3600          # client caches 1hr
Cache-Control: no-cache              # always validate with server
Cache-Control: s-maxage=600          # CDN caches 10min
ETag: "abc123"                       # content hash for conditional requests
If-None-Match: "abc123"              # client sends back; server returns 304 if unchanged
```

### API Response Caching (Redis)
```typescript
async function getProduct(id: string) {
  const cached = await redis.get(`product:${id}`);
  if (cached) return JSON.parse(cached);
  
  const product = await db.findById(id);
  await redis.setex(`product:${id}`, 3600, JSON.stringify(product));
  return product;
}
```

---

## Security at API Layer

### Authentication vs Authorization
- **AuthN** (Authentication): Who are you? (JWT, OAuth2, API Key)
- **AuthZ** (Authorization): What can you do? (RBAC, ABAC, OPA)

### JWT Best Practices
```
Header.Payload.Signature

- Use RS256 (asymmetric) for public services; HS256 only internal
- Short expiry: 15min access token + refresh token (7 days)
- Never store sensitive data in payload (it's base64, not encrypted)
- Validate: signature, expiry (exp), audience (aud), issuer (iss)
- Refresh token rotation: invalidate old on use
- Store in HttpOnly cookie (not localStorage — XSS safe)
```

### OWASP API Top 10 (Awareness Points for Principal Level)
1. **Broken Object Level Auth** — Validate user owns resource, not just is authenticated
2. **Broken Authentication** — Weak tokens, no rotation, exposed secrets
3. **Excessive Data Exposure** — Return only what's needed
4. **Lack of Rate Limiting** — Add per-user, per-IP limits
5. **Missing Function Level Auth** — Admin endpoints exposed to regular users
6. **Mass Assignment** — Don't bind request body directly to DB model; whitelist fields
7. **Security Misconfiguration** — Exposed debug info, default credentials
8. **Injection** — SQL, NoSQL, command injection; parameterized queries always
9. **Improper Asset Management** — Old API versions left running
10. **Insufficient Logging** — Can't detect breaches without logs

---

## Microservices Operational Concerns

### Health Checks
```
/health/live   — Is the process alive? (liveness probe in K8s)
/health/ready  — Is the service ready to serve traffic? (readiness probe)
/health/startup — Has the service finished initializing?
```

### Graceful Shutdown (Node.js)
```typescript
process.on('SIGTERM', async () => {
  server.close(async () => {      // stop accepting new connections
    await drainQueue();           // finish in-flight work
    await db.disconnect();        // release DB connections
    await redis.quit();
    process.exit(0);
  });
  // Force exit if drain takes too long
  setTimeout(() => process.exit(1), 30000);
});
```

### Configuration Management
- **Never hardcode** config or secrets
- Dev: `.env` files (not committed)
- Production: AWS Secrets Manager / Parameter Store, environment variables
- Feature flags: LaunchDarkly, AWS AppConfig — deploy without releasing
- Config changes should not require redeployment (reload on signal or watch)

### Service Versioning & Contracts
- **Consumer-Driven Contract Testing** (Pact): Consumer defines what it expects; provider verifies
- Non-breaking changes only (add fields, never remove/rename)
- Breaking changes: new API version, run both in parallel, deprecate old with sunset header
