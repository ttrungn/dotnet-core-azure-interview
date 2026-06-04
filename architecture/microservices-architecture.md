# Microservices Architecture

## What It Is

A **microservices architecture** decomposes a system into a set of small, independently deployable services. Each service:

- Owns one **business capability** (Orders, Billing, Inventory, Identity, Notifications).
- Owns its **data** — its own database, not shared tables.
- Exposes a network API (REST, gRPC, async messages).
- Is built and deployed by **one team**, on its own release cadence.
- Can be written in different languages/frameworks (though most shops standardize).

```text
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Orders   │   │ Billing  │   │Inventory │   │Shipping  │
   │ (.NET)   │   │ (.NET)   │   │ (.NET)   │   │ (.NET)   │
   ├──────────┤   ├──────────┤   ├──────────┤   ├──────────┤
   │ SQL DB   │   │ SQL DB   │   │ Cosmos   │   │ SQL DB   │
   └──────────┘   └──────────┘   └──────────┘   └──────────┘
        │             │              │              │
        └─────┬───────┴──────────────┴──────────────┘
              │ Azure Service Bus (events) / gRPC (queries)
```

Microservices is not the same as **distributed computing** in general, nor is it merely "split into APIs." The defining traits are:

1. **Independent deployment** — Orders ships at 11am without anyone else's involvement.
2. **Bounded context per service** — each service maps to one [DDD](domain-driven-design.md) bounded context.
3. **No shared database** — Billing cannot `JOIN` against Orders' tables.
4. **Async-first communication** — synchronous calls exist but events ([Event-driven architecture](event-driven-architecture.md)) are the default for non-blocking workflows.

## Why It Exists

A monolithic service eventually hits limits a single team can't bypass:

- One deploy reboots everything; a bug in module A blocks shipping module B.
- The DB becomes the integration point — every team's queries fight on the same tables.
- Different parts have different scaling needs (search 10x reads vs. write-heavy billing), but you scale the whole monolith.
- The codebase grows past the "two-pizza team" boundary; merge conflicts, slow builds, fear of releases.

Microservices were popularized by Netflix, Amazon, and Spotify in the early 2010s as a way to let many small teams ship in parallel. The core idea: **align service boundaries to team and business boundaries** so a single team owns a service end-to-end.

It exists to enable:

1. **Independent deployment** — team velocity is no longer gated by other teams.
2. **Independent scaling** — scale the search service to 100 pods without paying for 100 billing pods.
3. **Technology autonomy** — one team can pick Cosmos DB while another uses SQL Server.
4. **Failure isolation** — billing being down doesn't stop the catalog from rendering.

## When To Use It

**Use microservices for:**

- **Large organizations with many teams** (10+ engineers per service area). The autonomy is the point.
- **Systems with clearly separable business capabilities** that don't share too much data — e-commerce (catalog, cart, payment, fulfillment), banking (accounts, transactions, statements), SaaS platforms.
- **Workloads with very different scaling profiles** — search 100x more reads than writes, billing strictly transactional, image processing CPU-bound.
- **Compliance / isolation requirements** — PCI-scoped payment data segregated in its own service and database.
- **Polyglot needs** — ML inference in Python, transactional core in C#, real-time pricing in Go.

**Do not use microservices for:**

- **Small teams (1-5 engineers).** You'll create operational chaos and ship slower than a [monolith](monolith-vs-microservices.md).
- **Systems where everything shares a single business concept** (e.g., a tightly coupled invoicing engine) — bounded contexts overlap too much to split cleanly.
- **MVPs and early-stage products.** You don't yet know where boundaries are; refactoring service boundaries is far more expensive than refactoring a monolith.
- **Strictly transactional workloads.** Microservices give eventual consistency; if you need ACID across what would be services, keep them together.
- **Teams without DevOps maturity.** Microservices require CI/CD per service, distributed tracing, service mesh or API gateway, multiple DBs, broker, monitoring — operational overhead multiplies.

## Why It Is Important

Microservices change four properties that compound:

