# Health Checks

## What It Is

**Health checks** are HTTP endpoints (typically `/health/live`, `/health/ready`, `/health/startup`) that an application exposes so external systems — load balancers, Kubernetes, Azure App Service, Container Apps, monitoring agents — can ask "are you healthy enough to receive traffic?"

ASP.NET Core has first-class support via the `Microsoft.Extensions.Diagnostics.HealthChecks` package (built into the framework) plus an ecosystem of contributed checks (`AspNetCore.HealthChecks.*` published by Xabaril, maintained on `nuget.org`) for SQL Server, Azure Service Bus, Azure Storage, Redis, RabbitMQ, IdentityServer, and more.

There are **three distinct kinds** of health check, each answering a different question:

| Check | Question | Failure → action | Typical content |
|---|---|---|---|
| **Liveness** | Is this process responsive? | Restart the container | Always return 200 unless the process is wedged |
| **Readiness** | Should this instance receive traffic right now? | Remove from load balancer (no restart) | Check critical dependencies — SQL, Service Bus |
| **Startup** | Has this instance finished initializing? | Restart after timeout | Used to grant slow apps a grace period before liveness/readiness run |

The endpoints return:
- **`200 OK`** with body like `Healthy` when all checks pass.
- **`503 Service Unavailable`** when one or more checks are `Unhealthy`.
- **`200 OK`** by default for `Degraded` (the platform considers it healthy; only your dashboards see the degradation).

The orchestrator polls these endpoints on a schedule (Kubernetes default: every 10 seconds for liveness, with failure thresholds) and takes action when they return 5xx.

## Why It Exists

Pre-health-check world: "the process is running, so it must be working." Reality:

- The process is alive, but it's deadlocked on a thread-pool starvation issue and every HTTP request times out.
- The process started, but the connection pool to SQL Server is exhausted because credentials were rotated.
- The pod was scheduled and is "ready" by the orchestrator's definition, but it hasn't loaded its in-memory cache yet, so the first 30 seconds of traffic gets cache-miss errors.
- A rolling deploy brought up 5 new pods that immediately got traffic — but they hadn't connected to Service Bus yet, so messages were lost.

Health checks are the contract that lets the platform make smart decisions:

- **Don't restart a pod that's serving traffic fine but happens to have a degraded non-critical dependency.**
- **Don't send traffic to a pod that's still warming up.**
- **Do restart a pod whose process is wedged and has stopped responding entirely.**
- **Do gracefully drain a pod that's shutting down by returning 503 from readiness before terminating.**

They also surface dependency health in monitoring: a Grafana dashboard or Azure Monitor alert that fires when 3 of 6 pods report `/health/ready` returning 503 tells you "the database is having issues affecting half the fleet" — actionable, focused, and faster than chasing individual exceptions.

## When To Use It

**Always add health checks to:**

- Every ASP.NET Core API and worker service deployed to **AKS**, **Container Apps**, **App Service for Containers**, or **Azure Functions on Premium/Dedicated** (Consumption Functions are handled by the platform).
- Background workers behind a load balancer — even though they don't serve HTTP business traffic, they should expose a tiny health endpoint for liveness.
- Services with **long startup time** (warming caches, EF migrations, pre-fetching reference data) — add a startup probe.

**Don't overdo it for:**

- A throwaway CLI tool or one-shot job — exit code is the health signal.
- A function on Consumption plan — the platform monitors execution success; adding `/health` adds no value.

But the bar is low: **even a single `MapHealthChecks("/health")` is better than nothing** because it gives load balancers something to probe.

## Why It Is Important

Health checks are the single highest-ROI 30 lines of code you'll ship to a containerized .NET service. They unlock:

1. **Safer rolling deployments** — orchestrator only sends traffic to pods that are actually ready.
2. **Self-healing** — wedged processes get restarted automatically without paging anyone.
3. **Graceful shutdown** — readiness returns 503 immediately on SIGTERM; load balancer drains; in-flight requests finish; pod exits cleanly.
4. **Faster MTTR** — dashboards show *which dependency* failed, not just "the service is sad".
5. **Compliance signal** — health checks feed SLO/SLI metrics ("percentage of pods reporting ready over a 5-minute window").
6. **Cheap warning system** — Application Insights alerts can fire when `/health/ready` flips to 503 anywhere in production.

