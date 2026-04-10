# System Design — Real-World Case Studies

These are actual architectural decisions and trade-offs made by engineering teams at scale. Use these in interviews to ground abstract design in real precedent — "Netflix solved this by..." signals Principal-level awareness.

---

## Case Study 1: Netflix — Video Streaming at Scale

**Scale**: 250M+ subscribers, 15% of global internet traffic, 1B+ hours watched per week.

### The Core Architecture

```
User request → Netflix CDN (Open Connect) → Video file chunks (MPEG-DASH / HLS)
                                             Adaptive bitrate: switches quality based on bandwidth

Content pipeline:
  Studio master file (4K RAW)
    → Encoding farm (thousands of EC2 instances)
    → 1,200+ versions per title (codec × resolution × device × language)
    → Stored in Open Connect CDN appliances (Netflix's own CDN hardware in ISP racks)
```

### Key Decision: Build Their Own CDN (Open Connect)
```
Problem (2011): Akamai and Level 3 costs were enormous at Netflix's scale.
              Single CDN = single point of failure; also limited cache hit rate.

Decision: Deploy custom CDN hardware inside ISP networks globally.
  - ISPs get free hardware in exchange for hosting
  - Reduces ISP bandwidth costs (traffic stays local)
  - Netflix controls cache placement and pre-positioning

Pre-positioning: Each night, Netflix proactively pushes popular content to
  edge nodes based on predicted viewing patterns before users request it.
  Result: Most popular content is served entirely from edge with zero origin hits.

Trade-off accepted:
  + Massive cost reduction (~90% CDN cost savings estimated)
  + Sub-100ms start times globally
  - Significant upfront infrastructure investment
  - Ongoing operational complexity (proprietary hardware fleet)
```

### The Chaos Engineering Decision
```
Problem: Netflix ran on AWS. AWS had multiple outages (2011 East Coast failure
  killed Netflix for 3 days). Their monolith had hidden dependencies everywhere.

Decision: Chaos Monkey, then the Simian Army
  - Chaos Monkey: randomly kills production EC2 instances during business hours
  - Latency Monkey: introduces artificial latency in service calls
  - Chaos Gorilla: takes down an entire AWS Availability Zone
  - Janitor Monkey: cleans up unused AWS resources

Philosophy: "If something hurts, do it more often until it stops hurting."
  Services MUST be designed to handle failure because they WILL be killed.

Result: Netflix survived the 2012 AWS us-east-1 outage while competitors went down.
  Their preparation made them resilient to conditions they couldn't prevent.

Interview point: "We don't test our disaster recovery — we run it constantly."
```

### Microservices Migration (2008–2016)
```
The monolith problem:
  Single deployment of everything → one bad push = full outage
  Database: single Oracle cluster → scaling ceiling hit
  Teams blocked on each other to ship

Migration approach (Strangler Fig):
  Phase 1: Extract non-critical services first (recommendations, search)
  Phase 2: Move to DynamoDB / Cassandra per service (polyglot persistence)
  Phase 3: Decompose the core (streaming, billing, member management)

What they learned:
  - 700+ microservices at peak → distributed systems complexity is real
  - Started consolidating back: "micro" services sometimes too micro
  - Each service needs: circuit breakers, timeouts, fallbacks — always
  - "Hystrix" (now open-source) became their circuit breaker standard

Lesson for interview: Microservices solve organizational/deployment problems,
  not technical ones. The organizational benefit must justify the distributed
  systems complexity you're taking on.
```

---

## Case Study 2: Uber — Real-Time Dispatch at Scale

**Scale**: 5M+ trips per day, 3M+ drivers, location updates every 4 seconds per driver.

