# Handling System Design Questions

## What It Is

System design questions ask you to **design a non-trivial backend system in 45-60 minutes** with an interviewer who is mostly listening. Typical prompts: "Design a URL shortener", "Design Twitter's timeline", "Design Uber's dispatch system", "Design a notification service", "Design an e-commerce checkout". The interviewer is rarely looking for a "right" architecture — they're looking for **how you reason** about an open-ended problem.

A well-handled system design answer follows a predictable structure:

1. **Clarify requirements** (5-7 minutes) — functional and non-functional, with explicit scale numbers.
2. **Capacity estimation** (3-5 minutes) — back-of-envelope: QPS, storage, bandwidth.
3. **API design** (3-5 minutes) — the contract clients call.
4. **Data model** (5-7 minutes) — entities, relationships, partition keys.
5. **High-level architecture** (10 minutes) — boxes and arrows showing services, data stores, queues.
6. **Deep-dive on 1-2 components** (10-15 minutes) — driven by the interviewer's interest.
7. **Trade-offs and scaling** (5-10 minutes) — what fails as you grow, how you mitigate.

For a .NET / Azure engineer, the answer should be grounded in the stack you actually know — App Service, Azure SQL, Cosmos DB, Service Bus, Event Hubs, Redis, Front Door — rather than generic boxes labeled "API" and "queue".

## Why It Exists

System design rounds exist because coding rounds and behavioral rounds together don't tell hiring committees everything they need to know. Specifically:

- **Can the candidate operate at the level of services and data stores**, not just methods and classes?
- **Do they think about scale, failure, cost, and operations** before they think about cleverness?
- **Can they reason under ambiguity** — most real engineering work is open-ended, like this round?
- **Are they aware of the operational reality** of distributed systems (network failures, partial outages, eventual consistency)?
- **Do they communicate well enough** to align a team on an architecture?

For mid-to-senior roles, the system design round is often the **single highest-weight signal** in the loop. A strong coder with a weak system design round will typically be capped at SDE2; a candidate with the inverse profile is often hired in despite a softer coding round.

For .NET / Azure roles specifically, interviewers expect you to bring Azure vocabulary to the table. Generic "we'd use a load balancer and a database" is acceptable for a junior; "we'd front this with Azure Front Door for global routing, terminate TLS there, and rate-limit at the edge before hitting App Service" is senior.

## When To Use It

The structured system design approach is needed in:

- **Coding interview loops** for any backend role above entry level — almost always one dedicated 45-60 minute round.
- **Architecture interviews** for senior+ roles — sometimes 90 minutes with a panel.
- **"Tech screen" coding rounds** that include a 15-minute design tail.
- **On the job**: writing an RFC for a new service, leading a design review, proposing a re-architecture.
- **Internal promotions**: senior promotion packets typically require a design artifact (RFC, ADR, or technical proposal).
- **Customer-facing technical sales calls** when explaining how your platform handles a use case.

## Why It Is Important

System design skill is the dividing line between "engineer who codes well" and "engineer who can lead a team". Concretely:

- **Hiring outcomes** swing on this round — strong system design candidates routinely get offers from companies that decline candidates with stronger coding.
- **Leveling** at most companies caps engineers who can't design systems at the SDE2 / mid-level band, regardless of coding skill.
- **Cross-team trust** is built when other teams can read your designs and reason about how they interact with theirs.
- **Operational outcomes**: well-designed systems have fewer incidents, shorter MTTR, and lower long-term cost.
- **Customer outcomes**: B2B SaaS sales often depend on whether your architecture can support a customer's needs — answered by a system design conversation.

Communication is half the score. A candidate who designs the perfect system but explains it poorly will lose to a candidate who designs a less-optimal system but communicates choices clearly.

## How It's Used in C# / .NET

A .NET / Azure backend engineer is expected to map the generic system design vocabulary to specific Azure services. Knowing the mappings cold turns "I'd use a load balancer" into "I'd front this with Azure Front Door for global routing, with Application Gateway behind it for path-based routing and WAF".

### Generic concept → Azure service mapping

