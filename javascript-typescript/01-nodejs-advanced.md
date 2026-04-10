# JavaScript/TypeScript — Node.js Advanced

## The Event Loop (Deep Understanding)

```
Node.js is single-threaded, non-blocking via event loop

Phases (in order each tick):
┌──────────────────────────────┐
│ timers          (setTimeout, setInterval)
│ pending callbacks (I/O errors from prev tick)
│ idle, prepare   (internal)
│ poll            ← wait for I/O; execute I/O callbacks
│ check           (setImmediate)
│ close callbacks (socket.destroy, etc.)
└──────────────────────────────┘

Between each phase:
  process.nextTick queue   (highest priority!)
  Promise microtasks queue (then/catch/finally)
```

### Execution Order Quiz (Know This Cold)
```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
process.nextTick(() => console.log('4'));
setImmediate(() => console.log('5'));
console.log('6');

// Output: 1, 6, 4, 3, 2, 5
// Sync first → nextTick → microtasks (Promises) → timers → setImmediate
```

### Blocking the Event Loop (Never Do This)
```javascript
// BAD: blocks event loop for ALL requests
app.get('/report', (req, res) => {
  const data = computeHeavyReport(); // synchronous CPU work
  res.json(data);
});

// GOOD: offload to worker thread
import { Worker } from 'worker_threads';
app.get('/report', (req, res) => {
  const worker = new Worker('./reportWorker.js');
  worker.on('message', (data) => res.json(data));
  worker.postMessage({ params: req.query });
});
```

**What blocks the event loop**: JSON.parse of large objects, crypto operations (use `crypto.pbkdf2` async), regex with catastrophic backtracking, synchronous fs calls, `while(true)`.

---

## Streams

```
Types:
  Readable  — source of data (fs.createReadStream, http.IncomingMessage)
  Writable  — destination (fs.createWriteStream, http.ServerResponse)
  Duplex    — both readable and writable (net.Socket)
  Transform — duplex that transforms data (zlib.createGzip, crypto.createCipher)
```

### Why Streams
```javascript
// BAD: loads entire file into memory
const data = fs.readFileSync('huge-file.csv');
processData(data);

// GOOD: stream processes chunk by chunk
fs.createReadStream('huge-file.csv')
  .pipe(csv.parse({ columns: true }))
  .pipe(new Transform({
    objectMode: true,
    transform(record, enc, callback) {
      this.push(processRecord(record));
      callback();
    }
  }))
  .pipe(writableDestination);
```

### Backpressure
```javascript
// readable.pipe(writable) handles backpressure automatically

// Manual backpressure:
const canContinue = writable.write(chunk);
if (!canContinue) {
  readable.pause();
  writable.once('drain', () => readable.resume());
}
```

---

## Worker Threads vs Child Process vs Cluster

| | Worker Threads | Child Process | Cluster |
|---|---|---|---|
| Use for | CPU-intensive in same process | Separate process, different language | HTTP server scaling |
| Memory | Shared ArrayBuffer possible | Separate memory | Separate memory |
| Communication | SharedArrayBuffer, MessageChannel | IPC, stdin/stdout | IPC |
| Overhead | Low | High (fork) | Medium |

```javascript
// Cluster: distribute HTTP connections across cores
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  cluster.on('exit', (worker) => cluster.fork()); // restart dead workers
} else {
  startServer(); // each worker runs own event loop
}
```

---

## Memory Management & Garbage Collection

```javascript
// V8 GC: Generational
// Young generation (Scavenge): Short-lived objects, frequent collection
// Old generation (Mark-Sweep + Mark-Compact): Long-lived, less frequent

// Common memory leaks in Node.js:

// 1. Global variables accumulating data
global.cache = {};  // grows unbounded
// Fix: use Map with max size, or WeakMap for object keys

// 2. Closures holding references
function createServer() {
  const hugeBuffer = Buffer.alloc(1024 * 1024 * 100); // 100MB
  return {
    handler: (req) => {
      // hugeBuffer referenced here even if never used
      return req.path;
    }
  };
}

// 3. Event listener leaks
emitter.on('data', handler); // never removed
// Fix: emitter.off('data', handler) or use once()

// 4. Promises never settling (dangling promises)
// Fix: always handle rejection; set timeouts

// Heap profiling:
// node --inspect app.js → Chrome DevTools → Memory → Take heap snapshot
// node --heapsnapshot-signal=SIGUSR2 app.js → compare snapshots
```

### Memory Limits
```bash
# Default heap: ~1.5GB (64-bit)
node --max-old-space-size=4096 app.js  # 4GB heap

# Monitor heap
process.memoryUsage()
// { heapTotal, heapUsed, external, arrayBuffers, rss }
```

