# JavaScript/TypeScript — GraphQL Architecture

## Why GraphQL (and When Not To)

### Use GraphQL When
- Multiple clients (web, mobile, TV) need different shapes of the same data
- Rapid frontend iteration — teams can get exactly what they need without new endpoints
- Aggregating data from multiple services (BFF pattern)
- Schema is the contract — shared, versioned, auto-documented

### Don't Use GraphQL When
- Simple CRUD with few clients — REST is simpler and faster to implement
- File upload heavy (multipart is awkward in GraphQL)
- Streaming/real-time as primary use case (REST + SSE/WS is cleaner)
- Public API for third-party developers (REST with OpenAPI is more universal)

---

## Schema Design

### Type System Basics
```graphql
type User {
  id: ID!
  email: String!
  name: String!
  orders(status: OrderStatus, limit: Int = 20): [Order!]!
  createdAt: DateTime!
}

type Order {
  id: ID!
  status: OrderStatus!
  items: [OrderItem!]!
  total: Float!
  user: User!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

input CreateOrderInput {
  items: [OrderItemInput!]!
  shippingAddressId: ID!
  couponCode: String
}

type Query {
  user(id: ID!): User
  me: User
  orders(filter: OrderFilter, pagination: PaginationInput): OrderConnection!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(id: ID!): CancelOrderPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

### Schema Design Principles

**Mutation Payloads (never return the type directly)**
```graphql
# Bad — no room for errors or extra context
type Mutation {
  createOrder(input: CreateOrderInput!): Order!
}

# Good — extensible, consistent error handling
type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
}

type CreateOrderPayload {
  order: Order          # null on error
  errors: [UserError!]!
  clientMutationId: String  # idempotency
}

type UserError {
  field: String         # which input field caused the error
  message: String!
  code: String!         # machine-readable (INSUFFICIENT_STOCK, INVALID_COUPON)
}
```

**Pagination — Cursor-based (not offset)**
```graphql
type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}
type OrderEdge {
  node: Order!
  cursor: String!
}
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
# Relay-spec pagination: stable under inserts/deletes (unlike offset)
# cursor = base64(entity_id) or base64(timestamp + id)
```

---

## The N+1 Problem & DataLoader

### The Problem
```typescript
// Schema: User has many Orders; Order has one User
// Query: { orders { user { name } } }

// Naive resolver:
const resolvers = {
  Order: {
    user: async (order) => {
      return db.user.findById(order.userId); // called N times for N orders!
    }
  }
};
// 1 query to get 100 orders + 100 queries for users = 101 DB hits
```

### DataLoader Solution
```typescript
import DataLoader from 'dataloader';

// Batches individual loads into single DB call
const userLoader = new DataLoader<string, User>(async (userIds) => {
  const users = await db.user.findByIds([...userIds]);
  const userMap = new Map(users.map(u => [u.id, u]));
  return userIds.map(id => userMap.get(id) ?? new Error(`User ${id} not found`));
});

// Resolver — same API, batched automatically
const resolvers = {
  Order: {
    user: (order) => userLoader.load(order.userId)
    // 100 order resolvers each call load()
    // DataLoader batches into 1 DB call: WHERE id IN (1, 2, 3, ...)
  }
};

// Create per-request (not singleton!) to avoid cross-request caching
app.use((req, res, next) => {
  req.loaders = {
    user: new DataLoader(batchLoadUsers),
    product: new DataLoader(batchLoadProducts),
  };
  next();
});
```

### DataLoader with Caching
```typescript
// DataLoader caches within a single request by default
// Same key requested multiple times → only one load
userLoader.load('user-1');  // hits DB
userLoader.load('user-1');  // returns cached result (same request)

// Clear cache if you mutate
await updateUser('user-1', data);
userLoader.clear('user-1');  // force fresh load next time

// Prime the cache (if you fetched the data already)
const users = await db.user.findAll();
users.forEach(u => userLoader.prime(u.id, u));
```

---

## Apollo Server Setup (Node.js / TypeScript)

```typescript
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { useServer } from 'graphql-ws/lib/use/ws';

const schema = makeExecutableSchema({ typeDefs, resolvers });

const server = new ApolloServer({
  schema,
  plugins: [
    ApolloServerPluginDrainHttpServer({ httpServer }),
    // Disable introspection in production
    process.env.NODE_ENV === 'production'
      ? ApolloServerPluginLandingPageDisabled()
      : ApolloServerPluginLandingPageLocalDefault(),
  ],
  formatError: (formattedError, error) => {
    // Don't leak internal details in production
    if (process.env.NODE_ENV === 'production' && !isOperationalError(error)) {
      return { message: 'Internal server error', code: 'INTERNAL_ERROR' };
    }
    return formattedError;
  },
});

// Context — available in every resolver
app.use('/graphql', expressMiddleware(server, {
  context: async ({ req }) => ({
    user: await authenticate(req),
    loaders: createDataLoaders(),  // per-request DataLoaders
    db,
  }),
}));
```

---

## Apollo Federation (Microservices)

### The Problem
You have multiple teams owning separate services. Each has its own GraphQL schema. How do you expose a unified graph to clients?

```
Without Federation:
  Client → API Gateway → picks one service
  Each service has its own schema; no cross-service types