| Generic concept | Default Azure choice | When to consider alternatives |
|---|---|---|
| HTTP API | ASP.NET Core on App Service | Container Apps for KEDA scale; AKS for sidecars |
| Background worker | `BackgroundService` on App Service or Container Apps | Azure Functions for event-driven; AKS for heavy custom needs |
| Event-driven function | Azure Functions (Isolated worker, .NET) | Container Apps Jobs for longer runtimes |
| Relational DB | Azure SQL Database (Hyperscale for large) | PostgreSQL Flexible Server if team prefers |
| Document/NoSQL | Azure Cosmos DB (NoSQL API) | Table Storage for cheap/simple key-value |
| Cache | Azure Cache for Redis | In-process `IMemoryCache` for low-scale |
| Pub/Sub | Azure Service Bus topics | Event Grid for low-volume notifications |
| Stream / firehose | Azure Event Hubs | Service Bus is wrong; Event Hubs handles millions/sec |
| Object storage | Azure Blob Storage | Azure Files for SMB workloads |
| CDN / edge | Azure Front Door (Premium) | Azure CDN classic for simpler cases |
| Search | Azure AI Search | Cosmos full-text if simple enough |
| Secrets | Azure Key Vault + Managed Identity | App Configuration for non-secret config |
| Identity | Microsoft Entra ID (Azure AD) | B2C for customer-facing |
| Monitoring | Application Insights | Log Analytics + Workbooks for ops |

### Reusable English phrasing for system design

**Opening — clarifying questions**:
- "Before I sketch anything, let me make sure I understand what we're optimizing for."
- "Can you tell me a bit about the scale? Roughly how many users, requests per second, data volume?"
- "Is there a specific failure mode you want me to focus on?"

**Capacity estimation**:
- "Let me do quick back-of-envelope math…"
- "Assuming 1 million daily active users, each doing 10 actions, that's roughly 100 actions per second on average, peaking maybe 5x — 500 per second."
- "I'd round those generously since we usually under-estimate peak."

**High-level architecture**:
- "At a high level, I'd put four pieces in front of the data: an API tier, a worker tier, a cache, and the durable store."
- "Requests would flow like this: Front Door → App Service → Redis → Azure SQL."
- "The key boundary I'm drawing here is between synchronous request handling and asynchronous processing."

**Deep-diving when prompted**:
- "Happy to go deeper — there are three places I think are interesting: the partition strategy, the consistency model, and the failover plan. Which would you like first?"
- "Let me sketch the partition key choice in detail…"

**Acknowledging trade-offs**:
- "The cost of this choice is X — I think it's acceptable because Y."
- "We're explicitly favoring availability over strong consistency here for [reason]."

**Handling pushback**:
- "Good point — if the traffic profile changes that way, the design pivots. Here's what I'd change…"
- "I hadn't considered that. Let me think through the implications…"

## Advantages

- **Highest-leverage interview round** for senior leveling.
- **Transfers directly to job work** — RFCs, ADRs, architecture reviews all use the same skill.
- **Reveals depth that coding can't** — you can't fake "what happens when this Cosmos partition gets hot?" if you haven't actually run Cosmos.
- **Practiceable** — the question pool is finite (~30-50 canonical questions cover 90% of interviews).
- **Builds confidence** — a polished system design answer makes the rest of the loop easier.
- **Forces operational thinking** — what fails, how you detect it, how you recover.
- **Communication signal** — your phrasing and structure matter as much as your choices.

## Disadvantages

- **High variance in interviewer expectations** — some want depth on one component, others want breadth across the whole system.
- **Time pressure is brutal** — 45 minutes is barely enough for a real system.
- **Easy to spend too long on requirements** and run out of time for architecture.
- **Easy to skip requirements** and design the wrong system.
- **Stack bias** — if you've only used AWS, mapping mid-interview to Azure terminology is hard; same in reverse.
- **Overconfidence kills** — candidates who jump to a "standard" microservices design without listening to the prompt usually fail.
- **Whiteboard / virtual whiteboard skills matter** — drawing clear diagrams under pressure is itself a skill.

## Common Mistakes

### 1. Jumping To Architecture Before Clarifying

**Weak start**:
> "OK, for a URL shortener I'd use a load balancer, a microservice, a database, and a cache."

This signals you've memorized a template and aren't listening to the actual question.

**Strong start**:
> "Before I draw anything, can I check a few things? How many URLs do we expect to shorten per day? What's the read/write ratio — typical short-URL services are 100:1 read-heavy. Do shortened URLs expire? Do we need analytics on clicks? Is custom-alias support required? Are we global or single-region?"

### 2. Skipping Capacity Estimation

**Weak**:
> "We'll need a database and a cache."

What size? Throughput? Storage? Without numbers, your design has no constraints.

