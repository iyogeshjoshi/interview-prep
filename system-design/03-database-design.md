# System Design — Database Design

## Choosing a Database — Decision Framework

```
Ask yourself:
1. What are the access patterns? (key-value, range, full-text, graph?)
2. Read-heavy or write-heavy?
3. What consistency do I need?
4. How large is the dataset? Does it need to scale horizontally?
5. Does schema change frequently?
6. Do I need ACID transactions?
```

---

## SQL vs NoSQL

| | SQL (Relational) | NoSQL |
|---|---|---|
| Schema | Fixed, enforced | Flexible |
| Transactions | Full ACID | Varies (document: limited, Cassandra: none across partitions) |
| Scaling | Vertical + read replicas | Horizontal (built-in sharding) |
| Query power | Rich JOINs, aggregations | Limited (by design) |
| Consistency | Strong (default) | Eventual (default) |
| Best for | Complex relationships, financial | High scale, flexible schema, specific access patterns |

**Use SQL when**: relational data, complex queries, strong consistency required, data integrity critical (financial, ERP)

**Use NoSQL when**: high scale write throughput, known access patterns, flexible/evolving schema, global distribution

---

## SQL Deep Dive

### Indexes

**B-Tree Index** (default)
- Balanced tree; O(log n) lookup, range queries, ORDER BY
- Cost: write overhead (index maintenance), storage

**Hash Index**
- O(1) exact lookups only; no range queries

**Composite Index**
- `CREATE INDEX ON orders(user_id, created_at)`
- **Left-prefix rule**: query must use leading columns
- Good for: `WHERE user_id = X AND created_at > Y`
- Bad for: `WHERE created_at > Y` (without user_id)

**Covering Index**
- Index includes all columns needed by query → no table lookup
- `CREATE INDEX ON orders(user_id) INCLUDE (status, total)`

**Partial Index**
- Index only rows matching condition
- `CREATE INDEX ON orders(user_id) WHERE status = 'pending'`

### Query Optimization
```sql
-- Use EXPLAIN ANALYZE to see query plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- Watch for:
-- Seq Scan → missing index
-- Hash Join vs Nested Loop vs Merge Join
-- Rows estimate vs actual
```

### Normalization vs Denormalization
- **Normalize** (3NF): Eliminate redundancy, update anomalies. Use for write-heavy, consistency-critical.
- **Denormalize**: Embed/duplicate data for read performance. Use for read-heavy, analytics.
- **Practical**: Normalize first, denormalize specific hot paths.

### ACID
- **Atomicity**: All-or-nothing (rollback on failure)
- **Consistency**: DB constraints always satisfied
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data survives crashes (WAL)

### Isolation Levels & Anomalies
| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | ✓ possible | ✓ | ✓ |
| Read Committed | ✗ | ✓ possible | ✓ |
| Repeatable Read | ✗ | ✗ | ✓ possible |
| Serializable | ✗ | ✗ | ✗ |

- **PostgreSQL default**: Read Committed
- **MySQL default**: Repeatable Read
- Higher isolation = more locking = lower throughput

---

## NoSQL Types & Use Cases

### Key-Value Store
- Simple get/set by key; extremely fast
- **Redis**: In-memory; supports strings, hashes, lists, sets, sorted sets, streams
- **DynamoDB**: Managed, serverless, infinite scale; key + optional sort key
- Use for: sessions, caching, leaderboards (sorted sets), rate limiting

### Document Store
- Store JSON documents; flexible schema; query within document
- **MongoDB**: Rich queries, aggregation pipeline, Atlas for managed
- **Firestore**: Real-time sync, offline support
- Use for: product catalogs, user profiles, CMS, anything with varying attributes

### Wide-Column Store
- Table with rows and dynamic columns; optimized for write-heavy
- **Cassandra**: Decentralized, tunable consistency, excellent write throughput
- **HBase**: Hadoop ecosystem, strong consistency
- Use for: time-series data, IoT, activity feeds, audit logs
- **Data modeling**: Design around query patterns, not normalization. Denormalize aggressively.

