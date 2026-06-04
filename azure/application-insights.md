# Application Insights

## What It Is

Application Insights is Azure Monitor's application performance management (APM) service. It collects five telemetry streams from a running .NET application — **requests**, **dependencies**, **exceptions**, **traces (logs)**, and **metrics** — plus **availability tests**, and stores them in a Log Analytics workspace queryable via KQL.

A workspace-based Application Insights resource has a **connection string** that the SDK uses to send telemetry. The SDK is now built on **OpenTelemetry** (`Azure.Monitor.OpenTelemetry.AspNetCore`), making instrumentation vendor-portable.

```csharp
// One line in Program.cs ties an ASP.NET Core app to App Insights
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor(options =>
    {
        options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    });
```

From this point every HTTP request, every SQL call, every HttpClient call, every unhandled exception, and every `ILogger<T>` log line flows to the workspace, correlated by `operation_Id` and `operation_ParentId`.

## Why It Exists

Production .NET systems used to be observable only through:

1. **IIS / Windows Event Logs** — fragmented, server-local, no correlation.
2. **Log files on disk** — invisible until someone RDPs into the server.
3. **SQL Profiler** — only useful for the database, only by hand.
4. **"Bug came back, can you check the logs?"** — hours of grep, no causal chain.

What was missing:

- A single place to ask "what was happening during the 14:32 outage?"
- Correlation across services — when checkout fails, was it the payment API, Service Bus, or the database?
- Dependency timing — which call inside the request is slow?
- Distributed traces across HTTP, queues, and async boundaries.
- Live production debugging without a debugger attached.

Application Insights was Microsoft's answer (since 2014, now OpenTelemetry-based): one SDK, one workspace, correlated telemetry, KQL for analysis, Application Map for topology, Live Metrics for real-time.

## When To Use It

**Use Application Insights for:**

- Any production .NET API, Worker, or Function — request latency, error rate, dependency calls.
- Distributed tracing across microservices linked by W3C Trace Context.
- Centralizing `ILogger<T>` output across many instances and regions.
- Custom business events (`OrderPlaced`, `PaymentRefunded`) for KPI dashboards.
- Live Metrics during deploys to watch for regressions in real time.
- Availability tests (URL ping, multi-step) for synthetic monitoring.
- Application Map to see service topology and failure paths.
- Alert rules on metric thresholds and KQL queries.

**Do not use Application Insights for:**

- High-volume per-request audit trails (use Storage + queryable index) — App Insights sampling and cost models don't fit.
- Long-term security audit logs — use Microsoft Sentinel or Storage with immutability.
- Customer-facing analytics — use a product analytics tool. App Insights is for **engineers diagnosing engineering problems**.
- Logs with PII or PCI cardholder data — never log credit card numbers, passwords, or SSNs.

## Why It Is Important

Application Insights is the difference between "something is wrong" and "the dependency to Stripe is timing out at 14:31:42 on instance westeurope-5, originating from order 4b7a...". In production:

- **Mean time to detect (MTTD)** drops from hours (customer complaints) to seconds (alerts).
- **Mean time to resolve (MTTR)** drops from days (log spelunking) to minutes (Application Map → failing dependency → recent deploy → rollback).
- **Capacity planning** has data: P95 latency under load, dependency throughput, autoscale signals.
- **SLO tracking** becomes measurable: "99.9% of /api/checkout completes under 800ms over 28 days" is a KQL query.
- **Cross-service debugging** works because OpenTelemetry's W3C Trace Context flows through HttpClient, Service Bus, and SQL automatically.

Interviewers ask about App Insights to test whether you treat observability as engineering work or as an afterthought.

## How It's Used in C# / .NET

NuGet packages:

- `Azure.Monitor.OpenTelemetry.AspNetCore` — modern OpenTelemetry-based SDK (preferred).
- `Azure.Monitor.OpenTelemetry.Exporter` — exporter for non-ASP.NET workloads (Worker Services).
- `Microsoft.ApplicationInsights.AspNetCore` — legacy classic SDK (still supported, but new code should use OTel).
- `Microsoft.ApplicationInsights.WorkerService` — legacy for Worker Services.
- `Microsoft.Extensions.Logging.ApplicationInsights` — `ILogger<T>` provider.