**Strong**:
> "100 million URLs per day means about 1,200 writes/sec average, 12,000 peak. Reads at 100:1 are 120k average, 1.2 million peak. Each record is ~200 bytes including overhead, so storage grows ~20 GB per day, ~7 TB per year. That immediately rules out a single-region single-instance SQL setup for the read path — we need either a sharded relational store or a NoSQL store with horizontal partitioning."

### 3. Designing With Generic Boxes

**Weak**:
> "There's an API, a database, a queue, and a worker."

This works for a junior. For a .NET / Azure role, it signals you're not familiar with the actual stack.

**Strong**:
> "I'd put Azure Front Door at the edge for global routing and TLS termination. Behind it, ASP.NET Core on Azure App Service Premium V3 in two paired regions with traffic split 70/30. Writes go to a regional Azure SQL with active geo-replication. Reads hit Azure Cache for Redis (Premium with geo-replication) first, falling back to a read replica. For the async write path — analytics, expiration cleanup — I'd use Azure Service Bus and a `BackgroundService` worker."

### 4. Ignoring Failure Modes

**Weak**:
> "Once data is in the database, we're done."

Real systems fail constantly. Senior signal is naming the failure modes upfront.

**Strong**:
> "Three failures matter here. (1) Redis is down — we degrade to direct SQL reads with elevated latency, alert on cache miss rate above 20%. (2) SQL primary fails — automatic failover via geo-replication, with RPO of about 5 seconds; we accept losing the last few writes. (3) An entire region is unreachable — Front Door automatically routes to the healthy region. We'd test all three quarterly via game days."

### 5. Drawing The Same "Microservices" Diagram For Every Problem

A URL shortener doesn't need 12 microservices. A notification service might. The right architecture depends on the actual problem.

**Strong**:
> "For this volume and scope, I'd start with a single modular service, not microservices. The redirect path is so hot and so simple that splitting it adds latency without adding much value. If the analytics workload grew into its own bounded context with its own team, I'd extract it later."

### 6. Not Engaging With Pushback

Interviewers will challenge you to see if you cave or reason.

**Weak**:
> "Yeah, you're probably right, I'll change it."

**Strong**:
> "Good push — let me think about that. If we move to per-customer partitioning in Cosmos, we get hot partition protection but lose the ability to query across customers cheaply. For the dashboard use case you're describing, that matters. Let me revise: I'd keep the main store partitioned by customer for transactional access, and feed a separate read model — maybe Azure SQL Hyperscale — for cross-customer reporting queries."

### 7. Running Out Of Time

A common failure mode: spending 30 minutes on requirements and capacity, then panicking through the actual architecture in 10. Manage time actively.

**Strong**:
> "I want to make sure we hit the architecture. Let me lock in the requirements I have and move to the high level — I can refine assumptions if needed."

### 8. Forgetting Cost

Especially for Azure interviews, cost awareness is senior signal.

**Strong**:
> "Premium Service Bus is about $700/month vs Standard at $10/month. We'd start with Standard and move to Premium only if we hit the messaging quotas or need geo-disaster-recovery."

## Best Practices

### The 7-Step Framework

A reusable structure that fits 45-60 minutes:

| Step | Duration | What you do |
|---|---|---|
| 1. Clarify requirements | 5-7 min | Functional + non-functional + scale numbers |
| 2. Capacity estimation | 3-5 min | QPS, storage, bandwidth — back-of-envelope |
| 3. API design | 3-5 min | Public contract: endpoints, payloads, status codes |
| 4. Data model | 5-7 min | Entities, relationships, partition keys, indexes |
| 5. High-level architecture | 10 min | Boxes + arrows of services, stores, queues |
| 6. Deep-dive on 1-2 areas | 10-15 min | Whatever the interviewer probes |
| 7. Trade-offs + scale + failure | 5-10 min | What breaks at 10x, how you'd address |

### Lead With Numbers

Every step benefits from numbers — even rough ones. "100 million writes/day" is more useful than "high traffic". Numbers force design constraints.

### Pick Defaults, Mark Alternatives

Don't agonize on every choice. Pick a default (e.g., Azure SQL for relational data), say "I'd default to Azure SQL here; I'd consider Postgres if the team prefers, or Cosmos if multi-region writes become a constraint", and move on.

### Talk While You Draw

The interviewer wants narration, not silence. Even bad diagrams are fine if your narration is clear.

### Manage Scope Aggressively

