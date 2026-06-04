# Scalability Design

## What It Is

Scalability design is the discipline of making a backend system **absorb more load** — more requests per second, more concurrent users, more data, more tenants — without degrading latency, blowing the cost budget, or destabilizing operations. It is not "add more servers." It is the choice of *which* axis to scale (compute, database, cache, queue), *when* to scale it (now, at the next traffic event, never), and *what* trade-offs that scaling introduces (consistency, complexity, cost).

A scalable system is one where doubling load roughly doubles cost and leaves latency flat — instead of doubling latency and quadrupling cost.

```csharp
// Not scalable — state lives in process memory; two pods serve different truth.
public class CartService
{
    private static readonly Dictionary<Guid, Cart> _carts = new();   // in-process state
    public Cart Get(Guid customerId) => _carts.GetValueOrDefault(customerId) ?? new Cart();
    public void Save(Cart cart) => _carts[cart.CustomerId] = cart;
}

// Scalable — state in Redis. Any pod can serve any request.
public class CartService(IDistributedCache cache)
{
    public async Task<Cart> GetAsync(Guid customerId, CancellationToken ct)
    {
        var bytes = await cache.GetAsync($"cart:{customerId}", ct);
        return bytes is null ? new Cart() : JsonSerializer.Deserialize<Cart>(bytes)!;
    }
    public Task SaveAsync(Cart cart, CancellationToken ct) =>
        cache.SetAsync($"cart:{cart.CustomerId}", JsonSerializer.SerializeToUtf8Bytes(cart),
            new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromHours(2) }, ct);
}
```

## Why It Exists

Production load grows asymmetrically. A successful product never sees "uniform 20% growth across all axes." It sees:

- Reads grow 100x while writes grow 5x (catalog browsing vs purchases).
- One tenant becomes 60% of total traffic.
- A marketing campaign creates a 50x spike for 30 minutes, then nothing.
- Geographic expansion moves the latency-sensitive customer base to a region your service does not live in.
- A new feature introduces a query that joins three big tables and brings the database to its knees.

Scalability design exists because the things that worked at 100 requests/sec break at 100,000, and the breakage patterns are predictable: thread-pool exhaustion, database connection saturation, cache stampedes, hot partitions, lock contention, queue backlogs, and CDN misses. Each has a known remedy, but applying them in the wrong order — sharding before you indexed properly, caching data that needs to be fresh, adding instances when the bottleneck is SQL — wastes money and adds complexity without solving the problem.

The pattern that separates senior engineers from junior ones is **measure first, then choose the smallest effective scaling move**.

## When To Use It

**Design for scalability when:**

- Traffic is growing, even slowly, and current capacity will be exceeded in 6-12 months.
- You have a known traffic event coming (product launch, Black Friday, regulatory deadline).
- You are at the design stage of a system expected to grow (greenfield, replatform).
- A specific bottleneck has appeared in production (slow query, hot partition, queue backlog).
- You operate a multi-tenant system where one tenant can starve others.

**Do not over-engineer for scale when:**

- Traffic is flat and well within current capacity headroom.
- The product has not validated market fit (premature optimization).
- The workload is internal-only (10 admins) and stays that way.
- The "improvement" introduces distributed complexity (sharding, eventual consistency) for no measurable load benefit.

A common interview red flag: candidates who reach for sharding, microservices, and event sourcing for a system that handles 10 requests/sec. Senior engineers ask "what's the bottleneck today?" before reaching for the toolkit.

## Why It Is Important

Scalability is what separates a system that survives growth from one that gets replatformed every 18 months. Five production properties depend on it:

