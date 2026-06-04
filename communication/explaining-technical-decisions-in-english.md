# Explaining Technical Decisions in English

## What It Is

Explaining technical decisions in English is the skill of describing, **out loud and in clear language**, why your team chose one technical approach over the alternatives. It is not a presentation, not a written ADR (Architecture Decision Record), and not a buzzword pitch — it is a structured conversation that an interviewer, a non-technical PM, or a peer engineer can follow without prior context.

A complete decision explanation includes five elements, in this order:

1. **Context** — what system, what constraints, what the team was trying to achieve.
2. **Options** — what alternatives were on the table (and why).
3. **Decision** — what you actually chose.
4. **Trade-offs** — what you gave up by choosing it.
5. **Outcome** — what happened (or what you'd watch for) once it was in production.

For a .NET / Azure engineer, this skill shows up every day: defending a PR comment, explaining to your tech lead why you picked Azure Service Bus over Event Grid, walking a new hire through middleware ordering in `Program.cs`, or telling an interviewer why your team uses a modular monolith instead of microservices.

The "in English" part matters specifically for non-native speakers. The structure is the easy part; finding the right English phrasing under interview pressure is the hard part.

## Why It Exists

Software decisions are rarely "right" or "wrong" — they are trade-offs against constraints. A senior engineer who cannot articulate those constraints sounds like a junior who just copied a pattern. Interviewers, tech leads, and architecture review committees use decision explanations to evaluate:

- **Judgment** — did you actually consider alternatives, or did you reach for the first tool you knew?
- **Self-awareness** — can you name the cost you accepted?
- **Communication range** — can you explain the same decision to a PM, an SRE, and a junior engineer, adjusting depth without losing meaning?
- **Operational thinking** — did you consider what happens at 2 AM when this decision fails?

Without this skill, even excellent technical work gets misread. An engineer who says "We chose Cosmos DB because it scales" sounds shallow. An engineer who says "We chose Cosmos DB over Azure SQL because we needed multi-region writes with <50 ms read latency for the product catalog, and we were willing to give up complex joins and pay roughly 3x the per-RU cost compared to the SQL equivalent" sounds senior.

## When To Use It

Explaining decisions clearly is needed almost continuously in a backend role:

- **In interviews**: system design rounds, follow-up questions in coding rounds, behavioral questions about architecture, "tell me about a recent project" prompts.
- **In code reviews**: defending or critiquing a PR's structural choices (not just style).
- **In architecture / design reviews**: presenting a proposal to peers or a guild.
- **In ADRs and design docs**: the written form of the same skill.
- **In standup updates**: explaining why a task changed shape mid-sprint.
- **In incident postmortems**: explaining why a past decision contributed to the outage and what the team will change.
- **In customer conversations**: when a customer asks "why do we have to wait 24 hours for that report?" (the answer is usually an architectural trade-off).

## Why It Is Important

For mid-level engineers, decision explanation is the **most reliable promotion signal**. Calibration committees, hiring panels, and tech leads consistently report that two engineers with similar coding skill get different leveling based on how they reason about choices.

Specifically:

- **Interviewers will trust your code more** if your reasoning is clear, even when the code itself has minor issues.
- **PR reviewers will approve faster** because they can see your reasoning instead of inferring it.
- **Cross-team trust** is built when other teams understand your architectural choices and don't have to re-litigate them.
- **Postmortem learning** spreads through the org only if past decisions are explained well enough for others to apply the lesson.
- **In English specifically**, fluency at this skill multiplies the impact of every other skill — your code review comments land harder, your design docs get read, and your name gets associated with "thinks clearly".

## How It's Used in C# / .NET

A .NET / Azure engineer is constantly making decisions that are highly contextual. Here are common categories and the English phrasing patterns that work for each.

### Decision categories you'll explain often

| Decision | Typical pairs you'll compare |
|---|---|
| Hosting | Azure App Service vs Azure Container Apps vs AKS vs Azure Functions |
| Data store | Azure SQL vs Cosmos DB vs Azure Table Storage vs PostgreSQL |
| Messaging | Azure Service Bus vs Event Grid vs Event Hubs vs direct HTTP |
| Architecture style | Monolith vs modular monolith vs microservices |
| API style | REST vs gRPC vs GraphQL |
| Async pattern | `async/await` vs `BackgroundService` vs Azure Functions queue trigger |
| ORM strategy | EF Core tracked vs `AsNoTracking()` vs Dapper for hot paths |
| DI lifetime | `Transient` vs `Scoped` vs `Singleton` |
| Caching | In-process `IMemoryCache` vs Azure Cache for Redis vs CDN |
| Auth | JWT bearer vs cookies vs Managed Identity vs API keys |
| Secrets | `appsettings.json` vs Azure Key Vault vs Managed Identity |

### Reusable English sentence templates

**Opening — naming the context**:
- "At the time, the system was…"
- "The constraint we were optimizing for was…"
- "The business requirement that drove the decision was…"
- "We had two specific non-negotiables: X and Y."

**Listing alternatives without dismissing them**:
- "We seriously considered three options…"
- "The alternative was Y, which would have been simpler, but it didn't satisfy [constraint]."
- "Option A was attractive because of X; we didn't choose it because of Y."

**Stating the decision**:
- "We chose X because…"
- "After comparing them, we landed on X."
- "Our decision was X, on the understanding that we'd revisit it if traffic grew past N requests per second."

**Naming trade-offs honestly**:
- "The cost we accepted was…"
- "We gave up X in exchange for Y."
- "It's not free — we now have to maintain…"
- "The downside that surprised us later was…"

**Reflecting on outcome**:
- "Six months in, the decision held up because…"
- "In hindsight, I'd probably do X differently because…"
- "We had to revisit it once, when…"
- "If I were designing this fresh today, I'd…"

### Worked example — ASP.NET Core middleware ordering

> "Middleware in `Program.cs` runs in the order it's registered, and that order matters because each middleware can short-circuit the pipeline. We put authentication before authorization because authorization needs the user identity; we put exception handling outermost so it can convert anything thrown deeper into a `ProblemDetails` response; we put response caching after authentication so we don't cache responses across users. The trade-off of caring about order is that adding a new middleware requires thinking about where it belongs, but the alternative — middleware that 'sort of works' in production — is worse."

### Worked example — EF Core query strategy

> "For our order history endpoint, we use `AsNoTracking()` because the data is read-only for the response — we don't need EF's change tracker, and it cuts memory by about 30%. We project to a DTO with `Select` rather than returning the entity directly, which avoids over-fetching columns we don't render. We chose this over Dapper because the rest of the codebase already uses EF Core, and the consistency was worth the small performance gap. If this endpoint becomes a hot spot, we'd reconsider — but we'd measure first."

### Worked example — choosing an Azure tier

> "We're on the Azure App Service Standard tier. We picked it over Premium because we don't need VNet integration for this workload, and we picked it over Basic because we need the deployment slots for blue-green releases. Standard gives us auto-scale up to 10 instances, which covers our peak. The trade-off is no Premium-tier features like private endpoints — we accepted that because the API is public-facing anyway. If our compliance posture changes and we need to put it behind a private endpoint, we'll move to Premium V3."

### Worked example — modular monolith vs microservices

> "We chose a modular monolith for the new product because we're a 6-person team and we need to ship the first version in three months. Microservices would have given us independent deploys, but they would also have required us to set up service discovery, distributed tracing, schema-versioned events, and an integration test environment that crosses service boundaries — none of which we'd built yet. The modular monolith gives us logical separation through assembly boundaries and DI scopes; if a module needs to scale independently later, we can extract it. The cost we accepted is that a slow query in one module can affect the others until we extract it."

## Advantages

- **Differentiates senior signal** from junior signal in interviews.
- **Speeds up code review and architecture review** because reviewers can see reasoning, not just code.
- **Builds personal credibility** — "X explains things clearly" is one of the most career-positive reputations to have.
- **Reduces miscommunication** with PMs, customers, and other teams.
- **Forces you to actually think** — explaining badly often reveals a decision you didn't really make.
- **Creates teaching artifacts** — your decision explanations become onboarding material for new hires.
- **Transfers to writing** — same structure works for ADRs, design docs, and incident reviews.

## Disadvantages

- **Time investment** — preparing 5-10 polished decision narratives takes hours.
- **Over-explanation risk** — under interview pressure, candidates often run long and bury the point.
- **Can sound rehearsed** if you don't adapt to the specific question.
- **Translation friction** for non-native English speakers — getting natural phrasing takes deliberate practice.
- **Sensitivity** — explaining a decision honestly may require admitting a teammate's or manager's mistake; you have to phrase carefully without blaming.
- **Asymmetric difficulty** — explaining decisions you made yourself is easier than explaining decisions someone else made (but interviewers ask about both).

## Common Mistakes

### 1. Jumping Straight To The Tool

**Weak answer**:
> "We used Azure Service Bus because it's reliable."

The interviewer learned nothing. Why did you need reliability? What was the alternative? At what cost?

**Strong answer**:
> "Our order API needed to notify both the billing service and the inventory service when an order was placed. We considered three options: direct HTTP calls (rejected because billing has 30-second downtimes during deploys), Azure Event Grid (rejected because we needed ordered delivery per customer), and Azure Service Bus topics (chosen). Service Bus gave us at-least-once delivery, dead-letter queues, and per-customer session ordering. The trade-off was operational — we had to add a dead-letter monitor and idempotency on the consumer side."

### 2. Skipping The Trade-Off

**Weak answer**:
> "Cosmos DB is great because it scales globally with low latency, so we chose it."

This sounds like marketing copy. No engineering decision is cost-free.

**Strong answer**:
> "We chose Cosmos DB for the product catalog because we needed multi-region reads under 50 ms for our European and Asian users. The trade-offs we explicitly accepted were: per-RU cost roughly 3x our equivalent Azure SQL workload, no rich SQL joins (we denormalized the schema), and eventual consistency on reads outside the write region. We're comfortable with eventual consistency because product data only changes a few times per day, and stale reads of a few seconds are invisible to users."

### 3. Using Buzzwords Instead Of Reasoning

**Weak answer**:
> "We adopted CQRS, Clean Architecture, and microservices to be cloud-native."

This signals that you copied patterns without understanding them.

**Strong answer**:
> "We separated reads from writes in the order service because the read load was 50x the write load, and the read model needed fields from three different bounded contexts. We didn't go full CQRS — we just have a denormalized read model in Cosmos DB updated by domain events from the SQL write model. We kept everything else as plain ASP.NET Core services."

### 4. Talking In "We" Without Owning Your Role

**Weak answer**:
> "We thought about it, we considered options, we chose, we deployed."

The interviewer can't tell what *you* contributed. Did you make the decision? Did you write the proposal? Did you implement it?

**Strong answer**:
> "I wrote the design doc and proposed the three options. The tech lead and I made the final call together. I implemented the Service Bus integration and ran the load test that validated the throughput assumption."

### 5. Defending The Decision When The Interviewer Probes Weaknesses

**Weak answer**:
> "No, I don't think there were any real downsides — it was the right choice."

This signals lack of self-awareness. Every decision has downsides.

**Strong answer**:
> "The biggest downside has been onboarding cost — new engineers take about a week longer to ramp up because they have to learn the outbox + worker pattern before they can ship anything. We mitigated it with a one-page diagram in the repo README, but it's a real cost."

### 6. Failing To Adjust For The Audience

A PM doesn't need to hear about `IServiceCollection.AddScoped`. A staff engineer doesn't need to hear that Service Bus has dead-letter queues. The same decision should be explained at different depths.

**For a PM**:
> "We added a queue between the order API and the billing service so that if billing is slow, customer orders still go through immediately, and billing catches up later. The trade-off is that customers might see 'order placed' before they see the billing confirmation by a few seconds."

**For a senior backend engineer**:
> "We added a Service Bus topic with the order API as publisher and billing as subscriber. We use sessions for per-customer ordering, dead-letter for failures, and the consumer is idempotent on the order ID. The order API commits to SQL and writes to the outbox in the same transaction; a background worker drains the outbox to Service Bus."

### 7. Vague Numbers Or No Numbers

**Weak answer**:
> "It made things faster."

**Strong answer**:
> "P95 latency went from 1.8 seconds to 220 milliseconds, and we cut compute cost by about 40% by dropping from 6 to 3 App Service instances."

## Best Practices

### The COD-T-O Framework

A reusable structure for any decision explanation:

| Step | Question to answer | Time budget |
|---|---|---|
| **C — Context** | What was the system, what were the constraints? | 20 sec |
| **O — Options** | What alternatives did you consider? | 30 sec |
| **D — Decision** | What did you choose? | 15 sec |
| **T — Trade-offs** | What did you give up? | 30 sec |
| **O — Outcome** | What happened? What would you do differently? | 25 sec |

Target total: **~2 minutes spoken**.

### Practice "Why" Five Times

For any decision in your past work, ask "why" five times until you reach the business constraint. If your decision dead-ends at "because it's best practice", you don't really understand it.

### Prepare Three Tiers Of Depth

For each major decision in your portfolio, have a 30-second version, a 2-minute version, and a 10-minute deep-dive ready. Interviewers will signal which they want.

### Use Concrete Numbers Wherever Possible

Memorize: request rate, latency P50/P95/P99, error rate, $ cost per month, team size, time-to-ship. Decisions sound credible only when grounded in numbers.

### Build A "Pattern Library" Of Phrases

Keep a personal cheat sheet of English phrasings that work for you. Reuse them across interviews until they feel natural. Examples:

- "The reason we landed on X was…"
- "We seriously considered Y; the deal-breaker was…"
- "The cost we knowingly accepted was…"
- "Six months later, the decision still holds because…"
- "Honestly, in hindsight…"

### Record And Listen

Once a week, record yourself explaining one technical decision. Listen back. Notice filler words, unclear pronouns, missing trade-offs. This is the single highest-leverage practice for non-native English speakers.

### Cite Evidence, Mark Uncertainty

If you're uncertain about a fact, say so explicitly: "I think Cosmos DB charges per RU/s — I'd verify the exact pricing before quoting it to a customer." This signals maturity, not weakness.

## Related Concepts

- **[explaining-trade-offs.md](explaining-trade-offs.md)** — the same skill, focused on the cost side.
- **[handling-system-design-questions.md](handling-system-design-questions.md)** — decision explanation is the backbone of system design rounds.
- **[behavioral-questions-for-dotnet-developers.md](behavioral-questions-for-dotnet-developers.md)** — many behavioral answers turn on a decision narrative.
- **[code-review-discussion-questions.md](code-review-discussion-questions.md)** — same skill applied to defending or critiquing code.
- **Architecture Decision Records (ADRs)** — the written form of the same skill.
- **[../architecture/clean-architecture.md](../architecture/clean-architecture.md)** — a frequent topic of decision conversations.
- **[../architecture/monolith-vs-microservices.md](../architecture/monolith-vs-microservices.md)** — the most asked architectural decision in interviews.
- **C4 model / 4+1 architectural views** — frameworks for structuring written and visual explanations.
- **Active listening** — half of explaining is hearing what the interviewer is actually asking.

## Real-World Usage

### In Interviews

- **System design rounds**: every choice (SQL vs Cosmos, sync vs async, monolith vs microservices) becomes a 2-minute COD-T-O explanation.
- **Behavioral rounds**: "Tell me about a tough technical decision" — pure decision narrative.
- **Coding round follow-ups**: "Why did you pick that data structure?" — same skill, smaller scale.

### In Team Standups And Reviews

- Defending a design proposal at a guild meeting.
- Explaining why a sprint's scope changed (a decision the team made mid-sprint).
- Walking a new hire through the codebase's structural choices.

### In Code Reviews

- A review comment like "Why did you use `IEnumerable<T>` here instead of `IReadOnlyList<T>`?" deserves a one-paragraph decision explanation, not a one-line justification.

### In Postmortems

- A blameless postmortem is, in part, a decision review: "We chose X with assumption Y; assumption Y turned out to be false under load Z; here's what we'll do differently."

### In Customer Conversations

- A customer asking "why does this take 24 hours?" usually wants a decision explanation in business language: "We batch reports overnight to keep your transactional cost predictable; we can move to near-real-time on a higher tier."

## Code Example — Before and After

### Sample Answer — Weak vs Strong

**Question**: "Why did you choose to use Azure Service Bus in your last project?"

#### Weak Answer (transcript)

> "Azure Service Bus is a really powerful messaging system that's good for enterprise scenarios. It's reliable and it scales. We used it because we needed messaging between services. It worked well for us."

What's wrong:
- No context: what services, what scale, what problem?
- No alternatives considered.
- "Enterprise scenarios" is meaningless filler.
- No trade-offs named.
- No outcome.

#### Strong Answer (transcript)

> "Sure. At the time, we were building the checkout flow for a mid-size e-commerce platform — about 50 orders per second at peak. The checkout API needed to trigger three downstream effects: charge the customer in the payment service, reserve inventory, and send a confirmation email. (**Context**)
>
> We considered three options. (**Options**) The first was direct HTTP calls from checkout to each downstream service — we rejected it because the email service had occasional 30-second pauses during deploys and we didn't want that to fail customer checkout. The second was Azure Event Grid, which we use elsewhere for low-volume notifications — we rejected it for this case because we needed strict ordering per customer (you can't send a refund email before the order email) and Event Grid doesn't guarantee that. The third was Azure Service Bus topics with sessions enabled.
>
> We chose Service Bus topics with the customer ID as the session key, so each customer's messages process in order. (**Decision**)
>
> The trade-offs we accepted: (**Trade-offs**) operationally, we now own a dead-letter monitoring alert and a manual reprocessing tool — we built both during the rollout. Architecturally, we had to make every consumer idempotent on the order ID, which added about a day of work per consumer. Cost-wise, Service Bus standard tier is around $10 a month for this volume, which was a rounding error.
>
> Six months in, the decision is holding up. (**Outcome**) We've had two cases where billing was down for an hour during a deploy and Service Bus buffered everything — zero customer impact. The dead-letter queue has caught about a dozen poison messages, mostly malformed test events. In hindsight, I'd have built the dead-letter monitor earlier; we built it after a near-miss where messages sat there for two days unnoticed. That's now in our service template for any new Service Bus consumer."