1. **Team autonomy.** Two-pizza teams own one service end-to-end — code, database, deploy, on-call. They don't queue behind other teams to ship.
2. **Independent failure.** A bug in Notifications shouldn't take down Checkout. With circuit breakers and timeouts, partial failure is normal and tolerated.
3. **Independent scaling.** Black Friday traffic on Catalog autoscales to 200 pods while Billing stays at 8. Cost matches usage.
4. **Polyglot capability.** ML team picks Python; payments team picks .NET; search team picks Go. The contract between them is the API, not the language.

In an Azure context, microservices typically run on AKS (Kubernetes) or Azure Container Apps. They share a Service Bus namespace for events, an API Management gateway for ingress, Application Insights / Log Analytics for observability, and Azure AD for identity. The architecture is enabled by the platform — going microservices without that platform is reinventing it.

## How It's Used in C# / .NET

### 1. Service boundaries follow business capabilities

```text
OrderService            — places, modifies, cancels orders
PaymentService          — captures, refunds, settles payments
InventoryService        — reservations, stock counts
ShippingService         — labels, tracking, returns
NotificationService     — email, SMS, push
IdentityService         — auth, user profile
CatalogService          — product data, search
```

Each is a separate ASP.NET Core solution, repo (or folder), CI pipeline, and Azure resource.

### 2. Project structure per service

```text
src/
  Orders.Api/          (ASP.NET Core minimal API or controllers)
  Orders.Application/  (commands, queries, handlers via MediatR — CQRS)
  Orders.Domain/       (aggregates, value objects, domain events)
  Orders.Infrastructure/ (EF Core DbContext, repository implementations, Service Bus publisher)
tests/
  Orders.UnitTests/
  Orders.IntegrationTests/
deploy/
  Dockerfile
  k8s/
    deployment.yaml
    service.yaml
```

This matches [Clean Architecture](clean-architecture.md). The service is internally well-structured even if externally it's "just a microservice."

### 3. Synchronous communication — REST and gRPC

For queries that need an answer now:

```csharp
// In Orders service, calling Catalog over gRPC
builder.Services
    .AddGrpcClient<CatalogService.CatalogServiceClient>(o =>
        o.Address = new Uri("https://catalog.internal:5001"))
    .AddPolicyHandler(GetRetryPolicy())          // Polly retry
    .AddPolicyHandler(GetCircuitBreakerPolicy()); // Polly circuit breaker
```

For internal traffic, gRPC is preferred over REST: typed contracts via Protobuf, smaller payloads, streaming. REST stays for public APIs.

### 4. Async communication — Service Bus events

For everything that doesn't need an immediate answer:

```csharp
// Orders publishes via outbox
_outbox.Enqueue(new OrderPlacedIntegrationEvent(order.Id, ...));
await _uow.SaveChangesAsync(ct);
// Background worker reads outbox, publishes to Service Bus

// Payment, Inventory, Shipping, Notifications each subscribe
public class PaymentOrderPlacedConsumer : IConsumer<OrderPlacedIntegrationEvent> { /* ... */ }
```

See [event-driven architecture](event-driven-architecture.md) and [outbox pattern](outbox-pattern.md).

### 5. Each service owns its database

```csharp
// Orders service
builder.Services.AddDbContext<OrdersDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("OrdersDb")));

// Payment service
builder.Services.AddDbContext<PaymentsDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("PaymentsDb")));
```

Two databases, two connection strings, two schemas. No `JOIN` between them — if Orders needs payment data, it calls the Payment API or subscribes to Payment events.

### 6. Cross-cutting concerns

- **API gateway**: Azure API Management or YARP at the edge — auth, throttling, routing.
- **Authentication**: Azure AD / Entra ID issues tokens; each service validates with `Microsoft.Identity.Web`.
- **Configuration**: Azure App Configuration or environment variables; secrets in Key Vault.
- **Observability**: OpenTelemetry exporters → Application Insights / Log Analytics. Correlation IDs propagate across services automatically.
- **Resilience**: Polly for retry, circuit breaker, timeout, bulkhead. `IHttpClientFactory` with typed clients.
- **Service mesh** (optional): Linkerd or Istio for mTLS, retries, and traffic policies at the platform layer.

### 7. Deployment topology on Azure

