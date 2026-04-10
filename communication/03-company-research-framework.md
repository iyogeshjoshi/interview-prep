# Communication — Company Research Framework

## Why Research Matters at Principal Level

At senior/staff/principal level, interviewers expect you to show up knowing something about their stack, scale, and challenges. "Tell me about yourself" and generic questions are red flags for Principal candidates who should be asking sharp, specific questions that show preparation.

A well-researched candidate:
- Connects their experience to the company's actual problems
- Asks questions that signal understanding of the domain
- Doesn't waste time on things easily found on the website

---

## Research Checklist (2–3 hours before interview)

### 1. Engineering Blog & Tech Talks
```
Search: "[company name] engineering blog"
Search: "[company name] tech talk site:youtube.com"
Search: "[company name] engineering medium.com"

What to look for:
  - What stack are they running? (Node.js? Kotlin? Go?)
  - What problems have they written about? (scale, reliability, migration)
  - What decisions did they make and why? (shows values)
  - Recent posts (last 6 months) signal current priorities

Examples of great engineering blogs:
  Cloudflare Blog, Netflix Tech Blog, Shopify Engineering, Stripe Blog
```

### 2. Job Postings (On the Team You're Joining)
```
Check: their careers page + LinkedIn jobs

What to look for:
  - What technologies are listed? (confirms stack)
  - What problems are they trying to solve?
  - How do they describe the role's scope? (team size, ownership)
  - Other open roles on the team → what skills are missing → what you might own

Signal: if they're hiring for 5 SREs and an observability engineer, 
reliability is a current pain point.
```

### 3. LinkedIn Research
```
Search: "[company] engineer" → look at profiles of engineers on the team

What to look for:
  - Team size (how many engineers match that description?)
  - Tenure (high turnover is a signal)
  - What did former employees do after leaving?
  - Who is the hiring manager? What's their background?
    → Understand what they value (ex-FAANG may value rigor; startup background may value pace)
```

### 4. GitHub (if OSS / public repos exist)
```
Search: github.com/[company]

What to look for:
  - Primary languages used
  - Active vs stale repos (shows how they work in public)
  - Issues and PRs → actual engineering discussion
  - Architecture patterns in public code (monorepo? microservices? IaC?)
  - README quality → signals documentation culture
```

### 5. Glassdoor & Blind
```
What to look for:
  - Interview process description (helps you prepare)
  - Common complaints → things to probe in your own questions
  - Common praise → things to validate you care about
  
Treat with skepticism: reviews skew negative; read for patterns, not individual posts.
Signal: if 10 reviews mention "poor on-call culture", ask about it directly.
```

### 6. Their Product (Use It)
```
If it's a consumer/B2B product you can access:
  - Sign up and use it for 30 minutes
  - Note: what's slow? What feels polished? What seems held together with tape?
  - Think: what technical challenges does this UI suggest?
    (e.g., real-time features → WebSocket infra; complex filtering → search layer)

Shows: genuine curiosity; concrete observations in the interview
```

### 7. Recent News
```
Search: "[company name] site:techcrunch.com OR site:venturebeat.com"
Search: "[company name] funding OR acquisition OR launch 2024 2025"

What to look for:
  - Recent funding → growth phase → likely scaling challenges
  - Acquisitions → integration complexity → technical debt
  - Product launches → new areas → new infrastructure needs
  - Layoffs → efficiency pressure → cost optimization a concern
```

---

## Building Your Research Notes (Template)

