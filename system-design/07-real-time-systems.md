# System Design — Real-Time Systems

## Transport Mechanisms Comparison

| | Short Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| Protocol | HTTP/1.1 | HTTP/1.1 | HTTP/1.1 | WS (TCP upgrade) |
| Direction | Client → Server | Client → Server | Server → Client | Bidirectional |
| Latency | High (poll interval) | Low (~100ms) | Low | Low |
| Connection | New per request | Held open, re-estd on message | Persistent | Persistent |
| Overhead | High (headers each req) | Medium | Low (streaming) | Very low (frames) |
| Firewalls | Works everywhere | Works everywhere | Works everywhere | May need 443 |
| Scaling complexity | None | Medium | Medium | High |
| Auto-reconnect | N/A | Manual | Built-in | Manual |
| Best for | Simple dashboards, low frequency | Notifications, moderate scale | Live feeds, scores, 1-way push | Chat, gaming, collab |

---

## Short Polling
```typescript
// Client polls every N seconds
setInterval(async () => {
  const res = await fetch('/api/notifications');
  const data = await res.json();
  updateUI(data);
}, 3000);
```
- Simplest to implement and scale (stateless HTTP)
- Wastes requests when nothing changed
- Use when: update frequency is predictable, latency tolerance > poll interval, you want maximum simplicity

---

## Long Polling
```typescript
// Client holds connection open; server responds when data available
async function longPoll() {
  while (true) {
    try {
      const res = await fetch('/api/events?lastId=' + lastId, {
        signal: AbortSignal.timeout(30000),  // 30s timeout
      });
      const events = await res.json();
      process(events);
      lastId = events.at(-1)?.id ?? lastId;
    } catch {
      await sleep(1000); // backoff on error before reconnecting
    }
  }
}

// Server side (Express)
app.get('/api/events', async (req, res) => {
  const { lastId } = req.query;
  
  // Try to get immediately available events
  let events = await getEventsSince(lastId);
  if (events.length > 0) return res.json(events);
  
  // Wait up to 25s for new events
  const timeout = setTimeout(() => res.json([]), 25000);
  const handler = (event) => {
    clearTimeout(timeout);
    res.json([event]);
  };
  eventEmitter.once('new-event', handler);
  req.on('close', () => {
    clearTimeout(timeout);
    eventEmitter.off('new-event', handler);
  });
});
```

---

## Server-Sent Events (SSE)
```typescript
// Server
app.get('/api/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no');  // disable Nginx buffering
  
  res.write('retry: 3000\n\n');  // client reconnect after 3s
  
  const send = (event: string, data: unknown) => {
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };
  
  const interval = setInterval(() => {
    send('heartbeat', { ts: Date.now() });
  }, 15000);
  
  const handler = (data: any) => send('order-update', data);
  orderEvents.on('update', handler);
  
  req.on('close', () => {
    clearInterval(interval);
    orderEvents.off('update', handler);
  });
});

// Client — built-in reconnect + event listener API
const es = new EventSource('/api/stream');
es.addEventListener('order-update', (e) => {
  const data = JSON.parse(e.data);
  updateOrderStatus(data);
});
es.addEventListener('error', () => console.warn('SSE reconnecting...'));
```

### SSE Limitations
- Unidirectional: only server → client
- Max 6 concurrent SSE connections per origin in HTTP/1.1 (not an issue with HTTP/2)
- Need to send heartbeats to prevent proxy timeouts (every 15–30s)

---

## WebSockets

### Node.js WebSocket Server
```typescript
import { WebSocketServer, WebSocket } from 'ws';

// Connection registry: userId → Set of WebSocket connections
const connections = new Map<string, Set<WebSocket>>();

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  const userId = authenticate(req);  // extract from cookie/token
  if (!userId) return ws.close(4001, 'Unauthorized');
  
  // Register connection
  if (!connections.has(userId)) connections.set(userId, new Set());
  connections.get(userId)!.add(ws);
  
  ws.on('message', (raw) => {
    const msg = JSON.parse(raw.toString());
    handleMessage(userId, msg);
  });
  
  ws.on('close', () => {
    connections.get(userId)?.delete(ws);
    if (connections.get(userId)?.size === 0) connections.delete(userId);
  });
  
  // Heartbeat
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
});

// Ping interval — detect dead connections
setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);

// Push to specific user (from anywhere in app)
function pushToUser(userId: string, event: string, data: unknown) {
  const sockets = connections.get(userId);
  if (!sockets) return;
  const payload = JSON.stringify({ event, data });
  sockets.forEach((ws) => {
    if (ws.readyState === WebSocket.OPEN) ws.send(payload);
  });
}
```

---

## Scaling WebSockets — The Core Problem

A WebSocket is a **stateful, persistent connection** pinned to one server instance.

```
User A connected to Server 1
User B connected to Server 2

User A sends message to User B:
  → Server 1 receives it
  → Server 1 doesn't have User B's connection
  → ??? 
```