```text
                            ┌────────────────────┐
   internet  ──►  Azure ──► │ APIM / YARP        │
                  Front     │ (gateway)          │
                  Door      └────┬───────────────┘
                                 │
       ┌─────────────────┬──────┴───────┬─────────────────┐
       ▼                 ▼              ▼                 ▼
  AKS pod         AKS pod         AKS pod         Azure Function
  Orders (3 reps) Catalog (8 reps) Billing (2 reps) Notifications
       │                 │              │                 │
   SQL elastic     Cosmos DB         SQL Server      Service Bus
                                                     (consumes)
                            Azure Service Bus
                       (events shared by all)
```

KEDA scales consumers based on queue depth. Application Insights ties traces across pods.

### 8. Quick reference

| Concern                | Tool / approach                                                |
|------------------------|---------------------------------------------------------------|
| Sync communication     | gRPC for internal, REST for public, behind Polly + circuit breakers |
| Async communication    | Azure Service Bus topics + [outbox pattern](outbox-pattern.md)   |
| Service discovery      | Kubernetes DNS, Azure Front Door, APIM                        |
| Auth                   | Azure AD / Entra ID, JWT bearer middleware                    |
| Data per service       | One DB per service (SQL, Cosmos, Postgres — chosen per fit)   |
| Containerization       | Multi-stage Dockerfile, dotnet 8+ runtime base image          |
| Orchestration          | AKS (full Kubernetes) or Azure Container Apps (managed)       |
| Observability          | OpenTelemetry → Application Insights, Log Analytics, Grafana  |
| Configuration          | Azure App Configuration + Key Vault                           |
| API gateway            | Azure API Management, YARP                                    |
| Cross-service workflow | [Saga pattern](saga-pattern.md) (MassTransit Saga or Durable Functions) |

## Advantages

- **Independent deployment** — teams ship without coordination.
- **Independent scaling** — pay for what each service needs.
- **Failure isolation** — one slow service doesn't drag the others down (with circuit breakers).
- **Technology choice per service** — best tool per workload (Cosmos for high write, SQL for transactions, Python for ML).
- **Aligns with team structure** — Conway's law works *for* you instead of against you.
- **Smaller cognitive load per service** — a new engineer ramps up on Orders in days, not the whole platform in months.
- **Long-term evolvability** — replace one service entirely (rewrite from Java to .NET) without touching others.

## Disadvantages

- **Operational complexity explodes** — many DBs, many CI pipelines, many dashboards, distributed tracing, DLQs.
- **Distributed transactions are hard** — no DB-level ACID across services; you need sagas, idempotency, compensations.
- **Network latency and failures** — every hop is a potential failure mode; calls that were nanoseconds in-process are now milliseconds + retries.
- **Versioning of contracts** — events and APIs are public contracts; breaking changes ripple.
- **Debugging is harder** — a single business outcome spans 5+ services; you need OpenTelemetry to make sense of it.
- **Eventual consistency surprises** — UI must handle "I just saved this, why don't I see it yet?"
- **Higher cost** — more infrastructure (databases, brokers, gateways), more engineers per platform (DevOps, SRE).
- **Premature splits are painful to undo** — wrong service boundaries lock you in until you can do a major refactor.

## Common Mistakes

### 1. Splitting too early or by technical layer

```text
BAD: UserDataService, UserBusinessLogicService, UserApiService (split by layer)
GOOD: IdentityService (owns user data, auth, profile end-to-end)
```

Splitting by layer creates a "distributed monolith" — services that must deploy together. Split by **bounded context** ([DDD](domain-driven-design.md)).

### 2. Sharing the database

```csharp
// BUG: Orders service queries directly against Inventory's tables
var stock = await _inventoryDbContext.Stock.FindAsync(productId, ct);
```

Shared DB defeats the purpose. Inventory changes its schema → Orders breaks. Fix: Orders calls Inventory's API or subscribes to its events.

### 3. Building a "distributed monolith"

Symptoms: services must be deployed together, services share contracts that change frequently, one service down causes others to fail. The root cause is usually wrong service boundaries.

**Fix**: Re-examine bounded contexts; merge or split services until each one can release independently.

### 4. Synchronous chain calls

```text
Orders ──HTTP──► Payment ──HTTP──► Fraud ──HTTP──► Risk
```

Latency multiplies. One slow service slows the whole chain. One down service kills the chain.

**Fix**: Use events for fan-out. Each service reacts to `OrderPlaced` independently. Use a saga ([saga pattern](saga-pattern.md)) for explicit orchestration of multi-step workflows.

