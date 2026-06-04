# Monolith vs Microservices

## What It Is

**Monolith** and **microservices** are two ends of a spectrum for how you package and deploy a backend system.

- A **monolith** is a single deployable artifact — one `.dll`, one container, one Azure App Service, one process — that contains every feature. All modules share a process, a memory space, and (typically) a database. Calls between modules are **in-process method calls**.
- A **microservices** architecture splits the system into many small, independently deployable services, each owning its own data, its own deployment pipeline, and its own team. Calls between services are **over the network** (HTTP, gRPC, Service Bus).

Between them sits the **modular monolith** — a single deployable, but internally organized into well-defined modules with explicit boundaries (separate projects, separate schemas, no cross-module navigation in EF Core). It is often the right answer for systems that *might* become microservices later.

```
┌─────────────────────────┐     ┌──────────┐  ┌──────────┐  ┌──────────┐
│   Orders.Api (one app)  │     │ Orders   │  │ Payments │  │ Shipping │
│  ┌───────┐ ┌──────────┐ │     │  Service │  │  Service │  │  Service │
│  │Orders │ │Payments  │ │     └─────┬────┘  └────┬─────┘  └────┬─────┘
│  └───────┘ └──────────┘ │           │            │             │
│  ┌──────────┐           │           └─── Azure Service Bus ────┘
│  │Shipping  │           │           (events, commands, sagas)
│  └──────────┘           │
└────────┬────────────────┘
         │
   ┌─────▼──────┐                   ┌────┐  ┌────┐  ┌────┐
   │ One SQL DB │                   │ DB │  │ DB │  │ DB │
   └────────────┘                   └────┘  └────┘  └────┘
        MONOLITH                          MICROSERVICES
```

The choice is not "old vs modern" — it is a trade between **simplicity** and **independence**.

## Why It Exists

Late-2000s enterprise .NET teams hit a recurring wall: monoliths that had grown for 5–10 years became impossible to release safely. Deployments took hours, regression risk was high, scaling required spinning up the entire system, and a bug in the reporting module could take down checkout. The industry response — popularized by Netflix, Amazon, and Sam Newman's *Building Microservices* (2015) — was to split systems along business boundaries so each team could ship independently.

By 2018, the pendulum swung back. Teams that adopted microservices without the maturity to operate them (no distributed tracing, no service mesh, no saga discipline) ended up with **distributed monoliths** — all the operational cost of microservices and none of the independence. Martin Fowler coined the phrase *"MonolithFirst"*: start with a monolith, extract microservices only when the boundaries are obvious and the operational cost is justified.

The two patterns exist to solve different problems: monoliths optimize for **simplicity** and **transactional consistency**; microservices optimize for **independent deployment**, **team autonomy**, and **per-service scaling**. Choosing wrong is one of the most expensive architectural mistakes a team can make.

## When To Use It

### Use a (modular) monolith when:

- The team has **fewer than ~15 engineers** total.
- The system is **young** and domain boundaries are still shifting.
- **Transactional consistency** across business capabilities is important (orders + inventory + payments in one transaction).
- The team's **operational maturity is moderate** (no service mesh, no centralized tracing, no saga framework yet).
- **Latency budgets** are tight and network hops are expensive.
- Time-to-market matters more than independent scaling.

### Use microservices when:

- Multiple teams (3+ teams of ~5 engineers each) need to **deploy independently** without coordination meetings.
- Different parts of the system have **dramatically different scaling profiles** (a public catalog service handling 50k req/s next to an admin service handling 5).
- Different parts use **different technology stacks** for genuine reasons (Python for ML inference, Go for an edge proxy, .NET for the business core).
- The domain is **mature** — bounded contexts are stable enough that you will not constantly redraw service boundaries.
- The org has **production-grade observability** (Application Insights / OpenTelemetry, distributed tracing, centralized logging) and **resilient messaging** (Service Bus, sagas).

### Use a modular monolith — the middle ground — when:

- You expect the system to grow but cannot yet justify the operational cost of microservices.
- You want a "future microservices" exit strategy without paying the cost up front.
- You want clear internal boundaries (one project per module, one schema per module, no cross-schema queries) without crossing the network boundary.

