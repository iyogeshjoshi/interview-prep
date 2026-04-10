# Security Architecture

## Zero-Trust Architecture

### Core Principle
> "Never trust, always verify" — no implicit trust based on network location.

Traditional model: trust everything inside the perimeter (VPN/firewall).
Zero-trust model: every request must be authenticated, authorized, and encrypted — regardless of origin.

```
Traditional:  [Firewall] → inside = trusted
Zero-Trust:   Every request: Who are you? Are you allowed? Is this device healthy?

Pillars:
  1. Identity   — Verify user/service identity on every request (MFA, certificates)
  2. Device     — Is the device healthy? (MDM, endpoint detection)
  3. Least Privilege — Minimum access needed; just-in-time (JIT) access
  4. Microsegmentation — Services can't reach each other unless explicitly allowed
  5. Assume Breach — Monitor as if already compromised; log everything
```

### Zero-Trust on AWS
```
Network:
  - Services in private subnets (no direct internet access)
  - Security Groups: allow only named SG, not CIDR ranges
  - VPC Endpoints: S3/DynamoDB traffic never leaves AWS backbone
  - PrivateLink: service-to-service without traversing internet

Identity:
  - IRSA (IAM Roles for Service Accounts) in EKS: per-pod permissions
  - ECS Task Roles: per-task, not per-EC2
  - Never: shared access keys; always: short-lived role credentials

Service-to-Service:
  - mTLS via service mesh (Istio/App Mesh): services present certificates
  - API Gateway + Lambda authorizer: validate every call
  - No "trusted internal" APIs — all require auth token
```

---

## OAuth2 & OIDC — The Full Picture

### OAuth2 Roles
```
Resource Owner:    The user (owns the data)
Client:            Application requesting access (your frontend/mobile app)
Authorization Server: Issues tokens (Auth0, Cognito, Okta, Google)
Resource Server:   API that holds protected data (your backend)
```

### Grant Types (Know All Four)

**Authorization Code + PKCE** (Web & Mobile — Gold Standard)
```
1. User clicks "Login" in client app
2. Client generates: code_verifier (random), code_challenge = SHA256(code_verifier)
3. Client redirects to Auth Server:
   /authorize?client_id=X&redirect_uri=Y&response_type=code
   &scope=openid profile&code_challenge=Z&code_challenge_method=S256
4. User authenticates at Auth Server
5. Auth Server redirects back: /callback?code=AUTH_CODE
6. Client POSTs to token endpoint:
   /token  { code, code_verifier, client_id, redirect_uri }
7. Auth Server: verifies SHA256(code_verifier) == code_challenge → issues tokens
8. Client receives: { access_token, id_token, refresh_token }

Why PKCE: Prevents auth code interception attacks (no client_secret needed in browser)
```

**Client Credentials** (Machine-to-Machine)
```
POST /token
  { grant_type: client_credentials, client_id, client_secret, scope }
→ { access_token }  (no refresh token — just get a new one)

Use for: backend service A calling backend service B
Never use for: anything user-facing (client_secret would be exposed)
```

**Implicit Grant** — DEPRECATED. Never use. Tokens in URL fragment, no refresh.

**Device Code** (Smart TVs, CLIs)
```
1. Device POSTs /device_authorization → gets device_code + user_code + verification_uri
2. Device displays: "Go to example.com/activate, enter code: ABCD-1234"
3. Device polls /token with device_code
4. User authenticates on phone/browser, enters user_code
5. Device poll succeeds → receives tokens
```

### OIDC (OpenID Connect) — Identity Layer on OAuth2
```
OAuth2 = authorization (can app access resource?)
OIDC   = authentication (who is the user?) + authorization

OIDC adds:
  - id_token (JWT containing user identity claims)
  - /userinfo endpoint
  - scope: openid (required), profile, email, address, phone
  - Discovery: /.well-known/openid-configuration

id_token claims:
  sub: "user-uuid"          ← unique user identifier
  iss: "https://auth.co"   ← issuer
  aud: "your-client-id"    ← audience (must validate this!)
  exp: 1234567890          ← expiry (must validate!)
  iat: 1234567890          ← issued at
  email: "user@co.com"
  nonce: "random"          ← prevents replay attacks
```

