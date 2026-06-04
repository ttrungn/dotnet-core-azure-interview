# Reliability Design

## What It Is

Reliability design is the deliberate practice of making a backend system continue to behave **acceptably** when its dependencies are slow, throttled, partially broken, or completely down. It is not a single library or a single architectural diagram. It is a layered set of decisions — timeouts, retries, circuit breakers, bulkheads, health checks, idempotency, queueing, fallback paths, and observability — that together protect users from the failures that *will* happen in production.

A reliable system does not mean a system that never fails. It means a system whose failures are **bounded, recoverable, and visible**.

```csharp
// Unreliable: no timeout, no retry, no breaker, no idempotency.
// One slow call to Stripe pins an ASP.NET Core request thread for 90 seconds.
public async Task<ChargeResult> ChargeAsync(Order order)
{
    var resp = await _http.PostAsJsonAsync("https://api.stripe.com/v1/charges", order);
    return await resp.Content.ReadFromJsonAsync<ChargeResult>();
}

// Reliable: bounded latency, transient retry, breaker, idempotency key.
public async Task<ChargeResult> ChargeAsync(Order order, CancellationToken ct)
{
    var ctx = ResilienceContextPool.Shared.Get(ct);
    return await _pipeline.ExecuteAsync(async c =>
    {
        using var req = new HttpRequestMessage(HttpMethod.Post, "v1/charges");
        req.Headers.Add("Idempotency-Key", order.PaymentIntentKey);   // safe to retry
        req.Content = JsonContent.Create(order);
        var resp = await _http.SendAsync(req, c.CancellationToken);
        resp.EnsureSuccessStatusCode();
        return (await resp.Content.ReadFromJsonAsync<ChargeResult>(c.CancellationToken))!;
    }, ctx);
}
```

## Why It Exists

Every production backend eventually hits the same wall: the network is unreliable, dependencies have bad days, and one slow call upstream can take an entire fleet down. Reliability engineering exists because the failure modes that ruin a system in production are almost never the ones in the unit tests.

Concretely, reliability design exists to solve these recurring incidents:

- A payment provider's latency spikes from 200ms to 30s. Without timeouts, every ASP.NET Core thread is parked, the thread pool starves, and the entire checkout API returns 503 even for endpoints that do not call payments.
- A transient TCP reset on Azure SQL causes a single `SaveChangesAsync` to throw. Without retries, a checkout that would have succeeded on the next attempt is permanently lost.
- A dependency returns errors. Without a circuit breaker, the API keeps hammering it, the dependency cannot recover, and the upstream caller fans the failure into a retry storm.
- A consumer crashes mid-message. Without idempotency, replaying the message double-charges the customer.
- A deployment introduces a bug. Without health checks, the load balancer keeps routing traffic to the broken instance.

These are not theoretical. They are the standard incidents that mature teams design against from day one.

## When To Use It

**Use reliability patterns for:**

- Any call leaving the process: HTTP to payment/fraud/shipping providers, SQL/Cosmos/Redis, Azure Service Bus, Storage, Key Vault, Graph API.
- Any consumer reading from a queue or topic (idempotency, dead-letter, retry policy).
- Any background worker that processes business-critical work (orders, payments, invoices, notifications).
- Any service whose downtime has direct customer or revenue impact.
- Any system with an SLO commitment to customers or internal teams.

**Do not over-engineer for:**

- One-off internal admin tools used by 3 employees.
- Throwaway scripts and migrations.
- Pure in-memory computation with no external I/O.
- Prototypes and spikes — but mark them clearly so they do not silently become production.

A common mistake is wrapping every method in retries "just in case." Retries on a CPU-bound function add nothing but latency and confusion.

## Why It Is Important

In a senior backend interview and in a real on-call rotation, reliability design is what separates a system that survives Black Friday from one that goes down at 11:47 AM. Four production properties depend on it:

1. **Bounded blast radius.** When the fraud service fails, only fraud-dependent paths degrade — not checkout, not order history, not the admin UI. Bulkheads and circuit breakers enforce this boundary.
2. **Fast failure instead of slow death.** A 2-second timeout that returns "service unavailable, please retry" is dramatically better than a 90-second hang that exhausts threads. Latency budgets must be defended.
3. **Recoverability without data loss.** Idempotency keys, outbox tables, and dead-letter queues mean that a crash, redeploy, or transient outage can be re-driven without double-charging customers or losing orders.
4. **Operational visibility.** Health checks, structured logs, traces, metrics, and SLO-based alerts are what let a 3 AM on-call engineer identify the failing component in minutes instead of hours.