1. **Predictable latency under load.** Stateless app tiers, properly sized connection pools, and CDN-cached static content keep p95 flat as RPS rises.
2. **Cost efficiency.** Auto-scale rules add capacity when needed and remove it when idle — instead of paying for peak capacity 24/7.
3. **Isolation and fairness.** Sharding by tenant, rate-limiting per customer, and queue priorities prevent one noisy tenant from degrading service for everyone.
4. **Operational resilience.** A queue absorbing a 10x spike protects downstream services; without it, the spike becomes an outage.
5. **Geographic reach.** CDNs and multi-region deployments serve users near them with single-digit-ms latency — impossible from a single region.

In a cloud context (Azure App Service, AKS, Container Apps, Cosmos DB), the platform provides the *mechanisms* (auto-scale rules, replicas, partitions) but the application must be *built for them*: stateless, externalized session, partitioned cleanly, idempotent for safe retries.

## How It's Used in C# / .NET

### 1. Stateless app tier with externalized state

```csharp
// Program.cs — externalize session, cache, and data so the app tier is replaceable.
builder.Services.AddStackExchangeRedisCache(o =>
    o.Configuration = builder.Configuration["Redis:ConnectionString"]);

builder.Services.AddDataProtection()                       // share keys across instances
    .PersistKeysToAzureBlobStorage(builder.Configuration["DataProtection:BlobUri"]!)
    .ProtectKeysWithAzureKeyVault(new Uri(builder.Configuration["DataProtection:KvKey"]!),
                                  new DefaultAzureCredential());
```

Without shared Data Protection keys, two pods cannot decrypt each other's cookies — a classic sticky-session failure mode.

### 2. Azure App Service auto-scale rules

```jsonc
// Infrastructure as code (Bicep/Terraform) — scale on CPU and queue depth.
"autoscaleSettings": {
  "profiles": [{
    "name": "Default",
    "capacity": { "minimum": "3", "maximum": "20", "default": "3" },
    "rules": [
      { "metricTrigger": { "metricName": "CpuPercentage", "operator": "GreaterThan",
                            "threshold": 70, "timeWindow": "PT5M" },
        "scaleAction": { "direction": "Increase", "value": "2", "cooldown": "PT5M" } },
      { "metricTrigger": { "metricName": "CpuPercentage", "operator": "LessThan",
                            "threshold": 30, "timeWindow": "PT15M" },
        "scaleAction": { "direction": "Decrease", "value": "1", "cooldown": "PT10M" } }
    ]
  }]
}
```

For queue-driven workers, scale on **queue depth**, not CPU:

```jsonc
{ "metricTrigger": { "metricName": "ActiveMessageCount", "metricNamespace": "Microsoft.ServiceBus/namespaces",
                     "metricResourceUri": "/subscriptions/.../topics/orders/subscriptions/billing",
                     "operator": "GreaterThan", "threshold": 1000, "timeWindow": "PT5M" },
  "scaleAction": { "direction": "Increase", "value": "3" } }
```

### 3. Distributed cache with Redis

```csharp
public class ProductCatalog(IDistributedCache cache, ProductDb db)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct)
    {
        var key = $"product:{id}";
        var cached = await cache.GetStringAsync(key, ct);
        if (cached is not null) return JsonSerializer.Deserialize<Product>(cached);

        var p = await db.Products.FindAsync([id], ct);
        if (p is null) return null;

        await cache.SetStringAsync(key, JsonSerializer.Serialize(p),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) }, ct);
        return p;
    }
}
```

### 4. Output caching and response caching

```csharp
// ASP.NET Core 8+ output caching for read-heavy endpoints.
builder.Services.AddOutputCache(o =>
{
    o.AddPolicy("catalog", p => p.Expire(TimeSpan.FromMinutes(5))
                                  .SetVaryByQuery("category", "page")
                                  .Tag("catalog"));
});

app.MapGet("/api/products", async (string category, int page, ProductDb db) =>
    await db.Products.Where(p => p.Category == category).Skip(page * 20).Take(20).ToListAsync())
   .CacheOutput("catalog");
```

### 5. Queue-based load leveling — Service Bus + `BackgroundService`

