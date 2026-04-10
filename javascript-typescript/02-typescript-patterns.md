# JavaScript/TypeScript — Advanced Patterns & Fullstack Architecture

## Advanced TypeScript

### Utility Types (Internalize These)
```typescript
type Partial<T>       // all fields optional
type Required<T>      // all fields required
type Readonly<T>      // all fields readonly
type Pick<T, K>       // keep only K keys
type Omit<T, K>       // remove K keys
type Exclude<T, U>    // from union T, remove types assignable to U
type Extract<T, U>    // from union T, keep types assignable to U
type NonNullable<T>   // remove null and undefined
type ReturnType<T>    // return type of function
type Parameters<T>    // params tuple of function
type Awaited<T>       // unwrap Promise<T> to T
type Record<K, V>     // object with keys K and values V

// Deep examples:
type UpdateUserDto = Partial<Pick<User, 'name' | 'email' | 'avatar'>>;
type ApiHandler = (req: Request) => Promise<Response>;
type HandlerReturn = Awaited<ReturnType<ApiHandler>>; // Response
```

### Conditional Types
```typescript
type IsArray<T> = T extends any[] ? true : false;
type UnwrapArray<T> = T extends (infer Item)[] ? Item : T;

// Recursive conditional
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// Template literal types
type EventName = `on${Capitalize<string>}`;
type RouteParams<T extends string> = 
  T extends `${infer _}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof RouteParams<Rest>]: string }
    : T extends `${infer _}:${infer Param}`
    ? { [K in Param]: string }
    : {};
```

### Mapped Types
```typescript
// Make all methods async
type Asyncify<T> = {
  [K in keyof T]: T[K] extends (...args: infer A) => infer R
    ? (...args: A) => Promise<R>
    : T[K];
};

// Event handler map
type EventMap = { click: MouseEvent; keydown: KeyboardEvent; resize: UIEvent };
type EventHandlers = {
  [K in keyof EventMap as `on${Capitalize<K>}`]: (event: EventMap[K]) => void;
};
// { onClick: (e: MouseEvent) => void; onKeydown: ...; onResize: ... }
```

### Generic Constraints & Inference
```typescript
// Repository pattern with generics
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T>;
  update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Generic service with dependency injection
class BaseService<T extends { id: string }> {
  constructor(protected readonly repo: Repository<T>) {}

  async findOrFail(id: string): Promise<T> {
    const item = await this.repo.findById(id);
    if (!item) throw new NotFoundError('Entity', id);
    return item;
  }
}
```

---

## Design Patterns in TypeScript

### Dependency Injection
```typescript
// Without DI: hard to test, hard to swap
class OrderService {
  private db = new PostgresDB(); // tightly coupled
}

// With DI (interfaces + constructor injection)
interface IOrderRepository {
  create(order: CreateOrderDto): Promise<Order>;
  findById(id: string): Promise<Order | null>;
}

class OrderService {
  constructor(
    private readonly orderRepo: IOrderRepository,
    private readonly paymentService: IPaymentService,
    private readonly eventBus: IEventBus,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.orderRepo.create(dto);
    await this.eventBus.publish('order.created', order);
    return order;
  }
}

// In production: inject real implementations
// In tests: inject mocks/stubs
```

### Builder Pattern
```typescript
class QueryBuilder<T> {
  private filters: Partial<T>[] = [];
  private limitValue?: number;
  private orderField?: keyof T;

  where(filter: Partial<T>): this {
    this.filters.push(filter);
    return this;
  }

  limit(n: number): this {
    this.limitValue = n;
    return this;
  }

  orderBy(field: keyof T): this {
    this.orderField = field;
    return this;
  }

  async execute(): Promise<T[]> {
    return db.query({ filters: this.filters, limit: this.limitValue, order: this.orderField });
  }
}

// Usage
const users = await new QueryBuilder<User>()
  .where({ role: 'admin' })
  .where({ active: true })
  .orderBy('createdAt')
  .limit(10)
  .execute();
```

### Observer / Event Emitter
```typescript
type EventMap = Record<string, any>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners = new Map<string, Set<Function>>();

  on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): void {
    if (!this.listeners.has(event as string)) {
      this.listeners.set(event as string, new Set());
    }
    this.listeners.get(event as string)!.add(listener);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners.get(event as string)?.forEach(l => l(data));
  }

  off<K extends keyof Events>(event: K, listener: Function): void {
    this.listeners.get(event as string)?.delete(listener);
  }
}

type AppEvents = {
  'order:created': { orderId: string; userId: string };
  'payment:failed': { orderId: string; reason: string };
};

const bus = new TypedEventEmitter<AppEvents>();
bus.on('order:created', ({ orderId }) => console.log(orderId)); // typed!
```

---

## React Architecture (Principal Level)

### State Management Decision Tree
```
Local UI state (form, toggle, hover)?
  → useState / useReducer

Server state (fetched data, mutations)?
  → React Query / TanStack Query (best choice)
  → SWR (simpler alternative)

Global client state (user session, theme, feature flags)?
  → Zustand (lightweight) or Jotai (atomic)
  → Context only for stable, low-frequency data (theme, auth)
  → Redux only if you need time-travel debugging or existing codebase

Form state?
  → React Hook Form (best performance, minimal re-renders)