What's right:
- Specific volume, specific services, specific business stake.
- Three named alternatives with one-line rejection reasons.
- Decision is one sentence and unambiguous.
- Three categories of trade-off (ops, code, cost), each quantified.
- Outcome includes a concrete lesson learned.
- ~2 minutes spoken.

## Interview Questions and Answers

### 1. Walk me through how you'd explain a recent architecture decision to a non-technical PM.

**Why this matters**: Senior signal. Engineers who can only talk to other engineers cap out at mid-level.

**Answer**: "I'd start by naming the business outcome the decision protected. For example, if we recently moved order processing from synchronous to event-driven via Azure Service Bus, I'd tell the PM: 'We added a queue between order placement and billing so that if billing has a slow moment, customers can still complete checkout instantly — billing catches up within seconds. The visible change is that the order confirmation page might appear a moment before the billing confirmation email. We considered keeping it synchronous, but every billing slowdown would have shown up as failed checkouts.'

I'd avoid words like 'asynchronous', 'idempotency', 'dead-letter'. Those exist for engineers. For a PM, the right vocabulary is 'customer-visible', 'reliability', 'recovery time', and dollar impact."

**Real project**: A checkout flow where moving to event-driven cut peak-hour checkout failures from 0.8% to under 0.05%.

### 2. How would you explain why you chose ASP.NET Core minimal APIs over MVC controllers?