In a cloud context (Azure App Service, AKS, Container Apps, Functions), the platform gives you scaling, restarts, and load-balancer integration *only if* your application exposes the right hooks: readiness probes, graceful shutdown, structured logs to Application Insights, and resilient outbound calls.

## How It's Used in C# / .NET

The modern .NET reliability stack is built around three pillars: **Polly v8** for resilience policies, **`AddHealthChecks`** for liveness/readiness, and **`Microsoft.Extensions.Hosting`** for graceful shutdown of background workers.

### 1. Polly v8 — `ResiliencePipeline`

Polly v8 replaced the old `IAsyncPolicy<T>` API with a strategy-based `ResiliencePipelineBuilder`. Each strategy (retry, circuit breaker, timeout, hedging, rate limiter) composes into one pipeline.

```csharp
// NuGet: Polly, Polly.Extensions
using Polly;
using Polly.Retry;
using Polly.CircuitBreaker;
using Polly.Timeout;

var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddTimeout(TimeSpan.FromSeconds(2))                              // per-attempt timeout
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay            = TimeSpan.FromMilliseconds(200),
        BackoffType      = DelayBackoffType.Exponential,
        UseJitter        = true,
        ShouldHandle     = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>()
            .HandleResult(r => (int)r.StatusCode >= 500 || r.StatusCode == HttpStatusCode.RequestTimeout)
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio     = 0.5,                       // 50% of probes fail
        MinimumThroughput = 20,                       // need at least 20 calls in window
        SamplingDuration  = TimeSpan.FromSeconds(30),
        BreakDuration     = TimeSpan.FromSeconds(15)
    })
    .Build();
```

### 2. `AddResilienceHandler` — wire Polly into `HttpClient`

The cleanest production pattern is to attach the pipeline to an `IHttpClientFactory`-managed `HttpClient` so every outbound call inherits resilience.

```csharp
// NuGet: Microsoft.Extensions.Http.Resilience
builder.Services.AddHttpClient<IStripeClient, StripeClient>(c =>
{
    c.BaseAddress = new Uri("https://api.stripe.com/");
    c.Timeout = TimeSpan.FromSeconds(30);          // hard ceiling
})
.AddResilienceHandler("stripe", pipelineBuilder =>
{
    pipelineBuilder
        .AddTimeout(TimeSpan.FromSeconds(3))
        .AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(300),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true
        })
        .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.4,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(20)
        });
});
```

### 3. Hedging — racing slow primaries

When p99 latency matters more than minimizing call count (read-only, idempotent paths), hedging fires a second attempt if the first is still pending after a threshold.

```csharp
pipelineBuilder.AddHedging(new HttpHedgingStrategyOptions
{
    MaxHedgedAttempts = 2,
    Delay = TimeSpan.FromMilliseconds(150)
});
```

### 4. Bulkhead via `RateLimiter` strategy

Polly v8 ships a rate-limiter strategy that doubles as a bulkhead — bounding concurrent calls to a dependency so one slow service cannot consume the whole thread pool.

```csharp
pipelineBuilder.AddConcurrencyLimiter(permitLimit: 50, queueLimit: 20);
```

### 5. Health checks — `AddHealthChecks`

```csharp
// NuGet: AspNetCore.HealthChecks.SqlServer, .AzureServiceBus, .Redis
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Sql")!, name: "sql", tags: ["ready"])
    .AddAzureServiceBusQueue(builder.Configuration["ServiceBus:Connection"]!, "orders", tags: ["ready"])
    .AddRedis(builder.Configuration["Redis:Connection"]!, tags: ["ready"]);

var app = builder.Build();
app.MapHealthChecks("/health/live",  new HealthCheckOptions { Predicate = _ => false });        // process alive
app.MapHealthChecks("/health/ready", new HealthCheckOptions { Predicate = r => r.Tags.Contains("ready") });
```

Kubernetes / Azure App Service then probe `/health/live` and `/health/ready` separately. Failing readiness removes the instance from the load balancer; failing liveness restarts the pod.