### 5. No circuit breakers, no timeouts

```csharp
// BUG: blocking call with no timeout, no retry, no circuit breaker
var response = await _httpClient.GetAsync("https://billing/payments/123");
```

When billing is slow, this blocks. When it's down, you exhaust connections.

**Fix**: `IHttpClientFactory` + Polly with timeout, retry (with jitter and limits), circuit breaker.

```csharp
services.AddHttpClient<IBillingClient, BillingClient>(c => c.Timeout = TimeSpan.FromSeconds(5))
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(5))
    .AddPolicyHandler(Policy.Handle<HttpRequestException>()
        .WaitAndRetryAsync(3, attempt => TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt))))
    .AddPolicyHandler(Policy.Handle<HttpRequestException>()
        .CircuitBreakerAsync(10, TimeSpan.FromSeconds(30)));
```

### 6. Forgetting idempotency

Network retries cause duplicate calls. Without idempotency, you get duplicate charges, duplicate orders, duplicate emails.

**Fix**: Idempotency keys on commands, inbox table on consumers, unique constraints in the DB.

### 7. No distributed tracing

Without OpenTelemetry / Application Insights, debugging a cross-service issue requires manually correlating logs by timestamp. Impossible at scale.

**Fix**: Enable OTel from day one. Every service participates. Correlation IDs propagate automatically.

### 8. One team owning many services

If a 4-engineer team owns 12 services, each service is half-maintained. Microservices presume ~1 service per 4-6 engineers; otherwise consolidate.

## Best Practices

- **Follow bounded contexts** ([DDD](domain-driven-design.md)) for service boundaries.
- **One DB per service.** No sharing. Cross-service data via APIs or events.
- **Async-first.** Use events for anything that doesn't need an immediate answer.
- **Use the [outbox pattern](outbox-pattern.md)** for transactional publish.
- **Idempotent consumers and idempotent commands** — every retry must be safe.
- **Polly everywhere.** Timeout, retry with jitter, circuit breaker, bulkhead for every outbound call.
- **OpenTelemetry from day one.** No service ships without traces and structured logs.
- **API versioning** for public endpoints; event versioning for integration events.
- **Service ownership**: one team owns each service end-to-end (code, infra, on-call).
- **Standardize the platform** (logging, auth, base Docker image, CI templates) so teams don't reinvent it.
- **Start as a [monolith](monolith-vs-microservices.md) or modular monolith** if boundaries are unclear; extract microservices only when you know where the seams are.

## Related Concepts

- **[Monolith vs microservices](monolith-vs-microservices.md)** — the choice and the modular monolith path.
- **[Domain-driven design](domain-driven-design.md)** — bounded contexts define service boundaries.
- **[Event-driven architecture](event-driven-architecture.md)** — the standard async communication style.
- **[Outbox pattern](outbox-pattern.md)** — transactional publish from a service's database.
- **[Saga pattern](saga-pattern.md)** — long-running workflows across services.
- **[Reliability design](reliability-design.md)** — circuit breakers, retries, bulkheads, timeouts.
- **[Scalability design](scalability-design.md)** — horizontal scale per service.
- **[Azure Service Bus](../azure/azure-service-bus.md)** — the default broker for inter-service events.
- **[App Service / AKS](../azure/app-service.md)** and **[Container Apps](../devops/kubernetes-basics.md)** — hosting platforms.
- **[CI/CD pipelines](../devops/ci-cd-pipelines.md)** — pipeline per service.

## Real-World Usage

### E-commerce platform on Azure

12 services on AKS: Catalog, Search, Cart, Orders, Payments, Inventory, Shipping, Notifications, Identity, Loyalty, Reviews, Reporting. Each has its own DB (mix of SQL Server, Cosmos DB, Redis). Inter-service via Service Bus + REST/gRPC. APIM gateway at the edge. Application Insights for tracing. ~200 engineers across ~20 teams. Each team ships its service ~10 times/week independently.

### Banking core

Account service (SQL Server), Transactions service (event-sourced), Statements service (Cosmos DB read model), Notifications, Fraud (ML in Python). Hard regulatory boundaries — payment data isolated to PCI-scoped services. Sagas orchestrate transfers across accounts. Strict idempotency and audit logging.

