# Engineering Leadership — DevOps & Delivery

## Docker Best Practices for Node.js

### Production-Grade Dockerfile
```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && cp -R node_modules /prod_modules
RUN npm ci  # install all deps for build

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Production image (minimal)
FROM node:20-alpine AS runner
RUN addgroup -g 1001 -S nodejs && adduser -S nodeapp -u 1001
WORKDIR /app

# Copy only what's needed
COPY --from=deps    /prod_modules     ./node_modules
COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/package.json ./package.json

USER nodeapp
EXPOSE 8080
ENV NODE_ENV=production

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O /dev/null http://localhost:8080/health/live || exit 1

CMD ["node", "dist/index.js"]
```

### Why Each Decision
```
node:20-alpine:         Smaller base (~50MB vs ~200MB for full node)
Multi-stage build:      Final image has no dev tools, TypeScript compiler, source maps
npm ci:                 Exact versions from lockfile (not npm install)
Non-root user:          Container runs as UID 1001; prevents privilege escalation
HEALTHCHECK:            Orchestrators (ECS/K8s) know when container is unhealthy
NODE_ENV=production:    Disables dev middleware, enables optimizations
COPY specific files:    Don't COPY . . in final stage (leaks .env, .git, etc.)
```

### .dockerignore
```
node_modules
.git
.env*
*.test.ts
*.spec.ts
coverage
dist
.nyc_output
```

---

## Zero-Downtime Deployment Strategies

### Rolling Deployment
```
v1 v1 v1 v1  →  v2 v1 v1 v1  →  v2 v2 v1 v1  →  v2 v2 v2 v2
```
- Replace instances one at a time (or N at a time)
- Both versions run simultaneously during rollout
- **Risk**: Mixed versions serving traffic — APIs must be backward compatible
- ECS default: `minimum_healthy_percent=50, maximum_percent=200`

### Blue/Green Deployment
```
[ALB] → 100% → [Blue: v1]
               [Green: v2] ← deploy here first

After verification:
[ALB] → 100% → [Green: v2]
               [Blue: v1]  ← keep warm for instant rollback
```
- Zero mixed-version traffic; instant rollback
- Cost: double capacity during cutover
- AWS CodeDeploy + ECS or ALB weighted target groups

### Canary Deployment
```
[ALB] → 95% → [v1 fleet]
         5% → [v2 canary]  ← monitor error rates, latency
After 30min good metrics → 100% → [v2 fleet]
```
- Real traffic on new version; limited blast radius
- AWS CodeDeploy traffic shifting or ALB weighted routing
- **Essential**: alarm on error rate; auto-rollback if breached

### Feature Flags (Deploy without Releasing)
```typescript
// Code deployed to prod but feature is off
if (await featureFlags.isEnabled('new-checkout-flow', { userId })) {
  return newCheckoutFlow(cart);
} else {
  return legacyCheckoutFlow(cart);
}

// Benefits:
// - Decouple deploy from release
// - Gradual rollout: 1% → 10% → 50% → 100%
// - A/B testing
// - Kill switch without re-deploy
// - Ring-based rollout: internal users → beta → all

// AWS AppConfig, LaunchDarkly, Unleash (open source), Flagsmith
```

---

## Database Migrations (Zero-Downtime)

This is a Principal-level topic that trips up many engineers.

### The Challenge
```
Migration runs BEFORE new code deploys (usually)
Both old code and new code run simultaneously during deploy
Migration must be safe for old code to run against
```

### The Expand-Contract Pattern (3-phase migration)

**Adding a column (non-breaking)**
```
Phase 1 (Expand): Add new nullable column (old code ignores it, new code writes it)
  ALTER TABLE orders ADD COLUMN tracking_url TEXT;

Phase 2 (Migrate): Backfill existing rows; new code reads new column
  UPDATE orders SET tracking_url = ... WHERE tracking_url IS NULL;

Phase 3 (Contract): Add NOT NULL constraint once backfill complete
  ALTER TABLE orders ALTER COLUMN tracking_url SET NOT NULL;
  (safe now — all rows have the value)
```

**Renaming a column (3 deploys)**
```
Deploy 1:
  Add new column: ALTER TABLE users ADD COLUMN full_name TEXT;
  Write to both old (name) and new (full_name) columns
  Read from old column

Deploy 2:
  Backfill new column from old
  Read from new column; still write to both

Deploy 3:
  Stop writing to old column
  Drop old column: ALTER TABLE users DROP COLUMN name;
```

**Adding a NOT NULL column**
```
❌ Never: ALTER TABLE orders ADD COLUMN region TEXT NOT NULL;
   → Fails if table has existing rows (no default) or locks table during backfill

✓ Correct:
  1. ADD COLUMN region TEXT;                              -- nullable, instant
  2. UPDATE orders SET region = 'us-east-1' WHERE ...;   -- backfill in batches
  3. ALTER TABLE orders ALTER COLUMN region SET NOT NULL; -- safe after backfill
```

