# System Design — Distributed Systems

## Replication

### Why Replicate?
- **High availability**: Failover if primary goes down
- **Read scalability**: Distribute read load across replicas
- **Geographic distribution**: Serve users from nearest replica

### Replication Strategies

**Single-Leader (Master-Slave)**
- All writes go to leader; replicated to followers
- Followers serve reads
- Failover: promote follower to leader
- **Replication lag**: async replication means stale reads possible
- Used by: PostgreSQL, MySQL, MongoDB replica sets

**Multi-Leader**
- Multiple nodes accept writes; sync between leaders
- Good for multi-datacenter (each DC has a leader)
- **Conflict resolution needed**: last-write-wins, custom merge, CRDT
- Used by: CouchDB, multi-region DynamoDB

**Leaderless (Dynamo-style)**
- Any node can accept writes; replication to N nodes
- Quorum reads/writes: W + R > N for strong consistency
- Example: W=2, R=2, N=3 → any 2 nodes agree = consistent
- Used by: Cassandra, DynamoDB, Riak

### Quorum Formula
```
N = total replicas
W = write quorum (nodes that must confirm write)
R = read quorum (nodes that must confirm read)

W + R > N → strong consistency
W = 1, R = 1 → fastest, but stale reads
W = N → slowest write, fastest consistent read
```

---

## Partitioning (Sharding)

### Why Shard?
- Data too large for single node
- Write throughput exceeds single node capacity

### Partitioning Strategies

**Range Partitioning**
- Partition by sorted key range (e.g., user IDs 1–1M on shard 1)
- Pros: Range queries efficient
- Cons: Hot spots if data is skewed (e.g., all writes to today's date shard)

**Hash Partitioning**
- `shard = hash(key) % N`
- Pros: Uniform distribution
- Cons: Range queries require scatter-gather; resharding is expensive

**Consistent Hashing**
- Virtual ring of 2^32 positions; nodes placed on ring
- Key maps to nearest node clockwise
- Adding/removing node only affects adjacent partition
- Used by: Amazon DynamoDB, Apache Cassandra

**Directory-Based Sharding**
- Lookup table maps key → shard
- Flexible but lookup service is single point of failure

### Secondary Indexes with Sharding
- **Local index**: Each shard indexes its own data → scatter-gather for queries
- **Global index**: Separate distributed index → expensive to keep in sync

### Resharding
- Consistent hashing minimizes data movement
- Alternative: start with more shards than nodes (virtual shards), move virtual shards between nodes

---

## Consensus Algorithms

### Why Consensus?
- Needed for: leader election, distributed locks, atomic broadcast, configuration management
- Problem: Nodes can fail; messages can be delayed; how do all agree?

### Paxos
- Theoretical foundation; complex to implement correctly
- Two phases: Prepare (leader proposes) → Accept (nodes commit)

### Raft
- Designed for understandability; equivalent to Paxos
- **Leader election**: Nodes timeout → candidate requests votes → majority wins
- **Log replication**: Leader appends, replicates to followers, commits when majority ACK
- Used by: etcd (Kubernetes), CockroachDB, TiKV

### Zab (ZooKeeper Atomic Broadcast)
- Similar to Raft; used by ZooKeeper
- ZooKeeper used for: distributed locks, service discovery, config management

---

## Distributed Transactions

### Two-Phase Commit (2PC)
```
Phase 1 — Prepare:
  Coordinator → "Can you commit?" → all participants
  Participants → "Yes/No"

Phase 2 — Commit/Abort:
  If all YES → Coordinator → "Commit"
  If any NO  → Coordinator → "Abort"
```
- **Problem**: Coordinator failure after Phase 1 leaves participants blocked (blocking protocol)
- Used by: distributed SQL databases

### Saga Pattern (Preferred for Microservices)
- Long-running transaction = sequence of local transactions
- Each local tx publishes event; triggers next step
- **Compensating transactions** undo completed steps on failure

```
Order Service: CreateOrder → emit OrderCreated
Payment Service: ReservePayment → emit PaymentReserved
Inventory Service: ReserveStock → emit StockReserved
Shipping Service: CreateShipment

If stock fails:
  Shipping: (not started)
  Inventory: ReleaseStock (compensate)
  Payment: CancelReservation (compensate)
  Order: MarkOrderFailed (compensate)
```

**Choreography**: Services react to events (decoupled, complex to trace)
**Orchestration**: Central saga orchestrator coordinates (easier to reason, single point of knowledge)

---

## Message Queues & Event Streaming

### Message Queue vs Event Stream
| | Message Queue (SQS, RabbitMQ) | Event Stream (Kafka, Kinesis) |
|---|---|---|
| Consumption | Each message consumed once | Multiple consumers, replay |
| Retention | Deleted after consumption | Retained for days/weeks |
| Ordering | Per-queue (FIFO optional) | Per-partition |
| Use case | Task distribution, work queues | Event sourcing, audit log, analytics |

### Kafka Architecture
```
Topic → split into Partitions
Each Partition → replicated across Brokers
Producer → writes to partition (by key hash or round-robin)
Consumer Group → each partition assigned to one consumer
  (scale consumers = scale partitions)
```

- **Offset**: Position of consumer in a partition (committed to __consumer_offsets)
- **At-least-once delivery**: Commit offset after processing (reprocess on failure)
- **Exactly-once**: Transactional producers + idempotent consumers
- **Log compaction**: Keep latest value per key (good for state tables)

### Backpressure
- Consumer slower than producer → buffer fills → producer must slow down
- Strategies: bounded queues, consumer scaling, shed load, sampling

---

## Service Discovery

### Client-Side Discovery
- Client queries registry (Eureka, Consul) → gets instance list → load-balances itself
- More flexible, more client logic

### Server-Side Discovery
- Client → load balancer → queries registry → forwards to instance
- Client is dumb; discovery logic centralized (AWS ALB + ECS, Kubernetes)

### Health Checks
- Registry pings services; removes failed instances
- Types: HTTP `/health`, TCP connect, custom script

---

## Idempotency & Exactly-Once Semantics

### Idempotent Operations
- Safe to execute multiple times; same result
- GET, PUT, DELETE are idempotent; POST is not by default

### Idempotency Keys
```
POST /payments
Idempotency-Key: <uuid>

Server stores: key → response
If key seen again: return stored response (don't process again)
TTL on keys: 24–48 hours
```

### Deduplication Patterns
- **Unique constraint in DB**: INSERT with unique idempotency_key → catch duplicate key error
- **Redis SET NX**: Atomic set-if-not-exists with TTL
- **Kafka**: Producer idempotency + transactional API

---

## Observability in Distributed Systems

### Three Pillars
- **Metrics**: Aggregated numbers over time (Prometheus/CloudWatch) — QPS, error rate, latency p50/p95/p99
- **Logs**: Structured event records (CloudWatch Logs, ELK stack) — include trace/request IDs
- **Traces**: End-to-end request path across services (AWS X-Ray, Jaeger, OpenTelemetry)

### Key Metrics (USE + RED)
**USE** (for resources):
- **U**tilization — % time busy
- **S**aturation — queue depth
- **E**rrors — error rate

**RED** (for services):
- **R**ate — requests per second
- **E**rrors — error rate
- **D**uration — latency distribution

### Correlation IDs
- Generate at API gateway entry; pass via headers (`X-Request-ID`, `X-Trace-ID`)
- Every log line includes the correlation ID
- Critical for tracing requests across microservices
