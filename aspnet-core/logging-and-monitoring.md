# Logging and Monitoring

## What It Is

Logging is the act of emitting structured, queryable records of what your code did. Monitoring is the act of collecting metrics, traces, and alerts from those records — and from runtime probes — so you can answer "is the system healthy right now, and if not, why?" without attaching a debugger.

In .NET, logging is centered on `ILogger<T>` (Microsoft.Extensions.Logging), structured providers like Serilog, and exporters like Application Insights and OpenTelemetry that ship logs/metrics/traces to backends such as App Insights, Datadog, Seq, Grafana Loki, or Azure Monitor.

```csharp
// Bad: a string blob — unsearchable, unparseable.
_logger.LogInformation($"Captured payment {invoiceId} amount {amount}");

// Good: structured — InvoiceId and Amount become queryable fields.
_logger.LogInformation("Captured payment {InvoiceId} amount {Amount}", invoiceId, amount);
```

## Why It Exists

Before structured logging and centralized observability, on-call engineers SSH'd into machines and grepped text files. Distributed systems made that impossible — a single payment touches the API, Service Bus, a worker, SQL, and Stripe across five hosts. The CNCF observability stack (OpenTelemetry, W3C TraceContext) and Microsoft's `ILogger`/`ActivitySource` APIs exist so a correlation/trace ID flows automatically through every hop, and one query in App Insights or Datadog shows the full story.

## When To Use It

**Log and instrument:**
- Every request entering or leaving a service boundary (HTTP, message, scheduled job).
- Every failure with enough context to reproduce it (IDs, inputs, downstream status).
- Every state transition that matters for audit (payment captured, order shipped, user role changed).
- Latency-sensitive paths — emit a `Stopwatch` or `Activity` duration.

**Do not log:**
- Secrets, tokens, passwords, full PAN, full JWTs.
- High-volume tight loops at `Information` level — use `Debug` or sampling.
- Inside hot paths without checking `IsEnabled` or using `LoggerMessage` source generator.

## Why It Is Important

Production incidents are won or lost on observability:

- **MTTR (mean time to recovery)** drops from hours to minutes when one trace ID stitches API → worker → DB → Stripe.
- **Alerting on SLOs** (error rate, P99 latency) requires metrics — `_logger.LogError` alone does not page you.
- **Audit and compliance** (PCI, SOC 2, GDPR) require immutable, queryable logs of state changes.
- **Cost** — over-logging at `Information` in App Insights costs thousands per month; under-logging costs an incident.
- **Cold-path debugging** — for bugs that occur once a week in production, structured fields are the only artifact.

## How It's Used in C# / .NET

The standard stack:

| Concern | API / NuGet |
| --- | --- |
| Logging abstraction | `ILogger<T>` from `Microsoft.Extensions.Logging` |
| Structured backend | `Serilog.AspNetCore` + `Serilog.Sinks.Seq` / `.ApplicationInsights` |
| High-performance logging | `LoggerMessage` source generator (`[LoggerMessage]` attribute, .NET 6+) |
| Metrics | `System.Diagnostics.Metrics` (`Meter`, `Counter<T>`, `Histogram<T>`) |
| Tracing | `System.Diagnostics.ActivitySource` + OpenTelemetry SDK |
| Azure | `Microsoft.ApplicationInsights.AspNetCore`, `Azure.Monitor.OpenTelemetry.AspNetCore` |
| Correlation | W3C TraceContext (`traceparent` header) — built into `HttpClient` and ASP.NET Core |
| Health probes | `Microsoft.Extensions.Diagnostics.HealthChecks` |

### Log Levels

| Level | Use For | Example |
| --- | --- | --- |
| `Trace` | Method entry/exit, only in dev | "Entering CaptureAsync" |
| `Debug` | Diagnostic detail, off in prod by default | "Cache miss for key {Key}" |
| `Information` | State transitions, business events | "Order {OrderId} placed" |
| `Warning` | Recoverable failure, degraded path | "Stripe retried 2 times" |
| `Error` | Operation failed, request impacted | "Payment capture failed for {InvoiceId}" |
| `Critical` | System unable to continue | "Database unreachable" |