Conversely, missing or wrong health checks cause some of the **worst production incidents** — cascading restarts during a brief DB blip, traffic served by half-initialized pods, or load balancers that never notice a wedged instance.

## How It's Used in C# / .NET

### 1. Built-in package + basic registration

`Microsoft.Extensions.Diagnostics.HealthChecks` is built into the framework. For provider-specific checks, add packages from the **Xabaril** project:

```bash
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.AzureServiceBus
dotnet add package AspNetCore.HealthChecks.AzureStorage
dotnet add package AspNetCore.HealthChecks.Redis
dotnet add package AspNetCore.HealthChecks.AzureKeyVault
dotnet add package AspNetCore.HealthChecks.UI.Client    # for the JSON response writer
```

`Program.cs`:

```csharp
using HealthChecks.UI.Client;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddHealthChecks()
    // ---- liveness: only the process itself
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })

    // ---- readiness: critical dependencies
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("OrdersDb")!,
        healthQuery: "SELECT 1;",
        name: "sql-orders",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "ready", "db" })
    .AddAzureServiceBusQueue(
        connectionString: builder.Configuration["ServiceBus:ConnectionString"]!,
        queueName: "order-events",
        name: "servicebus-orders",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "ready", "messaging" })
    .AddAzureBlobStorage(
        connectionString: builder.Configuration["Storage:ConnectionString"]!,
        name: "blob-orders",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "ready", "storage" })
    .AddRedis(
        redisConnectionString: builder.Configuration["Redis:ConnectionString"]!,
        name: "redis-cache",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "ready", "cache" });

var app = builder.Build();

// Liveness — only checks tagged "live"
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    AllowCachingResponses = false
});

// Readiness — only checks tagged "ready"
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    ResultStatusCodes =
    {
        [HealthStatus.Healthy]   = StatusCodes.Status200OK,
        [HealthStatus.Degraded]  = StatusCodes.Status200OK,   // degraded ≠ unavailable
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    },
    AllowCachingResponses = false
});

// Startup — same as live for many apps; use it to gate liveness/readiness during warmup
app.MapHealthChecks("/health/startup", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("startup") || check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.Run();
```

This emits a JSON response like:

```json
{
  "status": "Healthy",
  "totalDuration": "00:00:00.0234567",
  "entries": {
    "sql-orders":      { "status": "Healthy", "duration": "00:00:00.0123456" },
    "servicebus-orders": { "status": "Healthy", "duration": "00:00:00.0042345" },
    "redis-cache":     { "status": "Degraded", "description": "Connection timeout", "duration": "00:00:00.0500000" }
  }
}
```

### 2. Custom health check for business invariants

Implement `IHealthCheck` for anything not covered by a package:

```csharp
public sealed class PricingApiHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _httpFactory;
    private readonly ILogger<PricingApiHealthCheck> _logger;

    public PricingApiHealthCheck(IHttpClientFactory httpFactory, ILogger<PricingApiHealthCheck> logger)
    {
        _httpFactory = httpFactory;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            using var client = _httpFactory.CreateClient("pricing");
            client.Timeout = TimeSpan.FromSeconds(2);

            var response = await client.GetAsync("/ping", cancellationToken);
            if (response.IsSuccessStatusCode)
                return HealthCheckResult.Healthy("Pricing API reachable");

            return HealthCheckResult.Degraded(
                $"Pricing API returned {(int)response.StatusCode}",
                data: new Dictionary<string, object> { ["statusCode"] = (int)response.StatusCode });
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Pricing API health check failed");
            return HealthCheckResult.Degraded("Pricing API unreachable", ex);
        }
    }
}

// Registration
builder.Services.AddHttpClient("pricing", c =>
    c.BaseAddress = new Uri(builder.Configuration["Pricing:BaseUrl"]!));

builder.Services.AddHealthChecks()
    .AddCheck<PricingApiHealthCheck>(
        name: "pricing-api",
        failureStatus: HealthStatus.Degraded,    // not critical — fallback exists
        tags: new[] { "ready", "external" });
```