If the interviewer says "design Uber", you do not have time to design every Uber service. Pick the **dispatch** flow as the core, mention the other flows briefly, and offer to deep-dive whichever the interviewer wants.

### Prepare 5-8 Canonical Designs

Walk through these end-to-end before any interview season. Most questions are variations of these:

| Canonical | Variant of |
|---|---|
| URL shortener | Read-heavy key-value with cache |
| Twitter / news feed | Fanout and timeline assembly |
| Notification service | Multi-channel pub/sub with delivery guarantees |
| Chat / messaging | Real-time + persistence + presence |
| Ride-share dispatch | Geospatial matching + low-latency state |
| E-commerce checkout | Distributed transaction + payment + inventory |
| File storage (Dropbox) | Object storage + sync + dedup |
| Video streaming | CDN + transcoding pipeline |
| Rate limiter | Distributed counters with consistency trade-off |
| Search autocomplete | Trie + caching + ranking |

### Always Cover Non-Functional Requirements

Functional ("what does it do") is half the picture. Non-functional ("how well must it do it") drives architecture:

- **Latency** (P50, P95, P99 targets)
- **Throughput** (peak QPS, sustained QPS)
- **Availability** (SLA target: 99.9%, 99.99%)
- **Consistency** (strong vs eventual, where each is needed)
- **Durability** (RPO, RTO)
- **Security & compliance** (PII, PCI, GDPR)
- **Cost** (rough $ / month or unit economics)

### Use The 4+1 Or C4 Mental Model

For longer rounds, structure your verbal walk-through using a known model:

- **C4**: Context (who/what) → Container (services/stores) → Component (modules within a service) → Code (rarely needed in interviews).
- **4+1**: Logical, Process, Development, Physical, plus Scenarios.

You don't need to draw all views — just be aware of the layers so you can switch between them when the interviewer asks "how would this be deployed?"

### Practice Out Loud, On A Whiteboard

System design is a verbal-and-visual skill. Reading articles is necessary; speaking through full designs (out loud, drawing on a whiteboard or tablet) is what builds the muscle.

## Related Concepts

- **[explaining-technical-decisions-in-english.md](explaining-technical-decisions-in-english.md)** — every system design choice is a decision explanation in miniature.
- **[explaining-trade-offs.md](explaining-trade-offs.md)** — system design is dense with trade-offs.
- **[../architecture/microservices-architecture.md](../architecture/microservices-architecture.md)** — common system design choice point.
- **[../architecture/monolith-vs-microservices.md](../architecture/monolith-vs-microservices.md)** — the canonical architecture trade-off.
- **[../architecture/event-driven-architecture.md](../architecture/event-driven-architecture.md)** — common pattern in system design answers.
- **[../architecture/scalability-design.md](../architecture/scalability-design.md)** — scaling reasoning is half of system design.
- **[../architecture/reliability-design.md](../architecture/reliability-design.md)** — failure modes and recovery.
- **[../azure/azure-service-bus.md](../azure/azure-service-bus.md)**, **[../azure/azure-sql.md](../azure/azure-sql.md)**, **[../azure/app-service.md](../azure/app-service.md)** — Azure-specific knowledge for design rounds.
- **C4 model, 4+1 architectural views, ADRs** — frameworks for structuring designs.
- **CAP theorem, PACELC** — consistency vs availability vocabulary.

## Real-World Usage

### In Interviews

- **Standalone system design round** (most common): 45-60 minutes, one prompt.
- **Mixed design + coding round**: 30 minutes coding, 30 minutes design.
- **Architecture deep-dive for senior roles**: 90+ minutes with a panel; expect to defend choices under repeated probing.

### In RFC / ADR Writing

The same 7-step structure works as a written design doc: Context → Requirements → Capacity → API → Data → Architecture → Trade-offs → Open Questions.

### In Architecture Review Meetings

Walking a peer team through your proposed design uses the same skill — talk through choices, accept pushback, name trade-offs.

### In Customer Sales Calls

When a customer asks "can your platform handle 10x our current load?", you essentially do a 5-minute system design walk-through of the relevant subsystem.

### In Postmortems

A blameless postmortem includes a "what would we design differently" section — same skill applied to the past.

## Code Example — Before and After

### Sample Answer — Weak vs Strong

**Question**: "Design a notification service that supports email, SMS, and push for an e-commerce platform with 5 million users."

#### Weak Answer (transcript)

