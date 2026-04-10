# AWS — Well-Architected Framework & Architecture Patterns

## The 6 Pillars

### 1. Operational Excellence
> Running and monitoring systems to deliver business value

Key practices:
- **Infrastructure as Code**: CloudFormation, CDK, Terraform — no manual console changes
- **CI/CD pipeline**: Automate build → test → deploy; enable fast, safe releases
- **Runbooks & playbooks**: Documented operational procedures; runbook automation via Systems Manager
- **Observability**: CloudWatch metrics/logs/traces; dashboards for each service
- **Event-driven operations**: Automate responses to operational events
- **Annotate events**: Mark deployments on dashboards; correlate issues with releases

Questions interviewers ask:
- "How do you deploy changes with zero downtime?"
- "How do you know your system is healthy?"
- "What happens when an on-call engineer gets paged at 2am?"

---

### 2. Security
> Protecting information and systems

**Identity**: IAM roles (not users), least privilege, MFA on root, SSO for humans
**Detective Controls**: CloudTrail (who did what), GuardDuty (threat detection), Config (compliance)
**Infrastructure Protection**: VPC, SGs, WAF, Shield, NACLs
**Data Protection**: Encryption at rest (KMS), in transit (TLS 1.2+), tokenization
**Incident Response**: IR runbooks, automated quarantine (GuardDuty → Lambda → isolate instance)

**Shared Responsibility Model**:
```
AWS manages: Physical hardware, network, hypervisor, managed service software
You manage:  OS patches (EC2), app code, IAM, data encryption, network config

Managed services (RDS, Lambda):
  AWS manages more (OS, runtime patches)
  You manage: Data, access controls, app config
```

---

### 3. Reliability
> Ability to recover from failures and dynamically acquire resources

**Design principles**:
- Test recovery procedures (chaos engineering, Game Days)
- Auto-recover from failure (ASG, multi-AZ, health checks)
- Scale horizontally; eliminate single points of failure
- Stop guessing capacity (auto-scaling)
- Manage change through automation (IaC)

**Key patterns**:
- Multi-AZ deployments for all stateful services
- Cross-region DR for critical services
- Backups + tested restore procedures (RTO/RPO defined)
- Circuit breakers for external dependencies

**RTO vs RPO**:
```
RPO (Recovery Point Objective): Maximum acceptable data loss
  e.g., RPO = 1hr → can lose at most 1hr of data → backups every hour

RTO (Recovery Time Objective): Maximum acceptable downtime
  e.g., RTO = 15min → must be back up within 15min of incident

More aggressive RPO/RTO = more expensive infrastructure
```

**DR Strategies (cost vs speed)**:
```
Backup & Restore (cheapest): Backup to S3; restore from scratch when needed
  RPO: hours, RTO: hours

Pilot Light: Core infrastructure running minimally in DR region; scale up when needed
  RPO: minutes, RTO: tens of minutes

Warm Standby: Smaller but functional copy running in DR region; scale up on failover
  RPO: seconds-minutes, RTO: minutes

Active-Active (multi-region): Both regions serving traffic; instant failover
  RPO: 0, RTO: seconds (most expensive)
```

---

### 4. Performance Efficiency
> Using computing resources efficiently

- Choose the right resource type (Lambda vs EC2 vs Fargate; gp3 vs io2 EBS)
- Use serverless/managed services to offload undifferentiated heavy lifting
- Global deployment with CloudFront, Global Accelerator, Aurora Global Database
- Experiment with advanced technologies (GPU instances for ML, Graviton for cost/perf)

**Performance testing**: Load test in staging before production; use AWS Distributed Load Testing

---

### 5. Cost Optimization
> Running systems at lowest price point

- **Right-size before committing** to Reserved/Savings Plans
- Use Spot for batch and fault-tolerant workloads
- **Delete unused resources**: Idle EC2, unattached EBS, old snapshots, unused Elastic IPs
- Serverless for intermittent workloads (Lambda, Fargate)
- S3 lifecycle policies, Glacier for archive
- **Data transfer**: Minimize cross-AZ/cross-region; use VPC endpoints for AWS services
- **AWS Cost Explorer**: Analyze spend; identify anomalies
- **Compute Savings Plans**: 1–3yr commitment on EC2/Lambda/Fargate; ~35–66% savings

---

### 6. Sustainability
> Minimizing environmental impact

