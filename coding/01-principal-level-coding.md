# Coding at Principal Level

## What to Expect (Different from Senior)

Principal-level coding rounds are **not LeetCode grinding**. Interviewers want to see:

- **Design thinking while coding**: "Before I write this, here's why I'm structuring it this way..."
- **Production-grade code**: error handling, edge cases, extensibility — not just "it works"
- **Trade-off awareness**: "I'm using a Map here for O(1) lookup; if memory were a constraint I'd use a sorted array with binary search"
- **Testability**: "I'd structure this so it's injectable, making it easy to test in isolation"
- **Incremental delivery**: Start with working solution, then optimize — don't spend 20 min on a "perfect" design

Common problem types at Principal level:
1. Implement a system primitive (rate limiter, circuit breaker, LRU cache, event emitter)
2. Refactor/extend a given messy codebase
3. Design + implement a small API or service layer
4. Live debugging: find the bug in provided code
5. Algorithm with system design context ("implement this efficiently at 10M ops/day scale")

---

## How to Behave During a Coding Round

```
1. Read the problem → restate it back ("So you want me to implement X which does Y?")
2. Clarify constraints BEFORE coding:
   - What inputs can be malformed or null?
   - What's the expected scale? (changes algorithm choice)
   - Should this be thread-safe? (Node: usually no, but worth asking)
   - What error behavior is expected?
3. State your approach in 1 sentence before writing it
4. Code with narration: "I'm using a Map here because..."
5. Write tests or test cases as you go (or after if pressed for time)
6. Review your own code before saying "done"
7. Proactively mention what you'd add: "In production I'd add logging here and handle the case where..."
```

---

## Pattern 1: LRU Cache

**Why asked**: Tests data structure knowledge (doubly-linked list + hash map), trade-off discussion (O(1) vs simpler O(n)), real-world relevance.

```typescript
class LRUCache<K, V> {
  private readonly capacity: number;
  private map: Map<K, V>;  // Map preserves insertion order in JS

  constructor(capacity: number) {
    if (capacity <= 0) throw new Error('Capacity must be positive');
    this.capacity = capacity;
    this.map = new Map();
  }

  get(key: K): V | undefined {
    if (!this.map.has(key)) return undefined;
    // Move to end (most recently used)
    const value = this.map.get(key)!;
    this.map.delete(key);
    this.map.set(key, value);
    return value;
  }

  set(key: K, value: V): void {
    if (this.map.has(key)) {
      this.map.delete(key); // remove to re-insert at end
    } else if (this.map.size >= this.capacity) {
      // Evict LRU: first key in Map is least recently used
      const lruKey = this.map.keys().next().value;
      this.map.delete(lruKey);
    }
    this.map.set(key, value);
  }

  get size(): number { return this.map.size; }
  
  has(key: K): boolean { return this.map.has(key); }
  
  clear(): void { this.map.clear(); }
}

// Tests
const cache = new LRUCache<string, number>(3);
cache.set('a', 1); cache.set('b', 2); cache.set('c', 3);
cache.get('a');       // access 'a' → now most recently used
cache.set('d', 4);    // evict 'b' (LRU), not 'a'
console.assert(!cache.has('b'), 'b should be evicted');
console.assert(cache.has('a'), 'a should be retained');
```

**Talking points**: "JS Map preserves insertion order, so LRU key is always first. This gives O(1) get/set. A pure HashMap approach would be O(n) for eviction."

---

## Pattern 2: Rate Limiter

**Why asked**: Direct system design → code translation. Tests algorithm knowledge, time handling, production concerns.

```typescript
// Sliding window rate limiter using timestamp log
class SlidingWindowRateLimiter {
  private readonly limit: number;
  private readonly windowMs: number;
  private readonly requests: Map<string, number[]>;  // key → sorted timestamps

  constructor(limit: number, windowMs: number) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.requests = new Map();
  }

  isAllowed(key: string): boolean {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    if (!this.requests.has(key)) {
      this.requests.set(key, []);
    }

    const timestamps = this.requests.get(key)!;
    // Remove expired timestamps (O(n) but bounded by limit)
    let i = 0;
    while (i < timestamps.length && timestamps[i] <= windowStart) i++;
    timestamps.splice(0, i);

    if (timestamps.length >= this.limit) {
      return false;  // rate limited
    }

    timestamps.push(now);
    return true;
  }

  // For production: would use Redis for distributed state
  getRemainingRequests(key: string): number {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    const timestamps = this.requests.get(key) ?? [];
    const active = timestamps.filter(t => t > windowStart);
    return Math.max(0, this.limit - active.length);
  }
  
  getResetTime(key: string): number {
    const timestamps = this.requests.get(key);
    if (!timestamps?.length) return Date.now();
    return timestamps[0] + this.windowMs;  // when oldest expires
  }
}

// Tests
const limiter = new SlidingWindowRateLimiter(3, 1000); // 3 per second
console.assert(limiter.isAllowed('user-1') === true);
console.assert(limiter.isAllowed('user-1') === true);
console.assert(limiter.isAllowed('user-1') === true);
console.assert(limiter.isAllowed('user-1') === false); // 4th → rejected
console.assert(limiter.isAllowed('user-2') === true);  // different key → allowed
```