### 6. Graceful shutdown

```csharp
public class OrderConsumer(ServiceBusProcessor processor, ILogger<OrderConsumer> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        processor.ProcessMessageAsync += HandleAsync;
        await processor.StartProcessingAsync(stoppingToken);
        try { await Task.Delay(Timeout.Infinite, stoppingToken); }
        catch (OperationCanceledException) { /* shutdown */ }
        finally
        {
            log.LogInformation("Stopping Service Bus processor, draining in-flight messages...");
            await processor.StopProcessingAsync(CancellationToken.None);  // let messages finish
        }
    }
}
```

### 7. Idempotency keys

```csharp
// Client supplies a key for every payment attempt; server stores key + result.
public async Task<PaymentResult> CapturePaymentAsync(Guid orderId, string idempotencyKey, CancellationToken ct)
{
    var existing = await _db.PaymentAttempts.FirstOrDefaultAsync(p => p.IdempotencyKey == idempotencyKey, ct);
    if (existing is not null) return existing.Result;        // safe replay

    var result = await _gateway.ChargeAsync(orderId, ct);
    _db.PaymentAttempts.Add(new PaymentAttempt(idempotencyKey, result));
    await _db.SaveChangesAsync(ct);
    return result;
}
```

## Advantages

- **Bounded latency** — timeouts mean one slow dependency cannot starve the thread pool.
- **Cascading-failure containment** — circuit breakers stop fanning load into a struggling service.
- **Higher effective availability** — transient failures recover invisibly through retries.
- **Faster incident diagnosis** — health checks and structured telemetry point to the failing component.
- **Safer deploys** — readiness probes pull broken pods out of rotation before customers notice.
- **Operational repair** — dead-letter queues and outbox tables preserve work that crashed mid-flight.

## Disadvantages

- **Configuration sprawl** — every dependency now has timeout, retry, breaker, and bulkhead knobs that must be tuned and tested.
- **Retry storms** when retries are added without backoff/jitter or without a breaker upstream.
- **Hidden failures** when fallbacks silently return empty/cached data without alerting.
- **Increased latency on the happy path** if timeouts are too aggressive or hedging fires too eagerly.
- **Higher cloud cost** for multi-region replicas, observability storage, and idle capacity headroom.
- **Test complexity** — chaos tests, failure injection, and DR drills require time and discipline.

## Common Mistakes

### 1. Retrying non-idempotent operations

```csharp
// BAD — POST /charges without idempotency key. Each retry charges the customer again.
await _httpRetryPipeline.ExecuteAsync(async ct =>
    await _http.PostAsJsonAsync("v1/charges", new { amount = 1000, source = "tok_visa" }, ct));
```

```csharp
// FIX — supply an idempotency key and let the provider deduplicate.
req.Headers.Add("Idempotency-Key", paymentAttempt.Key);
```

### 2. Missing per-attempt timeout

```csharp
// BAD — pipeline retries 3 times, each retry waits forever on a stuck socket.
new ResiliencePipelineBuilder().AddRetry(new() { MaxRetryAttempts = 3 }).Build();
```

```csharp
// FIX — add a per-attempt timeout INSIDE the retry boundary.
new ResiliencePipelineBuilder()
    .AddRetry(new() { MaxRetryAttempts = 3, Delay = TimeSpan.FromMilliseconds(200) })
    .AddTimeout(TimeSpan.FromSeconds(2))
    .Build();
```

### 3. Health check that calls every dependency

```csharp
// BAD — readiness probe takes 15s, runs every 5s, generates more load than real traffic.
builder.Services.AddHealthChecks()
    .AddCheck<DeepDependencyCheck>("everything");   // hits 12 services synchronously
```

```csharp
// FIX — readiness only checks dependencies the *instance* truly needs.
builder.Services.AddHealthChecks()
    .AddSqlServer(sqlConn, name: "sql", tags: ["ready"])
    .AddAzureServiceBusQueue(sbConn, "orders", tags: ["ready"]);
```

### 4. Fallback that hides failure

```csharp
// BAD — silently returns empty list; the dashboard shows "0 orders" instead of an error.
try { return await _orders.GetAsync(customerId, ct); }
catch { return Array.Empty<Order>(); }
```