```markdown
## [Company Name] — Interview Research

### Stack (confirmed)
- Backend: Node.js (TypeScript), Go for data pipeline
- Frontend: React, Next.js
- Infra: AWS (EKS, Aurora, MSK/Kafka, CloudFront)
- Observability: Datadog, PagerDuty

### Scale (from blog posts)
- ~10M DAU, 500M events/day
- ~80 engineers, 12 product teams

### Known Technical Challenges (from blog, job postings)
- Migrating from monolith → microservices (3 posts about this in 2024)
- Real-time notification system (recently rebuilt)
- Scaling their search from Elasticsearch to... (open question)

### Key People
- Hiring Manager: [Name] — came from Stripe, 3 years here, tweets about platform engineering
- CTO: [Name] — gave talk at AWS re:Invent 2024 on their event-driven architecture

### My Angle / Relevance
- My MFE experience directly relevant (they're building a platform team)
- Built similar event-driven checkout at [prev company] — same pattern they wrote about
- Have used Kafka at scale — matches their MSK usage

### Questions to Ask
1. "I read your post about migrating to microservices — where are you in that journey now and what's the next hard problem?"
2. "What does the engineering team look like around the Principal role — how many teams would this person work across?"
3. "How are technical decisions made here — is there an RFC process, or does it happen more organically?"
```

---

## Connecting Research to Interview Answers

### In System Design
```
Generic: "I'd use Kafka for the event streaming layer."

With research: "I noticed from your engineering blog that you already use MSK for 
your order pipeline. I'd extend that pattern here — producers push events to a topic, 
consumers handle downstream processing with dead-letter queues for failure cases. 
Does that match how you're approaching it on the current system?"
```

### In Behavioral
```
Generic: "I care about developer experience."

With research: "I saw you recently posted about rebuilding your deployment pipeline. 
That's something I've done — at [company] we were at 2-hour build times and got to 
12 minutes by [specific changes]. If you're still in that journey, I'd love to talk 
about what worked and what didn't."
```

### In Questions to Ask
```
Generic: "What's the tech stack?"
(They already told you. Don't ask this.)

Sharp: "Your Q4 2024 post mentioned moving to federated GraphQL — where did that land? 
I've run into the distributed tracing challenges that come with federation; 
is that something you've solved?"
```

---

## Red Flags to Watch For (During Research)

| Signal | What it might mean |
|---|---|
| Engineering blog hasn't posted in 2+ years | Engineering culture may not prioritize knowledge sharing, or team is heads-down in survival mode |
| Job posting lists 15+ required technologies | Either inexperienced hiring, or truly chaotic stack |
| High turnover on LinkedIn (avg tenure < 1yr) | Management, culture, or stability concerns |
| Glassdoor: consistent "on-call is brutal" | Reliability problems; ask directly in interview |
| CTO just left | Potential direction change; ask about technical vision stability |
| Product looks rough technically | Either opportunity (greenfield mandate) or liability (survival mode) |

### How to Probe Red Flags in Interview
```
Don't: "Your Glassdoor reviews say on-call is terrible."

Do: "How does on-call work here — what does a typical week look like for an 
engineer in this team? Is there a goal for number of pages per week?"

The answer (and the comfort with the question) tells you what you need to know.
```

---

## Day-Of Preparation

### The Night Before
- Re-read your research notes (30 min)
- Review 3 stories you'll tell using STAR (pick your strongest)
- Prepare your system design whiteboard setup (have a framework in your head)
- Prepare 5–7 questions to ask (you won't use all of them, but you need backup)

### Morning Of
- Recheck the role description — any specific wording that tells you what they care about?
- Have water nearby (voice matters)
- Know your first sentence for "Tell me about yourself" — this sets tone

### "Tell Me About Yourself" (2-minute version)
```
Structure:
  1. Current role + scope (1 sentence)
  2. Most relevant accomplishment to THIS company (1-2 sentences)
  3. Why THIS role / company now (1 sentence)

Example:
"I'm currently a Principal Engineer at [company], where I lead the platform 
architecture for our checkout and payments domain — a system processing 
$2B annually across 5 countries. Most recently I redesigned our event-driven 
order pipeline which cut our P99 latency by 60% and eliminated the class of 
data inconsistency issues we'd been fighting for 2 years.

I'm drawn to [this company] specifically because you're at the scale where 
those same distributed systems problems become critical, and I saw in your 
engineering blog that you're actively working through the microservices 
migration — that's exactly the kind of problem I want to be working on."
```