```csharp
// Producer: accept the request fast, queue the heavy work.
app.MapPost("/api/exports", async (ExportRequest req, ServiceBusSender sender) =>
{
    var jobId = Guid.NewGuid();
    await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.Serialize(new { jobId, req }))
        { MessageId = jobId.ToString() });
    return Results.Accepted($"/api/exports/{jobId}", new { jobId });
});

// Consumer: parallelism tuned to downstream capacity.
var processor = sbClient.CreateProcessor("export-jobs", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 16,
    PrefetchCount = 50,
    AutoCompleteMessages = false
});
```

### 6. Database read replicas

```csharp
// EF Core 8+ — separate DbContext for read replicas.
builder.Services.AddDbContext<OrderDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("OrdersPrimary")));

builder.Services.AddDbContextFactory<OrderReadDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("OrdersReadReplica"))
     .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));

// Use the read replica for reporting / read-heavy queries; primary for writes.
public class OrderReportService(IDbContextFactory<OrderReadDbContext> readFactory)
{
    public async Task<IReadOnlyList<OrderSummary>> GetMonthlySummariesAsync(int year, int month, CancellationToken ct)
    {
        using var db = await readFactory.CreateDbContextAsync(ct);
        return await db.Orders
            .Where(o => o.PlacedAt.Year == year && o.PlacedAt.Month == month)
            .GroupBy(o => o.CustomerId)
            .Select(g => new OrderSummary(g.Key, g.Count(), g.Sum(o => o.AmountCents)))
            .ToListAsync(ct);
    }
}
```

### 7. CQRS for read scale

When read patterns diverge from write patterns at high volume, split them. Writes go through the domain model; reads hit a denormalized projection (often in Cosmos DB or a separate SQL schema). See [cqrs.md](cqrs.md).

### 8. CDN for static and edge-cacheable content

```csharp
// Set Cache-Control so Azure Front Door / CloudFront / Akamai can cache aggressively.
app.MapGet("/api/products/{id}/image", (int id) =>
    Results.File(File.OpenRead($"images/{id}.webp"), "image/webp"))
   .WithMetadata(new ResponseCacheAttribute { Duration = 86400, Location = ResponseCacheLocation.Any });
```

### 9. Rate limiting

```csharp
// ASP.NET Core 8+ built-in rate limiter — per-user fixed window.
builder.Services.AddRateLimiter(o =>
{
    o.AddPolicy("per-user", ctx =>
        RateLimitPartition.GetFixedWindowLimiter(
            ctx.User.FindFirst("sub")?.Value ?? ctx.Connection.RemoteIpAddress?.ToString() ?? "anon",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                QueueLimit = 0
            }));
});

app.MapPost("/api/payments", PaymentHandler).RequireRateLimiting("per-user");
```

## Advantages

- **Linear cost scaling** — pay for what you use, not for peak capacity year-round.
- **Latency isolation** — caching and CDNs serve common reads in microseconds.
- **Spike absorption** — queues turn unpredictable load into predictable throughput.
- **Workload independence** — scale read and write paths independently with replicas / CQRS.
- **Tenant fairness** — sharding and rate limits prevent noisy-tenant impact.
- **Geographic reach** — multi-region and CDN cut latency for global users.

## Disadvantages

- **Operational complexity** — every new tier (cache, queue, replica, shard) is another thing to monitor, patch, and fail-over.
- **Cache invalidation** is famously hard; stale data causes subtle bugs.
- **Sharding** complicates joins, transactions, and re-balancing.
- **Replica lag** introduces visible eventual consistency.
- **Queueing** trades latency for throughput; users wait longer for results.
- **Auto-scale lag** — scaling up takes minutes; sudden spikes still hit cold capacity.
- **Cost surprises** — auto-scale without max limits can produce shocking bills during traffic anomalies.

## Common Mistakes

### 1. Adding instances when the database is the bottleneck