### Healthcare SaaS

EHR records service, Scheduling service, Billing service, Telehealth service, Reporting service. Each tenant's data isolated per service per tenant. Async events keep tenant-level read models in sync. Compliance audit requires every action published as a versioned event to a long-retention store.

### Streaming media

Catalog, Recommendations (Python ML), Playback, Billing, Notifications. Catalog and Recs use Cosmos DB for global low-latency reads; Billing uses SQL. Event Hubs streams playback events for analytics; Service Bus carries transactional events (subscriptions, billing).

## Code Example — Before and After

### Before: Monolith with shared DB

```csharp
// One ASP.NET Core app, one DbContext, one deploy
public class CheckoutController : ControllerBase
{
    private readonly AppDbContext _db; // Orders, Inventory, Payments tables all here

    [HttpPost("/checkout")]
    public async Task<IActionResult> Checkout(CheckoutRequest req, CancellationToken ct)
    {
        using var tx = await _db.Database.BeginTransactionAsync(ct);

        foreach (var line in req.Lines)
        {
            var stock = await _db.Stock.FindAsync(line.ProductId, ct);
            if (stock!.Available < line.Quantity) return BadRequest();
            stock.Available -= line.Quantity;
        }

        var order = new Order(req.CustomerId, req.Lines);
        _db.Orders.Add(order);

        var payment = new Payment(order.Id, order.Total);
        _db.Payments.Add(payment);

        await _db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
        return Ok(new { id = order.Id });
    }
}
```

Properties:
- One deploy. One team can change everything. ACID transactions are easy.
- But: when the team grows past ~6 engineers, releases become slow and risky. The catalog team blocks on the orders team. Inventory's scaling needs are different from payments'.

### After: Microservices

**Orders service**:
```csharp
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IInventoryClient _inventory;  // gRPC client to Inventory service
    private readonly IOutbox _outbox;
    private readonly IUnitOfWork _uow;

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        // Synchronous check — need answer to continue
        var reservation = await _inventory.ReserveAsync(cmd.Lines, ct);
        if (!reservation.Success) throw new OutOfStockException();

        var order = Order.Place(cmd.CustomerId, cmd.Lines);
        _orders.Add(order);
        _outbox.Enqueue(new OrderPlacedIntegrationEvent(order.Id, cmd.CustomerId, order.Total, ...));
        await _uow.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

**Payment service** (separate repo, separate DB, separate deploy):
```csharp
public sealed class OrderPlacedConsumer : IConsumer<OrderPlacedIntegrationEvent>
{
    private readonly IPaymentService _payments;
    private readonly IInbox _inbox;