### Token Validation (Backend Must Do All of These)
```typescript
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: 'https://your-auth-server/.well-known/jwks.json',
  cache: true, cacheMaxAge: 600000,  // cache public keys 10min
});

async function validateToken(token: string): Promise<JWTPayload> {
  const decoded = jwt.decode(token, { complete: true });
  if (!decoded) throw new Error('Invalid token');

  // Get public key by key ID (kid)
  const key = await client.getSigningKey(decoded.header.kid);
  const publicKey = key.getPublicKey();

  return jwt.verify(token, publicKey, {
    algorithms: ['RS256'],             // reject HS256 (symmetric — server must know secret)
    audience: process.env.API_AUDIENCE, // validate aud claim
    issuer: process.env.AUTH_ISSUER,    // validate iss claim
    clockTolerance: 30,                 // 30s clock skew tolerance
  }) as JWTPayload;
  // Also validates: signature, exp, nbf automatically
}
```

### Refresh Token Strategy
```
Access token:  Short-lived (15min–1hr). Stateless JWT. Use for API calls.
Refresh token: Long-lived (7–90 days). Opaque. Stored securely. Use to get new access token.

Rotation: On each use, issue new refresh token + invalidate old one
  → Detects token theft: if old token used again → revoke entire family

Storage:
  Access token:  Memory (JS variable) or sessionStorage — NOT localStorage (XSS risk)
  Refresh token: HttpOnly Secure cookie (not accessible from JS)
  
  Alternatively: BFF pattern — browser never holds tokens; BFF holds them server-side
```

---

## Threat Modeling (STRIDE)

Principal engineers are expected to think proactively about security threats.

```
S — Spoofing Identity      Can someone pretend to be another user/service?
T — Tampering              Can someone modify data in transit or at rest?
R — Repudiation            Can someone deny having performed an action?
I — Information Disclosure Can sensitive data be exposed?
D — Denial of Service      Can the system be made unavailable?
E — Elevation of Privilege Can someone gain permissions beyond their role?
```

### Applying STRIDE to an API
```
For each data flow in your system diagram, ask each STRIDE question.

Example: POST /orders endpoint
  S: Is the JWT signature validated? Is the user who they claim to be?
  T: Is the request over TLS? Is the payload validated? SQL injection possible?
  R: Are all mutations logged with user ID + timestamp + payload?
  I: Does the response include data this user shouldn't see? Error messages leak info?
  D: Is there rate limiting? Can one user create millions of orders?
  E: Does this endpoint check that user can only modify their OWN orders?
```

### Threat Model Template for Design Reviews
```markdown
## Threat Model: Order Service

### Assets
- User PII (email, address)
- Payment tokens
- Order history

### Trust Boundaries
- Internet → ALB (public)
- ALB → Order Service (private VPC)
- Order Service → Payment Service (private VPC, mTLS)
- Order Service → RDS (private subnet, SG-restricted)

### Threats & Mitigations
| Threat | STRIDE | Mitigation |
|--------|--------|------------|
| JWT stolen from localStorage | S | Use HttpOnly cookies |
| IDOR: access other user's orders | E | Validate userId == token.sub |
| SQL injection in order search | T | Parameterized queries (pg library) |
| Brute force order ID guessing | I | Use UUIDs, not sequential IDs |
| Bulk order creation (DoS) | D | Rate limit: 10 orders/min/user |
| Replay attack on payment | T | Idempotency keys + nonce |
```

---

## Supply Chain Security

### Dependency Risks
```
Risks:
  1. Malicious packages (typosquatting: "lodahs" instead of "lodash")
  2. Compromised legitimate packages (event-stream attack, 2018)
  3. Vulnerable dependencies (log4shell, protobuf-js, etc.)

Mitigations:
  - Lock files (package-lock.json, pnpm-lock.yaml) — pin exact versions
  - npm audit on every CI build; fail on high severity
  - Dependabot / Renovate — automated PRs for dependency updates (keep current)
  - Allowlist registries: .npmrc → registry = https://registry.npmjs.org (no fallback)
  - SBOM (Software Bill of Materials): generate with Syft/Cyclonedx in CI
  - Verify package integrity: npm uses SHA-512 hashes in lockfile
```

### Container Supply Chain
```
Base image hygiene:
  Use official images only (node:20-alpine, not random dockerhub images)
  Pin to digest, not tag: node@sha256:abc123... (tags are mutable)
  
Scanning in CI:
  Trivy: trivy image my-app:latest --exit-code 1 --severity HIGH,CRITICAL
  ECR enhanced scanning (Inspector): continuous, not just on push

Signing & attestation (advanced):
  Sigstore/Cosign: sign images with keyless signing (OIDC identity)
  Verify signature before deploy
  
Runtime protection:
  Read-only root filesystem in containers
  Drop all Linux capabilities, add back only needed ones
  Seccomp profiles (AWS Fargate applies a default)
```