---

## Error Handling Patterns

```typescript
// Typed error classes
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number,
    public readonly isOperational = true  // vs programmer errors
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} ${id} not found`, 'NOT_FOUND', 404);
  }
}

// Domain Result type (Railway-oriented)
type Result<T, E = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function findUser(id: string): Promise<Result<User>> {
  const user = await db.findById(id);
  if (!user) return { ok: false, error: new NotFoundError('User', id) };
  return { ok: true, value: user };
}

// Global unhandled rejection handler
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled rejection', { reason, promise });
  // Let process-supervisor restart gracefully
  process.exit(1);
});
```

---

## Performance Optimization in Node.js

### Async Patterns
```typescript
// Anti-pattern: sequential awaits when independent
const user = await getUser(id);
const orders = await getOrders(id);  // waits for user unnecessarily

// Better: parallel
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);

// Don't ignore errors in Promise.all — one failure rejects all
// Use Promise.allSettled for independent results:
const results = await Promise.allSettled([getUser(id), getOrders(id)]);

// Limit concurrency for batch operations
import pLimit from 'p-limit';
const limit = pLimit(10);
await Promise.all(items.map(item => limit(() => processItem(item))));
```

### Connection Pooling (pg/knex)
```typescript
import { Pool } from 'pg';

const pool = new Pool({
  max: 10,            // max connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// In Lambda: share pool across warm invocations
// In ECS: pool per container; set max based on DB max_connections / num_tasks
// Rule of thumb: max_pool_size = (core_count * 2) + effective_spindle_count
```

### HTTP Client Best Practices
```typescript
import axios from 'axios';

const client = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 3000,         // always set timeout
  headers: { 'Accept-Encoding': 'gzip' },
});

// Retry with exponential backoff
import axiosRetry from 'axios-retry';
axiosRetry(client, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (error) =>
    axiosRetry.isNetworkOrIdempotentRequestError(error) ||
    error.response?.status === 429,
});
```

---

## Testing Strategy (Principal Level)

### Test Pyramid
```
         ╱╲
        ╱E2E╲         (few, slow, expensive)
       ╱──────╲
      ╱Integration╲   (moderate, test real DB/services)
     ╱──────────────╲
    ╱   Unit Tests   ╲ (many, fast, isolated)
   ╱──────────────────╲
```

### Integration Tests (Don't Mock the DB)
```typescript
// Use real PostgreSQL via testcontainers
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let container: StartedPostgreSqlContainer;
let pool: Pool;

beforeAll(async () => {
  container = await new PostgreSqlContainer().start();
  pool = new Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
});

afterAll(async () => {
  await pool.end();
  await container.stop();
});

test('createOrder stores in DB', async () => {
  const service = new OrderService(pool);
  const order = await service.create({ userId: '1', items: [...] });
  
  const stored = await pool.query('SELECT * FROM orders WHERE id = $1', [order.id]);
  expect(stored.rows[0].status).toBe('pending');
});
```

### Contract Testing
```typescript
// Consumer (frontend) defines what it expects from provider (API)
// Pact: generates pact file; provider verifies against it
// Prevents breaking API changes from reaching production
```

### What NOT to test
- Trivial getters/setters
- Framework code (Express routing internals)
- Implementation details — test behavior, not how
- Infrastructure (VPC config, IAM) — use integration tests + AWS Config rules

---

## TypeScript Patterns for Production

### Branded Types (Prevent mixing IDs)
```typescript
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

const toUserId = (id: string): UserId => id as UserId;

function getUser(id: UserId): Promise<User> { ... }

getUser(toUserId('abc'));         // ✓
getUser('abc' as OrderId);       // ✗ TypeScript error
```

### Discriminated Unions
```typescript
type ApiResponse<T> =
  | { status: 'success'; data: T; requestId: string }
  | { status: 'error'; code: string; message: string; requestId: string }
  | { status: 'loading' };

function handle(res: ApiResponse<User>) {
  switch (res.status) {
    case 'success': return res.data.name;  // data typed as User
    case 'error':   return res.code;       // code typed as string
    case 'loading': return '...';
  }
}
```

### Zod for Runtime Validation
```typescript
import { z } from 'zod';

const CreateOrderSchema = z.object({
  userId: z.string().uuid(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive(),
  })).min(1),
  couponCode: z.string().optional(),
});

type CreateOrderDto = z.infer<typeof CreateOrderSchema>;

app.post('/orders', async (req, res) => {
  const result = CreateOrderSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({ errors: result.error.format() });
  }
  // result.data is fully typed as CreateOrderDto
});
```