Note `failureStatus: Degraded` — when the pricing API is down we still serve traffic (with cached prices or a fallback). It shouldn't pull the pod out of rotation.

### 3. Kubernetes probe wiring

```yaml
containers:
  - name: order-api
    image: contosoacr.azurecr.io/order-api:abc1234
    ports: [ { name: http, containerPort: 8080 } ]
    startupProbe:
      httpGet: { path: /health/startup, port: http }
      failureThreshold: 30      # allow 30 * 5s = 150s for startup
      periodSeconds: 5
    livenessProbe:
      httpGet: { path: /health/live, port: http }
      initialDelaySeconds: 0    # startupProbe handles initial grace
      periodSeconds: 30
      timeoutSeconds: 3
      failureThreshold: 3
    readinessProbe:
      httpGet: { path: /health/ready, port: http }
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
```

See [kubernetes-basics.md](kubernetes-basics.md) for the full Deployment context.

### 4. Azure App Service health check

In App Service blade → **Monitoring → Health check**, set Path = `/health/ready`. App Service will:

- Ping the path every minute from every instance.
- Stop sending traffic to instances returning 503 for 2 consecutive minutes.
- Replace the instance entirely after 1 hour of unhealthy responses (if 2+ instances).

For App Service for Containers, the same setting applies. The traffic load balancer respects it before the slot-swap completes.

### 5. Graceful shutdown integration

Wire readiness to the host lifecycle so the pod returns 503 *immediately* when shutdown begins:

```csharp
public sealed class ShutdownReadinessCheck : IHealthCheck
{
    private static volatile bool _isShuttingDown;

    public static void Trigger() => _isShuttingDown = true;

    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct)
        => Task.FromResult(_isShuttingDown
            ? HealthCheckResult.Unhealthy("Application is shutting down")
            : HealthCheckResult.Healthy());
}

// Registration
builder.Services.AddSingleton<ShutdownReadinessCheck>();
builder.Services.AddHealthChecks()
    .AddCheck<ShutdownReadinessCheck>("shutdown", tags: new[] { "ready" });

var app = builder.Build();

app.Lifetime.ApplicationStopping.Register(() =>
{
    ShutdownReadinessCheck.Trigger();
    // Let the load balancer notice (k8s polls every 10s) before we actually stop
    Thread.Sleep(TimeSpan.FromSeconds(15));
});
```

Combined with K8s `terminationGracePeriodSeconds: 30` and a `preStop: sleep 10`, in-flight requests complete and no new requests arrive during shutdown.

### 6. Application Insights / OpenTelemetry integration

Health-check results can be published as custom metrics:

```csharp
builder.Services.AddSingleton<IHealthCheckPublisher, ApplicationInsightsHealthPublisher>();

builder.Services.Configure<HealthCheckPublisherOptions>(o =>
{
    o.Delay   = TimeSpan.FromSeconds(5);
    o.Period  = TimeSpan.FromSeconds(30);
    o.Predicate = check => check.Tags.Contains("ready");
});
```

Then in Application Insights you can alert on `customMetrics | where name == "HealthCheck.sql-orders"` going to 0.

### 7. Health Checks UI dashboard

For internal dashboards:

```csharp
builder.Services
    .AddHealthChecksUI(o => o.AddHealthCheckEndpoint("Order API", "/health/ready"))
    .AddInMemoryStorage();

app.MapHealthChecksUI(o => o.UIPath = "/health-dashboard");
```

Useful in dev/staging; in production prefer Application Insights or Grafana.

## Advantages

