# AWS — Core Services

## Compute

### EC2
- **Instance families**: `t` (burstable), `m` (general), `c` (compute), `r` (memory), `g` (GPU)
- **Purchasing options**:
  - On-Demand: pay per hour, no commitment. Use for: unpredictable, short workloads
  - Reserved (1–3yr): 40–72% savings. Use for: baseline steady-state workloads
  - Spot: up to 90% savings, but can be interrupted with 2-min notice. Use for: fault-tolerant batch, ML training
  - Savings Plans: flexible commitment ($/hour), applies across instance families
- **Auto Scaling Group (ASG)**: Define min/max/desired; scale on CloudWatch metrics or schedules
- **Launch Template**: Defines AMI, instance type, SG, IAM role, user data script

### EC2 Networking
- **VPC**: Your isolated virtual network
- **Security Group**: Stateful firewall at instance level (allow rules only)
- **NACL**: Stateless firewall at subnet level (allow + deny rules, order matters)
- **Placement Group**: `Cluster` (low latency, same AZ), `Spread` (HA across hardware), `Partition` (Kafka/Hadoop style)
- **Elastic IP**: Static public IP; pay for unattached IPs
- **ENI**: Elastic Network Interface — attach multiple network interfaces to EC2

---

## Storage

### S3
- **Storage classes** (pick based on access frequency + retrieval time):

| Class | Access | Cost | Retrieval |
|---|---|---|---|
| Standard | Frequent | Highest | Instant |
| Intelligent-Tiering | Unknown pattern | Medium | Instant |
| Standard-IA | Infrequent | Lower storage, retrieval fee | Instant |
| One Zone-IA | Infrequent, less critical | Cheapest IA | Instant |
| Glacier Instant | Rare but immediate | Lower | Instant |
| Glacier Flexible | Rare | Low | Minutes–hours |
| Glacier Deep Archive | Archive | Cheapest | 12–48hrs |

- **S3 lifecycle policies**: Auto-transition objects between classes; auto-expire
- **Versioning**: Keep all versions; use with MFA delete for compliance
- **Replication**: Cross-Region (CRR) or Same-Region (SRR); requires versioning
- **S3 Transfer Acceleration**: Uses CloudFront edge locations for faster uploads
- **Multipart Upload**: Required >5GB; recommended >100MB; parallel upload
- **Pre-signed URLs**: Temporary access URL (e.g., upload directly from client, expire in N sec)
- **S3 Event Notifications**: Trigger Lambda/SQS/SNS on object create/delete

### EBS (Elastic Block Store)
- Block storage attached to EC2; single AZ; persists beyond instance life
- **Types**: `gp3` (general, 3000 IOPS baseline), `io2` (provisioned IOPS, databases), `st1` (throughput HDD), `sc1` (cold HDD)
- **Snapshots**: Incremental, stored in S3; use for backups + creating AMIs
- Multi-attach: `io2` supports attaching to multiple EC2 (for clustered apps)

### EFS (Elastic File System)
- Managed NFS; shared across multiple EC2/Lambda; auto-scales
- Multi-AZ; more expensive than EBS
- Use for: shared config files, CMS assets, ML training data, container volumes

---

## Databases

### RDS
- Managed relational DB: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server
- **Multi-AZ**: Synchronous standby in another AZ; auto-failover (~60s)
- **Read Replicas**: Async replication; up to 5 per DB; can be cross-region
- **RDS Proxy**: Connection pooling + IAM auth; reduces DB connection overhead
- **Automated backups**: 1–35 days PITR (Point-in-Time Recovery)
- **Enhanced Monitoring**: OS-level metrics (CPU, memory, disk)

### Aurora
- MySQL/PostgreSQL-compatible; AWS-built engine
- **6 copies of data across 3 AZs** (2 per AZ) — automatic, always-on
- **Up to 15 read replicas** with <10ms lag
- **Aurora Serverless v2**: Scale compute in 0.5 ACU increments; never idle
- **Global Database**: Primary region + up to 5 secondary regions; <1s cross-region replication
- **Aurora vs RDS**: Aurora is ~5x faster than MySQL; more resilient; pay more for compute but great for production