### Structured Logging with Serilog

```csharp
// Program.cs
builder.Host.UseSerilog((ctx, sp, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Service", "payments-api")
    .Enrich.WithProperty("Environment", ctx.HostingEnvironment.EnvironmentName)
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .WriteTo.ApplicationInsights(sp.GetRequiredService<TelemetryConfiguration>(), TelemetryConverter.Traces));
```

### LoggerMessage Source Generator (hot paths)

```csharp
public static partial class PaymentLog
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Captured payment {InvoiceId} amount {Amount:F2} {Currency}")]
    public static partial void PaymentCaptured(
        ILogger logger, Guid invoiceId, decimal amount, string currency);
}

// Usage — zero allocations for the format string
PaymentLog.PaymentCaptured(_logger, invoiceId, amount, "USD");
```

### Scopes for Per-Request Context

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["OrderId"] = orderId,
    ["CustomerId"] = customerId,
    ["CorrelationId"] = HttpContext.TraceIdentifier
}))
{
    _logger.LogInformation("Processing order");          // OrderId, CustomerId, CorrelationId attached
    await _gateway.CaptureAsync(...);                    // any logs inside inherit the scope
}
```

### Application Insights + OpenTelemetry

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("payments-api"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddSource("Contoso.Payments")
        .AddAzureMonitorTraceExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("Contoso.Payments")
        .AddAzureMonitorMetricExporter());
```

### Custom Metric

```csharp
private static readonly Meter Meter = new("Contoso.Payments");
private static readonly Counter<long> Captures = Meter.CreateCounter<long>("payments.captures.total");
private static readonly Histogram<double> CaptureDuration = Meter.CreateHistogram<double>("payments.capture.duration_ms");

Captures.Add(1, new KeyValuePair<string, object?>("currency", "USD"));
CaptureDuration.Record(stopwatch.Elapsed.TotalMilliseconds);
```

### Distributed Tracing — Correlation IDs

ASP.NET Core automatically reads/writes the W3C `traceparent` header. Outgoing `HttpClient` calls propagate it. App Insights stitches the whole trace into one Gantt chart with no extra code.

```csharp
using var activity = ActivitySource.StartActivity("Payment.Capture");
activity?.SetTag("invoice.id", invoiceId);
activity?.SetTag("amount", amount);
// ... work ...
activity?.SetStatus(ActivityStatusCode.Ok);
```

## Advantages

- **One query answers "what happened to order X"** across all services.
- **Source-generator logging** has near-zero overhead vs string interpolation.
- **OpenTelemetry is vendor-neutral** — swap App Insights for Datadog by changing the exporter.
- **Structured fields enable alerting** — "alert when `payment.capture.failed` > 1% per minute."
- **Audit trail is a side effect** of good event logging.

## Disadvantages

- **Cost** — App Insights and Datadog ingest fees scale with log volume; can run into thousands/month at scale.
- **PII risk** — careless logs leak emails, JWTs, card numbers; one Splunk breach can cost millions.
- **Cardinality explosions** — adding `user.id` as a metric dimension blows up the time-series DB.
- **Noise vs signal** — over-logging trains the team to ignore alerts.
- **Local-dev friction** — running OpenTelemetry collectors locally is non-trivial.

## Common Mistakes

### 1. String interpolation instead of structured templates

```csharp
// Bad — InvoiceId is baked into the message string; cannot query by it
_logger.LogInformation($"Captured payment {invoiceId}");
```

```csharp
// Fix — Serilog/App Insights store InvoiceId as a typed field
_logger.LogInformation("Captured payment {InvoiceId}", invoiceId);
```

### 2. Logging secrets and PII