> "OK so I'd have an API, a queue, and workers. The API receives a notification request, puts it on the queue, and workers pull from the queue and send. I'd use Azure for everything. We'd need a database to store notification status. That's basically it."

What's wrong:
- No requirements clarification.
- No capacity estimation.
- No API design.
- Generic boxes with no Azure specificity.
- No failure handling.
- No trade-offs.
- About 30 seconds; should be 30+ minutes.

#### Strong Answer (transcript, condensed)

> "Let me start with requirements. (**Clarify**)
>
> Functional: trigger notifications from various events (order placed, shipped, delivered, password reset). Support email, SMS, and mobile push. User preference per channel. Templates with personalization. Delivery status visible in admin. Optional rate limiting per user to avoid spam.
>
> Non-functional: 5M users, peak around Black Friday probably 100x baseline. Latency target: notification should fire within 10 seconds of the event — not real-time but not minutes. Email delivery is not in our hands once handed to the provider — we monitor accepted vs bounced. We want at-least-once delivery, with idempotency on the consumer to avoid duplicates. PII concerns mean we need encryption at rest and in transit. SLA: 99.9% for the API, best-effort for downstream providers.
>
> Capacity. (**Estimate**) 5M users, say 2 notifications/user/week average = 10M/week = ~17 notifications/sec average. Peak at maybe 100x for a Black Friday email blast = ~1,700/sec sustained. Burst could be tens of thousands/sec when a marketing campaign fires. Storage: ~30 days retention of delivery status, ~1KB per record = ~360M records × 1KB = 360GB. Manageable.
>
> API design. (**API**) Single endpoint: `POST /api/notifications`. Body has `userId`, `eventType`, `payload` (data for template), optional `channels` to override default. Returns 202 Accepted with a `notificationId` for status tracking. Separate `GET /api/notifications/{id}` for status. Internal-only, authenticated via Microsoft Entra ID with managed identity from the calling service.
>
> Data model. (**Data**) Two stores. (1) Notification record in Cosmos DB partitioned by `userId`: `id`, `userId`, `eventType`, `channels[]`, `status` per channel, `createdAt`, `attemptCount`. (2) User preferences in Azure SQL — joins to customer data. Templates live in Blob Storage with a metadata index in SQL.
>
> Architecture. (**High-level**) Let me sketch:
>
> ```
> Producers → API (App Service) → Service Bus topic ("notifications")
>                                         ↓
>                          ┌──────────────┼──────────────┐
>                       Email sub      SMS sub       Push sub
>                       (Worker A)    (Worker B)    (Worker C)
>                          ↓             ↓             ↓
>                    SendGrid /        Twilio       FCM/APNS
>                    Azure ACS
>                          ↓
>                    Cosmos DB (status)
> ```
>
> The API is ASP.NET Core on App Service Premium V3. It validates, persists the initial record to Cosmos, and publishes to Service Bus. Three subscriptions on the topic — one per channel — fan out to three `BackgroundService` workers in Container Apps with KEDA autoscale on queue depth. Each worker calls the channel provider, updates the status record, and ACKs the message. Failures retry with exponential backoff; after N retries the message goes to dead-letter and an alert fires.
>
> Deep-dive — let me cover idempotency and rate limiting since they're the interesting parts. (**Deep-dive**)
>
> Idempotency: the producer can retry, so the API uses an `Idempotency-Key` header. If we've seen it, we return the cached response. Inside, each Service Bus message carries the `notificationId` and the channel; the worker uses the pair as an idempotency key against Cosmos before sending. Cosmos's optimistic concurrency on the `ETag` prevents double-send.
>
> Rate limiting: per-user, per-channel, per-hour. We could do this in Redis with a sliding window counter — increment on send attempt, check before send. The trade-off is that Redis adds latency to the send path. Alternative: enforce at the API tier with the same Redis check before publishing — saves the message bus a round trip but means we reject at ingress.
>
> Trade-offs and scale: (**Trade-offs**) at 5M users we're easily inside single-region Azure capacity. At 50M, I'd consider regional sharding by `userId` hash with a regional Service Bus per region. At 500M, I'd reconsider Cosmos partitioning and probably move to a denormalized read model for status queries. Cost-wise, the dominant line item is the channel providers (SendGrid, Twilio); Azure infrastructure is probably under $3k/month at this scale.
>
> Failure modes: (**Failure**) Service Bus down → API returns 503, producer retries. Channel provider down → exponential backoff, dead-letter, ops alert. Cosmos write fails after channel send → the channel sent successfully but we'll retry the whole message, which is why idempotency on the channel call matters. We should add a reconciler job that scans for status updates older than X minutes and tries to query the provider for the actual delivery state."