## Why It Is Important

The choice drives five concrete production outcomes:

1. **Release cadence and risk.** A monolith ships everything at once — releases are higher-risk but coordinated. Microservices ship independently — daily releases per service are normal, blast radius is small, but coordinating a cross-cutting change becomes a saga.
2. **Failure modes.** A monolith fails as a unit — all features go down together. Microservices fail in patterns — circuit breakers, retries, dead-letter queues, partial degradation. Both require operational skill, but very different skills.
3. **Data consistency.** Inside a monolith, multi-table transactions are trivial (`SaveChangesAsync`). Across microservices, **two-phase commits do not work in cloud environments** — you accept eventual consistency and design with sagas + outbox + idempotent consumers.
4. **Cost.** A small monolith might run on a single Azure App Service B-tier (≈ $50/mo). The equivalent functionality split across 10 microservices typically costs 5–10× more in compute, networking, observability tooling, and engineering time.
5. **Team topology.** **Conway's Law** says system architecture mirrors org structure. One team → one deployable usually works. Six teams → one deployable becomes a constant integration nightmare. The architecture either follows the org or fights it.

Picking the wrong style is rarely fatal — but the cost of correcting it (extracting services from a tangled monolith, or merging microservices back into a modular one) is months of work. Choosing well from the start saves a year.

## How It's Used in C# / .NET

### Monolith — single ASP.NET Core app, internal modules

```
src/
  Orders.Api/                  // entry point, Program.cs, controllers
  Orders.Module.Orders/        // Order aggregate, handlers
  Orders.Module.Payments/      // Payment aggregate, Stripe adapter
  Orders.Module.Shipping/      // Shipment aggregate
  Orders.Module.Notifications/ // Email + push
  Orders.Infrastructure/       // One DbContext, one DB

// Program.cs
builder.Services.AddOrdersModule();
builder.Services.AddPaymentsModule();
builder.Services.AddShippingModule();
builder.Services.AddNotificationsModule();
app.MapOrdersEndpoints();
app.MapPaymentsEndpoints();
```

Modules communicate via **MediatR** in-process. A `PlaceOrderCommand` handler can call `_mediator.Send(new ReserveInventoryCommand(...))` and the work is one method call — same process, same transaction.

### Modular monolith — schema-per-module, no cross-module queries

```csharp
public sealed class OrdersDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.HasDefaultSchema("orders");
        mb.ApplyConfiguration(new OrderConfig());
    }
}

public sealed class PaymentsDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.HasDefaultSchema("payments");
        mb.ApplyConfiguration(new PaymentConfig());
    }
}
```

Two `DbContext`s, two schemas in one SQL Server database. Modules expose **public contracts** (commands, events, query DTOs) and keep entities `internal`. The compiler enforces the boundary — `Payments.Module` cannot reference `Orders.Order` directly. This is the easiest path to microservices later.

### Microservices — separate processes, separate databases, message bus

```
services/
  orders-api/        (Azure App Service)
    Orders.Domain
    Orders.Application
    Orders.Infrastructure (Azure SQL: orders-db)
    Orders.Api
  payments-api/      (Azure App Service)
    Payments.Domain ... Payments.Api (Azure SQL: payments-db)
  shipping-worker/   (Azure Functions, Service Bus triggered)
  notifications-fn/  (Azure Functions, Service Bus triggered)
```

Services communicate via **Azure Service Bus** topics and subscriptions. `Orders.Api` publishes `OrderConfirmedEvent` via the **outbox**; `Payments.Api`, `Shipping.Worker`, and `Notifications.Fn` each have their own subscription. Saga state lives in `Payments.Api` for the "charge → fulfill → notify" flow.

