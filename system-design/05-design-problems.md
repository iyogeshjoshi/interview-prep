# System Design — Practice Problems

## How to Approach Any Design Problem

```
1. Clarify (5 min)
   - Who uses it? Scale? SLA?
   - Read/write ratio? Latency requirement?
   - Core features for this session (MVP)?

2. Estimate (3 min)
   - QPS, storage, bandwidth
   - Define "large" concretely

3. High-Level Design (10 min)
   - Clients → LB → API tier → DB/Cache
   - Name the key services

4. Deep Dive (20 min)
   - Most interesting / critical component
   - Data model, algorithms, protocols

5. Scale & Edge Cases (10 min)
   - Bottlenecks, hotspots
   - What fails first? How do you handle it?

6. Trade-offs (5 min)
   - What did you give up? Why was it the right call here?
```

---

## Problem 1: Design a URL Shortener (e.g., bit.ly)

### Requirements
- Shorten long URL → 6-8 char short code
- Redirect short → long URL
- Analytics (clicks, referrers)
- Scale: 100M URLs, 10B redirects/day

### Estimation
```
Write QPS: 100M URLs / (365 * 86400) ≈ 3 writes/sec (easy)
Read QPS:  10B / 86400 ≈ 115,000 reads/sec  ← this is the challenge
Storage:   100M * (7B code + 2KB URL + metadata) ≈ 200GB
```

### Core Design
```
Client → CDN → API Gateway → Redirect Service → Cache (Redis) → DB

Short code generation:
  Option 1: MD5(longURL) → take first 7 chars (collision possible)
  Option 2: Base62 of auto-increment ID (predictable, sequential)
  Option 3: Random 7-char Base62 → check for collision (preferred)
  Option 4: Pre-generate codes in batch; assign from pool (best for scale)

  Base62 chars: a-z A-Z 0-9 = 62 chars
  62^7 = 3.5 trillion possible codes → plenty
```

### Data Model
```sql
CREATE TABLE urls (
  short_code  VARCHAR(8) PRIMARY KEY,
  long_url    TEXT NOT NULL,
  user_id     BIGINT,
  created_at  TIMESTAMP,
  expires_at  TIMESTAMP,
  click_count BIGINT DEFAULT 0
);
```

### Redirect Flow
```
1. Check Redis: GET short_code → 301/302 redirect
2. Cache miss: query DB → cache with TTL → redirect
3. 301 (Permanent): browser caches, no server hit → less analytics data
4. 302 (Temporary): every redirect hits server → more accurate analytics
```

### Analytics (Write-heavy stream)
```
Click event → Kafka → Analytics Consumer → ClickHouse/Cassandra
                                         ← aggregated dashboard
```
Don't synchronously increment `click_count` per redirect — too slow.

### Scaling the Read Path
- Redis cluster for short_code → long_url (fits in memory: 100M * ~200B ≈ 20GB)
- CDN caches 301 redirects for popular URLs
- DB read replicas for cold lookups

---

## Problem 2: Design a Chat System (e.g., WhatsApp)

### Requirements
- 1:1 and group chat (max 500 members)
- Delivery receipts (sent, delivered, read)
- Online presence
- Message history
- Scale: 50M DAU, 40M messages/day

### Estimation
```
Message QPS: 40M / 86400 ≈ 460/sec (peak ~2x = 920/sec)
Storage: 40M * 100 bytes avg = 4GB/day = 1.4TB/year
```

### Architecture
```
Client ←→ WebSocket Gateway ←→ Message Service → Cassandra
                              ↓
                         Presence Service → Redis
                              ↓
                        Notification Service → APNs/FCM
```

### Real-time Messaging (WebSocket)
- Persistent WebSocket connection per client to gateway
- Gateway is stateful (knows which connections are online)
- Use consistent hashing to route user → gateway server
- Scale: if user A and B on different gateway servers → route via message broker

### Message Flow
```
User A sends message:
1. WebSocket → Gateway A → Message Service
2. Message Service: save to Cassandra (async), publish to Kafka
3. Kafka Consumer: lookup recipient's gateway → push via WebSocket (if online)
4. If offline: store as pending → push notification
5. On delivery: update delivery receipt → push to sender
```

### Data Model (Cassandra)
```
-- Messages table
messages:
  PK: channel_id (chat room or user pair ID)
  SK: message_id (time-based UUID, TimeUUID → ordering for free)
  sender_id, content, type, created_at, status

-- One partition per conversation → efficient range queries
-- Tombstones: deletes in Cassandra create tombstones; use TTL for cleanup
```

### Presence Service
```
Online: Redis hash  user_id → {status, last_seen, gateway_id}
- Heartbeat every 30s from client (WebSocket ping)
- Expire key after 60s of no heartbeat
- Pub/Sub: when user goes online/offline → notify subscribed friends
```

### Group Chat Fanout
```
User sends to group of 500:
Option A: Fanout on write (push to each member's inbox)
  - Pros: fast read
  - Cons: 500 writes per message; expensive for large groups

Option B: Fanout on read (store once, each member fetches)
  - Pros: single write
  - Cons: read amplification

Hybrid: Small groups (<100) → fanout on write. Large groups → fanout on read.
```

---

## Problem 3: Design a News Feed (e.g., Twitter/Instagram)

### Requirements
- Follow/unfollow users
- Post updates (tweets/posts)
- See personalized feed (posts from followed users)
- Scale: 300M users, 20% DAU, 500M posts/day

### Fan-out Strategies