**Adding an index (non-blocking)**
```sql
-- Bad: locks table for entire index build (minutes on large tables)
CREATE INDEX ON orders(user_id);

-- Good: builds concurrently, doesn't lock writes
CREATE INDEX CONCURRENTLY ON orders(user_id);
-- Takes longer but safe for production
```

### Migration Tools
- **Flyway** (Java-based, works with any DB): versioned SQL files, checksum verification
- **Liquibase**: XML/YAML/JSON/SQL changelogs, more flexible
- **db-migrate** / **knex migrations** (Node.js): JS-based migrations
- **Prisma Migrate** (TypeScript): schema-first; generates migrations automatically
- **Rule**: migrations are **irreversible** (never edit a committed migration); always write a down migration for rollback

---

## GitHub Actions — CI/CD Pipeline Design

### Node.js Monorepo Pipeline
```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Fast feedback — run in parallel
  lint-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }

  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: testdb }
        options: >-
          --health-cmd pg_isready --health-interval 10s
      redis:
        image: redis:7-alpine
        options: --health-cmd "redis-cli ping"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  build-push:
    needs: [lint-typecheck, unit-test, integration-test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # OIDC for AWS auth (no stored secrets!)
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
      - name: Build and push
        run: |
          IMAGE=${{ env.ECR_REGISTRY }}/my-app:${{ github.sha }}
          docker build -t $IMAGE .
          docker push $IMAGE
          echo "IMAGE=$IMAGE" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build-push
    environment: staging  # requires manual approval if configured
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster staging \
            --service my-app \
            --force-new-deployment

  deploy-prod:
    needs: deploy-staging
    environment: production  # REQUIRED manual approval
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          aws ecs update-service --cluster prod --service my-app --force-new-deployment
```

### Security Best Practices in CI/CD
```
✓ Use OIDC federation (no stored AWS secrets in GitHub)
  → GitHub gets short-lived token via AWS trust policy
✓ Pin action versions with SHA: actions/checkout@8ade135...
  → Prevents supply chain attacks (malicious action updates)
✓ Least privilege IAM role for CI (deploy only to specific ECS service)
✓ Secrets in GitHub Environments (not repository secrets where possible)
✓ Dependency scanning: Dependabot or Renovate for automated PRs
✓ SAST: CodeQL workflow on PRs
✓ Container scanning: Trivy or ECR scan before deploy
```

---

## Infrastructure as Code (IaC) — CDK vs Terraform

### When to Choose CDK (TypeScript)
```typescript
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';

class ApiStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2 });

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    const taskDef = new ecs.FargateTaskDefinition(this, 'Task', {
      memoryLimitMiB: 1024,
      cpu: 512,
    });

    taskDef.addContainer('Api', {
      image: ecs.ContainerImage.fromEcrRepository(repo, 'latest'),
      portMappings: [{ containerPort: 8080 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'api' }),
      secrets: {
        DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password'),
      },
    });

    new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 2,
      assignPublicIp: false,
    });
  }
}
```

- **Choose CDK**: AWS-only, TypeScript team, want L2 constructs (sensible defaults), CDK Pipelines
- **Choose Terraform**: Multi-cloud, existing Terraform state, team familiar with HCL, better multi-region patterns

### CDK Best Practices
```typescript
// Use environment-specific stacks
new ApiStack(app, 'ApiStack-Prod', {
  env: { account: '111111111111', region: 'us-east-1' }
});

// Separate stateful (DB) from stateless (compute) stacks
// Stateful stacks: termination protection, retain on delete
// Stateless stacks: replaceable at any time

// Use context for environment config, not hardcoded values
const isProd = app.node.tryGetContext('environment') === 'prod';

// Always set removal policies on stateful resources
const bucket = new s3.Bucket(this, 'Bucket', {
  removalPolicy: isProd ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY,
});
```

---

## DORA Metrics (How to Measure Delivery Performance)

The industry-standard metrics for software delivery:

| Metric | Elite | High | Medium | Low |
|---|---|---|---|---|
| **Deployment Frequency** | Multiple/day | Weekly | Monthly | <6/year |
| **Lead Time for Changes** | <1 hour | 1day–1wk | 1wk–1mo | >1 month |
| **Change Failure Rate** | <5% | 5–10% | 10–15% | >15% |
| **MTTR** | <1 hour | <1 day | 1day–1wk | >1 week |

### Using DORA in Interviews
- "What's your current deployment frequency?" signals technical maturity
- If they don't know: "How do you currently measure delivery performance?" — tells you about culture
- Your prep angle: "At my last role, I helped move from monthly to weekly deployments by [X]. We tracked it with [Y]."

### Common Blockers to High Performance
```
Low deployment frequency → monolithic architecture, manual steps, lack of test coverage
High MTTR              → no automated alerting, poor observability, no runbooks
High change failure rate → insufficient testing, no canary/feature flags, large batch sizes
Long lead time         → too many handoffs, approval bottlenecks, environment provisioning lag
```