### The Geospatial Matching Problem
```
Core challenge: Given a rider at location X, find the nearest available driver.

Naive: SELECT * FROM drivers WHERE status='available' ORDER BY distance(lat,lng) LIMIT 1
  → Full table scan on 3M rows, every second, per city → impossible

Approach 1: Geohash
  Divide Earth into hierarchical grid cells (geohash)
  Each cell has a 12-char string prefix: "9q8yy" = ~5km cell in SF
  Driver at location → compute geohash → store driver in cell's bucket
  Matching: lookup driver's cell + adjacent 8 cells → small candidate set

  geohash prefix lengths:
    1 char  = ~5,000km × 5,000km
    5 chars = ~5km × 5km
    7 chars = ~150m × 150m  ← Uber uses ~7-8 chars for driver matching

Approach 2: H3 (Hexagonal Hierarchy) — Uber's current approach
  Uber open-sourced H3: hexagonal grid system
  Hexagons tile without gaps; better distance properties than squares
  Each hexagon has a unique 64-bit integer ID
  Resolution 9 hex ≈ 0.1km² — ~walking distance

Storage:
  Redis: GEOADD drivers:available <lng> <lat> <driver_id>
         GEORADIUS drivers:available <lng> <lat> 2km → nearby drivers
  (Redis GEORADIUS uses geohash internally; O(N+log M) where N=results, M=members)
```

### DISCO: Dispatch System Architecture
```
Problem: How do you match millions of riders to millions of drivers in <1 second?

Components:
  Supply Service:   tracks driver location + state (available/busy/offline)
  Demand Service:   tracks rider requests
  DISCO (Dispatch): the matching engine

Matching flow:
  1. Rider requests → Demand Service → publishes to dispatch queue
  2. DISCO queries Supply Service → nearby available drivers (Redis GEORADIUS)
  3. DISCO runs matching algorithm → selects best driver
  4. Offer sent to driver (WebSocket push)
  5. Driver accepts → trip created → both notified

City-level sharding:
  Each city is a shard → independent dispatch cluster
  Drivers cannot cross city shard boundaries mid-trip
  Inter-city: separate handling (airport runs)

Why not global matching?
  Latency: matching within a city takes <100ms; global would be ~10x slower
  Isolation: NYC outage doesn't affect London
  Scale: each city cluster sized independently for its demand
```

### The "Trip" State Machine
```
SEARCHING → DRIVER_FOUND → DRIVER_ACCEPTED → DRIVER_ARRIVING
→ TRIP_STARTED → TRIP_ENDED → PAYMENT_PROCESSING → COMPLETED

Each state transition:
  - Written to PostgreSQL (source of truth, per-city sharded)
  - Published to Kafka → downstream consumers (notifications, analytics, surge pricing)

Idempotency everywhere:
  Driver accepts → network timeout → driver retries accept
  Server must handle duplicate accept without creating two trips
  Key: idempotency_key = (rider_request_id + driver_id + timestamp_bucket)
```

### Surge Pricing Architecture
```
Problem: Price needs to update every ~30s based on supply/demand ratio in a geohash cell.

Pipeline:
  Driver location updates → Kafka → Stream processor (Flink)
  Rider requests          → Kafka → Stream processor (Flink)
    → Compute supply/demand ratio per H3 cell (5-min rolling window)
    → Publish surge multipliers → Redis (read by pricing service)
    → Rider app reads surge map → display

Why not SQL for surge?
  SQL aggregations on 5M rows every 30s per city × 100 cities = not feasible
  Flink maintains in-memory state per cell; event-driven updates
  Redis holds the materialized view: cell_id → surge_multiplier (read by API, <1ms)
```

---

## Case Study 3: WhatsApp — 1B Users, 50 Engineers

**Scale**: 1B+ users, 100B+ messages/day, 50 engineers (as of 2014 acquisition).

### The Architecture Philosophy
```
WhatsApp's guiding principle: Do one thing, do it very well. No games, no ads, no bloat.

Tech choices that enabled 50 engineers to serve 1B users:
  - Erlang + FreeBSD: Erlang VM (BEAM) handles millions of concurrent lightweight processes
    Erlang was built for telecom: fault-tolerant, concurrent by design
  - Each connection = one Erlang process (~2KB overhead)
    vs Node.js (2MB per connection at that time)
  - FreeBSD: better network stack than Linux for their use case (more tunable)

Server count (2014): ~900 servers for 1B users
  Each server: ~200K active connections
  Erlang scheduler: 1 OS thread per CPU core, millions of green threads (processes)
```

