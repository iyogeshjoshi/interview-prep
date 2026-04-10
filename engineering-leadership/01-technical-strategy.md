# Engineering Leadership — Technical Strategy

## What Principal Engineers Actually Do

Principal engineers operate at the **organization level**, not just the team level. The key shift from Staff/Senior is:

```
Senior Engineer:    Owns solutions within a team
Staff Engineer:     Owns solutions across a team; starting to influence adjacent teams
Principal Engineer: Sets technical direction for multiple teams or the entire org
                    Decisions affect 10s of engineers and multiple product areas
```

### The "Glue Work" That's Invisible But Critical
- Writing the design doc that gets 4 teams aligned on a shared direction
- Identifying that two teams are solving the same problem differently
- Reducing dependencies between teams so they can ship independently
- Onboarding senior hires so they're productive in weeks, not months
- Building the platform that 10 engineers use daily — force multiplier

---

## Writing a Technical Vision Document

A technical vision doc answers: "Where should our system be in 18–24 months and why?"

### Structure
```markdown
## Problem Statement
What is broken or missing today? Use data/incidents to show it's real.
(e.g., "Deployment takes 3 hours due to a monolithic build pipeline.
We deployed 52 times last year vs. 400+ at comparable companies.")

## Desired Outcome
What does success look like? Be specific and measurable.
(e.g., "Any team can deploy their service independently in <10 minutes,
with no coordination with other teams required.")

## Proposed Architecture
High-level: what changes? Diagrams help.
Not a detailed design — that comes later in individual RFCs.

## Migration Path
How do we get from here to there without burning the org down?
Phase 1: foundations. Phase 2: migrate first team. Phase 3: rollout.

## Principles / Guardrails
What trade-offs will guide decisions as this evolves?
(e.g., "We prefer boring technology over cutting-edge when both solve the problem.")

## Risks
What could go wrong? What are you not solving?

## Success Metrics
How will you know it's working? (deployment frequency, MTTR, cycle time)
```

### Getting Buy-in
1. **Share early as a "straw man"** — not a decree. "Here's my thinking, please punch holes in it."
2. **Talk to the people who will be affected** before the doc is "done"
3. **Address dissent in the doc itself** — "Team X raised concern Y. We addressed it by..."
4. **Find champions** — one senior engineer per affected team who believes in the direction
5. **Separate vision from plan** — vision aligns people on destination; plan details follow

---

## Build vs. Buy Decisions

Principal engineers are expected to make and defend these calls.

### Framework
```
1. Is this core to your competitive advantage?
   Yes → build (differentiating capability; don't outsource your moat)
   No  → strong lean toward buy/use managed service

2. What is the total cost of ownership?
   Build: engineering time × engineer cost × maintenance factor (2-3x initial)
   Buy: license + integration + vendor risk + switching cost

3. What's the risk of getting it wrong?
   Auth, payments, security → buy (the downside of getting it wrong is catastrophic)
   Core recommendation engine → build (this IS your product)

4. Time to market?
   Buy: weeks to integrate. Build: months.
   If speed matters and capability isn't differentiated → buy.

5. Vendor lock-in risk?
   Can you leave if the vendor raises prices 10x or gets acquired?
   Abstraction layer helps; standards-based (S3 API, OpenTelemetry) helps more.
```

### Examples & Reasoning
| Capability | Decision | Why |
|---|---|---|
| Authentication | Buy (Auth0, Cognito, Clerk) | Security-critical; not differentiating |
| Payment processing | Buy (Stripe, Braintree) | Compliance, security, fraud handling |
| Search | Buy (Elasticsearch, Algolia) | Specialized; hard to build well |
| Core recommendation engine | Build | Competitive advantage |
| Internal analytics dashboard | Buy (Metabase, Looker) | Not differentiating |
| Message queue | Buy (SQS, MSK) | Managed infra; operational cost if self-hosted |
| CI/CD pipeline | Buy (GitHub Actions, CircleCI) | Not differentiating |

---

## Platform Thinking

Platform teams exist to accelerate product teams. A platform is successful when product teams:
- Don't have to think about it (it just works)
- Can onboard to it in a day
- Find it faster than building themselves