### Setup with OpenTelemetry (recommended)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .UseAzureMonitor(options =>
    {
        // Connection string preferred over deprecated instrumentation key
        options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
        options.SamplingRatio = 1.0f;   // 100% in dev; tune in prod
    })
    .WithTracing(t => t.AddSource("Contoso.Orders"))          // custom ActivitySource
    .WithMetrics(m => m.AddMeter("Contoso.Orders"));          // custom Meter

builder.Logging.AddOpenTelemetry(o =>
{
    o.IncludeFormattedMessage = true;
    o.IncludeScopes = true;
});
```

### Connection string vs. instrumentation key

The **instrumentation key** (GUID only) is **deprecated**. Use the **connection string**, which includes the ingestion endpoint per region and supports sovereign clouds and Private Link. Format:

```
InstrumentationKey=00000000-0000-0000-0000-000000000000;IngestionEndpoint=https://westeurope-5.in.applicationinsights.azure.com/;LiveEndpoint=https://westeurope.livediagnostics.monitor.azure.com/
```

Store it in Key Vault or App Service settings; never commit to Git.

### Custom telemetry — events, metrics, traces

```csharp
public sealed class CheckoutService
{
    private static readonly ActivitySource Activity = new("Contoso.Orders");
    private static readonly Meter Meter = new("Contoso.Orders");
    private static readonly Counter<long> OrdersPlaced =
        Meter.CreateCounter<long>("orders.placed", unit: "{order}");

    private readonly ILogger<CheckoutService> _log;

    public CheckoutService(ILogger<CheckoutService> log) => _log = log;

    public async Task<OrderResult> PlaceAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        using var span = Activity.StartActivity("PlaceOrder", ActivityKind.Internal);
        span?.SetTag("order.customer_id", cmd.CustomerId);
        span?.SetTag("order.amount", cmd.Total);

        _log.LogInformation("Placing order for customer {CustomerId} amount {Amount}",
            cmd.CustomerId, cmd.Total);

        // ... domain work ...

        OrdersPlaced.Add(1, new KeyValuePair<string, object?>("region", "weu"));
        return new OrderResult(/* ... */);
    }
}
```

### TelemetryInitializer for cross-cutting context

Add tenant ID, build version, or feature-flag state to every telemetry item:

```csharp
public sealed class TenantTelemetryInitializer(IHttpContextAccessor accessor) : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        var tenantId = accessor.HttpContext?.User.FindFirst("tid")?.Value;
        if (!string.IsNullOrEmpty(tenantId))
            telemetry.Context.GlobalProperties["TenantId"] = tenantId;

        telemetry.Context.GlobalProperties["BuildVersion"] =
            Assembly.GetEntryAssembly()?.GetName().Version?.ToString() ?? "unknown";
    }
}

builder.Services.AddSingleton<ITelemetryInitializer, TenantTelemetryInitializer>();
```

(For OpenTelemetry, use a `BaseProcessor<Activity>` or `Resource` attributes; for the classic SDK, `ITelemetryInitializer` is the hook.)

### Sampling

| Mode | When to use | Cost impact |
|---|---|---|
| **No sampling (1.0)** | Dev, low-volume APIs (< 1 req/sec sustained) | Highest |
| **Fixed-rate (e.g., 0.25)** | Steady, predictable load | Predictable reduction |
| **Adaptive sampling (default)** | Most production workloads | SDK targets a per-instance items/sec budget; bursts are sampled down automatically |
| **Ingestion sampling** | When you cannot change the app | Server-side, less efficient, after billing |

Always preserve **exceptions** and **failed requests** — adaptive sampling does this by default.

### KQL — querying telemetry

```kql
// P95 checkout latency over the last hour by region
requests
| where timestamp > ago(1h) and name == "POST /api/checkout"
| summarize p95 = percentile(duration, 95) by bin(timestamp, 5m), tostring(customDimensions.region)
| render timechart