### Message Delivery Architecture
```
Delivery guarantees: at-least-once with deduplication

Message flow:
  Sender → WhatsApp server → ACK to sender (message received by server)
  Server → attempt delivery to recipient
    Recipient online: push via persistent WebSocket-equivalent connection
    Recipient offline: store in queue (Mnesia/custom storage)
      On recipient connect: deliver queued messages + send delivery receipt
      On recipient read: send read receipt to sender

Message IDs:
  Client-generated: timestamp + device_id + counter → globally unique
  Why client-side: avoids server round-trip for ID generation; works offline

Offline message storage:
  Store up to 30 days (now more)
  If recipient never connects: messages expire
  WhatsApp does NOT store messages after delivery (end-to-end encrypted; server can't read them)
  This is both a privacy feature AND a storage optimization
```

### End-to-End Encryption (Signal Protocol)
```
Architecture constraint that shapes everything:
  Messages are encrypted on sender's device, decrypted on recipient's device.
  WhatsApp servers see: ciphertext only, metadata (who → who, when, size), not content.

Key exchange: Double Ratchet Algorithm (Signal Protocol)
  Each message uses a new encryption key (forward secrecy)
  If one key is compromised, past messages are safe

Group messages:
  Sender encrypts once per group member (n messages for n members)
  Server delivers n separate encrypted copies
  At 1000-member groups: 1000 encrypt operations on sender's device → noticeable delay
  WhatsApp Sender Keys optimization: one encryption, members have the sender's chain key

Multi-device (2021):
  Problem: E2E encryption assumes one device per account
  Solution: Linked devices maintain their own key pairs
    Primary device re-encrypts messages for each linked device
    Phone-offline mode: companion devices get independent key material
```

### Scale Without a Database (Almost)
```
Mnesia: Erlang's built-in distributed database (not traditional SQL)
  In-memory with disk persistence
  Used for: session state, pending message queue, user metadata

What they don't store:
  Message content (delivered + deleted immediately on server)
  Media (media server separate, content-addressed storage, also deleted after delivery)

What this means at scale:
  No massive message history database to query
  Storage per user ≈ metadata only (contacts, settings, pending messages)
  Storage scales with pending undelivered messages, not message history

Lesson: The best data store for a problem is sometimes "don't store it."
```

---

## Case Study 4: Discord — Real-Time Messaging + Data at Scale

**Scale**: 19M+ active servers, 4B+ messages/day, storing messages permanently (vs WhatsApp).

### The Message Storage Problem
```
Discord stores ALL messages permanently (unlike WhatsApp).
4B messages/day × 365 = 1.46T messages/year accumulated.

Evolution:
  2015: MongoDB (single node) → hit WIRED TIGER engine limits at ~100M messages
  2016: Cassandra → chosen for: write-heavy, linear horizontal scale, no SPOF

Cassandra data model for Discord channels:
  Partition key: (channel_id, bucket)  ← bucket = month/year (time-based partitioning)
  Clustering key: message_id (Snowflake ID, time-sorted)
  
  Why bucket the partition?
    Without bucket: one channel's entire history = one partition → hot partition
    With bucket: channel history split across time-based partitions → even load
    Tradeoff: pagination across bucket boundaries requires multiple queries

  Message IDs: Snowflake (Twitter-style)
    64-bit: timestamp_ms (42 bits) + worker_id (10 bits) + sequence (12 bits)
    Globally unique, time-sortable, no coordination needed
```

### The 2023 Migration: Cassandra → ScyllaDB
```
Problem at scale (2023): 177 nodes of Cassandra, p99 latency spikes, GC pauses
  Java GC pauses of 10+ seconds during peak load
  Compaction storms: background compaction competing with read/write traffic

Decision: Migrate to ScyllaDB (Cassandra-compatible, written in C++)
  - C++ = no GC, predictable latency
  - Same CQL API: migration without app code changes
  - 177 Cassandra nodes → 72 ScyllaDB nodes (2.5x more efficient)
  - p99 latency: 40ms → 15ms

Migration strategy:
  1. Run ScyllaDB alongside Cassandra (dual-write)
  2. Backfill historical data from Cassandra → ScyllaDB
  3. Gradually shift reads to ScyllaDB (with consistency verification)
  4. Cut over writes, decommission Cassandra

Lesson: Technology choice has a lifespan. The right choice at 100M messages
  is not necessarily right at 1T messages. Plan for migration, don't over-optimize early.
```