What's right:
- All 7 framework steps covered, with clear time allocation.
- Specific Azure services with rationale (App Service Premium V3, Container Apps with KEDA, Cosmos partition strategy, Service Bus topics).
- Real numbers (5M users, 17 reqs/sec, 1,700 peak, ~360 GB).
- API contract explicit, data model explicit.
- Deep-dive prepared for idempotency and rate limiting (probable interviewer probes).
- Trade-offs named with watch signals (5M → 50M → 500M evolution).
- Failure modes addressed.
- ~25-30 minutes spoken with whiteboard.

## Interview Questions and Answers

### 1. Walk me through how you'd design a URL shortener.

**Why this matters**: Canonical first system design question. Tests fundamentals: read-heavy caching, key generation, scale estimation, storage choice.

**Answer**: "I'd clarify first: writes per second, read/write ratio, custom alias support, expiration, analytics, single vs multi-region. Assuming 100M URLs/day, 100:1 reads vs writes, no custom aliases, 1-year expiration, click analytics, and global users.

Capacity: 100M writes/day ≈ 1,200/sec average, ~12k peak. Reads at 100:1: 120k average, 1.2M peak. Storage: 100M × 200B × 365 days ≈ 7TB / year.

API: `POST /shorten` returns short code; `GET /{code}` 301 redirects.

Key generation: I'd use Base62 encoding of a distributed counter (Snowflake-style) — 7 chars gives 3.5 trillion unique codes. Avoid hashing because of collision handling cost.

Data model: Cosmos DB partitioned by short code (which is uniformly distributed by construction). Single document per URL: `{ code, longUrl, ownerUserId, createdAt, expiresAt }`. Click analytics flow separately.

Architecture: Azure Front Door at edge (cache hot redirects at edge for sub-10ms latency); ASP.NET Core API on App Service in two regions; Azure Cache for Redis Premium with geo-replication for warm cache; Cosmos DB multi-region with strong consistency on writes, session consistency on reads. Click events fire-and-forget to Event Hubs, processed by an Azure Function into a Cosmos analytics container partitioned by short code.

Trade-offs: Front Door caching for hot URLs gives huge cost wins (1.2M reads/sec mostly served at edge for cents); the cost is cache invalidation if URLs are deleted — I'd accept short staleness. At extreme scale (10x current), I'd reconsider key generation — distributed counter may become a bottleneck, in which case I'd partition the counter per region with a region-prefix in the code."

**Real project example**: Modeled on URL services like bit.ly, which publicly reports billions of clicks/day.

### 2. Design Twitter's home timeline.

**Why this matters**: Tests fanout-on-write vs fanout-on-read, the classic asymmetric workload problem.

**Answer**: "Clarify: 300M MAU, posts per user per day (say 0.5 average), follow graph (median ~200, but celebrities have millions), real-time vs eventual, retention.

Two designs:

**Fanout-on-write** (push): when a user posts, write the post ID into every follower's timeline. Read is fast (just read pre-built timeline). Cost: a celebrity with 50M followers triggers 50M writes per post — infeasible.

**Fanout-on-read** (pull): when a user opens timeline, fetch posts from everyone they follow and merge. Read is slow, write is cheap.

**Hybrid** (the realistic answer): fanout-on-write for normal users; fanout-on-read for celebrities. A user's timeline is the merge of their pre-fanned-out queue plus a live pull from celebrities they follow.

Architecture in Azure: posts land in Cosmos DB partitioned by `authorId`. A fanout worker (Azure Functions queue trigger) reads each new post and writes timeline entries to per-user Redis lists for normal authors. For celebrity authors, no fanout — readers merge at query time. Timeline API fetches the Redis list plus celebrity posts and returns the merged page.

Trade-offs: hybrid adds complexity (two code paths, threshold for 'celebrity'). Pure fanout-on-write is simpler but doesn't scale to celebrities. Pure fanout-on-read kills read latency. The hybrid is what Twitter actually does and what the interviewer wants you to discover."

**Trade-off**: Read latency vs write amplification vs operational complexity — the hybrid trades complexity for both ends working acceptably.

### 3. Design an e-commerce checkout flow that handles 50 orders per second at peak.