```csharp
// Orders.Api — confirms the order, writes the event to the outbox
public async Task<Result> Handle(ConfirmOrderCommand cmd, CancellationToken ct)
{
    var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
    if (order is null) return Result.NotFound();
    order.Confirm();
    _outbox.Enqueue(new OrderConfirmedIntegrationEvent(order.Id, order.CustomerId, order.Total));
    await _uow.SaveChangesAsync(ct); // atomic: order + outbox row
    return Result.Ok();
}

// Payments.Api — consumes via Service Bus
public sealed class OrderConfirmedHandler(IPaymentService payments)
{
    [Function(nameof(OrderConfirmedHandler))]
    public Task Run(
        [ServiceBusTrigger("order-confirmed", "payments-sub", Connection = "Sb")] OrderConfirmedIntegrationEvent evt,
        CancellationToken ct) =>
        payments.ChargeForOrderAsync(evt.OrderId, evt.Amount, ct);
}
```

### Key .NET infrastructure for microservices

- **Azure Service Bus** — topics + subscriptions for pub/sub, sessions for ordered delivery, dead-letter queues for poison messages.
- **MassTransit / NServiceBus** — saga support, retries, scheduled messages.
- **Polly** — retry, circuit breaker, bulkhead policies for HTTP/gRPC calls.
- **OpenTelemetry + Application Insights** — distributed tracing across service boundaries.
- **YARP** or **Azure API Management** — API gateway pattern.
- **Dapr** (optional) — sidecar abstraction over pub/sub, state, secrets.
- **Azure Container Apps** or **AKS** — orchestration for containerized services.

### Migrating monolith → microservices via Strangler Fig

```csharp
// API gateway routes /api/orders/* to the legacy monolith,
// /api/payments/* to the new Payments microservice.
app.MapReverseProxy(); // YARP
```

You extract one bounded context at a time, sit it behind the same gateway, dual-write or use change-data-capture during the transition, then cut over.

## Advantages

### Monolith

- **Simplest possible deployment** — one artifact, one pipeline, one Azure resource.
- **Cheap to operate** — minimal infrastructure cost.
- **Transactional consistency for free** — one database, one `SaveChangesAsync`.
- **Easy debugging** — one process, one stack trace.
- **Fast development** — no service boilerplate, no contract versioning, no cross-service tests.
- **Fast cross-feature refactoring** — rename a method everywhere with one Rider/Visual Studio operation.

### Microservices

- **Independent deployment** — teams ship at their own cadence without coordination.
- **Independent scaling** — scale the hot service, leave the rest alone.
- **Independent technology choices** — Python for ML, Go for the proxy, .NET for the business.
- **Small blast radius for failure** — one service down does not take everyone down (with circuit breakers).
- **Team autonomy** — bounded contexts map to teams; clear ownership.
- **Better long-term agility for large systems** — twenty teams working in twenty repos beat twenty teams working in one.

## Disadvantages

### Monolith

- **Deployment risk grows with codebase** — one bad commit can roll back the whole system.
- **Scaling is coarse** — you scale the entire monolith even if only one feature is hot.
- **Long-term entropy** — without strict module discipline, everything ends up coupled.
- **Slower releases at scale** — a 100-engineer monolith with a one-hour test suite ships slowly.

### Microservices

- **Operational complexity explodes** — service mesh, distributed tracing, log aggregation, secrets management, rolling deploys, schema versioning.
- **Distributed-system bugs are hard** — eventual consistency, partial failure, out-of-order messages, idempotency.
- **No distributed transactions** — sagas + outbox + compensating actions take real discipline.
- **Network is expensive** — every cross-service call adds 1–10ms of latency.
- **Higher infrastructure cost** — more compute, more storage, more observability tooling.
- **Cognitive load** — debugging an issue requires correlating logs across 5+ services.

## Common Mistakes

### 1. Choosing microservices because they are "modern"

**Problem:** a 4-engineer team builds a new product with 12 microservices. They spend 60% of their time on infrastructure (Service Bus glue, observability, deploys, sagas) and 40% on features. Velocity grinds to a halt.

**Fix:** start with a **modular monolith**. Extract microservices when you can name a concrete pain (release coupling, scaling mismatch, team independence) that a service split actually solves. *"It feels more mature"* is not a justification.

### 2. The "distributed monolith"

**Problem:** 10 microservices, but they all share one database (or all synchronously call each other in a chain). A change to the `Order` schema forces a coordinated deploy of all 10 services. You have all the cost of microservices and none of the independence.

```
// ❌ Sync chain — failures cascade, deploys couple
[POST] /api/orders  →  POST /api/payments  →  POST /api/inventory  →  POST /api/shipping
```