### Voice Architecture (Real-Time Audio/Video)
```
Discord's voice is separate from text messaging:

WebRTC for client media (peer-to-peer where possible, server relay otherwise)
Selective Forwarding Unit (SFU) architecture:
  Each participant sends ONE video/audio stream to server
  Server selectively forwards to each receiver (doesn't mix/decode)
  Scales better than MCU (Multipoint Control Unit) which decodes+mixes everything

Voice regions:
  ~14 voice regions globally; users connected to nearest voice server
  Why not AWS global? Voice latency requires physical proximity
  Discord co-located dedicated servers in key regions (not cloud VMs) for lower latency

Elixir for voice server (2022 migration from Go):
  Go goroutines handled concurrency but GC caused audio glitches
  Elixir on BEAM: same inspiration as WhatsApp, lightweight processes, no GC pauses
  Discord open-sourced their reasoning: "Why Discord is switching from Go to Rust" (for a different service)
```

### Read States (Who Read What)
```
Problem: Discord shows unread message counts per channel per user.
  19M servers × users per server × channels per server = enormous write amplification

Naive: UPDATE read_state SET last_read_id = X WHERE user_id = Y AND channel_id = Z
  Every message read = one DB write per user watching = impossible at scale

Solution: Eventual consistency + user-partitioned storage
  Read states stored in Cassandra, partitioned by user_id
  Last-read message ID stored per (user_id, channel_id)
  Unread count = count messages with id > last_read_id (computed client-side from cached data)
  Acknowledgment: fire-and-forget write; slight inconsistency across devices is acceptable

Trade-off: Occasionally a channel shows "1 unread" when there isn't one.
  Discord accepted this. The alternative (strong consistency) would cost orders of magnitude more.
  Business decision: perfect consistency isn't worth the infrastructure cost here.
```

---

## Case Study 5: Stripe — Payments Infrastructure

**Scale**: $1T+ in total payment volume, 100s of millions of API calls/day.

### Idempotency Keys — The Core Design
```
Problem: Payments are money. A retry must not double-charge.

Stripe's solution: Idempotency keys (client-supplied, UUID)
  POST /charges
  Idempotency-Key: a8098c1a-f86e-11da-bd1a-00112444be1e
  { amount: 2000, currency: "usd", source: "tok_xxx" }

Server behavior:
  1. Check if idempotency_key seen in last 24 hours
  2. If yes: return stored response (don't process again)
  3. If no: process charge → store (key, response) → return response

Storage: idempotency_key → { status, response, created_at } in PostgreSQL
  Unique constraint on key → concurrent requests for same key → only one executes
  Race condition: two simultaneous requests with same key
    → DB unique constraint → one gets 409 Conflict → client uses the 200 response

TTL: 24 hours. Keys expire after that.

Why 24 hours?
  Retry windows for network failures are typically minutes
  24h covers any reasonable retry storm
  Beyond 24h, user likely meant a new charge (different idempotency key)
```

### Distributed Transactions Across Services
```
Stripe's internal services: Payment Service, Card Network Service, Fraud Service,
  Ledger Service, Notification Service...

Challenge: Charging a card requires:
  1. Fraud check (internal)
  2. Authorization with card network (external: Visa/Mastercard API)
  3. Capture (another external call)
  4. Ledger entry (internal)
  5. Receipt email (internal)

If step 4 fails after step 3: customer charged but no ledger entry → accounting disaster

Stripe's approach: Optimistic locking + compensating transactions + event sourcing
  Every charge attempt creates a FinancialTransaction record
  State machine: INITIATED → AUTHORIZED → CAPTURED → SETTLED
  Each transition: atomic DB update + publish event
  Compensation: if capture succeeds but ledger fails → retry ledger (idempotent write)
                if auth succeeds but fraud blocks capture → void authorization (compensate)

Ledger design:
  Immutable append-only ledger entries (never UPDATE, only INSERT)
  Balance = sum of all ledger entries for account
  This is event sourcing: current state derived from event history
  Full audit trail for every dollar, always
```