**Why this matters**: Tests distributed transactions, idempotency, the most common backend pattern.

**Answer**: "Clarify: checkout includes cart validation, inventory reservation, payment authorization, order creation, and downstream effects (email, shipping, inventory commit). Peak 50/sec is modest — single-region single-database is plenty.

The hard part isn't throughput; it's correctness under partial failure. A user pressing 'Place Order' must result in exactly one order, even if the network drops or services restart.

Architecture: ASP.NET Core checkout API on App Service. The API does the synchronous part — cart validation, payment authorization, write to SQL — in a single database transaction using the Outbox pattern: in the same transaction, write the order row AND outbox rows for downstream events. A `BackgroundService` worker drains the outbox to Service Bus topics: `OrderPlaced`, `PaymentCaptured`, etc. Inventory, email, and shipping services subscribe.

Key safeguards: idempotency keys on the API (clients send `Idempotency-Key` header; we cache the response in Redis with a 24h TTL). Optimistic concurrency on inventory rows. Payment provider also supports idempotency keys — we use the same key.

Failure modes: payment succeeds but DB commit fails — we'd refund via reconciliation job. DB commit succeeds but Service Bus publish fails — the outbox worker retries until success, so downstream services eventually see the event. Service Bus delivers a message twice — the consumer checks the order ID before processing.

Trade-offs: the outbox pattern adds a worker and a polling delay (~1 sec) but it's the only honest way to coordinate a SQL commit with message publish. Pure 2PC across SQL and Service Bus doesn't exist in Azure. Choreography (sagas) vs orchestration is the bigger question for downstream — for a small flow like this, choreography (each service handles its event) is simpler; for more complex flows, a saga orchestrator becomes worth it."

**Real project**: The Outbox + saga pattern is what I've used in production for two e-commerce platforms.

### 4. Design a ride-share dispatch system (like Uber).

**Why this matters**: Tests geospatial queries, low-latency state, and real-time matching — harder than CRUD-style designs.

**Answer**: "Clarify: city scale to start (say 10k drivers, 100k riders/day). Match a rider to the nearest available driver within seconds. Drivers update location every 5 seconds while online. Surge pricing as a separate concern.

Core challenge: geospatial queries at scale. Storing driver locations in SQL and running `ST_DWithin` per request is too slow at scale.

Architecture: drivers' apps publish location updates to Event Hubs every 5 seconds. An Azure Function consumer writes the latest location to Redis with a geo index (`GEOADD`). Rider's app calls dispatch API with their pickup coords. Dispatch service does a `GEORADIUS` against Redis to find the 5 nearest available drivers, then queries Cosmos to filter by car type / rating / driver preferences. Sends the trip offer to one driver; on accept/reject, falls back to the next.

State machine: trip moves through `Requested → Offered → Accepted → Started → Completed`. Each transition is an event written to Cosmos partitioned by `tripId`. Real-time updates to the rider's app via Azure SignalR.

Scaling concerns: Redis geo index is great until you hit ~hundreds of thousands of active drivers per region; at that point shard by H3 geohash region. Event Hubs handles the location firehose comfortably at millions of events per second.

Trade-offs: Redis is in-memory and not durable for driver location — that's actually fine because locations are constantly refreshed; losing them on Redis restart just means a few seconds of degraded matching. For trip state we need durability, so Cosmos. Two stores is more ops but each is right-tool-for-the-job.

Failure: if dispatch API can't reach Redis, fall back to recent locations from the trip-state store (slower but correct). If matching takes >10 seconds, expand search radius and surface 'no drivers available' message."

**Real project**: Geospatial dispatch is well-documented in Uber engineering blogs; the Azure mapping is direct.

### 5. How would you handle 'design a system that scales to 10x current traffic'?

**Why this matters**: A common follow-up after the initial design. Tests scaling intuition and bottleneck identification.

**Answer**: "I'd identify the bottleneck order, because not everything scales at the same rate.

First: database. Most systems hit DB before they hit anything else. Read scaling — add a Redis cache, then read replicas. Write scaling — partition (sharding) by a natural key, or move write-heavy collections to Cosmos with partition by `customerId`.

Second: web tier. Usually horizontally scalable — add instances, push autoscale rules. App Service autoscale on CPU or queue depth; Container Apps with KEDA.

Third: synchronous fan-out. If service A synchronously calls B, C, D, then 10x A's traffic means 10x B, C, D's traffic. The fix is async — Service Bus between A and the downstreams.