```csharp
// FIX — return cached/last-known with a flag, and emit a metric so the failure is visible.
try { return await _orders.GetAsync(customerId, ct); }
catch (Exception ex)
{
    _metrics.OrdersFallback.Add(1);
    _log.LogWarning(ex, "Order service degraded, returning cached snapshot for {Customer}", customerId);
    return _cache.GetLastKnown(customerId) ?? throw;
}
```

### 5. Circuit breaker around a method with two dependencies

If one method talks to both Stripe and Service Bus, the breaker can't tell *which one* failed. Wrap each dependency in its own pipeline (its own bulkhead).

### 6. Retrying 4xx responses

`400 Bad Request` and `404 Not Found` are not transient. Retrying them wastes budget and masks bugs. Only retry on `408`, `429`, `5xx`, and connection-level exceptions — and respect `Retry-After`.

### 7. No graceful shutdown for queue consumers

```csharp
// BAD — SIGTERM kills the process mid-message; the message redelivers and double-processes.
protected override Task ExecuteAsync(CancellationToken ct) => Process(ct);
```

```csharp
// FIX — stop accepting new messages, wait for in-flight to complete, then exit.
finally { await processor.StopProcessingAsync(CancellationToken.None); }
```

## Best Practices

- **Every outbound call gets a timeout.** No exceptions. Pick a value smaller than the caller's timeout.
- **Retries always include backoff + jitter** and live *outside* the timeout.
- **Wrap retries with a circuit breaker** so a sustained outage does not turn into a retry storm.
- **Use one Polly pipeline per dependency**, named, tuned, and observable.
- **Bulkhead by dependency**, not by service — one bad downstream cannot exhaust shared resources.
- **Make writes idempotent** with an explicit key (Stripe, Service Bus message id, request id).
- **Liveness vs readiness are different probes**, and readiness should *not* check every dependency.
- **Drain background workers** on `SIGTERM` — `BackgroundService` + `StopProcessingAsync`.
- **Emit a metric every time a fallback fires.** Silent fallbacks become silent outages.
- **Test failure paths** — Polly's `Simmy` chaos strategies inject latency and faults in staging.
- **Define SLOs (e.g., 99.9% checkout success)** and alert on error budget burn rate, not raw error rate.

## Related Concepts

- [outbox-pattern.md](outbox-pattern.md) — reliable event publishing for at-least-once delivery.
- [saga-pattern.md](saga-pattern.md) — compensation when long-running workflows partially fail.
- [scalability-design.md](scalability-design.md) — capacity headroom is part of reliability.
- [aspnet-core/error-handling-and-problem-details.md](../aspnet-core/error-handling-and-problem-details.md) — surfacing failures to clients correctly.
- [devops/health-checks.md](../devops/health-checks.md) — `AddHealthChecks` deep dive.
- [devops/rollback-strategy.md](../devops/rollback-strategy.md) — recovery from bad deploys.
- [azure/application-insights.md](../azure/application-insights.md) — observability backbone.
- [csharp/async-await.md](../csharp/async-await.md) — `CancellationToken` is non-negotiable for timeouts.

## Real-World Usage

### ASP.NET Core checkout API on Azure App Service

The checkout API calls Stripe, Azure SQL, Azure Service Bus, and a fraud microservice. Each dependency has its own `HttpClient` with its own resilience pipeline. Readiness probes check SQL and Service Bus. Dead-letter queues catch poison messages. Application Insights tracks p95 latency per dependency.

### Background payment worker on AKS

A `BackgroundService` consumes `PaymentRequested` from Service Bus, calls the provider with an idempotency key, writes the result via the outbox pattern, and acknowledges. On `SIGTERM` (rolling restart) it drains in-flight messages before exiting. Failed messages move to a DLQ after 5 attempts; an alert fires when DLQ depth exceeds 10.

### Multi-region failover for a payment provider

Primary calls go to `provider-east`. A health check pings the provider every 30 seconds. When the breaker opens, requests route to `provider-west`. Reconciliation runs hourly against the provider's API to detect captured-but-unrecorded payments.

### Graceful degradation on the storefront

If the recommendations service returns 503, the product page still renders, hides the "you might also like" carousel, and emits `recommendations.degraded=1` to Application Insights. The customer can still buy.

