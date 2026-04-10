# System Design — Fundamentals

## The Design Framework (use this in every interview)

```
1. Clarify requirements        (5 min)
2. Define scale / constraints  (5 min)
3. High-level design           (10 min)
4. Deep dive components        (20 min)
5. Identify bottlenecks        (10 min)
6. Trade-offs & alternatives   (5 min)
```

Always ask before drawing:
- Who are the users? Expected QPS? Read-heavy or write-heavy?
- Latency requirements? Consistency vs availability needs?
- Global or single-region?

---

## Scalability Concepts

### Vertical vs Horizontal Scaling
| | Vertical (Scale Up) | Horizontal (Scale Out) |
|---|---|---|
| How | Bigger machine | More machines |
| Limit | Hardware ceiling | Virtually unlimited |
| Cost | Expensive, diminishing returns | Commodity hardware |
| Complexity | Simple | Requires load balancing, statelessness |
| Use when | DB primary, quick fix | Stateless services, web tier |

### Stateless vs Stateful Services
- **Stateless**: Any instance can handle any request → easy horizontal scale
- **Stateful**: Session/data tied to instance → sticky sessions or shared state (Redis)
- **Rule**: Push state to a dedicated store (Redis, DB), keep compute stateless

---

## Core Building Blocks

### Load Balancer
- Distributes traffic across servers
- **L4 (Transport)**: Routes by IP/TCP — fast, no content inspection
- **L7 (Application)**: Routes by URL, headers, cookies — enables smart routing
- Algorithms: Round Robin, Least Connections, IP Hash (for sticky sessions), Weighted
- AWS equivalent: ALB (L7), NLB (L4)

### CDN (Content Delivery Network)
- Caches static assets at edge locations close to users
- Reduces latency, offloads origin servers
- Use for: images, JS/CSS bundles, video, API responses (with TTL)
- AWS: CloudFront

### Reverse Proxy
- Single entry point → forwards to internal services
- Provides: SSL termination, compression, caching, rate limiting, auth
- Examples: Nginx, AWS API Gateway

---

## CAP Theorem

A distributed system can guarantee only **2 of 3**:

| Property | Meaning |
|---|---|
| **C**onsistency | Every read gets the most recent write |
| **A**vailability | Every request gets a response (not guaranteed latest data) |
| **P**artition Tolerance | System works despite network failures between nodes |

**Network partitions always happen** → real choice is **C vs A**:

- **CP systems** (prefer consistency): ZooKeeper, HBase, MongoDB (with write concern)
- **AP systems** (prefer availability): Cassandra, DynamoDB, CouchDB, DNS
- **CA** is only possible on single-node (no partition tolerance needed)

### PACELC Extension
Even without partition: choose between **Latency vs Consistency**
- DynamoDB: PA/EL (available + low latency)
- Aurora: PC/EC (consistent)

---

## Consistency Models

From strongest to weakest:

1. **Strong Consistency** — After write, all reads see it immediately. Expensive. (SQL DBs with sync replication)
2. **Linearizability** — Operations appear instantaneous and ordered
3. **Sequential Consistency** — Operations in program order, globally agreed
4. **Causal Consistency** — Causally related ops seen in order; concurrent ops may vary
5. **Eventual Consistency** — Given no new writes, all replicas converge. Fast but stale reads possible. (DynamoDB default, Cassandra)
6. **Read-your-writes** — You always see your own writes (not necessarily others')

---

## Availability & Reliability

### SLA / SLO / SLI
- **SLI** (Indicator): Actual metric — e.g., "request success rate = 99.95%"
- **SLO** (Objective): Internal target — e.g., "99.9% uptime"
- **SLA** (Agreement): Contract with customers — e.g., "99.9% or refund"

### Uptime Nines
| SLA | Downtime/year | Downtime/month |
|---|---|---|
| 99% | 3.65 days | 7.2 hrs |
| 99.9% | 8.76 hrs | 43 min |
| 99.99% | 52 min | 4.3 min |
| 99.999% | 5 min | 26 sec |

### Failure Handling Patterns
- **Retry with exponential backoff + jitter** — avoid thundering herd
- **Circuit Breaker** — stops calling failing service; half-open to probe recovery
- **Bulkhead** — isolate failures (separate thread pools per dependency)
- **Timeout** — always set timeouts; never wait indefinitely
- **Fallback** — serve stale data / degraded response rather than error
- **Idempotency** — safe to retry (use idempotency keys for payments, mutations)

---

## Rate Limiting

### Algorithms
| Algorithm | How | Pros | Cons |
|---|---|---|---|
| Token Bucket | Fill at rate R, consume 1/request | Allows bursts | Slightly complex |
| Leaky Bucket | Fixed output rate, queue excess | Smooths traffic | Drops on queue full |
| Fixed Window | Count in time window | Simple | Edge spike problem |
| Sliding Window Log | Track each request timestamp | Accurate | High memory |
| Sliding Window Counter | Weighted average of windows | Accurate + memory efficient | Approximate |

- **Where to store counters**: Redis (atomic INCR + TTL) — distributed, fast
- **Headers to return**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`

---

## Back-of-Envelope Estimation

Know these numbers:
```
L1 cache:        ~1 ns
L2 cache:        ~4 ns
RAM:             ~100 ns
SSD random read: ~100 µs
Network (same DC): ~0.5 ms
Network (cross-region): ~50-150 ms
HDD seek:        ~10 ms

1 KB   = 1,000 bytes
1 MB   = 10^6 bytes
1 GB   = 10^9 bytes
1 TB   = 10^12 bytes

1M requests/day  ≈  12 req/sec
10M requests/day ≈  115 req/sec
1B requests/day  ≈  11,574 req/sec
```

### Quick QPS Formula
`QPS = Daily Active Users × requests/user/day ÷ 86,400`

### Storage Estimation
`Storage = QPS × request_size × retention_days × replication_factor`

---

## Data Flow Patterns

### Synchronous Request-Response
- Client waits for response
- Simple, but tight coupling; cascading failures
- Use for: user-facing reads, simple CRUD

### Asynchronous (Event-Driven)
- Producer emits event, consumer processes independently
- Decoupled, resilient, but eventual consistency
- Use for: notifications, analytics, long-running tasks

### Push vs Pull
- **Push**: Server sends data when available (WebSocket, SSE, webhooks)
- **Pull**: Client polls server (REST polling, long polling)
- **When push**: Real-time feeds, live collaboration, notifications
- **When pull**: Batch jobs, dashboards with tolerable lag
