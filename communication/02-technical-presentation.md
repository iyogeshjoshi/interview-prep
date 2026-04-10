# Communication — Technical Presentation & System Design Communication

## Communicating in a System Design Interview

### Signal Clearly, Then Design

Principal engineers think out loud in a structured way. Interviewers aren't just evaluating the design — they're evaluating **how you think**.

```
Do say:
  "Let me start by understanding the problem space..."
  "Before I draw anything, I want to size this so we know what we're actually solving..."
  "I'll go broad first, then we can pick one area to go deep on — is that ok?"
  "Here's my trade-off: option A is simpler but won't scale past X. Option B handles that but adds operational complexity. For this problem, I'd go with..."

Don't say:
  "Hmm... uh... so... maybe we use Kafka?"  (jumping to tech before understanding problem)
  "This is the right way to do it."  (no trade-offs acknowledged)
  "I'm not sure if that's right."  (constant self-doubt signals)
```

### Signposting
Tell the interviewer what you're about to do before you do it:
- "I'm going to estimate scale first so we know what we're targeting."
- "I'll sketch the high-level flow now, then we can zoom in on the [most interesting component]."
- "I want to highlight a potential bottleneck here before moving on."
- "This is a trade-off worth calling out..."

### Recovering from Uncertainty
If you don't know something specific:
- "I haven't used that service in production, but based on what I know about how it works, I'd expect..."
- "I'd need to check the specifics on that, but the principle I'd apply is..."
- "Let me reason through it: if the problem is X, then we need Y property, which suggests Z approach..."
- Never bluff. Interviewers test depth. Acknowledging uncertainty + reasoning through it scores higher than a wrong confident answer.

---

## Architecture Decision Records (ADRs)

Writing clear ADRs is a key Principal engineer skill — shows you can make decisions collaboratively and leave a paper trail.

```markdown
# ADR 012: Use PostgreSQL for order data instead of DynamoDB

## Status
Accepted

## Context
We need to store order data. Orders have complex relational structure (order → line items → products → discounts). 
We evaluated DynamoDB (current default) and PostgreSQL.

## Decision
Use PostgreSQL (Aurora Serverless v2) for the order service.

## Rationale
- Orders have complex relationships requiring JOINs that are expensive to model in DynamoDB
- ACID transactions are critical (inventory decrement + order create must be atomic)
- Expected volume is 50K orders/day — well within PostgreSQL's sweet spot
- Query patterns for reporting (aggregations by date, user, product) are natural SQL

## Consequences
- Positive: Complex queries are simple; transactional integrity guaranteed
- Positive: Existing team PostgreSQL expertise
- Negative: Less elastic horizontal scaling than DynamoDB (mitigated by Aurora auto-scaling)
- Negative: Must manage schema migrations (use Flyway/Liquibase)

## Alternatives Considered
- DynamoDB: Would require single-table design with complex GSI patterns; no ACID across items; reporting queries would require DynamoDB Streams + aggregation pipeline
- MongoDB: Document model doesn't fit relational structure better than PostgreSQL; adds new technology to stack
```

---

## Writing RFCs (Requests for Comments)

For significant technical decisions, Principal engineers often write RFCs to drive consensus:

```markdown
# RFC: Migrate Order Service to Event Sourcing

## Summary
One paragraph. What are you proposing and why.

## Motivation
What problem does this solve? What are the pain points today?
Link to incidents, tickets, metrics that illustrate the problem.

## Detailed Design
How will this work? Include:
- Architecture diagram
- Data model changes
- API contract changes
- Migration strategy

## Drawbacks
What are the downsides? Complexity? Risk? Cost?
(Omitting this destroys credibility)

## Alternatives
What else did you consider and why didn't you choose it?

## Rollout Plan
How do we get from here to there safely?
Feature flag? Dark launch? Gradual migration?

## Open Questions
What's still uncertain? What do you need feedback on?
```

---

## Technical Deep Dive (Walk-Through a Past Project)

Common format: "Tell me about the most technically complex project you've worked on."

### Structure
```
1. Context (30 sec): company, team, business problem
2. Technical challenge (1 min): what made it hard
3. Your specific contribution (1-2 min): what YOU designed/built/decided
4. Trade-offs you navigated: what you chose and why
5. Result: measurable outcome
6. What you'd do differently: shows growth + honesty
```

### Power Phrases for Technical Impact
- "I was responsible for the technical direction of..."
- "The constraint that shaped everything was..."
- "We had three options; I advocated for X because..."
- "The key insight was..."
- "That decision saved us X / enabled Y / prevented Z"
- "In hindsight, I would have..."

---

## Whiteboarding / Diagramming Conventions

When drawing architecture diagrams on a whiteboard or in an interview:

```
Conventions to use:
  [Client/Browser] → box with role
  → solid arrow = synchronous request
  --> dashed arrow = async / event
  === double line = data store

Label everything:
  What protocol? (HTTP, gRPC, WebSocket)
  What data flows? (user object, event payload)
  What's the scale? (1K req/s, 50GB/day)

Group by:
  Zones: Client | CDN/Gateway | Services | Data
  OR by: Domain (order, payment, inventory)
```

---

## Handling Pushback on Your Design

Interviewers often deliberately challenge your choices. Treat pushback as a conversation, not a test.

```
Interviewer: "Why not use Kafka instead of SQS for this?"

Weak response: "You're right, I should use Kafka."
(capitulating without reasoning signals no conviction)

Weak response: "No, SQS is fine."
(dismissive, defensive)

Strong response:
"That's a good point — Kafka would give us message replay and fan-out, which could be useful if we add more consumers. I chose SQS here because:
1. We currently have one consumer and no replay requirement
2. SQS is ops-free; Kafka (even MSK) adds operational surface
3. The throughput is well within SQS limits

If we anticipate more consumer services or need the audit log, Kafka would be the right upgrade. Does the use case you're imagining require those properties?"
```

Always:
1. Acknowledge the merit in the challenge
2. State your reasoning explicitly
3. Name the conditions under which you'd change your mind
4. Ask a clarifying question to understand their concern

---

## Presentation: Proposing a Technical Initiative

When presenting a technical initiative to leadership (engineering director, VP):

### 5-minute pitch structure
```
1. The Problem (30 sec)
   "Our deployment frequency dropped from 3x/week to 1x/week over the past year."
   → Use data. Specific, quantified.

2. Why Now (30 sec)
   "We're about to hire 5 more engineers. If we don't fix this now, onboarding them will be slower and the problem compounds."

3. Proposed Solution (1 min)
   "I'm proposing we spend 3 sprints migrating to feature flags and improving our test pipeline. Here's what that looks like..."
   → Diagram or 3-bullet summary

4. Expected Impact (30 sec)
   "This should get us back to daily deployments and reduce our mean time to deliver a feature by ~40%."

5. Cost/Risk (30 sec)
   "3 sprints of 2 engineers. Main risk is scope creep — I'd mitigate with a strict definition of done."

6. Ask (30 sec)
   "I'd like 2 weeks to validate with a spike. Can I get buy-in to start next sprint?"
```

### Handling "Why does this matter to the business?"
Frame everything in terms the business cares about:
- Revenue: "Each hour of downtime costs ~$50K in lost orders"
- Velocity: "This will let us ship features 2x faster"
- Risk: "Without this, we're one outage away from an SLA breach that triggers customer credits"
- Cost: "This will reduce our AWS bill by ~$8K/month"

Never: "It's the right engineering practice" — this is correct but not persuasive to business stakeholders.