**Talking points**: "Memory grows with unique keys. In production, I'd use Redis with ZREMRANGEBYSCORE + ZADD in a pipeline for distributed, atomic sliding window. I'd also add a cleanup job for inactive keys."

---

## Pattern 3: Circuit Breaker

**Why asked**: Common production resilience pattern. Tests state machine thinking and async patterns.

```typescript
enum CircuitState { CLOSED, OPEN, HALF_OPEN }

interface CircuitBreakerOptions {
  failureThreshold: number;   // failures before opening
  successThreshold: number;   // successes in half-open before closing
  timeout: number;            // ms to wait before trying half-open
  onStateChange?: (from: CircuitState, to: CircuitState) => void;
}

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failures = 0;
  private successes = 0;
  private lastFailureTime: number | null = null;

  constructor(
    private readonly fn: (...args: any[]) => Promise<any>,
    private readonly opts: CircuitBreakerOptions
  ) {}

  async execute<T>(...args: any[]): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      // Check if timeout has passed → try half-open
      if (Date.now() - this.lastFailureTime! >= this.opts.timeout) {
        this.transition(CircuitState.HALF_OPEN);
      } else {
        throw new Error('Circuit breaker is OPEN — service unavailable');
      }
    }

    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    if (this.state === CircuitState.HALF_OPEN) {
      this.successes++;
      if (this.successes >= this.opts.successThreshold) {
        this.successes = 0;
        this.transition(CircuitState.CLOSED);
      }
    }
  }

  private onFailure(): void {
    this.lastFailureTime = Date.now();
    this.successes = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.transition(CircuitState.OPEN);
      return;
    }

    this.failures++;
    if (this.failures >= this.opts.failureThreshold) {
      this.transition(CircuitState.OPEN);
    }
  }

  private transition(to: CircuitState): void {
    this.opts.onStateChange?.(this.state, to);
    this.state = to;
  }

  getState(): CircuitState { return this.state; }
}

// Usage
const breaker = new CircuitBreaker(
  (userId: string) => paymentService.charge(userId),
  { failureThreshold: 5, successThreshold: 2, timeout: 30000,
    onStateChange: (from, to) => logger.warn('Circuit breaker state change', { from, to }) }
);

try {
  const result = await breaker.execute(userId);
} catch (err) {
  if (err.message.includes('OPEN')) {
    return { status: 'service_unavailable', fallback: true };
  }
  throw err;
}
```

---

## Pattern 4: Typed Event Emitter

**Why asked**: Tests TypeScript generics, event-driven patterns, memory management.

```typescript
type Listener<T> = (data: T) => void | Promise<void>;

class TypedEventEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<Listener<any>>>();
  private onceListeners = new Map<keyof Events, Set<Listener<any>>>();

  on<K extends keyof Events>(event: K, listener: Listener<Events[K]>): () => void {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener);
    // Return unsubscribe function
    return () => this.off(event, listener);
  }

  once<K extends keyof Events>(event: K, listener: Listener<Events[K]>): void {
    if (!this.onceListeners.has(event)) this.onceListeners.set(event, new Set());
    this.onceListeners.get(event)!.add(listener);
  }

  off<K extends keyof Events>(event: K, listener: Listener<Events[K]>): void {
    this.listeners.get(event)?.delete(listener);
    this.onceListeners.get(event)?.delete(listener);
  }

  async emit<K extends keyof Events>(event: K, data: Events[K]): Promise<void> {
    const listeners = [...(this.listeners.get(event) ?? [])];
    const onceListeners = [...(this.onceListeners.get(event) ?? [])];
    this.onceListeners.delete(event); // remove all once listeners

    const all = [...listeners, ...onceListeners];
    await Promise.all(all.map(l => l(data)));
  }

  listenerCount(event: keyof Events): number {
    return (this.listeners.get(event)?.size ?? 0) +
           (this.onceListeners.get(event)?.size ?? 0);
  }

  removeAllListeners(event?: keyof Events): void {
    if (event) {
      this.listeners.delete(event);
      this.onceListeners.delete(event);
    } else {
      this.listeners.clear();
      this.onceListeners.clear();
    }
  }
}

// Usage — fully typed
type AppEvents = {
  'order:created': { orderId: string; userId: string; total: number };
  'payment:failed': { orderId: string; reason: string; retryable: boolean };
};

const bus = new TypedEventEmitter<AppEvents>();
const unsub = bus.on('order:created', async ({ orderId, total }) => {
  await sendConfirmationEmail(orderId); // total is typed as number
});
// Cleanup
unsub();
```

---

## Pattern 5: Promise Pool (Bounded Concurrency)

