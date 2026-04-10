# Communication — Leadership & Behavioral

## The STAR Format

All behavioral answers follow: **Situation → Task → Action → Result**

- **Situation**: Set the context (team size, company stage, constraints)
- **Task**: What was YOUR responsibility (not the team's)
- **Action**: What YOU specifically did — be concrete, not "we"
- **Result**: Measurable outcome (%, $, time saved, incidents avoided)

Keep answers to 2–3 minutes. End with what you learned.

---

## Core Behavioral Questions for Principal Level

### "Tell me about a time you influenced a technical decision you didn't own"

**What they're testing**: Cross-functional influence without authority; persuasion through data

**Framework**:
1. Identify the decision and why you cared
2. Who were the stakeholders? What were their constraints/incentives?
3. How did you build the case (data, prototypes, risk analysis)?
4. How did you handle resistance?
5. What was the outcome?

**Example structure**:
> "Our payments team was about to build a custom retry mechanism. I noticed it would create inconsistency with how our notification and inventory services handled retries, and could lead to duplicate charges under failure scenarios. 
> I wrote a brief [RFC/design doc] showing three options, including reusing our existing idempotency layer with minor extension. I presented to the payments lead and their EM, focusing on the risk of inconsistency rather than 'my idea is better.' They adopted the shared approach. Six months later when we had a partial network outage, it worked without a single duplicate charge, whereas the original approach would have been untested in that scenario."

---

### "Describe a time you had to make a decision with incomplete information"

**What they're testing**: Decision-making under uncertainty; bias toward action

> "We were mid-launch for a major feature. At 10pm, we saw error rates climb to 3% — above our 1% SLA threshold. We didn't know if it was our new code or a downstream vendor issue. I made the call to roll back our changes even though we weren't certain they were the cause, because the rollback was low-risk and the vendor investigation would take hours. Error rates dropped immediately. Post-mortem confirmed it was our code after all. The key was: what's the reversible action with the lowest risk given what we know right now?"

---

### "Tell me about a time you disagreed with your manager/leadership"

**What they're testing**: Psychological safety, professionalism, ability to advocate without insubordination

**Don'ts**: Don't make your manager the villain. Don't say "I just did it their way."

> "My engineering director wanted to rewrite our main service in Go because of perceived performance issues. I disagreed — our performance problems were database queries, not the Node.js runtime, and a rewrite would take 8 months for zero user benefit. I put together a 2-page doc with profiling data, showing the top 10 query bottlenecks and estimating a 4-week optimization project would get us 10x the performance improvement. I asked for 2 weeks to prove it with one endpoint. We did. The director agreed to put the rewrite on hold. 18 months later we still haven't needed it. I learned that showing a faster path to the goal is more effective than debating the approach."

---

### "How do you raise the technical bar on your team?"

**What they're testing**: Multiplier effect; investment in others; influence at scale

Areas to cover:
- **Code review culture**: Reviews as teaching moments, not gatekeeping
- **Design reviews**: Lightweight RFC process for significant decisions; write decisions down
- **Pairing and mentoring**: Structured 1:1 technical sessions
- **Post-mortems**: Blameless; action items followed up on
- **Documentation**: ADRs (Architecture Decision Records) for decisions that were debated
- **Onboarding**: Strong Day 1 → Week 1 guide; pairing for first PR
- **Standards**: Codified in linting, tests, templates — not just vibes

---

### "Describe a time you had to balance technical debt with feature delivery"

**What they're testing**: Business awareness; pragmatism; ability to communicate risk

> "We had a growing backlog of technical debt around our authentication service — it was a 4-year-old module with no tests and a shared global state pattern that made it impossible to run multiple instances. Product was pushing hard for 3 new features in Q2. I mapped the debt against feature velocity and showed that 2 of the 3 features would require touching this module and take 3x longer because of the existing state. I proposed: one sprint on the auth refactor first, then features. The refactor made features 40% faster to build and we shipped all 3 before the original deadline. Key framing: tech debt isn't a tax, it's a velocity multiplier when addressed strategically."

---

### "Tell me about a production incident you led"

Structure:
1. What happened (brief)
2. How you detected it
3. How you coordinated the response (communication, delegation)
4. How you mitigated / resolved
5. Post-mortem actions you drove

**Principal-level focus**: How did you communicate? Who did you involve? How did you run the incident without doing all the work yourself?

> "During our biggest sale day, our order service started timing out. I was on-call. Within 5 min I had an incident channel open, paged the DB and infra leads, and posted a running timeline in Slack. I delegated: DB lead investigated query performance, infra lead checked connection pools, I monitored X-Ray traces and correlated errors. We found a query without an index that was fine under normal load but full-scanned under 10x traffic. DB lead added the index online; recovery in 23 minutes. I drafted the post-mortem before EOD and scheduled it for 48 hours later. Action items included alerting for slow queries and a load test in staging before future sales."

---

## Questions to Ask Your Interviewer

These signal you think at the right level:

**On engineering culture**:
- "How are significant technical decisions made — RFC process, architecture council, or individual teams own their decisions?"
- "What's the path from an engineer identifying a systemic problem to actually getting it fixed?"

**On scope and impact**:
- "What would distinguish a good first 6 months for this role from a great one?"
- "What are the biggest technical bets the company is making in the next year?"

**On team health**:
- "How do you handle disagreements between principal engineers on architectural direction?"
- "What does technical career growth look like beyond the principal level here?"

**On the work itself**:
- "What does the current system look like, and where do you feel it's holding back product velocity?"
- "Are there any areas where the team is knowingly accumulating risk that you'd want this role to address?"

---

## Communicating to Non-Technical Stakeholders

### The Pyramid Principle
```
Lead with the conclusion (what they care about)
  ↓
Support with key reasoning (2-3 points)
  ↓
Detail if asked (technical depth)
```

Never start with: "So first we have this service, and it talks to this database, and…"
Always start with: "The recommendation is X. Here's why it's the right call."

### Translating Technical Concepts
| Technical | Business translation |
|---|---|
| "We need to refactor" | "Current code will slow new features by 3x; 2-week investment now saves 8 weeks by Q3" |
| "We need monitoring" | "We currently find out about outages from customer complaints; we need to find them ourselves in under 2 minutes" |
| "Microservices migration" | "Right now, every deployment touches all of our system. We want teams to deploy independently, so we ship 3x faster with less risk" |
| "We need load testing" | "We don't know how our system will behave on Black Friday. This test tells us before it matters." |
| "Tech debt" | "We've been building fast; these shortcuts that worked at 10 users don't work at 1M. Addressing them now costs less than the incidents they'll cause later." |

### When Asked for Timeline Estimates
- Never give a number off the top of your head
- "I'd need to look at the code/data to give you a reliable estimate. I can have that for you by [tomorrow/end of week]."
- When you do estimate: give ranges, list assumptions, flag risks that could change the estimate
- "2–4 weeks, assuming we don't discover schema migration issues and the vendor API is stable"