```csharp
// Bad — JWT and card number land in App Insights forever
_logger.LogInformation("Request {Headers} body {Body}",
    Request.Headers, JsonSerializer.Serialize(request));
```

```csharp
// Fix — log identifiers, not payloads
_logger.LogInformation("Request {CorrelationId} for invoice {InvoiceId}",
    HttpContext.TraceIdentifier, request.InvoiceId);
```

### 3. Catch and swallow exceptions

```csharp
// Bad — the error vanishes; the support ticket is unsolvable
try { await _gateway.CaptureAsync(...); }
catch { return BadRequest(); }
```

```csharp
// Fix — log with exception, rethrow or return a meaningful response
catch (Exception ex)
{
    _logger.LogError(ex, "Capture failed for {InvoiceId}", invoiceId);
    throw;
}
```

### 4. Using Console.WriteLine in production code

```csharp
// Bad — no level, no structure, no correlation, not shipped to App Insights
Console.WriteLine("Order created");
```

```csharp
// Fix — always go through ILogger<T>
_logger.LogInformation("Order {OrderId} created", orderId);
```

### 5. Logging inside a hot loop at Information level

```csharp
// Bad — 50k log entries per request, App Insights bill triples
foreach (var line in order.Lines)
    _logger.LogInformation("Processing line {Sku}", line.Sku);
```

```csharp
// Fix — aggregate or downgrade to Debug
_logger.LogInformation("Processing {LineCount} lines for {OrderId}", order.Lines.Count, order.Id);
```

### 6. No correlation ID across service hops

```csharp
// Bad — worker logs cannot be tied back to the originating API request
await _bus.PublishAsync(new OrderPlaced(orderId));
```

```csharp
// Fix — flow the trace ID; W3C TraceContext does this automatically
// when using HttpClient + Azure Service Bus SDK ≥ 7.x. Verify with:
activity?.Id  // should be inherited by the consumer
```

## Best Practices

- Use `ILogger<T>` everywhere; never `Console.WriteLine`.
- Use **structured templates** (`{InvoiceId}`) not interpolation (`$"{invoiceId}"`).
- Use the **`LoggerMessage` source generator** for hot paths (.NET 6+).
- Set **log levels per category** in `appsettings.json` so you can dial verbosity without redeploying.
- Adopt **W3C TraceContext** end-to-end; verify `traceparent` flows API → bus → worker.
- Emit **business metrics** (`payments.captured.total`) alongside infra metrics (CPU, GC).
- Define **SLOs** (e.g., "99.9% of captures < 500ms") and alert on burn rate, not raw errors.
- Use **health checks** (`/healthz/live`, `/healthz/ready`) for Kubernetes/App Service probes — see [devops/health-checks.md](../devops/health-checks.md).
- **Redact PII** at the sink (Serilog destructuring policies, App Insights TelemetryProcessor).
- **Sample** high-volume traces (10–25%) once you exceed budget; never sample errors out.

## Related Concepts

- [aspnet-core/error-handling-and-problem-details.md](error-handling-and-problem-details.md)
- [aspnet-core/request-pipeline.md](request-pipeline.md)
- [azure/application-insights.md](../azure/application-insights.md)
- [devops/health-checks.md](../devops/health-checks.md)
- [architecture/reliability-design.md](../architecture/reliability-design.md)

## Real-World Usage

### ASP.NET Core APIs

Serilog request logging middleware (`app.UseSerilogRequestLogging()`) emits one structured record per HTTP request with method, path, status, elapsed ms — the foundation of every dashboard.

### Azure Functions

Functions runtime emits `ILogger` to Application Insights by default. Set `samplingSettings` in `host.json` to control cost — Functions can produce millions of logs/day.

### Azure Service Bus Workers

Worker services should `BeginScope` with the `MessageId` and `traceparent` from the message envelope, ensuring all subsequent logs tie back to the originating request.

### Azure Application Insights