### The Platform Product Mindset
```
Treat internal teams as customers.
The "product" is: golden paths, templates, SDKs, documentation, support.

Signs of a bad platform:
  - Teams build their own tooling to work around the platform
  - Documentation is always out of date
  - Onboarding takes weeks instead of days
  - Platform team gates every change

Signs of a good platform:
  - Teams can self-serve 90% of their needs
  - Platform team enables, not approves
  - New engineer productive on day 3
```

### Internal Developer Experience (DevEx)
Key metrics:
- **Onboarding time to first PR**: target < 1 day
- **Local dev setup time**: target < 30 min
- **Build / test cycle time**: fast feedback loop is critical
- **Documentation completeness**: can an engineer answer their own questions?

---

## Team Topologies (Know the Model)

Based on the "Team Topologies" book — useful mental model for org discussions.

```
Stream-aligned team:
  Aligned to a business domain (e.g., "Checkout team")
  Full-stack, full ownership: design → code → deploy → operate
  Goal: fast flow, minimal dependencies
  
Platform team:
  Builds internal products used by stream-aligned teams
  e.g., Infrastructure, Developer Tools, Design System
  Goal: reduce cognitive load on stream-aligned teams
  
Enabling team:
  Temporary: helps stream-aligned teams adopt new capability
  e.g., "Security enablement" helping teams adopt SAST
  Dissolves once teams are self-sufficient
  
Complicated Subsystem team:
  Owns code requiring specialist expertise
  e.g., Real-time engine, ML platform, payments
  Rare; avoid unless genuinely complex
```

### Conway's Law in Practice
> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of those organizations."

**Inverse Conway Maneuver**: Design your team structure to produce the architecture you want. If you want microservices with independent deployments, structure teams around services first.

---

## Tech Debt Strategy

### Classification
```
Type 1 — Intentional, strategic:
  "We shipped fast, knew this needed cleanup"
  → Schedule explicitly; pay down before it compounds

Type 2 — Unintentional, discovered:
  "We didn't know better at the time"
  → Prioritize when it blocks velocity or causes incidents

Type 3 — Environmental:
  "Dependencies aged out, security patches needed"
  → Treat as operational: regular dependency updates, automated (Dependabot)

Type 4 — Architecture:
  "The original design can't support current scale"
  → High impact, high effort; needs vision doc + phased migration
```

### Making the Business Case
```
Frame as: risk or velocity loss, not engineering aesthetics.

"The current authentication service has no tests, uses a deprecated library,
and two engineers left who understood it. Every time checkout needs to change
auth behavior, it takes 3 weeks instead of 1. In Q2 we need 3 such changes.
Investing 4 weeks now saves ~8 weeks in Q2 and eliminates the incident risk."
```

### The 20% Rule (Google's Model)
- Allocate 20% of sprint capacity to tech health (debt, reliability, developer experience)
- Non-negotiable; treated as real work, not "if we have time"
- Track tech health metrics as seriously as feature velocity

---

## Engineering Roadmap Alignment

### How Technical Work Fits in a Product Roadmap

```
Product roadmap:    Feature A → Feature B → Feature C (driven by business)
Technical roadmap:  Infra work → Migration → Optimization (driven by engineering)

Good Principal Engineer:
  Identifies technical work that enables/accelerates product roadmap items
  Frames technical investment in terms of product impact
  Negotiates capacity for foundational work before it becomes blocking
```

### OKRs for Engineering
```
Objective: Increase deployment reliability
KR1: MTTR (Mean Time to Recovery) < 15 min for P1 incidents (from 45 min)
KR2: Deployment success rate > 99% (from 94%)
KR3: 100% of services have runbooks and automated recovery for top-3 failure modes
```

---

## Onboarding Senior Engineers

Principal engineers often own this. A good onboarding plan:

**Week 1**: Context
- Architecture overview (30 min with you, not 5 hours of docs)
- The 3 most important systems they'll touch
- First PR: something real but bounded (a bug fix, small feature)

**Week 2-3**: Independence
- They own a small feature end-to-end
- You're available but not hand-holding
- Retrospective: what was confusing?

**Month 1**: Contribution
- They've shipped something meaningful
- They can unblock themselves 90% of the time
- They know who to ask for what

**Common mistakes**:
- Too much documentation, not enough doing
- No first PR in week 1 → engineer feels unproductive
- Assuming senior = no guidance needed (senior in previous context ≠ senior in yours)