- Right-size to reduce idle capacity
- Managed services have better utilization than single-tenant EC2
- Graviton (ARM) processors: better performance/watt
- Use serverless for spiky workloads (no idle compute)
- Multi-tenant architectures over single-tenant

---

## Architecture Patterns on AWS

### Serverless Web Application
```
Users
  → CloudFront (CDN + WAF)
  → S3 (static assets: React build)
  → API Gateway (HTTP API)
  → Lambda (Node.js/TypeScript handlers)
  → DynamoDB (data) + ElastiCache (cache)
  → Cognito (auth)
  → EventBridge (async events)
  → SQS → Lambda (background processing)
```
- Zero infrastructure management
- Auto-scales from 0 to millions
- Pay only for actual usage
- Cold starts: mitigate with Provisioned Concurrency on critical paths

### Container-Based Microservices
```
Route 53 → ALB → ECS Fargate (multiple services)
               → Aurora PostgreSQL (Multi-AZ)
               → ElastiCache Redis
               → MSK (Managed Kafka) / SQS
               → ECR (images)
               → CodePipeline → CodeBuild → ECR → ECS
```
- More control than Lambda (long-running processes, custom runtimes)
- Blue/green deployments with CodeDeploy
- Service discovery via Cloud Map or ALB routing

### Event-Driven Architecture
```
API Gateway/Lambda → EventBridge → [
  Lambda (process order)
  → SQS DLQ (failed events)
  
  Lambda (send email) → SES
  
  Kinesis Firehose → S3 → Athena (analytics)
  
  Step Functions (complex workflow)
]
```

### Data Lake Architecture
```
Sources: RDS, DynamoDB, S3, Kinesis (streaming)
  ↓
  S3 Raw Zone (landing, immutable)
  ↓ (Glue ETL jobs)
  S3 Curated Zone (cleaned, transformed, Parquet)
  ↓
  Glue Data Catalog (metadata)
  ↓
  Query: Athena (ad-hoc SQL), Redshift Spectrum (BI), EMR (Spark)
  
Stream processing:
  Kinesis Data Streams → Lambda/Flink → Kinesis Firehose → S3
```

---

## CDK (Cloud Development Kit) — Principal-Level Awareness

Write infrastructure in TypeScript/Python; synthesizes to CloudFormation.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class MyStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2 });
    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });
    
    new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: new ecs.FargateTaskDefinition(this, 'TaskDef'),
    });
  }
}
```

- **Constructs**: L1 (raw CloudFormation), L2 (high-level, sensible defaults), L3 (patterns)
- **CDK Aspects**: Apply rules across entire app (e.g., enforce tagging, encryption)
- **CDK Pipelines**: Self-mutating CDK pipeline (deploys itself, then your app)
- Prefer CDK for new greenfield; Terraform for multi-cloud or existing Terraform state

---

## Common Architecture Interview Questions & Answers

**"How would you architect a highly available Node.js API on AWS?"**
```
- ECS Fargate with multiple tasks across 2+ AZs
- ALB in front with health checks
- Aurora PostgreSQL Multi-AZ (auto-failover)
- ElastiCache Redis (cluster mode) for sessions/caching
- Secrets Manager for DB credentials; rotation configured
- CloudWatch alarms → SNS → PagerDuty
- CodePipeline: GitHub → CodeBuild → ECR → ECS blue/green
- WAF on ALB; Shield Standard; VPC with private subnets for services
```

**"How do you handle a production incident on AWS?"**
```
1. Detect: CloudWatch alarm triggers → PagerDuty page
2. Triage: Check CloudWatch dashboard; X-Ray service map for error hotspot
3. Mitigate: Route 53 failover / increase desired count / roll back deployment
4. Diagnose: CloudWatch Log Insights query; Contributor Insights for top offenders
5. Fix: Patch + deploy via pipeline or hotfix
6. Post-mortem: Document timeline, root cause, action items (blameless)
```

**"How do you reduce Lambda cold start latency for a customer-facing API?"**
```
1. Minimize bundle size (tree-shaking, avoid v2 SDK)
2. Use Provisioned Concurrency on critical paths (auth, checkout)
3. Keep Lambda in VPC only if needed (now fast, but still adds ~1-2ms)
4. Move Lambda initialisation outside handler (DB connections, config loading)
5. Node.js is fastest runtime for cold starts
6. Use Lambda@Edge or CloudFront Functions for auth/routing at edge
```