`AddApplicationInsightsTelemetry()` wires up dependency tracking (SQL, HTTP, Service Bus), exception capture, and live metrics. Use **Workbooks** for ops dashboards, **Smart Detection** for anomaly alerts.

### OpenTelemetry + Datadog/Grafana

If you are not all-in on Azure Monitor, OpenTelemetry's vendor-neutral exporters ship to Datadog, Honeycomb, Grafana Tempo/Loki, or Jaeger — same instrumentation code, swappable backends.

### Multi-Tenant SaaS

Enrich every log with `TenantId` and `Region`. Build per-tenant error-rate dashboards so noisy tenants do not mask a real outage.

## Code Example — Before and After

### Before — unstructured, untraceable

```csharp
public sealed class PaymentService
{
    private readonly IPaymentGateway _gateway;

    public PaymentService(IPaymentGateway gateway) => _gateway = gateway;

    public async Task<PaymentResult> CaptureAsync(Guid invoiceId, decimal amount)
    {
        Console.WriteLine($"Capturing payment for {invoiceId}, amount {amount}");
        try
        {
            var result = await _gateway.CaptureAsync(invoiceId, amount);
            Console.WriteLine($"Result: {result.Status}");
            return result;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error: {ex.Message}");
            throw;
        }
    }
}
```

Problems: no `ILogger`, no correlation, no structured fields, no metrics, no trace, no PII review, no level control. Production troubleshooting is impossible.

### After — structured, traced, measured

```csharp
namespace Contoso.Payments.Application;

public sealed partial class PaymentService
{
    private static readonly ActivitySource Activity = new("Contoso.Payments");
    private static readonly Meter Meter = new("Contoso.Payments");
    private static readonly Counter<long> CapturesTotal =
        Meter.CreateCounter<long>("payments.captures.total");
    private static readonly Counter<long> CapturesFailed =
        Meter.CreateCounter<long>("payments.captures.failed");
    private static readonly Histogram<double> CaptureDurationMs =
        Meter.CreateHistogram<double>("payments.capture.duration_ms");

    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(IPaymentGateway gateway, ILogger<PaymentService> logger)
    {
        _gateway = gateway;
        _logger = logger;
    }

    public async Task<PaymentResult> CaptureAsync(
        Guid invoiceId,
        decimal amount,
        string currency,
        CancellationToken cancellationToken)
    {
        using var activity = Activity.StartActivity("Payment.Capture");
        activity?.SetTag("invoice.id", invoiceId);
        activity?.SetTag("amount", amount);
        activity?.SetTag("currency", currency);

        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["InvoiceId"] = invoiceId,
            ["Amount"] = amount,
            ["Currency"] = currency
        });

        var sw = Stopwatch.StartNew();
        try
        {
            Log.CaptureStarted(_logger, invoiceId, amount, currency);

            var result = await _gateway.CaptureAsync(invoiceId, amount, cancellationToken);

            CapturesTotal.Add(1,
                new KeyValuePair<string, object?>("currency", currency),
                new KeyValuePair<string, object?>("outcome", result.Status.ToString()));

            if (result.Status == PaymentStatus.Approved)
            {
                Log.CaptureSucceeded(_logger, invoiceId, result.TransactionId);
                activity?.SetStatus(ActivityStatusCode.Ok);
            }
            else
            {
                CapturesFailed.Add(1, new KeyValuePair<string, object?>("reason", result.DeclineReason ?? "unknown"));
                Log.CaptureDeclined(_logger, invoiceId, result.DeclineReason);
                activity?.SetStatus(ActivityStatusCode.Error, result.DeclineReason);
            }

            return result;
        }
        catch (Exception ex)
        {
            CapturesFailed.Add(1, new KeyValuePair<string, object?>("reason", "exception"));
            Log.CaptureFailed(_logger, ex, invoiceId);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
        finally
        {
            sw.Stop();
            CaptureDurationMs.Record(sw.Elapsed.TotalMilliseconds);
        }
    }

    private static partial class Log
    {
        [LoggerMessage(EventId = 2001, Level = LogLevel.Information,
            Message = "Capture started for invoice {InvoiceId} amount {Amount:F2} {Currency}")]
        public static partial void CaptureStarted(ILogger logger, Guid invoiceId, decimal amount, string currency);

        [LoggerMessage(EventId = 2002, Level = LogLevel.Information,
            Message = "Capture succeeded for invoice {InvoiceId} txn {TransactionId}")]
        public static partial void CaptureSucceeded(ILogger logger, Guid invoiceId, string transactionId);

        [LoggerMessage(EventId = 2003, Level = LogLevel.Warning,
            Message = "Capture declined for invoice {InvoiceId} reason {Reason}")]
        public static partial void CaptureDeclined(ILogger logger, Guid invoiceId, string? reason);

        [LoggerMessage(EventId = 2004, Level = LogLevel.Error,
            Message = "Capture failed for invoice {InvoiceId}")]
        public static partial void CaptureFailed(ILogger logger, Exception ex, Guid invoiceId);
    }
}
```