- **Self-healing infrastructure** — orchestrators take action on health signals automatically.
- **Zero-downtime deploys** — readiness gates traffic during rolling updates.
- **Faster incident detection** — dashboards and alerts pivot on health endpoints.
- **Graceful shutdown** — `Lifetime.ApplicationStopping` + readiness signaling drains traffic cleanly.
- **Standardized across languages** — Kubernetes/Container Apps probe HTTP endpoints; not .NET-specific.
- **Free in the box** — `Microsoft.Extensions.Diagnostics.HealthChecks` is part of the framework.
- **Granular signals** — per-dependency status shows exactly what failed.
- **SLO/SLI primitive** — "percent of pods reporting Healthy over 5m" is a meaningful objective.

## Disadvantages

- **Tempting to over-check** — heavy probes (full DB query, full Service Bus enumeration) become a load source themselves.
- **Wrong probe → cascading failure** — liveness checking SQL means a DB blip restarts every pod, multiplying the outage.
- **Auth on health endpoints** is awkward — orchestrators usually probe anonymously; exposing them publicly leaks internal architecture.
- **Health-check latency adds startup time** — a slow `/health/ready` on every pod delays rolling deploys.
- **Degraded ≠ Unhealthy** trips up newcomers — by default Degraded returns 200.
- **External-dependency checks can flap** — an SLA-degraded vendor causes constant alert noise.
- **Doesn't replace monitoring** — health checks are point-in-time; you still need APM for trends.

## Common Mistakes

### 1. Liveness probe that checks SQL Server

```csharp
app.MapHealthChecks("/health");    // single endpoint, includes SQL
```

```yaml
livenessProbe:
  httpGet: { path: /health, port: 8080 }
```

When SQL has a 90-second incident, every pod fails liveness within 90 seconds, K8s restarts them all, the new pods also fail because SQL is still down — now you have a crash loop instead of a brief degradation. This is one of the most common production-incident amplifiers.

**Fix**: liveness must check only that the process is responsive — typically just `() => HealthCheckResult.Healthy()`. Readiness checks SQL.

### 2. No readiness probe at all

Default Kubernetes behavior: a pod becomes "ready" the moment its container is started. ASP.NET Core takes 1-3 seconds to compile pipelines and warm up the DI container — those first requests get failures.

**Fix**: always wire readiness, and the orchestrator waits for `/health/ready` to return 200 before adding the pod to the Service.

### 3. Slow or expensive checks

```csharp
.AddCheck("orders-table", async _ =>
{
    using var conn = new SqlConnection(connStr);
    await conn.OpenAsync();
    using var cmd = new SqlCommand("SELECT COUNT(*) FROM Orders WITH (NOLOCK)", conn);
    await cmd.ExecuteScalarAsync();
    return HealthCheckResult.Healthy();
});
```

A `COUNT(*)` over a billion-row table on every health probe melts the DB. Even cheap probes run *frequently* (every 10s × every pod × every dependency).

**Fix**: use `SELECT 1` or the connection establishment as the signal. Don't query business data.

### 4. Exposing health endpoints publicly without filtering

```csharp
app.MapHealthChecks("/health");     // reveals: connection strings (in errors), every dependency name
```

An attacker probing `/health/ready` learns your entire architecture — SQL, Service Bus queues, Redis, third-party APIs.

**Fix**: either restrict via IP allow-list (App Service `Authorization` access restrictions), or have two endpoints — `/health/live` minimal public, `/health/ready` internal only:

```csharp
app.MapHealthChecks("/health/live", new HealthCheckOptions { /* minimal */ });
app.MapHealthChecks("/health/ready", new HealthCheckOptions { /* full */ })
   .RequireAuthorization("HealthCheckPolicy");
```

### 5. Returning 200 for `Unhealthy`

```csharp
new HealthCheckOptions
{
    ResultStatusCodes = { [HealthStatus.Unhealthy] = 200 }   // BUG
}
```

Probes that always get 200 mean orchestrators can never act. Bizarrely common when copy-pasting from samples.

**Fix**: leave the defaults (200/200/503 for Healthy/Degraded/Unhealthy). Only customize after you understand the consequences.

### 6. No startup probe for slow apps

A service that takes 45 seconds to warm caches gets liveness-killed at 30 seconds.