```csharp
// BAD — scaling App Service from 3 to 30 pods didn't help; each pod still queues on the same SQL.
// p95 latency stayed flat at 4 seconds because Azure SQL CPU was already at 95%.
```

**Fix** — profile first. Use Azure SQL Query Performance Insight or `dotnet-counters`. If the bottleneck is SQL, add indexes, rewrite the query, scale up the tier, or add a read replica *before* adding app instances.

### 2. Caching data that must be fresh

```csharp
// BAD — caching payment status for 5 minutes; users see "pending" after their card was charged.
[ResponseCache(Duration = 300)]
public Task<PaymentStatus> GetStatus(Guid paymentId) => _payments.GetStatusAsync(paymentId);
```

**Fix** — only cache data that's tolerant to staleness (catalog, prices for non-promo periods, public profiles). Never cache user-specific state-critical data without invalidation.

### 3. In-process state in a horizontally scaled service

```csharp
// BAD — session, rate-limit counters, or cart in process memory. Pod B doesn't know what Pod A saw.
private static readonly ConcurrentDictionary<string, int> _attempts = new();
```

**Fix** — externalize to Redis or the database. Or accept "sticky session" only for low-stakes UX state, and even then prefer not to.

### 4. Cache stampede on cold cache

```csharp
// BAD — TTL expires; 5000 concurrent requests all miss cache and hit the database simultaneously.
var cached = await cache.GetStringAsync(key);
if (cached is null)
{
    var data = await db.GetAsync();                      // 5000 concurrent DB calls
    await cache.SetStringAsync(key, JsonSerializer.Serialize(data), ttl);
}
```

**Fix** — single-flight pattern (one request fills the cache, others wait), jittered TTLs across keys, or stale-while-revalidate semantics. `FusionCache` or `HybridCache` handles this for you.

### 5. Premature sharding

```csharp
// BAD — sharded a 50GB database across 8 shards "for the future"; now every join is application-level
// and analytics queries are a nightmare. Total RPS: 200.
```

**Fix** — exhaust simpler options first: indexes, query rewrites, vertical scale, read replicas, partitioning within one database. Shard only when one database physically cannot hold the workload.

### 6. Auto-scale without max limit

```jsonc
// BAD — no maximum. A retry storm caused App Service to scale to 200 instances overnight.
// Bill: $42,000 for one day.
"capacity": { "minimum": "3", "default": "3" }   // no maximum!
```

**Fix** — always set a maximum. Alert on `instance count > X` as a cost guardrail.

### 7. Scale on CPU for I/O-bound workers

```jsonc
// BAD — Service Bus consumer is mostly waiting for downstream calls; CPU stays low; never scales out.
// Queue depth grows to 100k while autoscale sleeps.
```

**Fix** — scale queue consumers on **queue depth or message age**, not CPU. KEDA on AKS does this elegantly for any keda-supported source.

## Best Practices

- **Measure before you scale.** Application Insights, `dotnet-counters`, SQL Query Store, queue-depth dashboards.
- **Stateless app tier** — externalize session, cart, rate-limit counters, and Data Protection keys.
- **Cache aggressively where staleness is acceptable**; never cache state-critical reads.
- **Use `HybridCache` (or `FusionCache`)** for stampede protection and L1/L2 layering.
- **Set both min and max** on auto-scale rules; alert when max is hit.
- **Scale workers on queue depth or message age**, not CPU.
- **Read replicas for read-heavy queries** — reporting, lists, dashboards.
- **Pre-warm caches and pre-scale instances** before known traffic events.
- **Rate-limit per user/tenant** to enforce fairness and protect dependencies.
- **Use a CDN** for any static or rarely-changing content.
- **Partition by a high-cardinality key** (customerId, tenantId, orderId) so no single partition is hot.
- **Plan for re-sharding** from day one — store the shard key with the data, not derived from it.

## Related Concepts