### The API Versioning Strategy
```
Problem: Stripe has thousands of integration partners. Any breaking API change
  = thousands of broken integrations.

Stripe's solution: Date-based versioning + perpetual backward compatibility
  API version: "2024-06-20" (date of last change you opted into)
  Each account is pinned to the version at time of API key creation
  Stripe maintains compatibility for all previous versions — forever

How it works internally:
  API Gateway reads Stripe-Version header (or account's pinned version)
  Request/response transformation layer converts between versions
  Every breaking change: write a version transform (old format ↔ new format)
  Deploy new behavior; old behavior preserved via transform

Cost:
  Stripe maintains version transform code going back to 2011
  Each new feature must consider how it appears across all versions
  Engineers say this is their biggest ongoing maintenance burden

Why they do it anyway:
  "Upgrade your Stripe integration" is a cost on every Stripe customer
  Eliminating that cost = better developer experience = competitive advantage
  Their moat is partially built on "Stripe never breaks you"

Alternative (what most companies do):
  v1/v2/v3 URL versioning → deprecated old versions → force migration
  Stripe explicitly rejected this model
```

### Handling Partial Failures in Card Network Calls
```
External card networks (Visa, Mastercard) are:
  - Not idempotent natively
  - Occasionally unreliable (timeouts, 5xx)
  - Slow (50-200ms typical, occasionally seconds)

Stripe's timeout strategy:
  Connect timeout: 3s (fail fast if can't connect)
  Read timeout: 15s (card networks can be slow)
  Total timeout: 20s before Stripe returns timeout error to merchant

Unknown outcomes (timeout after authorization request sent):
  Stripe doesn't know if the network authorized the charge
  They attempt a void/reversal to cancel any pending authorization
  If void fails: mark charge as "in unknown state", flag for manual review
  Merchants see: "charge failed" (conservative — better than double-charge)

Retry policy (for network errors only — not declines):
  Exponential backoff: 1s → 2s → 4s
  Max 3 retries
  Each retry is a NEW authorization request (card networks don't support retry on same request)
  Each request includes Stripe's internal idempotency → only one succeeds in their ledger

Why not more retries?
  Each retry = potential duplicate authorization hold on cardholder's card
  Too many retries = customer sees multiple "pending" charges → confusion + support tickets
```

---

## Case Study 6: Amazon DynamoDB — Design Principles from the Dynamo Paper

**The 2007 Dynamo Paper** is one of the most influential distributed systems papers. DynamoDB is the production evolution of those ideas.

### The Core Design Constraints
```
Amazon's requirements (2007):
  - Always writeable: shopping cart must accept writes even during network partitions
  - No single point of failure: availability > consistency for this use case
  - Horizontally scalable: add nodes without downtime
  - Predictable latency: p99.9 < 100ms (not just average)

Trade-off explicitly made:
  Chose A over C in CAP theorem
  Chose eventual consistency as the default
  "Sacrificing consistency under certain failure scenarios"

Why this was correct for shopping cart:
  Adding item to cart that isn't immediately visible to another browser = acceptable
  Failing to add item to cart = not acceptable
  The cost of unavailability (lost sale) > cost of inconsistency (briefly stale cart)

Key insight: Different data has different consistency requirements.
  Cart: eventual consistency OK
  Inventory: strong consistency critical (can't oversell)
  Price: strong consistency critical
  Analytics: eventual consistency fine
```

