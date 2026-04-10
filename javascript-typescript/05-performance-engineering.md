# Performance Engineering & Profiling

## The Diagnostic Mindset

Never guess. Profile first, optimize second.

```
Optimization workflow:
  1. Measure baseline (what is the actual metric? p50? p99? throughput?)
  2. Form hypothesis (what do you think is slow and why?)
  3. Profile to confirm (flame graph, query explain, trace)
  4. Fix the confirmed bottleneck
  5. Measure again (did it actually improve? by how much?)
  6. Document (what was the root cause, what was the fix)

Common mistake: optimizing what's convenient, not what's slow.
"We rewrote the sort algorithm" when the real bottleneck was a missing DB index.
```

---

## Node.js Profiling in Production

### Built-in Profiler (V8)
```bash
# Collect CPU profile (30 seconds)
node --prof app.js

# After collecting, isolate log file: isolate-0xXXXX-v8.log
node --prof-process isolate-0xXXXX-v8.log > profile.txt

# Or use clinic.js (much better DX)
npx clinic flame -- node app.js
npx clinic doctor -- node app.js
npx clinic bubbleprof -- node app.js
```

### Reading a Flame Graph
```
Y-axis: call stack depth (top = where CPU is spending time)
X-axis: time proportion (wider = more CPU time)

Patterns to look for:
  Wide flat top: CPU-bound computation in that function
  Wide stack with many layers: deep call chain, possible inefficiency
  "Stairs" pattern: many small allocations + GC pressure
  
Tools: 0x (npm), clinic flame, Chrome DevTools CPU profiler
```

### Chrome DevTools Remote Profiling
```bash
# Start Node with inspector
node --inspect app.js
# or for production (brief profiling window)
node --inspect-brk=0.0.0.0:9229 app.js

# Open Chrome: chrome://inspect
# Attach → Profiler tab → Record → Reproduce the issue → Stop

# For production containers: add --inspect flag, port-forward 9229
kubectl port-forward pod/my-app 9229:9229
```

### Heap Snapshot Analysis
```bash
# Programmatic heap snapshot
const v8 = require('v8');
const fs = require('fs');

// On SIGUSR2 signal: take snapshot without restarting
process.on('SIGUSR2', () => {
  const snapshotStream = v8.writeHeapSnapshot();
  console.log(`Heap snapshot written to: ${snapshotStream}`);
});

# Kill -USR2 <pid>
# Load .heapsnapshot in Chrome DevTools → Memory tab
# Compare two snapshots: find objects growing between them
# Sort by "Retained Size" — shows what's holding memory
```

### Memory Leak Detection Pattern
```typescript
// 1. Watch process.memoryUsage() over time
setInterval(() => {
  const { heapUsed, heapTotal, rss } = process.memoryUsage();
  logger.info('memory', {
    heapUsedMB: Math.round(heapUsed / 1024 / 1024),
    heapTotalMB: Math.round(heapTotal / 1024 / 1024),
    rssMB: Math.round(rss / 1024 / 1024),
  });
}, 60000);

// 2. If heapUsed grows steadily without plateau → leak
// 3. Take two heap snapshots (before / after traffic period)
// 4. Compare: "Objects allocated between snapshots" view
// 5. Look for: EventEmitter listeners, closures, cache objects, Promise chains

// Common leaks to check:
// - Map/Set that only grows (unbounded cache)
// - Event listeners never removed (especially in loops)
// - Async iterators not fully consumed
// - setTimeout chains with no exit condition
// - Global state accumulating request data
```

---

## APM Tools (Production Observability)

### Datadog APM (Node.js)
```typescript
// dd-trace must be the FIRST import (before everything else)
// In the entry file:
import 'dd-trace/init';  // reads DD_* env vars automatically

// Environment variables:
// DD_SERVICE=my-api
// DD_ENV=production
// DD_VERSION=1.2.3  (links traces to deployment)
// DD_TRACE_SAMPLE_RATE=0.1  (sample 10% of traces in high volume)

// Custom spans for business operations:
import tracer from 'dd-trace';

async function processOrder(orderId: string) {
  const span = tracer.startSpan('order.process', {
    childOf: tracer.scope().active() ?? undefined,
  });
  span.setTag('order.id', orderId);
  span.setTag('order.type', 'express');
  
  try {
    const result = await doProcessing(orderId);
    span.setTag('order.status', result.status);
    return result;
  } catch (err) {
    span.setTag('error', true);
    span.setTag('error.message', err.message);
    throw err;
  } finally {
    span.finish();
  }
}
```