**Why this matters**: Tests whether you can defend a specific .NET decision without dogma.

**Answer**: "For a small internal microservice with 8-10 endpoints, we chose minimal APIs because the boilerplate is lower, the routing is collocated with the handler, and the resulting `Program.cs` reads as a table of endpoints. We pair them with FluentValidation and a small `IEndpointGroup` extension to keep handlers organized.

For our larger customer-facing API, with 60+ endpoints, model binders, custom action filters, and a real need for OpenAPI grouping, we stayed on MVC controllers. The convention-based structure helps when many engineers are working in the same codebase.

The honest trade-off: minimal APIs are easy to make a mess of if you let the file grow. The discipline to split endpoints into separate static methods or modules is on the team — the framework doesn't enforce it the way controllers do."

**Trade-off**: Minimal APIs are leaner but less self-organizing; controllers are heavier but more opinionated.

### 3. Explain how you decide between Azure SQL Database and Cosmos DB for a new service.

**Why this matters**: The most common data-store decision in Azure interviews. Probes whether you understand consistency, scale, query model, and cost.

**Answer**: "I start with three questions. First, what's the access pattern? If it's mostly key-or-range lookups and the data is naturally hierarchical (like a catalog item with nested options), Cosmos with a thoughtful partition key fits well. If it's heavy relational joins, reporting, or full-text search, Azure SQL is a much better fit.