## Code Example — Before and After

### Before — fragile, blocks threads, double-charges on retry

```csharp
public class CheckoutService
{
    private readonly HttpClient _http = new HttpClient();      // shared static, no timeout
    private readonly OrderDbContext _db;

    public CheckoutService(OrderDbContext db) => _db = db;

    public async Task<Guid> CheckoutAsync(CheckoutRequest req)
    {
        // No timeout, no retry, no idempotency key.
        var resp = await _http.PostAsJsonAsync("https://api.stripe.com/v1/charges",
            new { amount = req.AmountCents, source = req.CardToken });
        resp.EnsureSuccessStatusCode();
        var charge = await resp.Content.ReadFromJsonAsync<ChargeResult>();

        var order = new Order(req.CustomerId, req.AmountCents, charge!.Id);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();   // if SQL has a transient blip, the customer is charged but no order exists

        return order.Id;
    }
}
```

What goes wrong in production:
- A 30s Stripe latency spike pins every checkout thread.
- A `429 Too Many Requests` from Stripe is bubbled as a 500 to the customer.
- A transient SQL deadlock loses the order — but the customer was already charged.
- A client double-click creates two charges.

### After — bounded latency, safe retry, atomic outbox, observable

```csharp
public class CheckoutService
{
    private readonly IStripeClient _stripe;            // HttpClient with resilience handler
    private readonly OrderDbContext _db;
    private readonly ILogger<CheckoutService> _log;
    private readonly TimeProvider _clock;

    public CheckoutService(IStripeClient stripe, OrderDbContext db,
                           ILogger<CheckoutService> log, TimeProvider clock)
        => (_stripe, _db, _log, _clock) = (stripe, db, log, clock);

    public async Task<Guid> CheckoutAsync(CheckoutRequest req, CancellationToken ct)
    {
        // 1. Deduplicate at the API boundary using the client-supplied idempotency key.
        var existing = await _db.Orders
            .FirstOrDefaultAsync(o => o.IdempotencyKey == req.IdempotencyKey, ct);
        if (existing is not null)
        {
            _log.LogInformation("Replay detected for {Key}, returning existing order {OrderId}",
                req.IdempotencyKey, existing.Id);
            return existing.Id;
        }

        // 2. Bounded, retried, breaker-protected outbound call. Provider-side idempotent.
        ChargeResult charge;
        try
        {
            charge = await _stripe.ChargeAsync(new ChargeRequest(
                Amount: req.AmountCents,
                Source: req.CardToken,
                IdempotencyKey: req.IdempotencyKey), ct);
        }
        catch (BrokenCircuitException)
        {
            _log.LogWarning("Stripe circuit open for {Key}", req.IdempotencyKey);
            throw new DependencyUnavailableException("payment", retryAfter: TimeSpan.FromSeconds(30));
        }

        // 3. Persist order + outbox event atomically. A background publisher delivers the event.
        var order = new Order(req.CustomerId, req.AmountCents, charge.Id,
                              req.IdempotencyKey, _clock.GetUtcNow());
        _db.Orders.Add(order);
        _db.OutboxMessages.Add(OutboxMessage.From(new OrderPlaced(order.Id, order.CustomerId)));
        await _db.SaveChangesAsync(ct);

        return order.Id;
    }
}
```

What changed:
- `HttpClient` is injected with timeout + retry + breaker.
- The Stripe `Idempotency-Key` makes retries safe.
- The order's own `IdempotencyKey` deduplicates at the API layer.
- The order and the `OrderPlaced` event commit in one transaction (see [outbox-pattern.md](outbox-pattern.md)).
- A `BrokenCircuitException` returns 503 with `Retry-After`, instead of hanging.

## Interview Questions and Answers

### 1. A single slow downstream call took down our whole API. What happened and how do you prevent it?

**Why this matters** — interviewers want to see you understand thread-pool starvation, not just "add a timeout."

**Answer** — Each slow request occupied an ASP.NET Core request thread. Once the thread pool was exhausted, even healthy endpoints returned 503. The fix is layered: enforce a per-attempt timeout on the outbound call (smaller than the upstream timeout), add a circuit breaker so repeated failures fail fast without waiting, and add a concurrency limiter (bulkhead) so this dependency can never consume more than N threads.