**Why asked**: Real-world async pattern. Tests understanding of Promise mechanics and backpressure.

```typescript
async function promisePool<T, R>(
  items: T[],
  concurrency: number,
  task: (item: T, index: number) => Promise<R>
): Promise<R[]> {
  const results: R[] = new Array(items.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < items.length) {
      const index = nextIndex++;
      results[index] = await task(items[index], index);
    }
  }

  // Start `concurrency` workers; they pull from the shared queue
  const workers = Array.from({ length: Math.min(concurrency, items.length) }, worker);
  await Promise.all(workers);
  return results;
}

// Usage: process 1000 images, max 10 at a time
const results = await promisePool(imageIds, 10, async (id) => {
  const image = await fetchImage(id);
  return processImage(image);
});

// With error handling variant — collect errors, don't fail-fast
async function promisePoolSettled<T, R>(
  items: T[],
  concurrency: number,
  task: (item: T) => Promise<R>
): Promise<PromiseSettledResult<R>[]> {
  return promisePool(items, concurrency, (item) =>
    task(item).then(
      value => ({ status: 'fulfilled', value } as const),
      reason => ({ status: 'rejected', reason } as const)
    )
  );
}
```

---

## Live Debugging: Common Node.js Gotchas

These come up in "find the bug" rounds:

```typescript
// Bug 1: Async forEach doesn't await
async function processOrders(orders: Order[]) {
  orders.forEach(async (order) => {  // ← BUG: forEach ignores returned Promise
    await processOrder(order);
  });
  console.log('done'); // runs immediately, before processing finishes
}
// Fix: for...of loop or Promise.all(orders.map(async (o) => ...))

// Bug 2: Closure in loop captures reference, not value
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // prints 3, 3, 3
}
// Fix: use let, or IIFE: setTimeout(((j) => () => console.log(j))(i), 100)

// Bug 3: Floating Promise (unhandled rejection)
async function handler(req, res) {
  sendAnalytics(req);  // ← not awaited; rejection is unhandled
  res.json({ ok: true });
}
// Fix: void sendAnalytics(req) (explicit fire-and-forget + global handler catches it)
// Or: await sendAnalytics(req).catch(e => logger.error(e))

// Bug 4: EventEmitter memory leak
function makeRequest() {
  emitter.on('data', handleData); // never removed → leak on repeated calls
}
// Fix: emitter.once(), or store reference + call emitter.off() when done

// Bug 5: Race condition in cache population
const cache = new Map();
async function getUser(id: string) {
  if (cache.has(id)) return cache.get(id);
  // Two concurrent calls both miss cache → two DB calls → last one wins
  const user = await db.findUser(id);
  cache.set(id, user);
  return user;
}
// Fix: store the Promise in the cache (in-flight deduplication)
const inFlight = new Map<string, Promise<User>>();
async function getUserSafe(id: string) {
  if (cache.has(id)) return cache.get(id);
  if (inFlight.has(id)) return inFlight.get(id);
  const promise = db.findUser(id).then(user => { cache.set(id, user); return user; })
                                  .finally(() => inFlight.delete(id));
  inFlight.set(id, promise);
  return promise;
}
```

---

## What Principal-Level Code Looks Like vs Senior

```typescript
// Senior level: correct, handles happy path
async function transferFunds(fromId: string, toId: string, amount: number) {
  const from = await db.accounts.findById(fromId);
  const to = await db.accounts.findById(toId);
  await db.accounts.update(fromId, { balance: from.balance - amount });
  await db.accounts.update(toId, { balance: to.balance + amount });
}

// Principal level: adds transaction, validation, idempotency, types, error clarity
async function transferFunds(
  fromId: AccountId,
  toId: AccountId,
  amount: Money,
  idempotencyKey: string,
): Promise<TransferResult> {
  // Guard against duplicate execution
  const existing = await db.transfers.findByIdempotencyKey(idempotencyKey);
  if (existing) return { status: 'already_processed', transfer: existing };

  if (amount.value <= 0) throw new ValidationError('Amount must be positive');
  if (fromId === toId) throw new ValidationError('Cannot transfer to self');

  return db.transaction(async (trx) => {
    // Lock rows in consistent order (lower ID first) → prevents deadlock
    const [first, second] = [fromId, toId].sort();
    const accounts = await trx.accounts.findByIdsForUpdate([first, second]);
    
    const from = accounts.find(a => a.id === fromId)!;
    const to = accounts.find(a => a.id === toId)!;

    if (from.balance.value < amount.value) {
      throw new InsufficientFundsError(fromId, from.balance, amount);
    }

    const transfer = await trx.transfers.create({
      fromId, toId, amount, idempotencyKey, status: 'completed',
    });
    await trx.accounts.update(fromId, { balance: from.balance.subtract(amount) });
    await trx.accounts.update(toId, { balance: to.balance.add(amount) });

    return { status: 'success', transfer };
  });
}
```