// Failed dependencies broken down by target
dependencies
| where timestamp > ago(1h) and success == false
| summarize count() by target, resultCode
| order by count_ desc

// Trace a single operation across services
union requests, dependencies, traces, exceptions
| where operation_Id == "4b7a8c2e1d9f4a8b9c0d1e2f3a4b5c6d"
| order by timestamp asc
```

## Advantages

- **Out-of-the-box correlation** — W3C Trace Context propagates across HttpClient, Service Bus, EF Core, and HTTP servers.
- **Application Map** — visual topology of dependencies and health.
- **Live Metrics** — sub-second metrics and sample telemetry without sampling, useful during deploys.
- **KQL** is a powerful, well-documented query language.
- **Smart Detection** — automatic anomaly alerts on response time, failure rate, exceptions.
- **OpenTelemetry-based** — vendor portability; the same instrumentation works with Jaeger, Datadog, etc.
- **Adaptive sampling** automatically protects ingestion cost during bursts.
- **Workspace-based** — share a Log Analytics workspace with infrastructure logs for unified queries.

## Disadvantages

- **Cost grows with volume** — chatty logs and unsampled dependencies can produce a six-figure annual bill.
- **Sampling can hide low-frequency bugs** — a failure that hits 1 user in 10,000 may be dropped.
- **Ingestion latency** — telemetry appears in queries 2–5 minutes after emission; not real-time for forensics (use Live Metrics for that).
- **PII leakage risk** — logging raw request bodies or user emails creates compliance liabilities.
- **Classic SDK vs OpenTelemetry confusion** — two SDKs coexist; mixing them double-counts telemetry.
- **Limited retention by default** — 90 days standard; longer requires Log Analytics archive tier.
- **Custom metrics dimensions** — high cardinality (e.g., per-customer-ID tag) explodes cost and breaks dashboards.

## Common Mistakes

### 1. Using the deprecated instrumentation key

```csharp
// ❌ Instrumentation key alone; deprecated, breaks in sovereign clouds
options.InstrumentationKey = "00000000-0000-0000-0000-000000000000";
```

**Fix:** Use the full connection string:

```csharp
// ✅
options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
```

### 2. Logging PII or secrets

```csharp
// ❌ Card number and password in App Insights, visible to anyone with reader role
_log.LogInformation("Charging {Card} for {Amount} with creds {Password}",
    request.CardNumber, request.Amount, request.Password);
```

**Fix:** Never log secrets or PII. Use a `TelemetryProcessor` (classic SDK) or `BaseProcessor` (OTel) to scrub if you cannot trust callers:

```csharp
// ✅
_log.LogInformation("Charging customer {CustomerId} amount {Amount}",
    request.CustomerId, request.Amount);