**Fix:** services own their own data (database-per-service). Services communicate via **asynchronous events**, not chained HTTP calls. Failure of one service should not take down the next.

### 3. Choosing monolith for a team of 50+ engineers shipping to a single repo

**Problem:** 50 engineers, one main branch, one CI pipeline, 90-minute test suite, daily merge conflicts. Release frequency drops from daily to weekly to monthly.

**Fix:** split along **bounded contexts**. Even if you keep the monolith deployable, split it into independent NuGet packages or sub-repos so teams can work in parallel. Or accept that you have outgrown the monolith and migrate via Strangler Fig.

### 4. No bounded contexts, just "split by entity"

**Problem:** the team creates an `OrdersService`, a `LinesService`, a `CustomersService`, a `ProductsService`. Every business operation requires 4 network calls and a saga because "an order has lines, customers, and products."

**Fix:** split by **business capability** (`Sales`, `Fulfillment`, `Catalog`, `Identity`), not by table. An aggregate stays in one service. Two aggregates that frequently change together belong in the same service.

### 5. Treating sagas as optional

**Problem:** a microservices team uses synchronous HTTP for cross-service workflows ("just call the next service"). When one service is slow or down, customers see timeouts and double-charges.

**Fix:** any workflow that touches more than one service is a **saga**. Use MassTransit, NServiceBus, or roll your own with Service Bus + state aggregate. Use **idempotency keys** and **compensating actions**.

### 6. Going microservices without observability

**Problem:** a production bug requires tracing a request through 5 services. There is no distributed trace ID, logs are not centralized, and metrics live in 3 different dashboards. Mean time to diagnosis: 4 hours.

**Fix:** **OpenTelemetry + Application Insights** with a shared trace ID on every request, structured logging with the trace ID, and one dashboard per service plus a top-level "service map." This is non-negotiable infrastructure for microservices.

### 7. Ignoring schema evolution between services

**Problem:** the `Payments` team renames a field in `PaymentSucceededEvent`. The `Orders` service consuming the event silently breaks. Production is on fire.

**Fix:** treat integration events as **public APIs**. Version them. Never remove fields, only add optional ones. Run consumer-driven contract tests (Pact) in CI.

## Best Practices

- **Default to a modular monolith** for new systems. Extract microservices when justified.
- **Split by bounded context**, not by table or by tier.
- **Database per service** in microservices. No shared schemas, no cross-service joins.
- **Asynchronous events between services** via Azure Service Bus + outbox. Synchronous HTTP only when the answer is needed immediately and the callee is reliable.
- **Idempotent consumers** — every integration event handler must be safe to run twice.
- **Sagas for multi-service workflows** — never a chain of HTTP calls.
- **Observability before extraction** — distributed tracing, centralized logs, service map. If you cannot trace a request, you cannot operate microservices.
- **Version integration contracts** — never break consumers without a deprecation window.
- **One team per service** as the long-term target. Multiple teams per service creates coordination overhead; multiple services per team is operationally fine.
- **Strangler Fig for migration** — extract one bounded context at a time behind an API gateway.
- **Right-size the runtime** — small services do not need AKS. Azure App Service, Container Apps, or Functions are usually enough.
- **Internal NuGet packages for shared contracts** — never a "Common" project that creates compile-time coupling between services.

## Related Concepts

- [architecture/microservices-architecture.md](microservices-architecture.md) — detailed microservices patterns.
- [architecture/event-driven-architecture.md](event-driven-architecture.md) — the messaging style that makes microservices work.
- [architecture/saga-pattern.md](saga-pattern.md) — coordinating multi-service workflows.
- [architecture/outbox-pattern.md](outbox-pattern.md) — reliable event publishing.
- [architecture/aggregates.md](aggregates.md) — aggregates often define service ownership.
- [architecture/domain-driven-design.md](domain-driven-design.md) — bounded contexts drive service boundaries.
- [architecture/scalability-design.md](scalability-design.md) — per-service scaling.
- [azure/app-service.md](../azure/app-service.md) — common monolith host.
- [azure/azure-functions.md](../azure/azure-functions.md) — common worker host for microservices.
- [azure/azure-service-bus.md](../azure/azure-service-bus.md) — backbone of inter-service messaging.
- [devops/ci-cd-pipelines.md](../devops/ci-cd-pipelines.md) — independent pipelines per service.