Now: every capture emits structured logs with shared scope, a distributed-trace span, three metrics (total/failed/duration), and zero string-interpolation overhead.

## Interview Questions and Answers

### 1. Why prefer structured logging over `string.Format` or interpolation?

**Why this matters:** Tests whether the candidate understands log backends as queryable databases.

**Answer:** Backends like App Insights, Seq, Datadog, and Splunk index named properties separately from the message template. `_logger.LogInformation("Captured {InvoiceId}", id)` stores `InvoiceId` as a typed column you can filter, group, and alert on. `$"Captured {id}"` produces a unique string per invoice — making "how many captures for invoice X?" impossible without regex parsing of millions of rows.

**Trade-off:** Structured templates require discipline; developers tempted by `$"..."` will silently bypass the system. Roslyn analyzers (`CA2254`) catch this at build time.

**Real project:** Migrating a payments service from interpolation to structured templates cut the Kusto query for "all failures for invoice X" from a 60-second `contains` scan to a 200ms indexed lookup.

### 2. How do you correlate logs across API → Service Bus → worker?

**Why this matters:** This is the single biggest productivity multiplier in distributed systems.

**Answer:** Use W3C TraceContext (`traceparent` header). ASP.NET Core, `HttpClient`, and the Azure Service Bus SDK ≥7.x all flow it automatically. On the worker, `ActivitySource` picks it up from the message's `DiagnosticId` property. App Insights and OpenTelemetry stitch the full Gantt chart so one click shows API → bus latency → worker → SQL → Stripe.

**Trade-off:** Older clients or custom transports may not propagate; verify with end-to-end tests that `Activity.Current?.TraceId` matches across hops.

**Real project:** Before correlation IDs, debugging "order stuck in pending" took ~3 hours of grep across five services. After: one query in App Insights, < 5 minutes.

### 3. What log level should "user not found" be?

**Why this matters:** A simple question that exposes whether the candidate thinks about alert noise.

**Answer:** `Information` or `Debug`, not `Error`. It is expected behavior — users mistype emails. `Error` should be reserved for things that need investigation. Logging "not found" as `Error` poisons your error-rate SLO and trains the team to ignore alerts.

**Real project:** A login service was logging `Error` on every wrong password. Dashboards showed a 12% "error rate"; the real one was 0.04%. Once reclassified, a real outage was actually visible.

### 4. How do you keep App Insights costs under control?

**Why this matters:** Observability bills can dwarf compute bills.

**Answer:** Four levers: **(1)** raise the floor — set `Microsoft.AspNetCore` to `Warning` in production; **(2)** sample — App Insights adaptive sampling keeps traces at a target rate (default 5 items/sec); **(3)** drop noisy categories with a `TelemetryProcessor`; **(4)** avoid logging in tight loops — aggregate. Always exempt exceptions and `Critical` from sampling.