### DynamoDB
- Serverless, fully managed, key-value + document
- **On-Demand**: Auto-scale, pay per request — for unpredictable workloads
- **Provisioned + Auto Scaling**: Set RCU/WCU ranges; cheaper for predictable traffic
- **DynamoDB Streams**: Change data capture → trigger Lambda → real-time processing
- **TTL**: Automatically delete items after a timestamp attribute expires
- **Transactions**: `TransactWriteItems` / `TransactGetItems` — up to 100 items, atomic
- **DAX**: DynamoDB Accelerator — in-memory cache, <1ms reads, API-compatible

### ElastiCache
- Managed Redis or Memcached
- **Redis use cases**: Caching, sessions, rate limiting, leaderboards, pub/sub, Lua scripting
- **Redis Cluster Mode**: Horizontal sharding; each shard is primary + replica pair
- **Memcached**: Multi-threaded, simple key-value, no persistence — pure caching

---

## Networking

### VPC Architecture Best Practices
```
VPC: 10.0.0.0/16

Public Subnets (per AZ):
  10.0.1.0/24 (us-east-1a) — ALB, NAT Gateway, Bastion
  10.0.2.0/24 (us-east-1b)

Private Subnets (per AZ):
  10.0.10.0/24 (us-east-1a) — EC2/ECS, Lambda (VPC-attached)
  10.0.11.0/24 (us-east-1b)

DB Subnets (per AZ):
  10.0.20.0/24 (us-east-1a) — RDS, ElastiCache
  10.0.21.0/24 (us-east-1b)
```

### Internet Connectivity
- **Internet Gateway (IGW)**: Allows public subnets to reach internet
- **NAT Gateway**: Allows private subnets to initiate outbound internet (updates, API calls); per-AZ; managed
- **VPC Peering**: Connect two VPCs; no transitive peering
- **Transit Gateway**: Hub for multiple VPC connections + on-prem; transitive routing
- **VPC Endpoints**: Private connection to AWS services (S3, DynamoDB) without internet
  - Gateway endpoint: S3 + DynamoDB (free)
  - Interface endpoint (PrivateLink): Other services (pay per hour + GB)

### Route 53
- **Routing policies**: Simple, Weighted (A/B deploy), Latency-based, Failover, Geolocation, Multivalue
- **Health checks**: Monitor endpoints; used by failover routing
- **Alias records**: Route to AWS resources (ALB, CloudFront) — free, works at zone apex

### CloudFront
- CDN with 400+ edge locations
- **Behaviors**: Route `/api/*` to origin, `/*` to S3 bucket
- **Cache policies**: Control TTL, headers, cookies in cache key
- **Origin groups**: Primary + secondary origin for failover
- **Lambda@Edge / CloudFront Functions**: Run code at edge (auth, redirects, A/B testing)
- **Signed URLs/Cookies**: Restrict access to premium content

---

## Monitoring & Observability

### CloudWatch
- **Metrics**: Default metrics free; custom metrics (put-metric-data); 1-second granularity available
- **Alarms**: Trigger SNS, Auto Scaling, EC2 actions on threshold breach
- **Logs**: Collect from EC2 (agent), Lambda (automatic), ECS; Log Insights for queries
- **Dashboards**: Cross-region visibility
- **Contributor Insights**: Identify top-N contributors (e.g., which IP is hitting your API most)

### CloudWatch Log Insights Query
```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errorCount by bin(5m)
| sort @timestamp desc
| limit 100
```

### X-Ray
- Distributed tracing; visualize request flow across services
- SDK instrumentation in Node.js:
  ```typescript
  import AWSXRay from 'aws-xray-sdk';
  const AWS = AWSXRay.captureAWS(require('aws-sdk'));
  AWSXRay.captureHTTPsGlobal(require('https'));
  ```
- **Service Map**: Visual DAG of service dependencies + error rates + latency

### AWS Health Dashboard
- Personal Health Dashboard: Alerts about AWS events affecting YOUR resources

---

## Cost Optimization Principles (Principal-Level Awareness)

1. **Right-size instances**: Use CloudWatch CPU data; start smaller than you think
2. **Reserved + Savings Plans for baseline**, Spot for batch/non-critical
3. **S3 lifecycle policies**: Don't pay Standard pricing for old data
4. **NAT Gateway**: Can be expensive at scale; use VPC endpoints for AWS services
5. **Data transfer costs**: Free in, pay out; same-AZ traffic free, cross-AZ ~$0.01/GB
6. **CloudWatch**: Custom metrics and logs cost money; be selective
7. **RDS Multi-AZ vs Aurora**: Multi-AZ for simple HA; Aurora for high-scale + global
8. **AWS Cost Explorer + Budgets**: Set alerts before you're surprised