## Real-World Usage

### Monolith — early-stage SaaS

A 5-engineer team building a B2B invoicing SaaS deploys a single ASP.NET Core 8 app to Azure App Service P1V3, backed by Azure SQL S2. Internally the code is organized into `Invoicing`, `Billing`, `Identity`, and `Notifications` modules. Releases happen daily via GitHub Actions. Time-to-first-feature is one week. They serve 200 paying customers on this setup before considering a split.

### Modular monolith — mid-stage scale-up

The same SaaS reaches 30 engineers and 10,000 customers. The monolith is still one deployable, but each module now has its own EF Core schema (`invoicing`, `billing`, etc.), exposes only commands/queries as public API, and communicates via in-process MediatR. The team adds **integration events** published to Azure Service Bus *in addition to* in-process events — so a future service extraction is a matter of changing one subscriber from in-process to remote.

### Microservices — large enterprise

A retail platform with 200 engineers operates a microservices architecture: `Catalog`, `Cart`, `Pricing`, `Inventory`, `Orders`, `Payments`, `Shipping`, `Returns`, `Loyalty`, `Notifications`, `Search`. Each service runs on Azure Container Apps with its own Cosmos DB or Azure SQL. Service Bus topics fan out integration events. Sagas live in `Orders` (orchestrating the checkout flow). Distributed tracing in Application Insights links every request end-to-end. Each team owns 1–2 services and deploys 5–20 times per day independently.

### Migration via Strangler Fig

A 10-year-old monolith with 1.2M lines of C# starts a 2-year extraction. **Azure API Management** sits in front. The team extracts `Loyalty` first (small, isolated, low risk), then `Notifications`, then `Catalog`. After each extraction, the corresponding routes in API Management flip from the monolith backend to the new service. The monolith shrinks; new features go into microservices; the team keeps shipping the whole time.

### Cost shape

- 4-engineer team, modular monolith: ~$200/mo Azure spend, ~5% of engineering time on platform.
- 20-engineer team, 5 microservices: ~$2,500/mo Azure spend, ~15% of engineering time on platform.
- 100-engineer team, 30 microservices: ~$20,000+/mo Azure spend, ~25% of engineering time on platform (a dedicated platform team).

## Code Example — Before and After

### Before — monolith hitting the wall

A single ASP.NET Core app, one DB, one team of 25. The catalog service handles 10k req/s during sales; the rest of the system handles 100 req/s. Every deploy scales the whole monolith and risks taking everything down.

```csharp
// One controller, one process, one DB
[ApiController]
public sealed class OrdersController : ControllerBase
{
    private readonly OrdersDbContext _db;
    private readonly IStripeClient _stripe;
    private readonly ICatalogService _catalog;   // in-process call to the catalog module
    private readonly IShippingService _shipping; // in-process call to the shipping module

    [HttpPost]
    public async Task<IActionResult> Place(PlaceOrderRequest req, CancellationToken ct)
    {
        var prices = await _catalog.GetPricesAsync(req.Skus, ct);    // shared DB
        var order = Order.Create(req.CustomerId, req.Lines, prices);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        await _stripe.ChargeAsync(req.CustomerId.ToString(), order.Total, ct);
        await _shipping.ScheduleAsync(order.Id, ct); // in-process
        return Ok(order.Id);
    }
}
```

Problems: scaling is all-or-nothing, Stripe call inside transaction window, shipping coupled to order release schedule.

### After — extracted catalog as a microservice, kept the rest modular

`Catalog` lives in its own Azure Container App, behind its own Azure SQL DB, scaled aggressively (10 to 100 replicas based on CPU). The order monolith calls it via HTTP for prices and consumes its `PriceChangedEvent` via Service Bus to cache. Payments and shipping are still in the monolith for now — they will be extracted next quarter.