### Key Datadog Queries
```
# P99 latency by endpoint
avg:trace.express.request.duration{env:production} by {resource_name}.rollup(p99, 60)

# Error rate
sum:trace.express.request.errors{env:production}.as_rate() /
sum:trace.express.request.hits{env:production}.as_rate()

# Slowest DB queries
avg:mongodb.query.duration{env:production} by {query_signature}.rollup(p99, 300)

# APM Service Map: shows dependency graph with error rates + latency
```

### OpenTelemetry (Vendor-Agnostic)
```typescript
// Instrument once, export to any backend (Datadog, Jaeger, Honeycomb, AWS X-Ray)
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://collector:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
    }),
  ],
});

sdk.start();
// Auto-instruments: HTTP, Express, pg, redis, mongodb, gRPC, etc.
// No manual span creation needed for most use cases
```

---

## Load Testing

### k6 — Load Test Script
```javascript
// load-test.js
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const orderDuration = new Trend('order_duration');

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up to 100 VUs
    { duration: '5m', target: 100 },   // hold at 100 VUs
    { duration: '2m', target: 500 },   // spike to 500 VUs
    { duration: '5m', target: 500 },   // hold at spike
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    errors: ['rate<0.01'],             // error rate < 1%
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const token = getAuthToken();
  
  const startTime = Date.now();
  const res = http.post(
    `${__ENV.BASE_URL}/orders`,
    JSON.stringify({ items: [{ productId: 'prod-1', quantity: 1 }] }),
    { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` } }
  );
  orderDuration.add(Date.now() - startTime);

  check(res, {
    'status is 201': (r) => r.status === 201,
    'has order id': (r) => JSON.parse(r.body).id !== undefined,
  });

  errorRate.add(res.status >= 400);
  sleep(1);
}
```

```bash
# Run load test
k6 run --env BASE_URL=https://staging.example.com load-test.js

# Run with cloud execution + real-time results
k6 cloud load-test.js

# Key metrics to watch:
# http_req_duration p50, p95, p99
# http_req_failed (4xx + 5xx / total)
# vus_max (peak concurrent users)
# iterations (total requests)
```

### Load Test Patterns
```
Smoke test:    1-2 VUs, 1 minute — baseline, confirm test works
Load test:     Expected production load — verify SLOs hold
Stress test:   Increase until breaking point — find limits
Spike test:    Sudden 10x jump — find bottlenecks under burst
Soak test:     Normal load for 24h — find memory leaks, connection leaks
Breakpoint:    Ramp until system fails — know your ceiling
```

---

## Database Query Performance

### PostgreSQL — Reading EXPLAIN ANALYZE
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 50;

-- Key things to check:
-- "Seq Scan" on large table → missing index
-- "rows=10000 (actual rows=1)" → bad statistics, run ANALYZE
-- "Buffers: shared hit=0 read=50000" → data not cached (cold)
-- "Sort Method: external merge" → sort spilled to disk (work_mem too low)
-- "Hash Batches: 4" → hash join spilled to disk
-- Nested Loop with large outer table → may need index on inner table
```

### Slow Query Log
```sql
-- PostgreSQL: log queries slower than 100ms
ALTER SYSTEM SET log_min_duration_statement = '100ms';
SELECT pg_reload_conf();

-- AWS RDS: enable Performance Insights
-- Shows: top SQL, wait events, DB load

-- pg_stat_statements: aggregated query stats
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### N+1 Detection in Production
```typescript
// Use query logging middleware to count queries per request
const queryLogger = {
  count: 0,
  log(query: string) {
    this.count++;
    if (process.env.LOG_QUERIES) logger.debug('DB query', { query });
  }
};

