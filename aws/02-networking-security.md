# AWS — Networking & Security

## IAM (Identity and Access Management)

### Core Concepts
- **Principal**: Who is making the request (IAM user, role, service, account)
- **Policy**: JSON document defining permissions (Allow/Deny + Action + Resource + Condition)
- **Role**: Identity assumed by services, users, or cross-account principals (no long-term credentials)

### Policy Evaluation Logic
```
1. Explicit Deny  → DENY (always wins)
2. SCPs (Org)     → restrict what accounts can do
3. Resource Policy → cross-account access
4. Identity Policy → what the principal can do
5. Default         → DENY
```

### Least Privilege Examples
```json
// EC2 reading from specific S3 bucket only
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }]
}
```

### IAM Best Practices
- **Never use root account** except for billing/account setup; enable MFA on root
- **Roles over Users**: EC2/Lambda/ECS get IAM roles, not access keys
- **No long-term access keys**: Use roles, OIDC federation, or SSO
- **Permission boundaries**: Limit maximum permissions a role can have (delegation control)
- **Service Control Policies (SCP)**: Organization-level guardrails (e.g., block all regions except us-east-1)
- **AWS Organizations**: Multi-account strategy; separate prod/dev/security accounts
- **Cross-account roles**: Account A assumes role in Account B; trust policy controls who can assume

### Instance Metadata Service (IMDS)
```bash
# Get temp credentials for EC2's attached role
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/RoleName
# IMDSv2 (required for new instances — prevents SSRF abuse)
TOKEN=$(curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" http://169.254.169.254/latest/api/token)
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/
```

---

## Secrets & Encryption

### Secrets Manager
- Store database credentials, API keys, OAuth tokens
- **Auto-rotation**: Built-in rotation for RDS; custom Lambda for others
- Versions: `AWSCURRENT`, `AWSPENDING`, `AWSPREVIOUS`
- Pay per secret + per 10K API calls
- Access from app:
  ```typescript
  const client = new SecretsManagerClient({});
  const response = await client.send(new GetSecretValueCommand({ SecretId: 'prod/myapp/db' }));
  const secret = JSON.parse(response.SecretString!);
  ```

### Parameter Store (SSM)
- **Standard**: Free, 4KB, no rotation. Good for: config, feature flags, non-sensitive
- **Advanced**: Pay, 8KB, rotation Lambda, references Secrets Manager. Good for: sensitive
- **SecureString**: Encrypted with KMS
- Hierarchical naming: `/prod/myapp/db/password`

### KMS (Key Management Service)
- **CMK (Customer Master Key)**: Your key, you control access
- **AWS managed keys**: Created by services (S3, RDS); you can see but not manage directly
- **Customer managed CMK**: You create, rotate, delete; fine-grained IAM control
- **Envelope Encryption**: KMS encrypts a data encryption key (DEK); DEK encrypts data locally
  - Why: KMS has 4KB limit; envelope lets you encrypt large data without sending it to KMS
- **Key Rotation**: Automatic annual rotation (new backing material, same key ARN)
- **Key Policy**: Resource-based policy on the key itself; must explicitly allow root account

---

## Network Security

### Security Groups vs NACLs

| | Security Group | NACL |
|---|---|---|
| Level | Instance/ENI | Subnet |
| Stateful? | Yes — return traffic auto-allowed | No — must allow in both directions |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (lowest # first) |
| Default | All outbound allowed, no inbound | All allowed (custom NACLs: all denied) |

### Typical 3-tier Security Group Setup
```
ALB Security Group:
  Inbound: 80, 443 from 0.0.0.0/0
  Outbound: 8080 to App SG

App Security Group:
  Inbound: 8080 from ALB SG only
  Outbound: 5432 to DB SG; 443 to 0.0.0.0/0 (for external APIs)

DB Security Group:
  Inbound: 5432 from App SG only
  Outbound: (none needed — stateful)
```

### WAF (Web Application Firewall)
- Protect ALB / CloudFront / API Gateway / AppSync
- **Managed Rule Groups**: AWS or Marketplace rules (OWASP top 10, bot control)
- **Custom Rules**: IP sets, geo-match, rate-based (auto-block IPs exceeding threshold), regex on body
- **Web ACL**: Container for rules; attach to resource

### Shield
- **Shield Standard**: Free; automatic DDoS protection for all AWS customers (L3/L4)
- **Shield Advanced**: ~$3K/month; L7 DDoS, real-time metrics, 24/7 DDoS Response Team (DRT), cost protection during DDoS

### GuardDuty
- ML-based threat detection; analyzes CloudTrail, VPC Flow Logs, DNS logs
- Detects: unusual API calls, port scanning, crypto mining, compromised EC2
- No agents; enable in minutes; automatic findings in Security Hub

---

## Certificate Management

### ACM (AWS Certificate Manager)
- Free SSL/TLS certificates for AWS resources (ALB, CloudFront, API Gateway)
- **Public certs**: DNS or email validation; auto-renewed
- **Private CA**: ACM Private CA for internal services (paid)
- Cannot export private key (intentionally) — only usable with AWS services
- For EC2 (non-managed): Use Let's Encrypt via Certbot, store cert+key in Secrets Manager

---

## AWS Organizations & Multi-Account Strategy

### Why Multi-Account?
- **Blast radius isolation**: Compromised dev account can't touch prod
- **Cost allocation**: Separate billing per business unit
- **SCPs**: Hard limits per account (e.g., dev can't create resources in prod regions)
- **Compliance**: Different accounts for different data classifications

### Common Structure
```
Management Account (billing only, no workloads)
├── Security Account (GuardDuty master, CloudTrail aggregation, Security Hub)
├── Shared Services Account (ECR, Artifact, Transit Gateway)
├── Production OU
│   ├── Prod App Account
│   └── Prod Data Account
├── Non-Production OU
│   ├── Staging Account
│   └── Dev Account
└── Sandbox OU (loose SCPs, dev experiments)
```

### AWS SSO / IAM Identity Center
- Centralized access management across accounts
- SAML 2.0 federation with corporate IdP (Okta, Active Directory)
- Assign permission sets → account/user combinations
- No need for IAM users in individual accounts

---

## Compliance & Audit

### CloudTrail
- Logs every API call across all AWS services (who, what, when, from where)
- **Management events**: Control plane (create/delete/modify resources) — free 1 copy
- **Data events**: S3 object-level, Lambda invocations — extra cost
- **Insights**: Detect unusual activity (sudden spike in API calls)
- Multi-region trail → S3 bucket (enable log file integrity validation)
- **Must enable in every account** — security requirement

### Config
- Continuous recording of resource configurations + changes
- **Config Rules**: Detect non-compliance (e.g., "all S3 buckets must block public access")
- **Conformance Packs**: Group of rules for a standard (PCI DSS, CIS)
- **Remediation**: Auto-remediate via SSM Automation or Lambda
- Config ≠ GuardDuty: Config is "what changed", GuardDuty is "suspicious behavior"

### Security Hub
- Aggregates findings from GuardDuty, Inspector, Macie, Config, third-party tools
- Scores against CIS AWS Foundations, PCI DSS, AWS Best Practices
- Single pane for security posture across all accounts

### Macie
- ML-based sensitive data discovery in S3 (PII, financial data, credentials)
- Alerts on publicly accessible buckets, unencrypted buckets