**Trade-off** — Tighter timeouts increase false-positive failures during legitimate slow periods. Tune against your dependency's p99 latency, not its mean.

**Real project** — A payment microservice on AKS had no timeout on calls to a 3DS provider. A provider incident pinned 200 pods. Adding a 2-second Polly timeout + breaker dropped checkout error rate from 90% to 4% within minutes during the next incident.

### 2. When should you retry a failed HTTP call and when should you not?

**Why this matters** — wrong retries cause double-charges, retry storms, and customer-visible errors that "look" like the service is broken.

**Answer** — Retry only when (a) the failure is transient (network reset, 5xx, 408, 429), and (b) the operation is idempotent or carries an idempotency key. Never retry 4xx other than 408/429. Respect `Retry-After`. Use exponential backoff with jitter, cap total attempts (3 is typical), and always wrap retries with a circuit breaker. Never retry non-idempotent POSTs without server-side deduplication.

**Trade-off** — Retries hide transient failures from users but amplify load on a struggling dependency. The breaker is what keeps retries safe.

**Real project** — An order export job retried every Service Bus send 5 times. When the namespace was throttled, the retries pushed the throttling from yellow to red and triggered a 4-hour incident. Adding a breaker and reducing to 3 attempts with jitter stopped the amplification.

### 3. What is the difference between liveness and readiness probes, and why does it matter?

**Why this matters** — confusing them causes either flapping restarts or traffic sent to broken pods.

**Answer** — Liveness asks "is the process healthy enough to keep running?" — if it fails, the platform restarts the pod. It should be cheap: usually just "the process responds." Readiness asks "should this pod receive traffic right now?" — if it fails, the load balancer stops routing to it but the pod keeps running. Readiness can check critical dependencies (SQL, Service Bus) so a pod that just started or temporarily lost SQL connectivity stops taking traffic instead of returning errors.

**Trade-off** — A readiness check that hits every dependency creates flapping: one transient SQL blip and the whole fleet leaves rotation simultaneously. Check only what's *essential* for the pod to serve requests, and use circuit-breaker style timeouts inside the check itself.

**Real project** — An API on AKS used the same endpoint for both probes and called 8 downstream services. A 5-second downstream slowdown triggered liveness failures and Kubernetes restarted every pod simultaneously. Splitting the probes and slimming the readiness check eliminated the cascade.

### 4. Walk me through how you would make a payment-capture endpoint safe to retry.

**Why this matters** — payments are the canonical interview test for idempotency.

**Answer** — Three layers. (1) The client sends an `Idempotency-Key` header (or the order id acts as the key). (2) The server stores a `PaymentAttempts` table keyed by that idempotency key; on retry, it returns the stored result without calling the provider again. (3) The outbound call to the provider also passes the same key so the provider deduplicates if the server's record was lost. Each layer covers a different failure: client double-submit, server crash between provider response and DB write, server crash before provider response.

**Trade-off** — Adds a write per attempt and requires storage retention. The cost is trivial compared to the cost of double-charging a customer.

**Real project** — A subscription billing service replayed a Service Bus message after a deploy and double-charged 200 customers. Adding an `IdempotencyKey` unique constraint on `PaymentAttempts` ensured the replay was a no-op.

### 5. How does a circuit breaker actually help — isn't a timeout enough?

**Why this matters** — checks whether you understand systemic effects, not just per-request behavior.

**Answer** — A timeout protects *one* request from hanging. A circuit breaker protects the *system* from repeatedly attempting calls that are almost certainly going to fail. When a dependency is down, even fast-failing timeouts still consume thread-time, connection slots, and downstream load. The breaker trips after a failure threshold and fails fast for a cooldown period, then half-opens to probe recovery. This both protects the upstream caller and gives the downstream service room to recover instead of being hammered.

**Trade-off** — A breaker that's too sensitive trips on benign blips and degrades the user experience unnecessarily. Tune `MinimumThroughput`, `FailureRatio`, `SamplingDuration`, and `BreakDuration` against the dependency's real behavior.

**Real project** — A fraud-screening service had a 10-minute outage. Without a breaker, checkout retried each request 3 times for 10 minutes, generating ~30x normal load on a service that was already down. Adding `AddCircuitBreaker` with a 50% failure ratio over 30 seconds opened the breaker in under a minute and routed orders to "manual review" fallback.

