# AWS — Serverless & Containers

## Lambda

### How Lambda Works
```
Invocation → Cold Start (if no warm instance)
           → Container init: download code, start runtime
           → Execution environment reuse (warm start)
           → Function runs → return result → environment idle (up to 15min)
           → Environment frozen/recycled
```

### Cold Start Factors & Mitigation
- **What affects cold start**: Runtime (Node.js/Python fastest, Java/C# slowest), bundle size, VPC attachment
- **VPC cold start**: Was ~10s due to ENI creation; now uses hyperplane ENIs — much faster (<1s)
- **Mitigation**:
  - Provisioned Concurrency: Keep N instances warm (pay for idle time)
  - SnapStart (Java): Snapshot after init, restore from snapshot
  - Minimize bundle size: tree-shaking, avoid large AWS SDK v2 (use v3 modular)
  - Keep Lambda warm via scheduled ping (hack; Provisioned Concurrency is better)

### Lambda Configuration
```
Memory:   128MB to 10GB (also scales CPU proportionally)
Timeout:  15 min max (use Step Functions for longer workflows)
Disk:     512MB–10GB ephemeral /tmp
Concurrency: 1000/region default (soft limit); 1 concurrent exec per request
```

### Concurrency Model
```
Reserved Concurrency:
  - Guarantee N concurrent for this function
  - Also acts as throttle (prevent runaway)

Provisioned Concurrency:
  - Pre-warm N instances
  - No cold starts for those N

Unreserved Concurrency:
  - Shares the 1000 account limit
  - First-come-first-served
```

### Lambda Triggers (Event Sources)
| Source | Invocation | Notes |
|---|---|---|
| API Gateway / ALB | Sync | HTTP request-response |
| S3 | Async | Object events; retry on failure |
| SQS | Polling | Batch size configurable; DLQ for failures |
| SNS | Async | Fan-out pattern |
| Kinesis | Polling | Ordered; each shard = 1 concurrent Lambda |
| DynamoDB Streams | Polling | Like Kinesis |
| EventBridge | Async | Rule-based event routing |
| Step Functions | Sync | Orchestration |

### Lambda + SQS Pattern (Reliable Async)
```
Producer → SQS → Lambda (batch 10)
  │
  └── DLQ (after N retries)
      └── Lambda (inspect + alert)

Configure:
  visibility_timeout = 6 × function_timeout (prevent double-processing)
  max_receive_count = 3 (before DLQ)
  batch_item_failure: return failed items → only those requeued
```

### Node.js Lambda Best Practices
```typescript
// Initialize expensive resources OUTSIDE handler (reused across warm invocations)
const dbClient = new DynamoDBClient({});
const secretsClient = new SecretsManagerClient({});
let cachedDbPassword: string | null = null;

export const handler = async (event: APIGatewayProxyEvent) => {
  // Lazy-load and cache secrets
  if (!cachedDbPassword) {
    const secret = await secretsClient.send(new GetSecretValueCommand({ SecretId: 'db/password' }));
    cachedDbPassword = secret.SecretString!;
  }
  
  // Use Power Tools for Lambda
  // - Logger with correlation IDs
  // - Tracer for X-Ray
  // - Metrics for CloudWatch EMF
};
```

### Lambda Power Tools (TypeScript/Python)
- Structured logging with correlation IDs
- Metrics using Embedded Metric Format (CloudWatch)
- X-Ray tracing
- Idempotency utility
- Event source data classes

---

## Step Functions

### When to Use
- Orchestrate multi-step workflows with branching, retry, parallel, wait
- Long-running processes (beyond Lambda's 15min)
- Human-in-the-loop workflows (`.waitForTaskToken`)
- Replace complex Lambda chains with explicit state machines

### Workflow Types
- **Standard**: Up to 1 year; exactly-once execution; audit history; pay per state transition
- **Express**: Up to 5 min; at-least-once; pay per execution duration; high-throughput

### State Types
```json
{
  "ProcessOrder": { "Type": "Task", "Resource": "arn:aws:lambda:...", "Next": "CheckPayment" },
  "CheckPayment": { "Type": "Choice",
    "Choices": [{ "Variable": "$.status", "StringEquals": "approved", "Next": "FulfillOrder" }],
    "Default": "RejectOrder"
  },
  "SendNotifications": { "Type": "Parallel",
    "Branches": [
      { "StartAt": "EmailStep", ... },
      { "StartAt": "SMSStep", ... }
    ]
  },
  "WaitForApproval": { "Type": "Task",
    "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
    "Parameters": { "QueueUrl": "...", "MessageBody": { "TaskToken.$": "$$.Task.Token" } },
    "HeartbeatSeconds": 3600
  }
}
```

---

## API Gateway

### Types
- **HTTP API**: Cheaper (~70% less), lower latency, supports JWT authorizers, Lambda, HTTP backends
- **REST API**: Full features: usage plans, API keys, request/response transformation, AWS service integrations
- **WebSocket API**: Persistent bidirectional connections (chat, gaming, live dashboards)

### REST API Features
- **Stages**: `dev`, `staging`, `prod`; stage variables for environment-specific config
- **Usage Plans + API Keys**: Rate limit + quota per customer
- **Authorizers**:
  - Lambda authorizer: Custom auth logic → return IAM policy
  - Cognito authorizer: Validate Cognito JWT
  - JWT authorizer (HTTP API): Validate any OIDC-compatible JWT
- **Request/Response Mapping**: Transform payloads without Lambda (Velocity Templates)
- **Models + Validation**: Validate request body against JSON Schema before Lambda runs

---

## ECS (Elastic Container Service)

### Launch Types
- **Fargate**: Serverless containers; no EC2 management; pay per vCPU+memory per second
- **EC2**: You manage instances; better for GPU workloads, custom AMIs, cost predictability at scale
- **ECS Anywhere**: Run ECS tasks on your own on-prem hardware

### Task Definition
```json
{
  "family": "my-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::...role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::...role/myAppRole",
  "containerDefinitions": [{
    "name": "api",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest",
    "portMappings": [{ "containerPort": 8080 }],
    "secrets": [{ "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:..." }],
    "logConfiguration": { "logDriver": "awslogs", "options": { "awslogs-group": "/ecs/my-api" } },
    "healthCheck": { "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"] }
  }]
}
```

### ECS Service Auto Scaling
```
Target Tracking: ALBRequestCountPerTarget = 100 req/target
Step Scaling: CPU > 70% → add 2 tasks; CPU < 30% → remove 1 task
Scheduled: Scale up at 8am, down at 8pm
```

### Blue/Green Deployments with ECS + CodeDeploy
```
Blue: current production traffic
Green: new version deployed

CodeDeploy shifts traffic:
  Canary: 10% to green → wait → 100% to green
  Linear: 10% every 5min
  All-at-once: immediate

Rollback: shift all traffic back to blue in seconds
```

---

## EKS (Elastic Kubernetes Service)

### When EKS over ECS
- Multi-cloud portability requirement
- Complex orchestration needs (custom schedulers, complex networking)
- Team already has Kubernetes expertise
- Advanced features: custom CRDs, Operators, Helm charts ecosystem

### EKS Node Types
- **Managed Node Groups**: AWS provisions, patches, terminates EC2 for you
- **Fargate Profiles**: Serverless; each pod gets its own Fargate VM (isolation)
- **Self-managed**: Full control; more ops burden

### Key EKS Integrations
- **ALB Ingress Controller**: Create ALB from Ingress resources
- **EBS/EFS CSI Driver**: Persistent volumes on EBS/EFS
- **IRSA (IAM Roles for Service Accounts)**: Fine-grained IAM per pod (not node-level)
- **Cluster Autoscaler** / **Karpenter**: Auto-scale nodes based on pending pods (Karpenter is faster, more flexible)
- **AWS Load Balancer Controller**: Manage ALB/NLB from Kubernetes

---

## ECR (Elastic Container Registry)

- Private Docker registry; IAM-controlled
- **Image scanning**: Basic (CVE on push) and Enhanced (Inspector, continuous)
- **Lifecycle policies**: Auto-delete old images (e.g., keep last 10 tags)
- **Cross-region/account replication**: Replicate images for DR or multi-account pipelines
- **Pull-through cache**: Cache Docker Hub images in ECR (avoid rate limits)

---

## EventBridge

- Serverless event bus; route events between services
- **Sources**: AWS services, custom applications, SaaS (Datadog, Zendesk, PagerDuty)
- **Rules**: Pattern match on event JSON → route to target (Lambda, SQS, Step Functions, API Gateway)
- **Event Buses**: Default (AWS), custom, partner
- **Scheduler**: Cron or rate expressions; one-time or recurring; replaces CloudWatch Events cron
- **Pipes**: Point-to-point between source (SQS, Kinesis, DynamoDB) → optional enrichment (Lambda) → target

### EventBridge vs SNS vs SQS
| | EventBridge | SNS | SQS |
|---|---|---|---|
| Model | Event routing (content-based filter) | Fan-out pub/sub | Work queue |
| Filtering | Rich JSON pattern matching | Attribute filtering | Message visibility |
| Targets | 20+ AWS services | SQS, Lambda, HTTP, email | Lambda, EC2, ECS |
| Retention | None (fire-and-forget) | None | Up to 14 days |
| Ordering | No | No (FIFO available) | FIFO available |
| Use for | Complex event routing, SaaS integration | Notifications, fan-out | Async task processing |