Fourth: hot keys. Sharding doesn't help if 80% of traffic goes to one shard. Identify hot keys, consider replicating just those.

Fifth: dependencies you don't control. Payment provider, email provider, external APIs — they often have rate limits. Backpressure (queues with bounded consumers) is the fix.

For each bottleneck I'd estimate: when do we hit it (at what RPS), what's the fix, how long does the fix take. That gives the team a 'next thing to fix' roadmap rather than a panic when the wall hits."

**Trade-off**: Premature optimization vs reactive scaling. Knowing the next bottleneck lets you fix in advance with a feature flag instead of under outage pressure.

### 6. How do you handle the situation where the interviewer asks about a technology you don't know?

**Why this matters**: Honesty under pressure. Most interviewers respect "I don't know but here's how I'd approach it" more than bluffing.

**Answer**: "I'd be honest and pivot to what I do know. 'I haven't used Kafka in production, but I've used Azure Event Hubs which solves a similar problem — appending to a partitioned log with consumer groups. If you want, I can design with Event Hubs and we can map back to Kafka concepts, or you can give me a quick orientation on the Kafka-specific constraints you care about.'

This signals: (1) I know what I know and what I don't, (2) I have an analogous mental model, (3) I'm comfortable with mid-conversation course-correction. Most interviewers will pick one of the two options and you continue from there.

Bluffing — claiming you've used something you haven't, then fumbling the follow-up — is a fast way to fail the round."

**Real project**: I've used this exact phrasing when interviewers ask about DynamoDB; I bridge to Cosmos and we proceed.

### 7. How do you push back when the interviewer suggests a design choice you disagree with?

**Why this matters**: Tests whether you can defend a position without being defensive.

**Answer**: "I'd genuinely consider their suggestion first — they often see something I missed. I'd say: 'Interesting — let me think through it. If we use [their suggestion], we'd get [benefits], but we'd need to handle [costs]. The reason I leaned toward [my original] was [specific constraint]. Is that constraint still in play in what you're describing?'

If they have a good reason and I missed it, I update: 'You're right, that's a stronger fit for [reason] — let me revise.' Updating in public is senior signal, not weakness.

If I still think mine is correct after considering, I defend with evidence: 'I see why [theirs] is appealing — for our specific case of [constraint], I think mine still holds because [reason]. I'd love to be wrong here — what's the experience you're drawing on?'

Either way, I treat it as a collaborative whiteboard session, not an adversarial one."

**Trade-off**: Capitulating immediately makes you seem unsure; defending blindly makes you seem rigid. The middle is genuine consideration plus evidence-based response.

### 8. The interviewer says you have 5 minutes left. How do you wrap up?

**Why this matters**: Time management is itself an interview signal.

**Answer**: "I'd consolidate: 'Let me summarize where we are and call out what I'd want to come back to with more time. The core architecture is [X], we covered [Y] in depth, and I'd want to deep-dive on [Z]. The things I'd flag for a real implementation are: [list 2-3 risks I haven't fully addressed].'

This shows: I know what I built, I know what I didn't address, and I'm comfortable saying 'incomplete' instead of pretending I covered everything. Most interviewers leave the last 5 minutes open for candidate questions — having a clean wrap-up before that conversation lands well."

**Trade-off**: Five extra minutes of design vs five minutes of articulate summary — the summary is almost always the higher-value use of the time.

## Summary Checklist

- [ ] I follow a consistent 7-step framework (Clarify → Estimate → API → Data → Architecture → Deep-dive → Trade-offs).
- [ ] I always clarify functional AND non-functional requirements before designing.
- [ ] I do explicit back-of-envelope capacity estimation with real numbers.
- [ ] I use specific Azure services (App Service, Service Bus, Cosmos, etc.) instead of generic boxes.
- [ ] I cover idempotency, retries, dead-letter, and failure modes as first-class concerns.
- [ ] I name trade-offs explicitly for every major choice, with the deciding constraint and watch signal.
- [ ] I have 5-8 canonical designs (URL shortener, Twitter, e-commerce, notifications, ride-share) practiced end-to-end.
- [ ] I can do a deep-dive on partitioning, consistency model, or scaling when the interviewer probes.
- [ ] I manage time actively and produce a clean summary when 5 minutes remain.
- [ ] I can push back on interviewer suggestions with evidence, and update gracefully when their suggestion is better.