- [reliability-design.md](reliability-design.md) — scalability without reliability creates beautiful outages.
- [cqrs.md](cqrs.md) — separate read and write models for asymmetric scale.
- [event-driven-architecture.md](event-driven-architecture.md) — queues are the scaling glue.
- [microservices-architecture.md](microservices-architecture.md) — independent scaling per service.
- [data-access/query-performance.md](../data-access/query-performance.md) — indexes and query plans are usually the first lever.
- [azure/app-service.md](../azure/app-service.md) — auto-scale rules and Premium tier.
- [azure/azure-service-bus.md](../azure/azure-service-bus.md) — partitioned queues for throughput.
- [azure/azure-sql.md](../azure/azure-sql.md) — read replicas, geo-replication, Hyperscale.

## Real-World Usage

### Storefront API on Azure App Service

The product catalog and search endpoints are output-cached for 5 minutes and tagged for selective invalidation when inventory changes. Static assets (images, JS) are served by Azure Front Door CDN. The app tier auto-scales 3→20 instances on CPU. Azure SQL has one read replica for catalog queries and a primary for write operations. p95 latency stays under 200ms even during a 30x flash-sale spike.

### Payment processing on AKS

Payment requests land on Azure Service Bus. A `BackgroundService` consumer in AKS uses KEDA to scale 2→50 pods based on `ActiveMessageCount`. Each pod handles 16 messages concurrently. Downstream calls to the payment provider have a concurrency limiter (bulkhead) of 200 across the cluster to respect the provider's rate limit. Queue depth alerts at 5000 messages.

### Multi-tenant SaaS

A SaaS platform shards customer data by `tenantId`. Small tenants share a "shared shard" Azure SQL database; large tenants get their own dedicated database. A tenant router maps `tenantId → connection string` via a cached lookup. Background workers are partitioned by tenant prefix to avoid noisy-tenant impact.

### Event ingestion at scale

A telemetry pipeline ingests 80k events/sec. Events land in Event Hubs (32 partitions, partitioned by `deviceId` for ordering per device). A consumer group writes batches to Cosmos DB partitioned on `deviceId`. Read-side analytics queries hit Azure Data Explorer (Kusto), not the operational store.

### Cart on Redis

Cart state lives in Redis with a 24-hour sliding TTL. Any App Service pod can serve any customer because the cart is not in process memory. Redis is sized for 2x peak working set, and the cart key includes a tenant prefix so multi-tenant evictions are contained.

## Code Example — Before and After

### Before — single instance, in-process state, synchronous heavy work

```csharp
public class OrderController : ControllerBase
{
    private static readonly ConcurrentDictionary<Guid, OrderDraft> _drafts = new();   // in-process
    private readonly OrderDbContext _db;
    private readonly IInvoiceRenderer _renderer;
    private readonly IEmailSender _email;

    [HttpPost]
    public async Task<IActionResult> Submit(SubmitOrderRequest req)
    {
        // Heavy synchronous work inside the request thread.
        var order = await CreateOrderAsync(req);
        var invoicePdf = await _renderer.RenderAsync(order);                  // 4-6 seconds
        await _email.SendAsync(order.CustomerEmail, "Your invoice", invoicePdf);  // 2-3 seconds

        _drafts.TryRemove(req.DraftId, out _);                                // local only
        return Ok(order);
    }
}
```

Production failure modes:
- One App Service instance + sticky drafts means horizontal scale is impossible.
- 6-9 second request latency exhausts the thread pool under load.
- Email provider outage takes down checkout entirely.
- A traffic spike causes 502s because there is no buffer.

### After — stateless, queued, autoscalable, cacheable