```csharp
// Orders monolith — calls Catalog over HTTP with resilience
public sealed class CatalogClient : ICatalog
{
    private readonly HttpClient _http;
    private static readonly AsyncPolicy<HttpResponseMessage> _policy = Policy<HttpResponseMessage>
        .HandleResult(r => (int)r.StatusCode >= 500)
        .WaitAndRetryAsync(3, attempt => TimeSpan.FromMilliseconds(100 * attempt))
        .WrapAsync(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(2)));

    public CatalogClient(HttpClient http) => _http = http;

    public async Task<IReadOnlyList<Price>> GetPricesAsync(IReadOnlyList<Sku> skus, CancellationToken ct)
    {
        var response = await _policy.ExecuteAsync(ct =>
            _http.PostAsJsonAsync("/api/prices", new { Skus = skus }, ct), ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<Price>>(cancellationToken: ct) ?? new();
    }
}

// Program.cs
builder.Services.AddHttpClient<ICatalog, CatalogClient>(c =>
    c.BaseAddress = new Uri(builder.Configuration["Catalog:BaseUrl"]!));

// Outbox publishes OrderPlacedIntegrationEvent for downstream consumers
public async Task<Result> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var prices = await _catalog.GetPricesAsync(cmd.Skus, ct);
    var order = Order.Create(cmd.CustomerId, cmd.Lines, prices);
    await _orders.AddAsync(order, ct);
    _outbox.Enqueue(new OrderPlacedIntegrationEvent(order.Id, order.Total));
    await _uow.SaveChangesAsync(ct);
    return Result.Ok();
}
```

A `Payments.Worker` and a `Shipping.Worker` will subscribe to `OrderPlacedIntegrationEvent` when they are extracted — no change required in `Orders` at that point. The architecture grew incrementally without a big-bang rewrite.

## Interview Questions and Answers

### 1. A 5-engineer team is starting a new SaaS. They want microservices. What is your advice?

**Why this matters:** the most common architectural mistake of the last decade, and the one interviewers genuinely want to hear you push back on.

**Answer:** Start with a **modular monolith**. With 5 engineers, the operational cost of microservices (service mesh, distributed tracing, sagas, contract versioning, per-service pipelines) will consume more time than the features you are trying to build. Organize the monolith into modules with clear boundaries — one project per bounded context, one schema per module, no cross-module navigation in EF Core. When the team grows past ~15 engineers and a specific bounded context needs to scale or release independently, extract that one. Migration is straightforward because the boundaries already exist.

**Real project:** I joined a startup that had launched with 8 microservices and 4 engineers. They spent more time fighting Service Bus and tracing than shipping features. We consolidated back to one ASP.NET Core monolith with clear modules. Velocity 4x'd in a quarter.

### 2. What is a "distributed monolith" and why is it the worst outcome?

**Why this matters:** identifies whether the candidate has seen the pattern fail in the wild.

**Answer:** A distributed monolith is microservices that **share a database** or **call each other synchronously in chains**. Every change requires coordinated deploys of multiple services; every failure cascades; every request crosses 5 networks. You pay the full operational cost of microservices (latency, complexity, infrastructure) and get none of the benefits (independent deploy, independent scaling, independent failure). It is strictly worse than either a clean monolith *or* clean microservices.

**Fix:** database per service, asynchronous events between services, **never** a synchronous HTTP chain for write workflows.

### 3. How do you handle a transaction that touches three services?

**Why this matters:** distinguishes candidates who understand distributed systems from those who do not.

**Answer:** You **cannot** have a distributed ACID transaction in modern cloud environments — MSDTC over the public internet is operationally a non-starter. Instead, use a **saga**: orchestrated (one service drives the workflow, listening for events and dispatching commands) or choreographed (each service reacts to events from the previous step). Each step is **idempotent** and supports **compensating actions** for rollback. Reliable event publishing uses the **outbox pattern** so business state and outgoing events commit together.

**Real project:** a checkout flow spanned Orders, Payments, Inventory. We modeled it as an orchestrated saga in the Orders service. On payment failure, a `RefundInventoryReservationCommand` compensated. End-to-end success rate was tracked in Application Insights; failures dead-lettered for human review.

### 4. When is the right time to extract the first microservice from a monolith?