### 6. You see queue depth growing on a Service Bus subscription. Walk me through how to diagnose and respond.

**Why this matters** — tests operational reasoning under live pressure.

**Answer** — First check **consumer health**: are pods running? Are they erroring? Look at `MessageReceived/sec` per consumer vs `MessageProcessed/sec`. Check the **DLQ depth** — if poison messages are accumulating, fix the bug and replay. Check **downstream dependencies** the consumer calls — a slow payment provider can stall consumers even though they look "healthy." Short term: scale out consumers (more pods or higher `MaxConcurrentCalls`) and ensure prefetch is sized for the workload. If the spike is upstream (a marketing event), throttle producers or shed lower-priority work. Long term: tune autoscale rules on queue depth.

**Trade-off** — Scaling consumers aggressively can overwhelm downstream dependencies. Coordinate with downstream capacity.

**Real project** — An order-fulfillment consumer hit a 30s timeout on a shipping API. Each message took 30s to fail. Scaling consumers didn't help — the bottleneck was downstream. Reducing the timeout to 3s, adding a breaker, and DLQ'ing fast cleared a 50k message backlog in 20 minutes.

### 7. How do you decide what to alert on?

**Why this matters** — checks whether you understand SLOs vs symptoms.

**Answer** — Alert on **customer-impacting symptoms** tied to SLOs: checkout success rate, p95 latency on critical endpoints, error rate on payment capture, queue age for time-sensitive workflows. Do not alert on every warning log, every transient 500, or every restart. Use **error-budget burn rate** alerts (Google SRE style): a 2% budget burned in 1 hour is a page; 10% burned in a day is a ticket. Each alert must have a runbook and an owner — if no one knows what to do, delete it.

**Trade-off** — Aggressive alerting creates fatigue and missed real incidents. Loose alerting misses customer impact.

**Real project** — A team had 400 active alerts; on-call ignored most. We deleted 380, defined SLOs for the 5 critical user journeys, and built burn-rate alerts. Page count dropped from 40/week to 3/week, and MTTR improved because every page now meant something.

### 8. How would you design disaster recovery for a payment service?

**Why this matters** — tests whether you understand RTO/RPO, data integrity, and operational realism.

**Answer** — Start from business requirements: **RTO** (how long can we be down?) and **RPO** (how much data can we lose?). For payments, RPO must be near zero — losing capture records means losing money. Use Azure SQL geo-replication (synchronous for primary region pair, asynchronous to secondary region) or Cosmos DB multi-region writes. Replicate Service Bus namespaces and keep idempotency keys consistent. Test restore from backup quarterly — an untested backup is not a backup. Have a runbook for region failover that includes DNS cutover, downstream notification, and reconciliation against the payment provider's records to detect any in-flight gaps.

**Trade-off** — Multi-region active-active is expensive and complex (conflict resolution, latency). Active-passive with documented failover is usually the right starting point.

**Real project** — A payment platform had a 4-hour RPO but no tested restore. During a region incident, the restore failed because of an undocumented schema change. After: weekly automated restore validation in a sandbox subscription and a quarterly full DR drill became mandatory before any release.

## Summary Checklist

- [ ] Every outbound HTTP/SQL/queue call has a bounded timeout smaller than the caller's timeout.
- [ ] Retries use exponential backoff with jitter, capped attempts, and only retry transient failures.
- [ ] Every retried POST/PUT carries an idempotency key, and the server deduplicates on it.
- [ ] Each external dependency is wrapped in its own Polly v8 `ResiliencePipeline` (timeout + retry + breaker + bulkhead).
- [ ] `HttpClient` instances are created via `IHttpClientFactory` with `AddResilienceHandler`.
- [ ] Liveness and readiness probes are separate endpoints; readiness checks only essential dependencies.
- [ ] `BackgroundService` consumers drain in-flight work on `SIGTERM` before exiting.
- [ ] Dead-letter queues exist for every Service Bus subscription, with alerts on depth.
- [ ] Fallback paths emit a metric so silent degradation is visible.
- [ ] SLOs are defined for the top critical user journeys, with burn-rate alerts and runbooks.
- [ ] DR strategy has documented RTO/RPO, geo-replicated data, and tested restore drills.