**Push (Fan-out on Write)**
```
User posts → write to all followers' feed caches
Reads are fast (pre-computed feed in cache)
Cons: celebrity with 10M followers = 10M writes per post (write amplification)
```

**Pull (Fan-out on Read)**
```
User reads feed → query all N followed users' posts → merge, sort, paginate
Cons: Slow reads; N queries per page load
```

**Hybrid (Twitter's actual approach)**
```
Regular users:  Push to followers' feed cache (Redis sorted set by timestamp)
Celebrities (>1M followers): Skip push; pull at read time and merge
Feed generation:
  1. Load cached feed for user (pre-built from regular follows)
  2. Pull recent posts from celebrity accounts user follows
  3. Merge + deduplicate + rank
```

### Feed Ranking
- Reverse chronological: Simple, predictable, users understand
- Ranked/ML: Engagement score, recency, relationship strength, content type
- At Principal level: mention ML pipeline exists; don't design it unless asked

### Data Models
```
-- Post store: Cassandra (high write throughput)
posts: post_id (PK), user_id, content, media_urls, created_at, like_count

-- Social graph: Either DynamoDB or graph DB
follows: follower_id (PK), followee_id (SK)

-- Feed cache: Redis Sorted Set
ZADD feed:{user_id} <timestamp_score> <post_id>
ZREVRANGE feed:{user_id} 0 49  -- get latest 50 posts
```

---

## Problem 4: Design a Distributed Rate Limiter

### Requirements
- Rate limit API calls per user/IP/API key
- Multiple services need rate limiting
- Distributed (multiple API server instances)
- High performance (<5ms added latency)

### Algorithm Choice: Sliding Window Counter (best balance)

```
Current rate = (prev_window_count * overlap_ratio) + current_window_count

Example: 60 req/min limit
  prev window (11:00): 40 requests
  current window (11:01): 15 requests, now at 11:01:30 (50% into window)
  
  Estimated count = 40 * 0.5 + 15 = 35 → allow (< 60)
```

### Redis Implementation
```typescript
async function isAllowed(userId: string, limit: number, windowSec: number): Promise<boolean> {
  const now = Date.now();
  const windowMs = windowSec * 1000;
  const key = `rate:${userId}`;
  
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, now - windowMs);  // remove old entries
  pipeline.zadd(key, now, `${now}-${Math.random()}`);  // add current request
  pipeline.zcard(key);                                  // count requests in window
  pipeline.expire(key, windowSec);
  
  const results = await pipeline.exec();
  const count = results[2][1] as number;
  return count <= limit;
}
```

### Distributed Coordination
- Redis as shared counter store → all API server instances share same view
- Redis Cluster for HA
- Lua scripts for atomicity (instead of pipeline for exact correctness)
- **Token Bucket in Redis**: Store last_refill_time + tokens; update atomically with Lua

---

## Problem 5: Design an E-Commerce Order System

### Requirements
- Browse products, add to cart, checkout
- Payment, inventory, order management
- Scale: 1M orders/day, flash sales (spike 100x)

### Services
```
Product Service    → PostgreSQL (product data) + Elasticsearch (search)
Inventory Service  → DynamoDB (atomic decrements) + Redis (reservation)
Cart Service       → Redis (ephemeral, session-based)
Order Service      → PostgreSQL (order records) + Kafka (events)
Payment Service    → PostgreSQL (transactions) + Stripe API
Notification Svc   → Kafka consumer → Email/SMS/Push
```

### Flash Sale: Inventory Oversell Problem
```
Race condition: 100 users click "Buy" at same time, only 1 item left
  
Option 1: DB-level optimistic locking
  UPDATE inventory SET quantity = quantity - 1, version = version + 1
  WHERE product_id = X AND quantity > 0 AND version = current_version
  → retry on failure; doesn't scale under extreme contention

Option 2: Redis pre-check + async DB update
  1. DECR inventory:{product_id} in Redis (atomic)
  2. If result >= 0: allowed (optimistic reservation)
  3. Create order, enqueue payment
  4. On payment success: commit DB decrement
  5. On payment failure: INCR inventory (release reservation)
  
Option 3: Queue-based serialization
  All purchase requests → queue (Kafka partition per product)
  Single consumer per product → serial processing, no race
  Works for extreme flash sales
```

### Order State Machine
```
PENDING → PAYMENT_RESERVED → INVENTORY_CONFIRMED → CONFIRMED
                                                  ↓
                                              SHIPPED → DELIVERED
PENDING → PAYMENT_FAILED → CANCELLED
CONFIRMED → CANCELLATION_REQUESTED → CANCELLED (with compensations)
```
Use Saga orchestration for cross-service order lifecycle.

---

## Common Interview Discussion Points for Principal Level

### "How would you handle 10x traffic tomorrow?"
```
1. Identify bottleneck first (profiling, metrics)
2. Horizontal scale stateless services (auto-scaling groups)
3. Cache aggressively (CDN, Redis)
4. DB read replicas + connection pooling
5. Async expensive operations (queue them)
6. Rate limit + shed load gracefully
7. Long-term: shard DB, partition data
```

### "What would you do differently if you redesigned this?"
- Good answer shows: you learned from past mistakes, you think in trade-offs, not just tech
- Example: "I'd use event sourcing for the order service to give us a full audit trail — we later had a compliance requirement that cost 3 months of retroactive work"

### "How do you ensure data consistency across services?"
- "It depends on the business requirement. For financial operations, I use Saga with compensating transactions and idempotency keys. For analytics data, eventual consistency is fine — I use event streaming and accept a few seconds of lag. The key is being explicit about consistency guarantees per domain."