```

### 3. Sampling at 100% in high-volume production

```csharp
// ❌ A high-traffic API will burn through the workspace cap and lose all telemetry mid-month
options.SamplingRatio = 1.0f;
```

**Fix:** Let adaptive sampling run (the default), or set a fixed rate appropriate to volume:

```csharp
// ✅ Adaptive sampling preserves exceptions and failed requests automatically
// Leave SamplingRatio unset for adaptive behavior
```

### 4. High-cardinality custom dimensions

```csharp
// ❌ One series per customer ID = millions of series; query times explode
OrdersPlaced.Add(1, new KeyValuePair<string, object?>("customer_id", cmd.CustomerId));
```

**Fix:** Keep metric dimensions bounded (region, tier, status). Use the **log** side for per-customer detail:

```csharp
// ✅ Metric is low-cardinality
OrdersPlaced.Add(1, new KeyValuePair<string, object?>("tier", customer.Tier));
// Per-customer detail goes to logs
_log.LogInformation("Order placed for customer {CustomerId}", cmd.CustomerId);
```

### 5. Mixing the classic SDK with OpenTelemetry

```csharp
// ❌ Both packages registered — every request gets counted twice
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddOpenTelemetry().UseAzureMonitor(...);
```

**Fix:** Pick one. For new code, OpenTelemetry. For legacy apps, finish migration before adding new instrumentation.

### 6. No correlation across queue boundaries

```csharp
// ❌ Sending a Service Bus message without propagating the Activity context
await _sender.SendMessageAsync(new ServiceBusMessage(payload), ct);
```

**Fix:** The modern `Azure.Messaging.ServiceBus` SDK propagates `traceparent` automatically when an Activity is active. Ensure the publish runs inside an Activity:

```csharp
// ✅
using var span = Activity.StartActivity("Publish OrderPlaced", ActivityKind.Producer);
await _sender.SendMessageAsync(new ServiceBusMessage(payload), ct);
// Consumer span on the other side will be linked via traceparent
```

### 7. No availability tests for critical endpoints

A silent outage where the API returns 500 to every request but no one notices because there's no synthetic monitor.

**Fix:** Configure a standard availability test (URL ping or custom TrackAvailability) for each critical endpoint with multi-region origins and an alert.

## Best Practices

- **Use the OpenTelemetry SDK** for new applications.
- **Connection string, not instrumentation key**.
- **Store the connection string in Key Vault or App Service settings**, never in Git.
- **Adaptive sampling** by default; review per-instance items/sec cap monthly.
- **Always preserve exceptions and failed requests** — these are the events you cannot afford to lose.
- **Add a TelemetryInitializer / BaseProcessor** that attaches tenant, build version, region, and request-id to every telemetry item.
- **Use `ActivitySource` and `Meter`** for custom traces and metrics — these are OpenTelemetry-native and vendor-portable.
- **Bounded cardinality on metric dimensions** (< 50 distinct values per tag).
- **Strip PII** at the SDK boundary, not at the dashboard.
- **Configure alert rules** on failure rate, P95 latency, dependency failures, and KQL business signals (e.g., `OrderPlaced` rate dropping > 30%).
- **Use Application Map** in incident reviews — it visualizes the failure topology.
- **Set workspace daily cap and alerts** before someone deploys a `Debug` log level to production.
- **Test Live Metrics** during every deploy — it catches regressions before users do.

## Related Concepts

- [aspnet-core/logging-and-monitoring.md](aspnet-core/logging-and-monitoring.md) — `ILogger<T>`, log enrichment, structured logging.
- [aspnet-core/error-handling-and-problem-details.md](aspnet-core/error-handling-and-problem-details.md) — exception propagation to telemetry.
- [azure/azure-service-bus.md](azure/azure-service-bus.md) — cross-queue distributed tracing.
- [azure/deployment-to-azure.md](azure/deployment-to-azure.md) — Live Metrics during deploys.
- [devops/health-checks.md](devops/health-checks.md) — health endpoints feed availability tests.
- [architecture/reliability-design.md](architecture/reliability-design.md) — SLOs and alerting strategy.

## Real-World Usage

### Payment APIs

A payments microservice instruments every charge with an `ActivitySource` span tagged with `payment_provider`, `amount_bucket`, and `result`. Dashboards show provider-by-provider success rate; an alert fires when Stripe success rate drops below 99.5% over 5 minutes, distinct from a generic API failure-rate alert.

### Order processing across services

`POST /api/checkout` → publishes `OrderPlaced` to Service Bus → consumed by Billing, Inventory, and Notification workers. All four services share a single `operation_Id`. When the Notification worker fails, the Application Map shows the broken edge and a single KQL query on the operation ID returns the full causal chain across four services.

### Multi-region failover

App Insights resources in West Europe and North Europe feed a shared Log Analytics workspace. Geographic dashboards show per-region P95. Front Door routes traffic away from a degraded region; the Live Metrics chart confirms the failover within 30 seconds.

### Managed Identity for telemetry

Production App Services use **Entra-only ingestion** with Managed Identity to send telemetry — no connection string secret to leak. Configured via the `AAD` ingestion option in the OpenTelemetry distro.

### KPI dashboards

Custom metrics `orders.placed`, `payments.captured`, `refunds.issued` populate a Power BI dashboard refreshed from a Log Analytics query. The product team gets near-real-time business KPIs from the same pipeline that engineers use for SLOs.

### CI/CD validation

A GitHub Actions release job runs a 60-second Live Metrics validation after each production deploy: if failure rate exceeds 1% or P95 latency exceeds 2× baseline, the job auto-rolls back via slot swap.

## Code Example — Before and After

### Before — classic SDK, secrets in config, no correlation, PII in logs

```csharp
// appsettings.Production.json (committed)
{
  "ApplicationInsights": {
    "InstrumentationKey": "00000000-0000-0000-0000-000000000000"   // ❌ deprecated, in Git
  }
}