### Consistent Hashing for Data Distribution
```
Problem: Add/remove nodes → minimize data movement

Virtual nodes (vnodes):
  Each physical node owns multiple virtual positions on the hash ring
  Benefits:
    More uniform distribution (vs few large gaps)
    Gradual rebalancing when adding nodes
    Heterogeneous hardware: powerful nodes get more vnodes

Amazon's numbers (original Dynamo):
  Typical N=3 (three replicas), W=2, R=2 → W+R > N → consistent reads
  Or W=1, R=3 → faster writes, slower consistent reads
  Or W=3, R=1 → very fast reads (but slow writes)

Quorum with sloppy quorum:
  Strict quorum: must get W of the N designated nodes
  Sloppy quorum: if designated node down, use next available node
    "Hinted handoff": node holds data temporarily, forwards when original returns
  This improves availability but risks stale reads
```

### Version Vectors for Conflict Resolution
```
Problem: Two clients write to same key simultaneously (partition scenario)
  Client A: { cart: ["book"] }   → written to replica 1+2 (partition!)
  Client B: { cart: ["phone"] }  → written to replica 2+3

After partition heals: replica 2 has both versions. Which is correct?

Dynamo's answer: Vector clocks (version vectors)
  Each write carries a vector: { node_id: counter } per node that processed it
  { A: 1 } → { A: 1, B: 1 } ← B updated based on A's version (no conflict)
  { A: 1 } and { B: 1 } ← concurrent writes (conflict!)

Conflict resolution:
  If one version "happens before" the other → discard older
  If concurrent → return BOTH versions to client
  Client must resolve and write back the merged version

Shopping cart example:
  Conflict: [book] and [phone]
  Application logic: merge = [book, phone] (union)
  Write back: { cart: ["book", "phone"] }

This is application-level conflict resolution — the DB can't know what "merging a cart" means.
DynamoDB simplified this: last-write-wins by default, conditional writes for optimistic locking
```

---

## How to Use Case Studies in Interviews

### When to Reference Them
```
Interviewer: "How would you handle eventual consistency in this cart service?"
You: "This is actually the exact problem Amazon faced with Dynamo — 
     they explicitly chose availability over consistency because an unavailable 
     cart loses sales, while a briefly stale cart is just a minor UX issue. 
     I'd apply the same reasoning here: [your specific design]."

Interviewer: "How would you handle retry in the payment service?"
You: "Stripe's approach is instructive here — they use idempotency keys to 
     make retries safe, and they separate the 'unknown outcome' case (timeout 
     after request sent) from the 'clean failure' case. I'd structure this 
     similarly: [your design]."
```

### What They Actually Signal
```
Referencing real case studies signals:
  1. You read engineering blogs and papers (curiosity + depth)
  2. You've thought about problems beyond your own codebase
  3. You can reason from first principles, not just patterns
  4. You know that "the right answer" depends on context (as these show)

But: Don't just name-drop. Explain WHY the company made that choice
  and whether those same trade-offs apply to the problem in front of you.
  Sometimes they don't — and saying so is equally impressive.
```

### Quick Reference: Which Case Study for Which Topic

| Topic | Company | Key Lesson |
|---|---|---|
| CDN / content delivery | Netflix | Pre-position content; build if scale justifies it |
| Resilience / chaos | Netflix | Test failure constantly, not just in staging |
| Microservices trade-offs | Netflix | Org benefit must justify distributed systems cost |
| Geospatial / real-time matching | Uber | Geohash/H3 partitioning; city-level sharding |
| Surge/dynamic pricing | Uber | Stream processing for real-time aggregation |
| E2E encryption architecture | WhatsApp | Encryption shapes every downstream decision |
| Minimal storage design | WhatsApp | The best store for some data is "don't store it" |
| Time-series message storage | Discord | Cassandra time-bucket partitioning; Snowflake IDs |
| Technology migration | Discord | Right choice at 100M ≠ right choice at 1T; plan for it |
| Idempotency in payments | Stripe | Client-supplied keys; DB unique constraint as lock |
| API versioning | Stripe | Date-based pinning; never break integrations |
| Partial failure in external APIs | Stripe | Unknown outcome is its own state; void conservatively |
| CAP theorem applied | Amazon/Dynamo | Be explicit about which consistency you need per entity |
| Consistent hashing | Amazon/Dynamo | Vnodes for uniform distribution + gradual rebalance |
| Conflict resolution | Amazon/Dynamo | Application-level merge; DB can't know business logic |