**Why this matters:** maturity to recognize the trigger, not just the destination.

**Answer:** When **at least two** of the following are true: (a) two teams genuinely block on each other in the monolith's release pipeline; (b) one bounded context has dramatically different scaling needs (10x more traffic, or expensive ML workloads); (c) the bounded context is mature and stable — its API has not changed materially in 6+ months; (d) the team has the operational maturity (distributed tracing, centralized logs, alerting, saga discipline) to run it in production. If only one of those is true, fix that problem differently before extracting.

### 5. How do you choose service boundaries?

**Why this matters:** wrong boundaries are the second most common microservices mistake.

**Answer:** By **bounded context** (business capability), never by entity or technical layer. *"Sales", "Fulfillment", "Catalog", "Identity"* are good boundaries — each owns a cohesive set of aggregates and a database. *"OrderService", "OrderLineService", "CustomerService"* is a classic anti-pattern — those entities change together. A good test: a typical feature should require changes to **one** service. If features routinely span three services, the boundaries are wrong.

**Real project:** an early version of a platform split `Orders` and `OrderLines` into separate services. 90% of features touched both; deploys had to be coordinated; performance was terrible. We merged them into one `Sales` service and the pain disappeared.

### 6. How does data consistency change when moving from monolith to microservices?

**Why this matters:** consistency model is the single biggest cognitive shift.

**Answer:** In a monolith, multi-aggregate consistency is **immediate** via a database transaction. In microservices, it becomes **eventual** — there is a window (milliseconds to seconds, sometimes minutes) where the system is in an intermediate state. The UI, the business stakeholders, and downstream consumers must accept this. You design with idempotency, outbox-driven events, sagas, and compensating actions. You also build read models / projections so query-side consistency can be optimized separately. If a feature genuinely requires immediate cross-aggregate consistency, those aggregates probably belong in the same service.

### 7. What infrastructure do you need before you can run microservices in production?

**Why this matters:** going to production without this list is the most common cause of microservices regret.

**Answer:** Non-negotiable: **distributed tracing** (OpenTelemetry + Application Insights or equivalent), **centralized logging** with a shared correlation ID, **per-service health checks** with readiness/liveness, **circuit breakers and retries** (Polly), **reliable async messaging** (Azure Service Bus + outbox), **secrets management** (Key Vault + Managed Identity), **independent CI/CD pipelines per service**, **API gateway** (APIM / YARP), and a **service catalog** so engineers can find what exists. Nice-to-have: service mesh, schema registry, contract testing (Pact), feature flags.

### 8. Conway's Law — how does it apply to monolith vs microservices?

**Why this matters:** distinguishes senior architectural thinking from feature-focused thinking.

**Answer:** Conway's Law states that *systems mirror the communication structures of the organizations that build them*. Practically: if you have one team, you will build one deployable — fighting it with a microservices architecture creates an integration nightmare. If you have eight teams, one monolith forces every team through one release cadence — fighting it creates contention. The architecture should follow team topology. When the org changes (a team is split, two teams merge), expect the architecture to evolve too. **Inverse Conway maneuver** is the senior version: change the org structure on purpose to drive the architecture you want.

**Real project:** a 60-engineer monolith team was failing to ship. We restructured into four ~15-engineer teams, each owning a vertical slice. Over 18 months, the monolith naturally split into 4 services along those lines — no big-bang rewrite, just team-by-team extraction.

## Summary Checklist

- [ ] I can articulate the simplicity vs independence trade-off and where each pattern fits.
- [ ] I default to a modular monolith for new systems and small teams.
- [ ] I split by bounded context, never by entity or technical layer.
- [ ] I avoid the distributed monolith anti-pattern (shared DBs, synchronous chains).
- [ ] I use asynchronous events + outbox + idempotent consumers for inter-service communication.
- [ ] I design multi-service workflows as sagas with compensating actions.
- [ ] I have distributed tracing, centralized logs, and circuit breakers before going microservices.
- [ ] I version integration contracts and never break consumers silently.
- [ ] I can apply Conway's Law to align architecture with team topology.
- [ ] I migrate monolith → microservices incrementally via Strangler Fig, not big-bang rewrite.