// After request: if queryLogger.count > 10 for a single endpoint → N+1 suspected
// Fix: eager loading, DataLoader, JOIN, or preload

// Sequelize: use { include: [Model] } for eager loading
// TypeORM: relations('order', 'order.items') in QueryBuilder
// Prisma: include: { items: true }
```

---

## Node.js Performance Patterns

### Caching Expensive Operations
```typescript
// Simple TTL cache (use node-cache or lru-cache in production)
import { LRUCache } from 'lru-cache';

const cache = new LRUCache<string, Product>({
  max: 1000,           // max 1000 entries
  ttl: 5 * 60 * 1000, // 5 minute TTL
  allowStale: true,    // serve stale while refreshing
  updateAgeOnGet: false,
});

async function getProduct(id: string): Promise<Product> {
  const cached = cache.get(id);
  if (cached) return cached;

  const product = await db.products.findById(id);
  if (product) cache.set(id, product);
  return product;
}

// For distributed: Redis with stale-while-revalidate pattern
async function getProductCached(id: string): Promise<Product> {
  const raw = await redis.get(`product:${id}`);
  if (raw) {
    const { data, expiresAt } = JSON.parse(raw);
    // Serve stale, trigger background refresh
    if (Date.now() > expiresAt - 30000) {
      refreshProductInBackground(id); // don't await
    }
    return data;
  }
  return refreshProduct(id);
}
```

### Avoiding Serialization Bottlenecks
```typescript
// JSON.stringify of large objects is synchronous and blocks event loop
// Measured: 10MB object stringify ≈ 50ms on modern hardware

// Option 1: Stream JSON (for HTTP responses)
import JSONStream from 'jsonstream';
app.get('/products', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  const stream = db.products.find().cursor(); // DB cursor, not array
  stream.pipe(JSONStream.stringify()).pipe(res);
});

// Option 2: Fast-json-stringify (schema-based, 2-5x faster)
import fastJsonStringify from 'fast-json-stringify';
const stringify = fastJsonStringify({
  type: 'object',
  properties: {
    id: { type: 'string' },
    name: { type: 'string' },
    price: { type: 'number' },
  },
});
res.send(stringify(product)); // much faster than JSON.stringify
```

### HTTP Keep-Alive (Client)
```typescript
import { Agent } from 'https';
import axios from 'axios';

// Without keep-alive: new TCP+TLS handshake per request (~100ms overhead)
// With keep-alive: reuse connection

const agent = new Agent({ keepAlive: true, maxSockets: 50 });
const client = axios.create({
  httpsAgent: agent,
  timeout: 5000,
});
// Critical for high-frequency service-to-service calls
```

---

## Diagnosing a Latency Spike (Interview Scenario)

**"Our API p99 jumped from 200ms to 2000ms at 2pm. How do you diagnose it?"**

```
Step 1 — Isolate timing
  Was it all endpoints or specific ones?
  → APM service map / Datadog APM → which service first?
  Was it correlated with a deploy? Traffic change? External event?
  → Deployment markers on graphs

Step 2 — Check the four horsemen
  CPU: Was any service pegged at 100%? (compute bottleneck)
  Memory: Was GC thrashing? (heap near limit → constant GC pauses)
  Connections: DB connection pool exhausted? (requests queueing)
  I/O: Was a downstream service slow? (X-Ray trace waterfall)

Step 3 — Read the trace
  X-Ray / Datadog APM → find a slow request
  Waterfall view: where did time go?
    Long DB span → slow query (check explain analyze)
    Long external HTTP span → downstream service
    Time between spans → CPU work / serialization
    Gap at start → cold start or connection pool wait

Step 4 — DB specifically
  RDS Performance Insights: what wait events? what top SQL?
  connection count near max_connections → connection pool pressure
  Replication lag on read replicas (if reads were routed there)

Step 5 — Recent changes
  What deployed at 2pm?
  Was there a data migration running?
  Did traffic pattern change (new feature launched)?
  Was there an autoscaling event that caused cold starts?
```