// Program.cs
builder.Services.AddApplicationInsightsTelemetry();   // classic SDK

// CheckoutController.cs
[HttpPost]
public async Task<IActionResult> Checkout(CheckoutRequest req)
{
    _log.LogInformation("Checkout request: {@Request}", req);   // ❌ logs full body incl. card
    try
    {
        await _stripe.ChargeAsync(req.CardNumber, req.Amount);
        await _bus.SendAsync(new OrderPlaced(req.OrderId));     // ❌ no Activity, no correlation downstream
        return Ok();
    }
    catch (Exception ex)
    {
        _log.LogError(ex, "Checkout failed");                    // ❌ no context, just the exception
        return StatusCode(500);
    }
}
```

**Problems:**
- Instrumentation key deprecated and in Git.
- Card number logged — PCI violation.
- Service Bus publish not correlated with the request.
- Generic error log gives no operational context.

### After — OpenTelemetry, connection string from Key Vault, correlation, PII-safe

```csharp
// appsettings.json (no secret)
{
  "ApplicationInsights": { "ConnectionString": "" }   // populated from Key Vault
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Connection string sourced from Key Vault by the configuration provider
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor(opts =>
    {
        opts.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    })
    .WithTracing(t => t.AddSource("Contoso.Checkout"))
    .WithMetrics(m => m.AddMeter("Contoso.Checkout"));

builder.Services.AddSingleton<ITelemetryInitializer, TenantTelemetryInitializer>();
builder.Services.AddHttpContextAccessor();

// CheckoutController.cs
public sealed class CheckoutController : ControllerBase
{
    private static readonly ActivitySource Activity = new("Contoso.Checkout");
    private static readonly Meter Meter = new("Contoso.Checkout");
    private static readonly Counter<long> CheckoutSucceeded =
        Meter.CreateCounter<long>("checkout.succeeded");
    private static readonly Counter<long> CheckoutFailed =
        Meter.CreateCounter<long>("checkout.failed");

    private readonly IStripeClient _stripe;
    private readonly ServiceBusSender _bus;
    private readonly ILogger<CheckoutController> _log;

    public CheckoutController(IStripeClient stripe, ServiceBusSender bus, ILogger<CheckoutController> log)
    {
        _stripe = stripe;
        _bus = bus;
        _log = log;
    }

    [HttpPost]
    public async Task<IActionResult> Checkout(CheckoutRequest req, CancellationToken ct)
    {
        using var span = Activity.StartActivity("Checkout", ActivityKind.Server);
        span?.SetTag("order.id", req.OrderId);
        span?.SetTag("customer.id", req.CustomerId);
        span?.SetTag("amount", req.Amount);

        // ✅ Log identifiers, never raw PII
        _log.LogInformation("Checkout starting for order {OrderId} customer {CustomerId}",
            req.OrderId, req.CustomerId);

        try
        {
            // ✅ Stripe call becomes a child dependency, automatically correlated
            await _stripe.ChargeAsync(req.PaymentMethodToken, req.Amount, ct);

            // ✅ Service Bus publish propagates traceparent to downstream workers
            await _bus.SendMessageAsync(new ServiceBusMessage(
                BinaryData.FromObjectAsJson(new OrderPlaced(req.OrderId, req.CustomerId, req.Amount)))
            {
                MessageId = req.OrderId.ToString(),
                Subject = nameof(OrderPlaced)
            }, ct);

            CheckoutSucceeded.Add(1);
            span?.SetStatus(ActivityStatusCode.Ok);
            return Ok();
        }
        catch (StripeDeclineException ex)
        {
            CheckoutFailed.Add(1, new KeyValuePair<string, object?>("reason", "declined"));
            span?.SetStatus(ActivityStatusCode.Error, ex.DeclineCode);
            _log.LogWarning(ex, "Checkout declined for order {OrderId} code {Code}",
                req.OrderId, ex.DeclineCode);
            return BadRequest(new ProblemDetails { Title = "Payment declined", Detail = ex.DeclineCode });
        }
        catch (Exception ex)
        {
            CheckoutFailed.Add(1, new KeyValuePair<string, object?>("reason", "error"));
            span?.SetStatus(ActivityStatusCode.Error, ex.Message);
            _log.LogError(ex, "Checkout failed for order {OrderId}", req.OrderId);
            throw;   // global handler converts to ProblemDetails
        }
    }
}
```

**Why this is better:**
- Connection string in Key Vault, not Git.
- OpenTelemetry SDK, vendor-portable.
- Custom span with rich tags for slicing in KQL.
- No PII logged.
- Service Bus correlation works automatically — Billing/Inventory workers join the same trace.
- Domain-specific metrics (`checkout.succeeded` / `checkout.failed` with `reason`) drive business dashboards and alerts.

## Interview Questions and Answers

### 1. P95 latency on `POST /api/checkout` doubled after this morning's deploy. Walk me through the diagnosis.

**Why this matters:** Tests practical use of App Insights for incident response.

**Answer:** Open the Application Map first to see if the slowdown is in the API itself or a downstream dependency — usually a colored edge highlights the change. Then in KQL, compare P95 by dependency over the last hour vs. the prior day baseline. If Stripe shifted from 200ms to 800ms, the issue is external; correlate with Stripe's status page. If the slowdown is in our SQL `EXEC sp_GetCart`, check Live Metrics for an active long-running query, and the most recent EF Core migration for an index change. End-to-end I'd grab one slow `operation_Id` and run a union query across requests, dependencies, and exceptions to see the full causal chain.

**Trade-off:** App Insights ingestion has a 2–5 minute lag; for the first 5 minutes after deploy, Live Metrics is the right tool.

**Real project:** On a payments service a deploy doubled P95 because a new dependency added 300ms HMAC verification per request. Application Map showed the new dependency immediately, and we rolled back via slot swap inside 8 minutes.

### 2. Why connection string instead of instrumentation key?

**Why this matters:** Currency of knowledge — instrumentation key has been deprecated since 2020.

**Answer:** The instrumentation key alone uses a default global ingestion endpoint that doesn't exist in sovereign clouds (US Government, China). It also doesn't support Private Link. The connection string contains the ingestion endpoint, live diagnostics endpoint, and (optionally) the AAD audience for Entra-only ingestion. New SDKs require it; the GUID-only path will eventually stop working.

### 3. How does cross-service correlation work?

**Why this matters:** Distributed tracing literacy.

**Answer:** OpenTelemetry propagates the **W3C Trace Context** header (`traceparent`) on every outbound HTTP call. The receiving server reads it and starts a new span linked to the parent. The Azure SDKs do the same for Service Bus, Event Hubs, Storage, and Cosmos — they attach the trace context as a message property or header. As long as every hop has the OpenTelemetry SDK and an active Activity, all spans share an `operation_Id`. A single KQL query on that ID returns the request, all dependencies, all logs, and all exceptions in causal order.

**Trade-off:** Manual `HttpRequestMessage` construction outside the standard `HttpClient` pipeline breaks propagation. Always use `HttpClient` with `IHttpClientFactory`.

### 4. We're paying $40K/month for App Insights. How do you cut cost without losing visibility?

**Why this matters:** Cost optimization is a senior skill.

**Answer:** Four levers, in order. (1) **Sampling** — switch to adaptive at, say, 5 items/sec/instance; this often cuts ingestion by 70% while preserving all exceptions and failures. (2) **Reduce log verbosity** — `Information` for business events, `Warning` and above for noise like EF Core SQL. (3) **Strip noisy dependencies** — health check pings, Application Insights' own self-telemetry, internal heartbeats. Use a TelemetryProcessor to drop them. (4) **Workspace daily cap with alert** — set a hard ceiling at 90% of budget so a runaway log doesn't burn the month's budget overnight. Then revisit retention — most teams don't need the default 90 days for everything.

**Trade-off:** Aggressive sampling can hide bugs that hit < 1% of requests. Always preserve exceptions and failed requests.

### 5. When would you use `IOptionsMonitor<T>` for App Insights configuration?

**Why this matters:** Cross-topic understanding.

**Answer:** Rarely directly, because the connection string shouldn't change in a running process. But for **sampling rate** or **enabled/disabled telemetry channels**, an `IOptionsMonitor<T>` driven by App Configuration lets ops dial sampling up during an incident without redeploying. The TelemetryProcessor consults the monitor's `CurrentValue` per item.

### 6. A teammate proposes logging the full HTTP request and response body for every API call. Push back.

**Why this matters:** Security and cost awareness.

**Answer:** No, for three reasons. (1) **Security/compliance** — bodies routinely contain PII, payment tokens, auth headers. Storing them in App Insights creates breach exposure and GDPR/PCI violations. (2) **Cost** — bodies dominate ingestion volume; a 2KB request becomes 4–8KB of telemetry, multiplied by traffic. (3) **Diagnostic value** — bodies are rarely the missing context; structured logs of identifiers and outcomes are. If a specific endpoint genuinely needs body capture for an investigation, do it on-demand via a feature flag, scrub sensitive fields, and emit to a separate workspace with restricted access.

### 7. How do you SLO `99.9% of /api/checkout completes successfully under 800ms over a rolling 28 days`?

**Why this matters:** Reliability engineering literacy.

**Answer:** Two ingredients. **Definition** as a KQL query: total successful requests under 800ms divided by total requests over 28 days. **Alerting** at three thresholds: budget-burn alert (fast — when burn rate would exhaust the monthly budget in 1 hour), warning (medium — 6-hour exhaustion), and informational (slow — 24-hour). Publish the SLO and remaining error budget on a dashboard. When budget runs low, freeze risky deploys until it recovers — an organizational lever, not just a chart.

### 8. The Live Metrics blade shows our app, but Application Map is empty. What's wrong?

**Why this matters:** Practical debugging.

**Answer:** Live Metrics works off the live channel; Application Map builds from the last ~15 minutes of ingested dependencies and requests. Common causes: (1) telemetry is being sampled to zero (rare); (2) the app is brand new and hasn't received traffic yet; (3) outbound dependencies aren't being captured because the app uses raw `HttpWebRequest` instead of `HttpClient`; (4) the wrong workspace is selected in the portal. Check the dependencies table directly: if rows exist, the Map will populate within a few minutes.

## Summary Checklist

- [ ] I configure App Insights with the OpenTelemetry SDK and a connection string.
- [ ] I source the connection string from Key Vault, never from Git.
- [ ] I emit custom traces and metrics via `ActivitySource` and `Meter`.
- [ ] I add an `ITelemetryInitializer` or `BaseProcessor` for tenant/build context.
- [ ] I run KQL queries to slice request latency, dependency failures, and business metrics.
- [ ] I use Application Map for topology and Live Metrics for real-time deploy validation.
- [ ] I configure alert rules on failure rate, P95 latency, and KQL business signals.
- [ ] I keep metric cardinality bounded and strip PII before ingestion.
- [ ] I rely on adaptive sampling to control cost while preserving exceptions and failures.
- [ ] I track SLOs and error budgets from telemetry, not from gut feel.