With Federation:
  Client → Apollo Router (gateway) → subgraphs
  Router merges schemas; resolves cross-service references
```

### Subgraph Example
```typescript
// Products Service (owns Product type)
import { buildSubgraphSchema } from '@apollo/subgraph';
import gql from 'graphql-tag';

const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key"])
  
  type Product @key(fields: "id") {
    id: ID!
    name: String!
    price: Float!
  }
  
  type Query {
    product(id: ID!): Product
    products: [Product!]!
  }
`;

const resolvers = {
  Product: {
    __resolveReference: (ref) => productDb.findById(ref.id),
  }
};

const schema = buildSubgraphSchema({ typeDefs, resolvers });
```

```typescript
// Orders Service (references Product from Products Service)
const typeDefs = gql`
  extend schema @link(url: "...", import: ["@key", "@external"])

  type Product @key(fields: "id") {
    id: ID!
    @external  # defined in Products Service
  }

  type OrderItem {
    product: Product!  # Router knows how to resolve this
    quantity: Int!
    price: Float!
  }

  type Order @key(fields: "id") {
    id: ID!
    items: [OrderItem!]!
    status: OrderStatus!
  }
`;
```

### Router (supergraph)
```yaml
# router.yaml
supergraph:
  subgraphs:
    products:
      routing_url: http://products-service:4001/graphql
    orders:
      routing_url: http://orders-service:4002/graphql
    users:
      routing_url: http://users-service:4003/graphql
```

The Router fetches the schemas (subgraph introspection), composes them into a supergraph, and executes queries by planning which subgraphs to call.

### Federation Query Planning
```
Query: { order(id: "1") { items { product { name price } } user { email } } }

Plan:
  1. Fetch order from Orders Service: { id, items { product { id } quantity }, userId }
  2. Fetch products in parallel from Products Service: { id, name, price } for each product id
  3. Fetch user from Users Service: { email } for userId
  4. Merge results, return to client
```

---

## Security in GraphQL

### Query Complexity & Depth Limiting
```typescript
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  schema,
  validationRules: [
    depthLimit(7),                              // max 7 levels of nesting
    createComplexityLimitRule(1000, {           // max complexity score 1000
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
    }),
  ],
});
```

### Persisted Queries (Production Hardening)
```
Instead of sending arbitrary query strings:
  Client: hash query → send { id: "sha256-abc123" }
  Server: look up hash → execute known query

Benefits:
  - Block ad-hoc queries (attackers can't send malicious queries)
  - Smaller request payloads
  - Better CDN caching for GET queries

Apollo: Automatic Persisted Queries (APQ) — falls back to full query on miss
```

### Authentication in Resolvers
```typescript
const resolvers = {
  Query: {
    myOrders: async (_, __, { user }) => {
      if (!user) throw new GraphQLError('Not authenticated', {
        extensions: { code: 'UNAUTHENTICATED' }
      });
      return orderService.findByUserId(user.id);
    }
  },
  Mutation: {
    deleteProduct: async (_, { id }, { user }) => {
      if (!user?.roles.includes('ADMIN')) throw new GraphQLError('Forbidden', {
        extensions: { code: 'FORBIDDEN' }
      });
      return productService.delete(id);
    }
  }
};

// Better: use a shield / directive for declarative auth
import { shield, rule, and } from 'graphql-shield';
const isAuthenticated = rule()((_, __, { user }) => !!user);
const isAdmin = rule()((_, __, { user }) => user?.roles.includes('ADMIN'));

const permissions = shield({
  Query: { myOrders: isAuthenticated },
  Mutation: { deleteProduct: and(isAuthenticated, isAdmin) },
});
```

---

## GraphQL Subscriptions (at Scale)

```typescript
// Server — graphql-ws (modern, replaces subscriptions-transport-ws)
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';

const wsServer = new WebSocketServer({ server: httpServer, path: '/graphql' });
useServer({
  schema,
  context: async (ctx) => ({
    user: await authenticateWs(ctx.connectionParams?.token),
  }),
}, wsServer);

// Resolver
const resolvers = {
  Subscription: {
    orderStatusChanged: {
      subscribe: async function* (_, { orderId }, { user, pubsub }) {
        if (!user) throw new GraphQLError('Not authenticated');
        
        for await (const event of pubsub.subscribe(`ORDER_STATUS:${orderId}`)) {
          if (event.userId === user.id) yield { orderStatusChanged: event.order };
        }
      }
    }
  }
};
```

### Subscriptions at Scale
```
Single server: in-memory EventEmitter works
Multi-server: need Redis pub/sub

graphql-redis-subscriptions:
  const pubsub = new RedisPubSub({ publisher: redisClient, subscriber: redisClient });
  pubsub.publish('ORDER_STATUS:abc123', { order: updatedOrder });
  pubsub.asyncIterator(['ORDER_STATUS:abc123']);

At large scale (10K+ subscribers):
  Consider: dedicated subscription servers, fan-out via Kafka
  Or: use Hasura / AppSync (managed subscription scaling)
```