**Trade-off:** Sampling hides individual requests; for compliance-critical events (payments, auth), use a separate non-sampled sink.

**Real project:** A team's App Insights bill went from $400/mo to $11k/mo after a deploy that logged every cache hit. Lowering that one category to `Debug` reverted the bill within a day.

### 5. How do you handle PII in logs?

**Why this matters:** GDPR, PCI-DSS, and SOC 2 violations start here.

**Answer:** Never log payloads — log identifiers only (`OrderId`, `CustomerId`, `InvoiceId`). For required fields (email for auth troubleshooting), mask at the sink with a Serilog `IDestructuringPolicy` or App Insights `ITelemetryProcessor` that redacts known property names. Add an automated test: send a PII string in a known field and assert it does not appear in the sink output.

**Real project:** A pen test found JWTs in `RequestHeaders` logs because someone added `app.UseSerilogRequestLogging()` with default config. Fix took one config line, but disclosure obligations took weeks.

### 6. What's the difference between logs, metrics, and traces?

**Why this matters:** The "three pillars" are not interchangeable; senior engineers know when to reach for each.

**Answer:** **Logs** are individual events with rich context — best for "what happened to this specific request?" **Metrics** are pre-aggregated numbers — best for "what's the error rate over the last hour?" Cheap to store, expensive to add high-cardinality dimensions. **Traces** are causal chains of spans across services — best for "where did the 800ms come from?" Use logs for forensics, metrics for SLOs/alerts, traces for latency analysis.

**Trade-off:** Some teams over-rely on logs and try to compute SLOs by counting log lines — slow and expensive. Emit metrics for things you alert on.

### 7. Should `Error` logs include the exception or just the message?

**Why this matters:** Stack traces are the single most valuable diagnostic artifact.

**Answer:** Always pass the exception as the first argument: `_logger.LogError(ex, "Capture failed for {InvoiceId}", id)`. This preserves the stack trace, inner exception, and exception data — `ex.ToString()` in the message string loses structured data and is harder to format consistently across sinks.

**Real project:** A team lost three hours of debugging because logs had `"Capture failed: Object reference not set"` with no stack — the dev had `LogError("Capture failed: " + ex.Message)`. The actual NRE was in a downstream library.

### 8. How would you instrument a new payment endpoint from scratch?

**Why this matters:** Synthesizes the whole topic into a practical answer.

**Answer:** **(1)** `ILogger<PaymentService>` with `BeginScope` over `InvoiceId`/`CustomerId`; **(2)** `LoggerMessage` source generator for the three or four expected events; **(3)** `ActivitySource` span around the gateway call with `invoice.id` and `amount` tags; **(4)** three metrics — `payments.captures.total`, `.failed`, `.duration_ms`; **(5)** App Insights or OTel exporter wired in `Program.cs`; **(6)** dashboard with success rate, P95 latency, error breakdown; **(7)** alert on error rate > 1% over 5 min and P99 > 2s; **(8)** healthcheck for Stripe connectivity.

**Real project:** Following this template, a new "refund" endpoint was production-debuggable on day one — first incident was diagnosed in 4 minutes via the auto-generated trace.

## Summary Checklist

- [ ] I use `ILogger<T>` with structured templates and never string interpolation.
- [ ] I use the `LoggerMessage` source generator on hot paths.
- [ ] I `BeginScope` for per-request fields (`OrderId`, `CustomerId`, `CorrelationId`).
- [ ] I emit business metrics with `Meter` and `Counter<T>` / `Histogram<T>`.
- [ ] I instrument distributed traces with `ActivitySource` and rely on W3C TraceContext.
- [ ] I wire Application Insights or OpenTelemetry in `Program.cs`.
- [ ] I never log secrets, JWTs, full card numbers, or PII payloads.
- [ ] I set per-category log levels in `appsettings.{Environment}.json`.
- [ ] I define SLOs and alert on error-budget burn, not raw error counts.
- [ ] I add health checks for liveness/readiness probes.