    public async Task Consume(ConsumeContext<OrderPlacedIntegrationEvent> ctx)
    {
        if (await _inbox.AlreadyProcessed(ctx.Message.EventId)) return;
        await _payments.CaptureAsync(ctx.Message.OrderId, ctx.Message.Total, ctx.CancellationToken);
        await _inbox.MarkProcessed(ctx.Message.EventId);
    }
}
```

**Inventory service** (separate repo, gRPC):
```csharp
public sealed class InventoryGrpcService : Inventory.InventoryBase
{
    public override Task<ReserveResponse> Reserve(ReserveRequest req, ServerCallContext ctx) =>
        // ...
}
```

Trade-offs delivered: each team ships their service independently. Inventory autoscales differently from Payments. ACID inside Orders DB; eventual consistency across services. Sagas handle multi-step flows. APIM gateway is the single front door.

## Interview Questions and Answers

### 1. When is microservices the wrong choice?

**Why this matters**: Senior engineers know when patterns add cost without value. Junior engineers often default to microservices without justification.

**Answer**: Microservices is wrong when (1) the team is small (< 10 engineers), (2) the product is early-stage and boundaries are unclear, (3) the workloads share too much data — every "service" boundary requires API calls or events, (4) the org lacks DevOps maturity (no CI/CD per service, no distributed tracing, no DLQ monitoring), (5) you need strict ACID transactions across what would be services. In those cases, a well-modularized monolith ships faster, is easier to debug, and costs less to operate.

**Trade-off**: A monolith that grows beyond its bounded contexts eventually needs to split. The mistake is splitting too early *or* too late. Start as a modular monolith; extract microservices when the team grows past one service's worth of engineers.

**Real project**: An early-stage fintech split their MVP into 8 microservices for "scalability." After 6 months they had spent 60% of engineering on infrastructure and had no users. They consolidated to a monolith, shipped the MVP, then split out the payment processing service when PCI scope forced it.

### 2. How do you decide service boundaries?

**Why this matters**: Wrong boundaries are the #1 cause of microservices failure.

**Answer**: Use [DDD](domain-driven-design.md) — each service should be one bounded context with one team owning it end-to-end. Look for natural seams: different terminology (an "order" in Sales means something different from in Fulfillment), different change rates, different scaling needs, different data ownership. A boundary is good if the service can release independently for weeks without coordinating with neighbors. A boundary is bad if every change touches multiple services or if neighbors must deploy in lockstep.

**Trade-off**: Boundaries chosen by org chart (Conway's law) often work, because team communication is the bottleneck anyway. Boundaries chosen by technical layer always fail.

**Real project**: Split based on "team's release cadence and data ownership" rather than abstract domain models. Catalog (one team, denormalized read model) + Inventory (different team, transactional stock) ended up correct. An earlier split of "User Service" and "Account Service" had to be merged because they always changed together.

### 3. How do services communicate?

**Why this matters**: Mixing sync and async correctly is essential.

**Answer**: Sync (REST or gRPC) when the caller needs an immediate answer and the answer drives the next step — auth checks, fraud scores, inventory holds. Async (Service Bus events, [outbox pattern](outbox-pattern.md)) for fan-out — `OrderPlaced` triggers billing, shipping, notifications, analytics with no caller waiting. Use gRPC over REST for internal sync (typed contracts, smaller payloads, streaming). Always wrap sync calls in Polly (retry + circuit breaker + timeout) and prefer events whenever possible.

**Trade-off**: Sync is simpler to debug but coupling increases and latency multiplies through chains. Events decouple but introduce eventual consistency and require idempotency.

**Real project**: An e-commerce platform exposes gRPC internally for "is this product available?" (sync, sub-100ms), and publishes `OrderPlaced` to Service Bus for everything else. APIM gateway at the edge translates external REST → internal gRPC.

### 4. How do you handle a transaction that spans services?

**Why this matters**: Distributed transactions are one of the hardest problems in microservices.

**Answer**: You don't. Two-phase commit across services is operationally infeasible. Instead use the [saga pattern](saga-pattern.md): break the transaction into a sequence of local transactions, each in its own service. If a step fails, run compensating actions to undo earlier steps. Sagas can be choreographed (each service reacts to events) for short flows or orchestrated (a central coordinator sends commands and reacts to events) for complex ones. Make every step idempotent so retries are safe.

**Trade-off**: Sagas give eventual consistency, not ACID. The user may see "order placed" before "payment captured." Build the UI to handle that or sequence the saga so the visible step is last.

**Real project**: Travel booking saga: ReserveFlight → ReserveHotel → ChargeCard → ConfirmBooking. If ChargeCard fails, run ReleaseHotel + ReleaseFlight compensations. Implemented with MassTransit Saga state machine; state persisted to SQL Server.

### 5. How do you scale individual services on Azure?

**Why this matters**: Independent scaling is one of the main reasons to do microservices.

**Answer**: On AKS, use Horizontal Pod Autoscaler (HPA) on CPU/memory metrics, plus KEDA for event-driven scaling on Service Bus queue depth or HTTP RPS. For Azure Container Apps, scale rules are baked in (HTTP, Service Bus, KEDA scalers). For Azure Functions, scale is automatic based on event source. Database scaling is per-service: SQL elastic pool for spiky workloads, Cosmos DB autoscale RU/s, Redis premium for shared cache. Don't scale every service the same way — Catalog might run 50 pods, Billing 4.

**Trade-off**: Autoscaling has a warm-up time (30s-2min for pods, longer if cold start). Pre-scale for predictable spikes (Black Friday); reactive scale for unexpected traffic. Bursting too fast can melt downstream services (DBs, third-party APIs).

**Real project**: A media service runs Recommendations on 40-100 pods (KEDA scaling on RPS) and Billing on 2-4 pods. During the World Cup, Recommendations briefly hit 250 pods; Billing stayed at 4 because billing rate is independent of view rate.

### 6. What's the operational cost of microservices?

**Why this matters**: Candidates who say "microservices is always better" haven't operated them.

**Answer**: Microservices multiply operational surface. You need: CI/CD per service, multiple databases to back up and patch, broker to monitor, gateway to maintain, distributed tracing, structured logging with correlation IDs, DLQ monitoring per consumer, schema versioning for events, runbooks per service, on-call rotations per team, contract tests between services. The platform team becomes a real organization — SRE, DevOps, security. A small team carrying that load instead of building features will fall behind a monolithic competitor.

**Trade-off**: A platform team is overhead until you have ~5+ product teams to amortize over. Smaller orgs should use managed PaaS (Container Apps, App Service, Functions) to reduce the platform burden.

**Real project**: A 30-engineer company moved from monolith to microservices and discovered they needed 3 SREs to keep the lights on. Net engineering productivity went *down* for 9 months before recovering. They would have been better served by a modular monolith.

### 7. How do you trace a request across 5 services?

**Why this matters**: Distributed tracing is essential, not optional.

**Answer**: OpenTelemetry. Every service auto-instruments ASP.NET Core, HTTP clients, gRPC clients, EF Core, Service Bus, SQL Server. Trace context propagates via `traceparent` HTTP header and Service Bus message properties. Exporter sends spans to Application Insights / Jaeger / Tempo. A single business outcome becomes one trace tree. Every log line includes `TraceId` + `SpanId` so log search by trace finds every related entry across services. Sample at the edge (5-10% for healthy, 100% for errors).

**Trade-off**: Tracing adds ~1-5% overhead and emits a lot of data. Sample aggressively for high-volume systems; keep 100% for low-volume or error traces.

**Real project**: A 12-service order pipeline produces a single flame graph in Application Insights — APIM → Orders → gRPC to Inventory → Service Bus to Billing → Service Bus to Shipping → Service Bus to Notifications. Mean time to root cause dropped from hours to minutes after OTel was rolled out.

### 8. How do you migrate a monolith to microservices?

**Why this matters**: Most "microservices" projects are actually migrations, not greenfield.

**Answer**: Use the **strangler fig pattern**. Keep the monolith running. Identify the first bounded context to extract — pick one with clear boundaries, low coupling, and a real reason to split (different scaling, separate team, compliance). Build the new service alongside. Put an API gateway (APIM, YARP) in front of both, routing the extracted endpoints to the new service. The monolith calls the new service for what it owns. Migrate data with dual-writes or events. Once stable, remove the old code from the monolith. Repeat for the next context. Never try a big-bang rewrite; almost all of them fail.

**Trade-off**: The strangler approach is slow (months to years for a large monolith) and requires running two systems in parallel. But the failure rate of big-bang rewrites is ~70%, so slow-and-safe wins.

**Real project**: A 500k-LOC monolith was strangled over 18 months. First extraction: Notifications (low risk, clear boundary). Then Search (different scaling). Then Payments (compliance). After 3 extractions, the team had the muscle memory and the platform; subsequent extractions accelerated. The monolith still exists as the "core" service but is now 40% smaller.

## Summary Checklist

- [ ] I can define microservices as small, independently deployable services owning one bounded context each.
- [ ] I can articulate when microservices is overkill (small team, MVP, no DevOps maturity) and when it pays off (many teams, separable contexts).
- [ ] I can choose service boundaries via [DDD](domain-driven-design.md) bounded contexts rather than technical layers.
- [ ] I can mix sync (gRPC/REST + Polly) and async (Service Bus + [outbox pattern](outbox-pattern.md)) appropriately.
- [ ] I can explain why each service must own its database and how to share data through APIs/events.
- [ ] I can handle multi-service workflows with the [saga pattern](saga-pattern.md) and idempotent consumers.
- [ ] I can deploy on AKS or Container Apps with KEDA / HPA for independent scaling.
- [ ] I can wire OpenTelemetry for distributed tracing across services.
- [ ] I can recognize a "distributed monolith" (services that must deploy together) and fix the boundaries.
- [ ] I can lead a strangler-fig migration from monolith to microservices without a big-bang rewrite.