### Secrets Never In
```
❌ Never:
  - Source code (git blame never forgets)
  - Docker image (docker history reveals ENV vars)
  - CI/CD environment variables shown in logs
  - package.json scripts
  - Error messages / stack traces sent to client

✓ Always:
  - AWS Secrets Manager / Parameter Store
  - Runtime injection via ECS secrets / K8s secrets
  - Rotate on suspicion of compromise (Secrets Manager auto-rotation)
  - Pre-commit hooks: git-secrets, detect-secrets, TruffleHog
  - GitHub secret scanning (automatic, free)
```

---

## Common Security Vulnerabilities (OWASP Deep Dive)

### Injection (SQL, NoSQL, Command)
```typescript
// SQL Injection — NEVER do this:
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`);
// Attack: email = "' OR '1'='1"

// Always parameterized:
const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);

// NoSQL injection (MongoDB):
// Never: db.users.find({ email: req.body.email })
// If req.body.email = { $gt: "" } → returns all users!
// Always validate/sanitize input with Zod before DB operations

// Command injection:
// Never: exec(`convert ${filename} output.pdf`)
// Always: execFile('convert', [filename, 'output.pdf']) — no shell interpolation
```

### Broken Object Level Authorization (BOLA / IDOR)
```typescript
// Bad: trusts user-supplied ID without ownership check
app.get('/orders/:id', async (req, res) => {
  const order = await db.orders.findById(req.params.id);
  res.json(order);
});
// Attack: GET /orders/12345 (other user's order)

// Good: always scope to authenticated user
app.get('/orders/:id', authenticate, async (req, res) => {
  const order = await db.orders.findOne({
    id: req.params.id,
    userId: req.user.id,  // ← ownership enforced
  });
  if (!order) return res.status(404).json({ error: 'Not found' });
  res.json(order);
});
```

### Mass Assignment
```typescript
// Bad: spread entire request body onto DB record
app.put('/users/:id', async (req, res) => {
  await db.users.update(req.params.id, req.body);
  // Attack: { "role": "admin", "isVerified": true }
});

// Good: explicit allowlist with Zod
const UpdateUserSchema = z.object({
  name: z.string().optional(),
  email: z.string().email().optional(),
  // NOT: role, isVerified, createdAt, deletedAt
});
app.put('/users/:id', async (req, res) => {
  const data = UpdateUserSchema.parse(req.body);
  await db.users.update(req.params.id, data);
});
```

### Security Headers (Express)
```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-{RANDOM}'"],  // no 'unsafe-inline'
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  noSniff: true,             // X-Content-Type-Options: nosniff
  frameguard: { action: 'deny' },  // X-Frame-Options: DENY (clickjacking)
  xssFilter: true,
}));

// CORS — be explicit
app.use(cors({
  origin: ['https://app.yourdomain.com'],  // never '*' for credentialed requests
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
}));
```

---

## mTLS (Mutual TLS) for Service-to-Service

```
Regular TLS: Server proves identity to client
mTLS:        Server AND client prove identity to each other

Use for: microservice-to-microservice calls in zero-trust environments

Implementation options:
  1. Service mesh (Istio/Linkerd): automatic, no code changes
     Sidecar proxy handles certificate issuance + rotation + verification
     
  2. Manual (AWS ACM Private CA):
     Each service gets a cert from your private CA
     Validate client cert in server: req.socket.getPeerCertificate()
     
  3. AWS API Gateway + mutual TLS:
     Upload truststore (CA cert)
     Client must present valid cert signed by that CA

Certificate rotation:
  Certs must rotate before expiry
  Service mesh: automatic (typically 24hr cert lifetime)
  Manual: use cert-manager (K8s) or AWS Certificate Manager
```

---

## Principal-Level Security Talking Points

**"How do you think about security in your architecture decisions?"**
> "Security is a design constraint, not an afterthought. I apply threat modeling during architecture reviews — specifically STRIDE — to identify threats before code is written. I also enforce security at the platform level: our service templates include helmet, rate limiting, parameterized queries, and structured logging by default, so teams get a secure baseline without thinking about it."

**"What's the most important security control in a Node.js API?"**
> "Probably input validation at the boundary with something like Zod — it prevents injection, mass assignment, and type confusion in one place. But close second is proper authorization checking: BOLA is the most common API vulnerability, and it's purely an application-level concern that no WAF can catch."

**"How do you handle secret rotation without downtime?"**
> "AWS Secrets Manager with auto-rotation. The key is dual-active: during rotation, both old and new credentials work for a transition window. Applications fetch the secret by name at startup and cache it — if a 401 is received, they re-fetch (secrets may have rotated). For DB credentials, RDS rotation via Secrets Manager handles this transparently."