### Graph Database
- Nodes + edges; optimized for relationship traversal
- **Neo4j**, **Amazon Neptune**
- Use for: social networks, fraud detection, recommendation engines, knowledge graphs
- Bad fit: high write throughput, simple key lookups

### Time-Series Database
- Optimized for time-ordered data; efficient range queries, downsampling, TTL
- **InfluxDB**, **TimescaleDB** (Postgres extension), **Amazon Timestream**
- Use for: metrics, IoT sensor data, financial tick data

---

## Caching Strategy

### Cache-Aside (Lazy Loading)
```
1. Check cache → hit: return data
2. Miss: query DB → store in cache → return data
```
- App controls caching logic
- Stale data possible (use TTL)
- Works well for read-heavy with irregular access patterns

### Write-Through
```
1. Write to cache AND DB simultaneously
```
- Cache always in sync
- Write latency increased
- Cache filled with data that may not be read (cold data waste)

### Write-Behind (Write-Back)
```
1. Write to cache
2. Async: flush to DB
```
- Fast writes
- Risk of data loss if cache fails before flush

### Read-Through
- Cache sits in front of DB; cache populates itself on miss
- App just talks to cache
- Used by: ElastiCache in front of RDS

### Cache Eviction Policies
- **LRU** (Least Recently Used) — most common
- **LFU** (Least Frequently Used) — better for frequency patterns
- **TTL-based** — always set a TTL to prevent stale data

### Cache Invalidation (Hard Problem)
- **TTL expiry**: Simple; accepts some staleness
- **Event-driven invalidation**: Service publishes "data changed" event; cache listener deletes key
- **Write-through**: Invalidate immediately on write
- **Cache stampede / Thundering herd**: Many misses simultaneously → DB overwhelmed
  - Prevention: Mutex lock on cache miss; probabilistic early expiration

---

## Database Patterns for Scale

### Read Replicas
```
Writes → Primary
Reads  → Replicas (1 to many)
```
- Replication lag: replicas may be slightly behind
- Use connection pooling (PgBouncer for PostgreSQL)
- AWS: RDS read replicas, Aurora up to 15 replicas with <10ms lag

### CQRS (Command Query Responsibility Segregation)
- Separate read model from write model
- Write side: normalized, transactional
- Read side: denormalized, optimized for query (materialized views, separate DB)
- Often paired with Event Sourcing

### Event Sourcing
- Store **events** (facts) not current state
- Reconstruct state by replaying events
- Pros: Full audit log, time travel, event replay for new projections
- Cons: Complexity, eventual consistency, schema evolution hard
- Use for: financial ledger, audit trails, collaborative editing

### Materialized Views
- Pre-computed query results stored as a table
- Refresh: on write (synchronous) or periodically (async)
- Use for: expensive aggregations, reporting queries

### Database Connection Pooling
- DB connections are expensive (~5ms to establish)
- Pool: maintain N open connections, reuse them
- **PgBouncer** (PostgreSQL): transaction-level or session-level pooling
- **RDS Proxy** (AWS): managed connection pooling + IAM auth

---

## DynamoDB Specifics (AWS)

### Key Concepts
- **Partition Key** (hash key): determines which partition data lives on
- **Sort Key** (range key): orders items within a partition
- **GSI** (Global Secondary Index): alternate partition + sort key; eventually consistent
- **LSI** (Local Secondary Index): same partition key, different sort key; strongly consistent

### Capacity Modes
- **On-Demand**: Auto-scales; pay per request; good for unpredictable traffic
- **Provisioned**: Specify RCU/WCU; cheaper if predictable; use Auto Scaling

### Access Patterns → Data Model
```
1 RCU = 1 strongly consistent read of ≤4KB (or 2 eventually consistent)
1 WCU = 1 write of ≤1KB

Hot partition: all traffic to one partition key → throughput throttling
Fix: distribute writes across partition keys (add random suffix, use write sharding)
```

### Single-Table Design
- Model multiple entity types in one table
- Use composite keys: `PK = USER#<id>`, `SK = ORDER#<date>#<orderId>`
- Overload GSIs for different access patterns
- Avoids JOINs; efficient for known access patterns; complex to evolve