Second, what's the consistency requirement? Azure SQL gives strong consistency essentially for free; Cosmos forces an explicit choice among five consistency levels, and the right one depends on tolerance for stale reads. For an order ledger I want strong consistency in a single region. For a product catalog I'm fine with session or bounded staleness across regions.

Third, what's the cost model the team can reason about? Azure SQL is DTUs or vCores, which the team can predict. Cosmos is RU/s, which is easy to underestimate — a single unindexed query can burn 10x what a well-shaped one does, and your bill grows quietly.

In my last project, we chose Azure SQL for the order system (strong consistency, complex joins, our team's deep SQL skill) and Cosmos for the product catalog (multi-region reads, simple document shape, no joins needed). Different stores for different bounded contexts is fine — what matters is the rationale per store."

**Trade-off**: Operating two stores doubles operational surface (backup, monitoring, IaC), but trying to force one store to do both jobs is usually worse.

### 4. How would you explain why your team is on a modular monolith instead of microservices?

**Why this matters**: Architecture maturity. Engineers who blindly advocate microservices signal junior.

**Answer**: "We're an 8-person team and we shipped V1 in four months. Microservices would have required us to invest first in service discovery, distributed tracing, a contract-testing setup, a deployable integration environment per service, and event schema governance. None of that work would have moved customer outcomes in V1.

A modular monolith gives us logical separation through .NET project boundaries, internal `internal` visibility per module, and DI registration boundaries per module. We treat each module as if it could be extracted later — no shared mutable state, only domain events crossing module boundaries, separate database schemas per module.

The trade-off we accepted is that a slow query in one module can affect another (shared process, shared DB connection pool), and that we can't deploy modules independently. We've been fine — we deploy 5-10 times a day and have no per-module scaling pressure. When we hit one, we'll extract that module, and because we've kept the boundaries clean, that extraction is weeks of work, not months."

**Real project**: 8-person team, B2B SaaS, ~200 paying customers, ~5 RPS peak.

### 5. How would you defend a controversial decision like "we deleted our unit tests"?

**Why this matters**: An extreme case that tests honesty and structural reasoning. You may have to defend a decision you didn't even agree with at the time.

**Answer**: "I'd be honest about the context. On that team, our test suite had drifted into a state where 60% of failures were flaky environment issues, and the team had stopped trusting it. Engineers were merging on red. We had two real options: invest two months to repair the suite, or delete it and rebuild incrementally.

We deleted it. The reasoning: a test suite the team ignores is worse than no suite, because it produces false confidence. We immediately introduced a discipline that every new PR had to include tests, and we wrote tests for any bug we fixed. After three months we had ~400 reliable tests instead of 4,000 unreliable ones, and the team trusted them.

In hindsight, I'd probably have done a hybrid — deleted the flaky ones, kept the green ones. We were too aggressive. But I'd defend the underlying decision: when a process has lost credibility, restoring it sometimes requires a reset, not a polish."

**Trade-off**: Decisive resets create short-term risk; preserving broken processes creates long-term decay.

### 6. Walk through how you'd explain a complex technical decision in a written ADR.

**Why this matters**: The written counterpart of the same skill. Many companies require ADRs for any significant decision.

**Answer**: "I use a standard ADR template with five sections.

**Title** — short, decision-oriented: 'Use Azure Service Bus topics for inter-service events'.

**Status** — Proposed / Accepted / Superseded with date.

**Context** — what's true about the system today, what business pressure prompted the decision. This is the longest section. I write it assuming the reader joins the team in two years and has never heard of the project.

**Decision** — what we're doing, in one paragraph. No conditionals.

**Consequences** — explicitly positive, negative, and neutral. I force myself to write at least three negatives. If I can only think of positives, I haven't thought hard enough.

I also add an **Alternatives Considered** section because future readers always want to know what else was on the table.

The biggest mistake I see in ADRs is writing them after the fact as PR justification. Real ADRs are written *while* the decision is being made — they're a tool for thinking, not a tool for documentation."

**Trade-off**: Writing ADRs takes 1-2 hours per decision. Skipping them saves time today and costs much more time in re-litigation later.

### 7. How do you explain a decision when the original constraints have changed?

**Why this matters**: Real systems outlive their original assumptions. Senior engineers can frame the past honestly without blaming.

**Answer**: "I separate the past decision from the current situation. For example: 'When we built the order service three years ago, our peak was 5 RPS and we were a 3-person team. SQL Server with EF Core was the right call — minimal operational surface, the team knew it cold, and the read load was easy to serve.

Today we're at 200 RPS peak, the team is 15 engineers, and read load is dominating the database. The original decision was correct in its context; the context has changed. We're now planning to add a read replica and a Cosmos-backed read model for the high-volume product lookups, while keeping SQL as the system of record for orders.

I make a point of saying the original decision was correct then, because junior engineers often look at legacy code as 'mistakes' when it was usually 'fit-for-the-moment'."

**Trade-off**: Reframing the past charitably costs you nothing and builds trust with whoever made the original call.

### 8. The interviewer says "I'd have done it differently — defend your choice." How do you respond?

**Why this matters**: Interviewers sometimes apply pressure to see if you fold under disagreement.

**Answer**: "First, I'd genuinely consider their alternative — sometimes they spot something I missed. I might say: 'Interesting — let me think about what that would look like in our constraints. With X approach, we'd get [benefits], but we'd need to handle [cost]. The reason we ruled it out at the time was [specific constraint]. Has that constraint changed in what you're describing?'

If their alternative is genuinely better, I say so: 'You're right, that's a stronger approach for [reason]. If I were redesigning this today, I'd seriously consider it.' Senior signal isn't being right — it's updating in public when you're wrong.

If after thinking it through I still believe my original choice was correct, I defend it with the same evidence: 'I see why X is appealing — for our specific constraint of Y, though, I think the original choice still holds because of Z. I'd love to be wrong here — what's the experience you're drawing on?'

Either way, I treat it as collaboration, not combat."

**Trade-off**: Disagreeing politely with an interviewer is risky if done poorly and high-value if done well. Practiced phrases ('that's a stronger approach for…', 'has that constraint changed?') make it land naturally.

## Summary Checklist

- [ ] I can structure any decision explanation using Context → Options → Decision → Trade-offs → Outcome within 2 minutes.
- [ ] I name at least two alternatives I seriously considered, with one-line rejection reasons.
- [ ] Every decision I describe includes at least one explicit trade-off — never "no downsides".
- [ ] I quantify outcomes with numbers (latency, error rate, $, RPS, team size).
- [ ] I adjust the depth of my explanation for the audience (PM, peer engineer, junior, staff/principal).
- [ ] I use specific .NET / Azure vocabulary correctly (App Service tiers, Service Bus features, EF Core query strategies, DI lifetimes).
- [ ] I have a personal library of English sentence templates I've practiced out loud.
- [ ] I can defend a decision under disagreement without becoming defensive, and I update publicly when the alternative is stronger.
- [ ] I can mark uncertainty explicitly ("I think it's X — I'd verify before quoting it") instead of bluffing.
- [ ] I can write the same explanation as an ADR (Architecture Decision Record) with Context, Decision, Consequences, and Alternatives sections.