**Fix**: add a startup probe that grants extra time:

```yaml
startupProbe:
  httpGet: { path: /health/startup, port: http }
  failureThreshold: 30        # 30 × 5s = 150s grace
  periodSeconds: 5
```

Liveness and readiness don't run until startup passes.

### 7. Forgetting to wire shutdown signaling

Pod gets `SIGTERM`, ASP.NET Core starts gracefully stopping, but `/health/ready` still returns 200 until the entire `Lifetime.ApplicationStopped` chain finishes. Meanwhile new requests arrive and get refused.

**Fix**: `ShutdownReadinessCheck` (see §5) flips readiness to Unhealthy *immediately* on shutdown, giving the load balancer time to drain.

### 8. Caching responses

```csharp
new HealthCheckOptions { AllowCachingResponses = true }    // BAD
```

A cached 200 from 60 seconds ago doesn't reflect that SQL just died.

**Fix**: explicit `AllowCachingResponses = false` (which is the default — don't override it).

## Best Practices

- **Three endpoints**: `/health/live`, `/health/ready`, `/health/startup`. Use **tags** to route checks to the right endpoint.
- **Liveness checks only the process** — almost always `() => HealthCheckResult.Healthy()` plus maybe a watchdog for stuck background services.
- **Readiness checks critical dependencies** — SQL, Service Bus, Key Vault — using `failureStatus: Unhealthy`.
- **Non-critical dependencies use `failureStatus: Degraded`** so they don't pull the pod from rotation.
- **Keep probes fast** (< 1 second). Use `SELECT 1` for SQL, peek-don't-receive for Service Bus.
- **Reasonable probe schedules**: liveness every 30s, readiness every 10s, startup every 5s with high failure threshold.
- **Use `UIResponseWriter.WriteHealthCheckUIResponse`** for structured JSON output.
- **Wire `Lifetime.ApplicationStopping`** to flip readiness Unhealthy on shutdown.
- **Restrict access** — internal-only or with auth — at least for `/health/ready` which exposes architecture.
- **Publish health-check metrics** to Application Insights / OpenTelemetry for trend analysis and alerting.
- **Test health checks in CI** — at minimum a smoke test that hits `/health/ready` against a fully wired test environment.
- **Document the runbook** — what does each Unhealthy/Degraded check mean, who pages, what to do.

## Related Concepts

- **[kubernetes-basics.md](kubernetes-basics.md)** — the probe configuration consumes these endpoints.
- **[docker.md](docker.md)** — the image must `EXPOSE` the port the probes hit.
- **[blue-green-deployment.md](blue-green-deployment.md)** — App Service slot swap uses health checks as a gate.
- **[ci-cd-pipelines.md](ci-cd-pipelines.md)** — post-deploy smoke tests hit `/health/ready`.
- **[rollback-strategy.md](rollback-strategy.md)** — sustained 503s on `/health/ready` should trigger automated rollback.
- **[../aspnet-core/logging-and-monitoring.md](../aspnet-core/logging-and-monitoring.md)** — where health-check failures get logged and alerted.
- **[../azure/application-insights.md](../azure/application-insights.md)** — destination for health-check telemetry.
- **[../azure/app-service.md](../azure/app-service.md)** — `Health check Path` setting in App Service.
- **[../architecture/reliability-design.md](../architecture/reliability-design.md)** — the broader resilience picture health checks contribute to.

## Real-World Usage

**Scenario: an order-processing platform on AKS**

A logistics company runs `order-api` (HTTP), `order-processor` (Service Bus worker), and `shipment-tracker` (HTTP) on a 6-node AKS cluster. Health-check setup:

- **All three services** expose `/health/live`, `/health/ready`, `/health/startup` via `Microsoft.Extensions.Diagnostics.HealthChecks`.
- **Liveness** is `() => HealthCheckResult.Healthy()` for all three — the only thing that can fail it is a deadlocked process, which `kubectl` will then restart.
- **Readiness** for `order-api`: SQL (Unhealthy), Service Bus (Unhealthy), Redis (Degraded — fallback to DB).
- **Readiness** for `order-processor`: Service Bus connection only (Unhealthy if can't peek the queue).
- **Startup probe** for `order-api`: gives 90s for EF schema validation + cache warmup.
- **Shutdown signaling**: every service flips a `ShutdownReadinessCheck` to Unhealthy on `Lifetime.ApplicationStopping`, then sleeps 15s before allowing exit. K8s `terminationGracePeriodSeconds: 30` + `preStop: sleep 10` ensures the load balancer drains.
- **Monitoring**: Application Insights collects health-check results every 30s via `IHealthCheckPublisher`. Alert: "3+ pods reporting `/health/ready` Unhealthy for SQL for >5 minutes" pages the on-call DBA.

During a recent SQL maintenance window, readiness flipped to Unhealthy on all 8 `order-api` pods. K8s stopped sending traffic; the load balancer returned 503 to clients (handled gracefully by client retry logic). When SQL came back, readiness flipped to Healthy, and traffic resumed. **No pod was restarted** — the process was fine, only the dependency was down. This is exactly the behavior you want.

## Code Example — Before and After

### Before: single endpoint, mixed signals, no shutdown handling

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Db")!)
    .AddRedis(builder.Configuration["Redis"]!);

var app = builder.Build();

app.MapHealthChecks("/health");    // one endpoint for everything

app.Run();
```

```yaml
livenessProbe:
  httpGet: { path: /health, port: 80 }
readinessProbe:
  httpGet: { path: /health, port: 80 }
```

Problems:
- SQL outage causes liveness failure → pod restart → restart loop → cluster-wide cascading failure.
- Redis outage takes a pod out of rotation that could have served traffic via the DB fallback.
- No startup grace; slow EF migration triggers immediate liveness restart.
- No shutdown signaling — pod returns 200 from `/health` while load balancer sends final requests, which die during shutdown.
- Health response is plain text — hard to dashboard.

### After: three endpoints, tags, custom shutdown check, structured response

```csharp
using HealthChecks.UI.Client;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ShutdownReadinessCheck>();

builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddCheck<ShutdownReadinessCheck>("shutdown", tags: new[] { "ready" })
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Db")!,
        healthQuery: "SELECT 1;",
        name: "sql",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "ready" })
    .AddRedis(
        redisConnectionString: builder.Configuration["Redis"]!,
        name: "redis",
        failureStatus: HealthStatus.Degraded,        // fallback to DB
        tags: new[] { "ready" });