```csharp
public class OrderController(IDistributedCache cache, OrderDbContext db,
                             ServiceBusSender sender, ILogger<OrderController> log) : ControllerBase
{
    [HttpPost("draft")]
    public async Task<IActionResult> SaveDraft(OrderDraft draft, CancellationToken ct)
    {
        await cache.SetStringAsync($"draft:{draft.Id}",
            JsonSerializer.Serialize(draft),
            new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromHours(1) }, ct);
        return Ok();                                                       // any pod can serve later
    }

    [HttpPost("submit")]
    public async Task<IActionResult> Submit(SubmitOrderRequest req, CancellationToken ct)
    {
        // 1. Synchronous: just persist the order. Fast (< 50ms).
        var order = new Order(req.CustomerId, req.Items, req.AmountCents);
        db.Orders.Add(order);
        db.OutboxMessages.Add(OutboxMessage.From(new OrderPlaced(order.Id, order.CustomerId, order.AmountCents)));
        await db.SaveChangesAsync(ct);

        // 2. Asynchronous: invoice + email handled by workers, scale independently.
        await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.Serialize(
            new RenderInvoiceJob(order.Id))) { MessageId = $"invoice-{order.Id}" }, ct);

        return Accepted($"/api/orders/{order.Id}", new { order.Id });
    }
}

// Separate worker, scaled by queue depth via KEDA.
public class InvoiceWorker(ServiceBusClient sb, IInvoiceRenderer renderer,
                           IEmailSender email, OrderDbContext db) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var processor = sb.CreateProcessor("invoice-jobs", new ServiceBusProcessorOptions
        {
            MaxConcurrentCalls = 8,
            AutoCompleteMessages = false
        });
        processor.ProcessMessageAsync += async args =>
        {
            var job = JsonSerializer.Deserialize<RenderInvoiceJob>(args.Message.Body.ToString())!;
            var order = await db.Orders.FindAsync([job.OrderId], ct);
            var pdf = await renderer.RenderAsync(order!);
            await email.SendAsync(order!.CustomerEmail, "Your invoice", pdf);
            await args.CompleteMessageAsync(args.Message);
        };
        processor.ProcessErrorAsync += e => { /* log */ return Task.CompletedTask; };
        await processor.StartProcessingAsync(ct);
        try { await Task.Delay(Timeout.Infinite, ct); }
        catch (OperationCanceledException) { }
        finally { await processor.StopProcessingAsync(CancellationToken.None); }
    }
}
```

What's better:
- API responds in < 100ms; the customer is not blocked on PDF rendering or email.
- API tier and worker tier scale independently — workers can burst to 50 pods for an invoice backlog while the API stays at 5.
- Draft state in Redis works across any pod; sticky sessions are gone.
- An email provider outage stalls the queue, but checkout continues unaffected.

## Interview Questions and Answers

### 1. Walk me through how you'd diagnose a sudden latency spike on the checkout API.

**Why this matters** — tests measurement-first instinct over guess-and-scale.

**Answer** — Open Application Insights and look at the failure/duration breakdown by dependency. Most spikes are downstream: SQL CPU, Service Bus throttling, an upstream provider, or a thread-pool exhaustion. Check `dotnet-counters` for the thread pool, GC, and `ThreadPool.PendingWorkItemCount`. Check SQL Query Store for any new top-CPU query. Only after locating the actual bottleneck do I consider scaling — and I scale the bottleneck, not the API tier.

**Trade-off** — Scaling app instances feels productive but does nothing if SQL is at 95% CPU.

**Real project** — A retail checkout API showed 5x latency on Black Friday. Initial reaction was to scale the App Service plan. The real cause was a missing index on a new `OrderItems.ProductId` query. Adding the index dropped p95 from 4s to 180ms; scale-out was reverted.

### 2. When would you choose horizontal scaling over vertical scaling?

**Why this matters** — tests judgment on simplicity vs. ceiling.

**Answer** — Vertical scaling (bigger VM, higher SQL tier) is simpler and right when (a) the workload is hard to parallelize, (b) the database is the bottleneck and you haven't hit the tier ceiling, or (c) you want a quick fix while engineering a real solution. Horizontal scaling is right for stateless workloads (web tiers, queue consumers), when you've hit the vertical ceiling, or when you need redundancy across instances/zones for availability. Most real systems use both: scale up the database, scale out the app tier.