### Solution: Pub/Sub Adapter (Redis)
```typescript
import { createClient } from 'redis';

const pub = createClient();
const sub = createClient();

await sub.subscribe('ws:messages', (raw) => {
  const { targetUserId, event, data } = JSON.parse(raw);
  
  // Deliver to local connections for this user
  const sockets = connections.get(targetUserId);
  sockets?.forEach((ws) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ event, data }));
    }
  });
});

// When a message needs to reach any user:
async function sendToUser(userId: string, event: string, data: unknown) {
  // Try local delivery first
  if (connections.has(userId)) {
    pushToUser(userId, event, data);
    return;
  }
  // Publish to Redis → all servers will check their local connections
  await pub.publish('ws:messages', JSON.stringify({ targetUserId: userId, event, data }));
}
```

### Alternative: Socket.io with Redis Adapter
- Socket.io handles reconnection, rooms, namespaces, fallback to long polling
- `@socket.io/redis-adapter` handles cross-server message delivery automatically
- Trade-off: more abstraction, less control; fine for most use cases

### Sticky Sessions (Alternative Approach)
- Load balancer routes user → always same server (by IP hash or cookie)
- Simpler: no cross-server messaging needed
- Downside: uneven load distribution; server failure loses all its connections
- Not recommended for large scale

---

## WebSocket Scaling Architecture
```
Clients
  ↓
L4 Load Balancer (NLB on AWS — preserves TCP connection)
  ↓
WebSocket Gateway Tier (EC2 / ECS — horizontally scalable)
  ↓            ↓
Redis Pub/Sub  Redis (presence: userId → serverId)
  ↓
Message Service / Business Logic
  ↓
DB (Cassandra/DynamoDB for messages)
```

### Presence Service
```
User connects to WS Server 1:
  Redis HSET presence:{userId} server server-1 last_seen <ts>
  Expire: 60s; refresh via heartbeat

User disconnects (or times out):
  Redis HDEL presence:{userId}
  Publish "user:offline" event

Check if user is online:
  Redis HGET presence:{userId}
  → present + recent last_seen → online
```

---

## Real-Time Design Patterns

### Fan-Out Patterns
```
Small group (<100 users, e.g., document collaborators):
  Push message to all connected users directly (in-memory room map)
  
Large broadcast (>10K users, e.g., live sports score):
  Single producer → topic → subscriber fan-out at edge
  Don't send 10K individual messages; use pub/sub or SSE stream

Hybrid (news feed, 1M followers):
  Pre-push to active users (online, high engagement)
  Pull on demand for inactive users
```

### Read Your Own Write Consistency (Real-Time Context)
```
User sends chat message → server saves to DB → broadcasts to room
Problem: user might see their own message delayed if using eventual consistency

Solution:
  Optimistic UI: show message immediately on sender's side
  Server confirms with message ID: update local state with server ID
  On reconnect: sync messages since last known ID
```

### Reconnection & Message Ordering
```typescript
// Client tracks last received message ID
let lastMessageId = localStorage.getItem('lastMsgId') ?? '0';

es.addEventListener('message', (e) => {
  const msg = JSON.parse(e.data);
  lastMessageId = msg.id;
  localStorage.setItem('lastMsgId', msg.id);
});

// On reconnect, pass lastEventId → server sends missed messages
const es = new EventSource(`/stream?lastEventId=${lastMessageId}`);
// SSE protocol: Last-Event-ID header sent automatically if you set `id:` in events
```

---

## Common Interview Questions

**"How would you build a real-time collaborative document editor (like Google Docs)?"**
```
Conflict resolution: OT (Operational Transformation) or CRDT
  - CRDT (Conflict-free Replicated Data Types) preferred now:
    Each operation is mathematically conflict-free
    Can merge out-of-order; works offline-first
    Libraries: Yjs, Automerge

Transport: WebSocket (bidirectional, low latency)
Architecture:
  - Client buffers ops while offline
  - Client sends op → server applies to CRDT state → broadcasts to room
  - New joiner: server sends full document snapshot + ops since snapshot
  - Persist: periodic snapshots to S3 + op log to Cassandra
```

**"Design a live leaderboard for a gaming platform (10M concurrent users)"**
```
Redis Sorted Set per game/tournament:
  ZADD leaderboard:{gameId} <score> <userId>
  ZREVRANK leaderboard:{gameId} <userId>  → user's rank
  ZREVRANGE leaderboard:{gameId} 0 99     → top 100

Updates: score change → ZADD → publish delta to Kafka → WebSocket broadcast
Fan-out: SSE stream for top-100 leaderboard (refresh every 5s or on delta)
Scale: shard leaderboard by score range if single sorted set becomes a bottleneck
Near-real-time: accept ~1s lag; batch micro-updates to avoid Redis flood
```