var app = builder.Build();

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = c => c.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    AllowCachingResponses = false
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = c => c.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    AllowCachingResponses = false
});

app.MapHealthChecks("/health/startup", new HealthCheckOptions
{
    Predicate = c => c.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.Lifetime.ApplicationStopping.Register(() =>
{
    ShutdownReadinessCheck.Trigger();
    Thread.Sleep(TimeSpan.FromSeconds(15));    // give k8s a poll cycle to notice
});

app.Run();

public sealed class ShutdownReadinessCheck : IHealthCheck
{
    private static volatile bool _shuttingDown;
    public static void Trigger() => _shuttingDown = true;

    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext _, CancellationToken __)
        => Task.FromResult(_shuttingDown
            ? HealthCheckResult.Unhealthy("Shutting down")
            : HealthCheckResult.Healthy());
}
```

```yaml
startupProbe:
  httpGet: { path: /health/startup, port: http }
  failureThreshold: 30
  periodSeconds: 5
livenessProbe:
  httpGet: { path: /health/live, port: http }
  periodSeconds: 30
readinessProbe:
  httpGet: { path: /health/ready, port: http }
  periodSeconds: 10
```

Now:
- SQL outage flips readiness Unhealthy; traffic stops; pods stay alive; no cascading restart.
- Redis outage marks Degraded but doesn't pull pods from rotation.
- Slow startup gets 150 seconds of grace; once warm, normal probes apply.
- Graceful shutdown: 15-second drain window prevents lost requests.
- JSON response feeds Application Insights and Health Checks UI dashboards.

## Interview Questions and Answers

### 1. Difference between liveness, readiness, and startup probes — and how do you implement them in ASP.NET Core?

**Why this matters**: Tests whether the candidate knows the operational nuance, not just the API.

**Answer**:
- **Liveness** → is the process responsive? Failure restarts the container. In ASP.NET Core, `AddCheck("self", () => HealthCheckResult.Healthy())` tagged "live", mapped to `/health/live`.
- **Readiness** → can the app serve traffic right now? Failure removes it from the load balancer but doesn't restart. Map dependency checks (SQL, Service Bus) tagged "ready" to `/health/ready`.
- **Startup** → has the app finished initializing? Disables liveness/readiness until passed. Mapped to `/health/startup`, used with `startupProbe.failureThreshold: 30` to allow slow warmup.

**Critical rule**: liveness must NEVER check external dependencies. A 90-second SQL blip with liveness=SQL means every pod restarts within 90s, the new pods also fail, and you have a cluster-wide crash loop instead of a transient outage.

### 2. Walk me through wiring health checks for an ASP.NET Core API that uses SQL Server, Azure Service Bus, and Redis.

**Answer**: Install the relevant Xabaril packages (`AspNetCore.HealthChecks.SqlServer`, `.AzureServiceBus`, `.Redis`, `.UI.Client`). Register with appropriate tags:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddSqlServer(connStr, healthQuery: "SELECT 1;", name: "sql",
                  failureStatus: HealthStatus.Unhealthy, tags: new[] { "ready" })
    .AddAzureServiceBusQueue(sbConnStr, "orders", name: "servicebus",
                  failureStatus: HealthStatus.Unhealthy, tags: new[] { "ready" })
    .AddRedis(redisConnStr, name: "redis",
              failureStatus: HealthStatus.Degraded, tags: new[] { "ready" });
```

Map three endpoints filtered by tag; SQL and Service Bus are critical (Unhealthy → pod out of rotation); Redis is a cache (Degraded → still in rotation, just slower).

### 3. The team set up health checks but during a Service Bus outage, every pod got restarted and the service was down for 10 minutes. What went wrong?

**Answer**: Liveness probe was wired to the same endpoint as readiness, which checked Service Bus. When the outage started:

1. Probe failed → liveness threshold (3 × 10s = 30s) hit → K8s restarted the pod.
2. New pod started, probe checks Service Bus, still down → restart loop.
3. All pods stuck in CrashLoopBackOff for the duration of the SB outage.

The fix: separate `/health/live` (process only) from `/health/ready` (dependencies). Liveness should keep returning 200 even during the SB outage; readiness flips to 503, the load balancer drains traffic, and recovery is automatic once SB comes back — **no pod restart**, no cascading failure.

**Real project**: After exactly this incident, the team refactored to the three-endpoint pattern. The next SB outage was invisible to liveness; recovery time dropped from 10 minutes (cascading restarts + warmup) to 90 seconds (load balancer rerouting + retry).

### 4. How do you implement graceful shutdown so in-flight requests aren't dropped during a rolling deploy?

**Answer**: Three coordinated pieces:

1. **App-side**: a `ShutdownReadinessCheck` flips readiness to Unhealthy when `IHostApplicationLifetime.ApplicationStopping` fires, then sleeps ~15 seconds before allowing exit.
2. **Kubernetes**: `terminationGracePeriodSeconds: 30` (or longer) plus `lifecycle.preStop.exec: ["sh","-c","sleep 10"]` so the load balancer notices the readiness flip before SIGTERM hits.
3. **ASP.NET Core**: Kestrel honors SIGTERM and stops accepting new connections while letting in-flight requests finish (default behavior; configurable via `HostOptions.ShutdownTimeout`).

End result: when a pod is deleted, readiness goes Unhealthy → load balancer routes new requests to other pods → in-flight requests finish → pod exits cleanly.

### 5. Should health endpoints require authentication?

**Answer**: Trade-off:

- **Pro auth**: prevents enumeration of internal architecture; reduces attack surface (an unauthenticated `/health/ready` reveals every dependency name).
- **Con auth**: orchestrator probes are anonymous; either you add probe-friendly auth (token in headers, IP allow-list, mTLS) or you split endpoints.

The pragmatic pattern:

- **`/health/live`** — minimal info, anonymous (returns just "Healthy" + a duration). Safe for K8s liveness probes.
- **`/health/ready`** — full info, IP-restricted to the cluster (App Service Access Restrictions, K8s NetworkPolicy, or App Gateway WAF).
- **`/health/details`** — fully authenticated, for human ops dashboards.

Avoid embedding bearer tokens in K8s probes — token rotation breaks everything.

### 6. How would you test health checks?

**Answer**: Three layers:

1. **Unit tests** for custom `IHealthCheck` implementations: assert the right status given mocked dependencies.
2. **Integration tests** using `WebApplicationFactory`: hit `/health/ready` against a fully wired test host (Testcontainers SQL, in-memory Service Bus). Confirm 200 when all green, 503 when SQL is killed.
3. **Smoke tests in CD pipeline**: post-deploy, `curl --fail https://app.azurewebsites.net/health/ready` with retry. Block promotion if it doesn't pass.

See [../testing-quality/integration-testing.md](../testing-quality/integration-testing.md).

### 7. How do health checks integrate with App Service slot swaps?

**Answer**: App Service has a built-in **health check** feature (Monitoring → Health check, set Path = `/health/ready`). During slot swap:

1. App Service warms the staging slot by pinging configured paths (including the health check) and applying slot-sticky settings.
2. The swap only proceeds when the warmup succeeds (`WEBSITE_SWAP_WARMUP_PING_PATH` and the health check both pass).
3. After swap, App Service continues polling — if the new production slot returns 503 for too long, you can auto-swap back.

Combine with deployment slot auto-swap settings and Application Insights alerts on `/health/ready` for a fully automated blue-green pattern. See [blue-green-deployment.md](blue-green-deployment.md).

### 8. The `/health/ready` endpoint is taking 5 seconds to respond. How do you fix it?

**Answer**: A slow ready endpoint slows every probe, every smoke test, and every dashboard refresh. Sources and fixes:

- **SQL check running `SELECT COUNT(*)` over a large table** → switch to `"SELECT 1"`.
- **Sequential dependency checks** → they already run in parallel via `HealthCheckService`; if not, you've serialized them somehow (e.g., shared lock). Remove the bottleneck.
- **External HTTP check with no timeout** → set `HttpClient.Timeout = TimeSpan.FromSeconds(2)`.
- **Probe runs deep API call** (peek-all messages from a queue) → switch to a lightweight "connection-open" check.
- **Cold connection** to SQL/Service Bus on first probe → pre-warm by opening connections during startup, keep them in a pool.
- **DNS resolution** (esp. private endpoints) — measure with a custom check that times `Dns.GetHostEntryAsync`.

Goal: `/health/ready` should respond in under 500 ms in the steady state.

## Summary Checklist

- [ ] I can implement three distinct endpoints (`/health/live`, `/health/ready`, `/health/startup`) with tag-based routing.
- [ ] I can explain why liveness must NOT check external dependencies and what happens when you ignore that.
- [ ] I can wire `AddSqlServer`, `AddAzureServiceBusQueue`, `AddRedis`, and a custom `IHealthCheck` correctly.
- [ ] I can use `failureStatus: HealthStatus.Degraded` for non-critical dependencies with fallbacks.
- [ ] I can configure Kubernetes liveness/readiness/startup probes that match the endpoint contract.
- [ ] I can configure App Service health check path and explain its role in slot swaps.
- [ ] I can implement graceful shutdown via `ShutdownReadinessCheck` + `Lifetime.ApplicationStopping` + `preStop`.
- [ ] I can integrate health-check results into Application Insights via `IHealthCheckPublisher`.
- [ ] I can secure health endpoints (IP allow-list, internal-only) without breaking platform probes.
- [ ] I can write integration tests that verify probes return 503 when a critical dependency is down.