**Trade-off** — Horizontal scaling requires the application to be stateless; that engineering work is the real cost.

**Real project** — An invoicing API was running on a single P2v3 App Service plan, hitting CPU limits during month-end. We scaled vertically to P3v3 first (5-minute fix) to survive the night, then refactored to externalize state and enabled horizontal scale-out 3→10 for the next cycle.

### 3. Caching seems magical. What can go wrong?

**Why this matters** — surfaces understanding of cache invalidation and stampede.

**Answer** — Five common failure modes: (1) **Stale data** when the source changes but the cache isn't invalidated — solved by event-driven invalidation or short TTL on freshness-sensitive data. (2) **Cache stampede** when many requests miss simultaneously and all hit the database — solved by single-flight (one request fills, others wait), jittered TTLs, or `HybridCache`. (3) **Cache poisoning** when a bad write enters the cache and persists — solved by cache versioning and TTLs. (4) **Memory pressure** when working set exceeds Redis size and evictions cause every request to miss — solved by sizing for working set + headroom. (5) **Caching too much** — caching user-specific state-critical data (payment status, balances) causes bugs.

**Trade-off** — Caching is the highest-leverage scalability tool but the highest-blame-radius bug when it goes wrong.

**Real project** — A catalog cache stampede during a sale brought down Azure SQL — 8000 concurrent cache misses generated 8000 simultaneous queries. Moving to `FusionCache` with single-flight semantics eliminated the stampede.

### 4. Walk me through designing for a flash-sale traffic spike.

**Why this matters** — tests a full mental model of layered scaling.

**Answer** — Layered approach. (1) **CDN** caches product images, descriptions, and "is this product available?" reads — most traffic never reaches origin. (2) **Pre-scale** the App Service plan to the expected peak 30 minutes before the sale (autoscale lag is real). (3) **Rate-limit per user** to prevent bots from grabbing 1000 units. (4) **Queue purchase attempts** into Service Bus so backend processing is decoupled from the spike. (5) **Hot inventory in Redis** with atomic decrement (`DECR`) to handle 50k reservations/sec without hitting SQL. (6) **Graceful degradation** — hide recommendations, simplify the page during peak. (7) **Reconcile** post-sale: queued attempts → final orders → settled payments.

**Trade-off** — Queuing makes purchases asynchronous, so users see "your order is being processed" instead of instant confirmation.

**Real project** — A footwear drop expected 100k concurrent customers for 50k units. CDN absorbed 92% of traffic. Redis `DECR` reserved inventory atomically. Service Bus queued payment captures. The site stayed up at p95 < 500ms; SQL CPU peaked at 40%.

### 5. When would you shard a database, and what does that cost you?

**Why this matters** — most candidates either shard too early or never; senior engineers know the criteria.

**Answer** — Shard only when (a) one database physically cannot hold the working set or sustain the IOPS even at the highest tier, (b) the workload partitions cleanly by a single high-cardinality key (tenantId, customerId), and (c) you've already exhausted indexes, query optimization, vertical scale, read replicas, and table partitioning. Costs: cross-shard queries become application-level joins, transactions cannot span shards, re-balancing requires careful migration, and reporting/analytics gets much harder (usually solved by ETL to a separate analytical store).

**Trade-off** — Sharding solves a real scale problem at the cost of permanent engineering complexity. Avoid it until it's unavoidable.

**Real project** — A multi-tenant SaaS hit 4TB on a single Azure SQL Hyperscale instance and had a tenant doing 60% of IOPS. Sharded by `tenantId`: large tenants got dedicated databases, small tenants shared a pool. Took 6 months to migrate; eliminated the noisy-tenant problem permanently.

### 6. Why scale queue consumers on queue depth instead of CPU?