```

### React Query Patterns
```typescript
// Queries
const { data: user, isLoading, error } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => api.getUser(userId),
  staleTime: 5 * 60 * 1000,  // don't refetch for 5min
  retry: (count, error) => error.status !== 404 && count < 3,
});

// Mutations with optimistic update
const mutation = useMutation({
  mutationFn: (data: UpdateUserDto) => api.updateUser(userId, data),
  onMutate: async (data) => {
    await queryClient.cancelQueries({ queryKey: ['user', userId] });
    const previous = queryClient.getQueryData(['user', userId]);
    queryClient.setQueryData(['user', userId], old => ({ ...old, ...data }));
    return { previous };
  },
  onError: (err, vars, ctx) => {
    queryClient.setQueryData(['user', userId], ctx?.previous);
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['user', userId] }),
});
```

### Performance Optimization
```typescript
// Memoization — use sparingly, measure first
const ExpensiveList = memo(({ items, onSelect }) => {
  // Only re-renders if items or onSelect reference changes
  return items.map(i => <Item key={i.id} item={i} onSelect={onSelect} />);
});

// Stable callbacks
const handleSelect = useCallback((id: string) => {
  dispatch({ type: 'SELECT', id });
}, [dispatch]); // dispatch from useReducer is stable

// Derived state — compute, don't store
const filteredItems = useMemo(
  () => items.filter(i => i.status === filter),
  [items, filter]
);

// Virtual lists for 1000+ rows
import { FixedSizeList } from 'react-window';
```

### Code Splitting
```typescript
// Route-level splitting
const OrdersPage = lazy(() => import('./pages/OrdersPage'));
const AdminPage  = lazy(() => import('./pages/AdminPage'));

// Component-level splitting (heavy components)
const RichTextEditor = lazy(() => import('./components/RichTextEditor'));

// Preload on hover
const preloadOrdersPage = () => import('./pages/OrdersPage');
<Link onMouseEnter={preloadOrdersPage} to="/orders">Orders</Link>
```

---

## Node.js API Architecture Patterns

### Layered Architecture
```
HTTP Layer    (Express/Fastify routes — parse, validate, respond)
     ↓
Service Layer (business logic — no HTTP/DB knowledge)
     ↓
Repository Layer (data access — DB queries, cache interaction)
     ↓
Infrastructure (DB connection, Redis, external APIs)
```

```typescript
// Route (thin — validate + delegate)
router.post('/orders', authenticate, async (req, res, next) => {
  try {
    const dto = CreateOrderSchema.parse(req.body);
    const order = await orderService.create(req.user.id, dto);
    res.status(201).json(order);
  } catch (e) { next(e); }
});

// Service (business logic — no req/res)
class OrderService {
  async create(userId: string, dto: CreateOrderDto): Promise<Order> {
    const inventory = await this.inventoryRepo.reserve(dto.items);
    const order = await this.orderRepo.create({ userId, ...dto });
    await this.eventBus.publish('order.created', order);
    return order;
  }
}

// Repository (pure data access)
class OrderRepository implements IOrderRepository {
  async create(data: NewOrder): Promise<Order> {
    const result = await this.db.query(
      'INSERT INTO orders(user_id, status, ...) VALUES($1, $2, ...) RETURNING *',
      [data.userId, 'pending', ...]
    );
    return toDomain(result.rows[0]);
  }
}
```

### Fastify vs Express (Know the Difference)
| | Express | Fastify |
|---|---|---|
| Performance | ~15k req/s | ~45k req/s |
| Schema validation | Manual (Zod/Joi) | Built-in (AJV, JSON Schema) |
| TypeScript | Good (types package) | Native (first-class) |
| Plugins | Middleware pattern | Plugin system (encapsulation) |
| Ecosystem | Enormous | Growing |
| Choose when | Familiarity, existing codebase | New project, performance matters |

### Middleware Pattern in Express
```typescript
// Error handling middleware (must be last, 4 params)
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError && err.isOperational) {
    return res.status(err.statusCode).json({
      error: { code: err.code, message: err.message },
      requestId: req.id,
    });
  }
  // Programmer error — log + crash
  logger.error('Unexpected error', { err, requestId: req.id });
  res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: 'Internal server error' } });
  process.exit(1); // Let supervisor restart
});
```

---

## CI/CD Pipeline Architecture

```yaml
# .github/workflows/deploy.yml (conceptual)
# 1. Lint + Type Check
# 2. Unit Tests (fast)
# 3. Integration Tests (testcontainers, isolated)
# 4. Build Docker image
# 5. Push to ECR
# 6. Deploy to staging (ECS/EKS)
# 7. Smoke tests against staging
# 8. Manual approval gate (prod)
# 9. Deploy to prod (blue/green or canary)
# 10. Health check → rollback on failure
```

### Quality Gates
- **Branch protection**: All tests must pass, required reviewers
- **Coverage threshold**: >80% lines (not metric to game, metric to trust)
- **Dependency scanning**: npm audit, Snyk, GitHub Dependabot
- **SAST**: SonarQube, CodeQL (GitHub)
- **Container scanning**: ECR image scan (Inspector), Trivy
- **Secrets scanning**: git-secrets, TruffleHog in pre-commit + CI