**Why this matters** — common mistake; tests understanding of bottleneck types.

**Answer** — Queue consumers are typically I/O-bound — they spend most of their time waiting on database writes, downstream HTTP calls, or other dependencies. Their CPU stays low even when overloaded. Autoscaling on CPU means the scaler never triggers, the queue grows unbounded, and message age explodes. The real signal of "we need more consumers" is **how much work is waiting** — `ActiveMessageCount` or `MessageAge`. KEDA on AKS exposes these as native scaling triggers; Azure Functions and Service Bus scale this way automatically.

**Trade-off** — Queue-depth scaling can overshoot if the downstream is the actual bottleneck — add a concurrency limit to prevent overwhelming downstream.

**Real project** — A notification worker consumed Service Bus and called SendGrid. CPU autoscale never triggered (workers waited on HTTP). A 200k-message backlog grew overnight. Switching to KEDA with `ActiveMessageCount > 1000` triggers scaled 2 → 30 pods in 90 seconds and drained the backlog in 15 minutes.

### 7. How do you keep p95 latency flat as RPS grows 10x?

**Why this matters** — flat latency under load is the canonical scalability outcome.

**Answer** — Three principles. (1) **Remove serial dependencies** — every request that goes through SQL inherits SQL's contention curve; cache or precompute what you can. (2) **Size connection pools** so the app tier never queues for SQL/Redis/HTTP connections under peak load. (3) **Isolate slow work** — anything that takes > 200ms goes through a queue and a background worker, not the request thread. Combine with auto-scale and proper concurrency limits, and p95 stays flat because each request now does less synchronous work and contends for fewer shared resources.

**Trade-off** — Async-via-queue improves latency for the API but adds a "your request is processing" UX layer.

**Real project** — A reporting API had p95 of 8s at 100 RPS. Moved the report generation to a queued worker, served the request from a denormalized read model in Cosmos, and added Redis cache for hot reports. At 1000 RPS, p95 was 120ms.

### 8. Replica lag bit you in production. How do you handle it?

**Why this matters** — eventual consistency is a real customer-visible problem.

**Answer** — First, understand which queries actually need read-after-write consistency. Most reads (reporting, history, search) tolerate seconds of lag; some (the user's own freshly created order, the payment status they just paid) do not. Route read-after-write critical queries to the primary; route everything else to replicas. For user-visible operations, the simplest pattern is "after a write, the next read for that user goes to primary for N seconds" (session-bound read consistency). Surface lag in your application's telemetry and alert if it exceeds the threshold the product can tolerate.

**Trade-off** — Routing read-after-write to primary costs replica scale benefit; route only what truly needs it.

**Real project** — An order confirmation page hit a read replica and sometimes showed "order not found" right after creation. Adding session-pinned reads to primary for 5 seconds after any write eliminated the user-visible inconsistency without re-routing 90% of read traffic.

## Summary Checklist

- [ ] Measure the bottleneck before scaling — CPU, SQL, queue depth, downstream latency.
- [ ] App tier is stateless; session/cart/rate-counters live in Redis or SQL, not process memory.
- [ ] Data Protection keys are shared across instances (Blob + Key Vault).
- [ ] Auto-scale rules have both minimum and maximum, and the maximum has a cost-guardrail alert.
- [ ] Queue consumers scale on queue depth or message age, not CPU.
- [ ] Reads tolerant to staleness are cached (`HybridCache` or Redis); reads needing freshness are not.
- [ ] Read-heavy queries hit a read replica; writes hit the primary.
- [ ] Heavy or slow work is queued; the request thread returns fast.
- [ ] Rate limiting is in place per user/tenant for any expensive endpoint.
- [ ] Static and cacheable content is served via CDN with explicit `Cache-Control`.
- [ ] Partition/shard keys are chosen for even distribution; no single hot partition.
- [ ] Capacity headroom is documented and a load test validates it before known traffic events.

