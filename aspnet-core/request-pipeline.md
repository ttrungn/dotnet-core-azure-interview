# ASP.NET Core Request Pipeline

## What It Is

The **request pipeline** is the ordered chain of components — called **middleware** — that ASP.NET Core runs for every incoming HTTP request and outgoing response. Each middleware can inspect, modify, short-circuit, or pass the request on to the next component. The final component is usually an **endpoint** (an MVC action or a minimal API handler) that produces the response.

You build the pipeline in `Program.cs` by calling extension methods on `WebApplication` in the **order** you want them to execute:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();

app.UseExceptionHandler();   // 1. catch unhandled exceptions
app.UseHsts();               // 2. force HTTPS via header
app.UseHttpsRedirection();   // 3. redirect HTTP to HTTPS
app.UseRouting();            // 4. match the endpoint
app.UseCors("Default");      // 5. handle cross-origin headers
app.UseAuthentication();     // 6. identify the user
app.UseAuthorization();      // 7. check permissions
app.MapControllers();        // 8. execute the endpoint

app.Run();
```

Mentally, picture an onion: the request travels **inward** through each layer, the endpoint runs at the center, and the response travels **outward** through the same layers in reverse — which is why a middleware can both add a request header on the way in and a response header on the way out.

## Why It Exists

Before ASP.NET Core, classic ASP.NET (System.Web) used `HttpModules`, `HttpHandlers`, and a tangled IIS lifecycle with ~20 named events (`BeginRequest`, `AuthenticateRequest`, `PostAuthorizeRequest`, …). The order was implicit, configuration lived in `web.config`, and you could not easily move the same hosting code between IIS and a console app.

The middleware pipeline (inspired by OWIN and projects like Connect/Express in Node.js) solves three concrete problems:

1. **Explicit ordering** — the order you call `Use*` is the order it runs. No hidden lifecycle events.
2. **Host-agnostic hosting** — the same pipeline runs on Kestrel, IIS, HTTP.sys, Azure App Service, Azure Container Apps, and Docker images on AKS.
3. **Composable cross-cutting concerns** — logging, authentication, exception handling, compression, CORS, and rate limiting plug in without touching controllers.

It also lets you compose third-party middleware (Serilog request logging, Application Insights, OpenTelemetry, Polly retry handlers) with one line per concern instead of writing custom filters or HTTP modules.

## When To Use It

**Use middleware for:**

- **Cross-cutting concerns** that apply to many endpoints — authentication, authorization, exception translation, request logging, correlation IDs, rate limiting, CORS, response compression.
- **Request enrichment** — adding a `traceparent` header, attaching a tenant ID, parsing a custom header.
- **Response shaping** — adding security headers (`X-Content-Type-Options`, `Strict-Transport-Security`).
- **Short-circuiting** — health checks, maintenance mode, IP allowlists that should return immediately.

**Do not use middleware for:**

- **Per-endpoint business logic** — keep it in controllers, minimal API handlers, or MediatR handlers.
- **Validation of a specific request body** — use model validation, [FluentValidation](https://docs.fluentvalidation.net/), or endpoint filters instead.
- **Anything you only want on one route** — endpoint filters (minimal APIs) or action filters (MVC) are cheaper and easier to test.
- **Long-running work** — middleware runs synchronously inside the request; offload to a [BackgroundService](../architecture/event-driven-architecture.md) or queue.

## Why It Is Important

The pipeline is the **first place a production bug shows up** and the **first place a security audit reads**. Three concrete reasons it matters:

1. **Security correctness depends on ordering.** If `UseAuthentication` runs after `UseAuthorization`, every request looks anonymous and your `[Authorize]` attributes fail closed for legitimate users. If you forget `UseHttpsRedirection`, tokens leak over HTTP.
2. **Observability is built here.** Correlation IDs, request logging, and distributed tracing (Application Insights, OpenTelemetry) are middleware. A misplaced `UseSerilogRequestLogging` produces logs without HTTP context.
3. **Performance is shaped here.** Response caching, response compression, output caching (.NET 7+), and rate limiting all live in the pipeline. A `UseResponseCompression` placed after the endpoint executes does nothing for the body that was already written.

In a cloud / microservices context (Azure App Service, AKS, Container Apps), the pipeline also integrates with the host: `ForwardedHeaders` middleware reads the load balancer's `X-Forwarded-*` headers so `HttpContext.Request.Scheme` reports `https` even when traffic is terminated at the front door.

## How It's Used in C# / .NET

### 1. The three building blocks: `Use`, `Run`, `Map`

```csharp
// Use — runs middleware AND calls the next one (or short-circuits)
app.Use(async (ctx, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(ctx);                                           // call the next middleware
    ctx.Response.Headers["X-Elapsed-Ms"] = sw.ElapsedMilliseconds.ToString();
});

// Run — terminal middleware, never calls next (pipeline ends here)
app.Run(async ctx =>
{
    ctx.Response.StatusCode = StatusCodes.Status404NotFound;
    await ctx.Response.WriteAsync("Endpoint not found");
});

// Map — branches the pipeline on a path prefix
app.Map("/health", branch =>
{
    branch.Run(async ctx => await ctx.Response.WriteAsync("OK"));
});
```

### 2. The canonical built-in middleware order

The order below is the one Microsoft documents and the one that survives production audits. Memorize it:

| # | Middleware                       | Purpose                                                  |
|---|----------------------------------|----------------------------------------------------------|
| 1 | `UseExceptionHandler` / `UseDeveloperExceptionPage` | Translate unhandled exceptions to `ProblemDetails`       |
| 2 | `UseHsts`                        | Send `Strict-Transport-Security` header (production)     |
| 3 | `UseHttpsRedirection`            | Redirect HTTP to HTTPS                                   |
| 4 | `UseStaticFiles`                 | Serve `wwwroot` files (if any)                           |
| 5 | `UseRouting`                     | Match the request to an endpoint                         |
| 6 | `UseCors`                        | Apply CORS policy (must run between routing and auth)    |
| 7 | `UseAuthentication`              | Populate `HttpContext.User` from token / cookie          |
| 8 | `UseAuthorization`               | Enforce `[Authorize]` and policy requirements            |
| 9 | `UseRateLimiter` (.NET 7+)       | Apply rate limit policies                                |
| 10 | `UseOutputCache` (.NET 7+)      | Serve cached responses for matched endpoints             |
| 11 | `MapControllers` / `MapGet` …   | Execute the endpoint                                     |

### 3. Writing a custom middleware (convention-based)

```csharp
public sealed class CorrelationIdMiddleware
{
    private const string HeaderName = "X-Correlation-Id";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, ILogger<CorrelationIdMiddleware> logger)
    {
        var correlationId = context.Request.Headers.TryGetValue(HeaderName, out var existing)
            ? existing.ToString()
            : Guid.NewGuid().ToString("N");

        context.Response.Headers[HeaderName] = correlationId;
        using (logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
        {
            await _next(context);
        }
    }
}

// Registration helper
public static class CorrelationIdExtensions
{
    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder app)
        => app.UseMiddleware<CorrelationIdMiddleware>();
}

// Wire-up
app.UseCorrelationId();    // place early, right after UseExceptionHandler
```

### 4. Factory-based middleware (`IMiddleware`)

Use this when your middleware needs **scoped** dependencies (a `DbContext`, a request-scoped `ITenantContext`):

```csharp
public sealed class TenantResolutionMiddleware : IMiddleware
{
    private readonly ITenantStore _tenants;
    public TenantResolutionMiddleware(ITenantStore tenants) => _tenants = tenants;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var host = context.Request.Host.Host;
        var tenant = await _tenants.ResolveByHostAsync(host, context.RequestAborted);
        context.Items["Tenant"] = tenant ?? throw new InvalidOperationException("Unknown tenant");
        await next(context);
    }
}

// Must be registered in DI
builder.Services.AddScoped<TenantResolutionMiddleware>();
app.UseMiddleware<TenantResolutionMiddleware>();
```

### 5. Endpoint filters (minimal APIs, .NET 7+) — middleware's lighter cousin

```csharp
app.MapPost("/api/orders", (CreateOrderRequest req, IOrderService svc, CancellationToken ct)
        => svc.PlaceAsync(req, ct))
    .AddEndpointFilter(async (ctx, next) =>
    {
        var sw = Stopwatch.StartNew();
        var result = await next(ctx);
        ctx.HttpContext.Response.Headers["X-Handler-Ms"] = sw.ElapsedMilliseconds.ToString();
        return result;
    });
```

Endpoint filters run **after** routing has matched and **only** for the endpoints you attach them to — perfect when middleware would be too broad.

### 6. Short-circuiting and `UseWhen` / `MapWhen`

```csharp
// Only apply the rate limiter to API routes
app.UseWhen(ctx => ctx.Request.Path.StartsWithSegments("/api"), branch =>
{
    branch.UseRateLimiter();
});

// Branch a sub-pipeline for the admin area
app.MapWhen(ctx => ctx.Request.Path.StartsWithSegments("/admin"), admin =>
{
    admin.UseAuthentication();
    admin.UseAuthorization();
    admin.Run(async ctx => await ctx.Response.WriteAsync("admin"));
});
```

### 7. Reading `HttpContext` safely from outside the pipeline

```csharp
builder.Services.AddHttpContextAccessor(); // register once
public class AuditService(IHttpContextAccessor accessor) { /* accessor.HttpContext */ }
```

Note: Avoid `IHttpContextAccessor` inside background services — it returns `null` outside a request.

## Advantages

- **Explicit order** — what you wire is what runs; easy to reason about and review in PRs.
- **Host-agnostic** — the same pipeline runs on Kestrel, IIS, Azure App Service, Azure Container Apps, and Docker on AKS.
- **Composable** — first-party and third-party middleware (Serilog, OpenTelemetry, Polly) plug in with one line.
- **Performance-friendly** — runs as a tight `RequestDelegate` chain (no reflection per request).
- **Short-circuiting** — terminal middleware (health checks, maintenance mode) skips the rest of the pipeline.
- **Async-first** — every step is `Task`-returning, with `CancellationToken` available via `HttpContext.RequestAborted`.

## Disadvantages

- **Order bugs are silent** — `UseAuthorization` before `UseAuthentication` looks fine; everything 401s in production.
- **No type-safe contract between middlewares** — they communicate via `HttpContext.Items` (a `Dictionary<object, object?>`).
- **Lifetime trap** — convention-based middleware is effectively singleton; injecting a scoped service into its constructor captures it.
- **Hard to unit test in isolation** — most teams test middleware via `WebApplicationFactory` integration tests instead.
- **Easy to put logic in the wrong place** — business logic in middleware becomes invisible to controllers and filters.
- **Branching with `Map`/`UseWhen` can fragment configuration** — overuse makes the pipeline hard to follow.

## Common Mistakes

### 1. Wrong order: authorization before authentication

```csharp
// BUG: UseAuthorization runs first → HttpContext.User is never populated
app.UseRouting();
app.UseAuthorization();        // ❌ wrong order
app.UseAuthentication();
app.MapControllers();
```

**Fix**: Authentication must come first; authorization reads what authentication produced.

```csharp
app.UseRouting();
app.UseAuthentication();       // ✅
app.UseAuthorization();
app.MapControllers();
```

### 2. CORS placed after authentication

```csharp
// BUG: Preflight OPTIONS requests are rejected by auth before CORS replies
app.UseAuthentication();
app.UseAuthorization();
app.UseCors("Default");        // ❌ too late
app.MapControllers();
```

**Fix**: `UseCors` must run between `UseRouting` and `UseAuthentication`.

### 3. Forgetting `UseRouting` before `UseAuthorization`

```csharp
// BUG: endpoint-specific policies (e.g., [Authorize(Policy = "Admin")]) are unknown
app.UseAuthentication();
app.UseAuthorization();
app.UseRouting();              // ❌ runs after the policies tried to read the endpoint
app.MapControllers();
```

**Fix**: `UseRouting` must run before `UseAuthorization` so policies can see the matched endpoint.

### 4. Injecting a scoped service into convention-based middleware

```csharp
public class AuditMiddleware
{
    private readonly RequestDelegate _next;
    private readonly AppDbContext _db; // ❌ captured forever — singleton holds scoped DbContext
    public AuditMiddleware(RequestDelegate next, AppDbContext db) { _next = next; _db = db; }
}
```

**Fix**: Inject scoped services into `InvokeAsync` parameters, or implement `IMiddleware`:

```csharp
public async Task InvokeAsync(HttpContext ctx, AppDbContext db) { /* ... */ }
```

### 5. Reading the request body twice

```csharp
public async Task InvokeAsync(HttpContext ctx, RequestDelegate next)
{
    using var reader = new StreamReader(ctx.Request.Body);
    var body = await reader.ReadToEndAsync();     // ❌ consumes the stream
    await next(ctx);                              // controller sees an empty body
}
```

**Fix**: Call `ctx.Request.EnableBuffering()` first, then rewind:

```csharp
ctx.Request.EnableBuffering();
using var reader = new StreamReader(ctx.Request.Body, leaveOpen: true);
var body = await reader.ReadToEndAsync();
ctx.Request.Body.Position = 0;
await next(ctx);
```

### 6. Writing to the response after `await next`

```csharp
await next(ctx);
ctx.Response.Headers["X-Trace-Id"] = traceId;    // ❌ headers may already be sent
```

**Fix**: Use `HttpContext.Response.OnStarting` to register a callback that runs before headers are flushed:

```csharp
ctx.Response.OnStarting(() =>
{
    ctx.Response.Headers["X-Trace-Id"] = traceId;
    return Task.CompletedTask;
});
await next(ctx);
```

### 7. Putting global exception handling inside try/catch in every controller

Catching exceptions inside controllers leaks infrastructure concerns and is easy to forget. Use one `UseExceptionHandler` + `IExceptionHandler` (see [error-handling-and-problem-details.md](error-handling-and-problem-details.md)).

## Best Practices

- **Follow the canonical order**: `ExceptionHandler → HSTS → HttpsRedirection → StaticFiles → Routing → CORS → Authentication → Authorization → RateLimiter → Endpoints`.
- **Place exception handling first** so it catches everything below it, including middleware bugs.
- **Use `UseExceptionHandler` + `IExceptionHandler`** for production; keep `UseDeveloperExceptionPage` for Development only.
- **Prefer `IMiddleware` over convention-based** when your middleware needs scoped dependencies.
- **Use endpoint filters (.NET 7+) for per-endpoint concerns** instead of branching middleware with `MapWhen`.
- **Add a correlation ID middleware early** and log it with `ILogger<T>` scopes so every line for a request is correlatable.
- **Register `IHttpContextAccessor` only when needed** (small per-request cost; not safe outside a request).
- **Cover the pipeline with integration tests** (`WebApplicationFactory<Program>`) — unit-testing middleware in isolation rarely catches order bugs.
- **Document the pipeline** with a short comment block in `Program.cs` listing the order and rationale.
- **Treat middleware as the API contract layer**: status codes, security headers, and error shape live here, not in controllers.

## Related Concepts

- **Endpoints and routing** — `UseRouting` chooses the endpoint; `UseEndpoints` / `MapControllers` runs it. See [controllers-and-minimal-apis.md](controllers-and-minimal-apis.md).
- **Endpoint filters / action filters** — per-endpoint counterpart to middleware.
- **`IExceptionHandler`** (.NET 8+) — modern hook used by `UseExceptionHandler`. See [error-handling-and-problem-details.md](error-handling-and-problem-details.md).
- **Authentication and authorization** — built on `UseAuthentication` / `UseAuthorization`. See [authentication-and-authorization.md](authentication-and-authorization.md).
- **Dependency injection** — middlewares are resolved (or constructed) through `IServiceCollection`. See [../csharp/dependency-injection.md](../csharp/dependency-injection.md).
- **CORS, HSTS, HTTPS redirection** — security middleware that protects every endpoint by default.
- **Health checks** (`MapHealthChecks`) — terminal endpoints exposed for Azure App Service / Kubernetes probes.
- **Output caching and response compression** — pipeline-level performance features (.NET 7+).
- **OpenTelemetry / Application Insights** — observability is plugged in as middleware + DI.

## Real-World Usage

### ASP.NET Core API on Azure App Service

A payments API behind Azure Front Door, deployed to App Service Linux, typically wires the pipeline like this:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
builder.Services.AddAuthentication().AddJwtBearer(/* Microsoft Entra ID */);
builder.Services.AddAuthorization();
builder.Services.AddRateLimiter(/* per-tenant policy */);
builder.Services.AddApplicationInsightsTelemetry();

var app = builder.Build();

app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});                                  // App Service / Front Door TLS termination
app.UseExceptionHandler();           // returns Problem Details
app.UseHsts();
app.UseHttpsRedirection();
app.UseCorrelationId();              // custom middleware shown above
app.UseRouting();
app.UseCors("PartnerPortal");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers();
app.MapHealthChecks("/health/ready");

app.Run();
```

### Azure Functions (isolated worker)

Functions has its own middleware concept via `IFunctionsWorkerApplicationBuilder`:

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults(worker =>
    {
        worker.UseMiddleware<CorrelationIdFunctionMiddleware>();
        worker.UseMiddleware<ExceptionHandlingMiddleware>();
    })
    .Build();
```

The mental model is identical — ordered chain, short-circuit, async — even though the host is different.

### Microservices

In a multi-service estate (Orders, Payments, Inventory), every service registers the **same** base middleware (correlation ID, exception handler, auth, rate limit) via a shared NuGet package (`Contoso.AspNetCore.Defaults`). Each service then adds its own domain-specific middleware on top. This guarantees consistent observability and security across the fleet.

### Multi-tenant SaaS

A tenant-resolution middleware runs early, reads the host header (or a JWT claim), looks up the tenant in a cache, and stashes it on `HttpContext.Items["Tenant"]`. Downstream services (repositories, feature flags, telemetry enrichers) read it from there.

### Testing

```csharp
public class PipelineTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public PipelineTests(WebApplicationFactory<Program> factory) => _client = factory.CreateClient();

    [Fact]
    public async Task Unauthenticated_request_returns_401()
    {
        var response = await _client.GetAsync("/api/orders/123");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task Response_includes_correlation_id_header()
    {
        var response = await _client.GetAsync("/health/ready");
        response.Headers.Should().ContainKey("X-Correlation-Id");
    }
}
```

`WebApplicationFactory` boots the full pipeline in-process — the only reliable way to test ordering.

## Code Example — Before and After

### Before: Logic Scattered, Order Implicit

```csharp
// Program.cs (legacy style)
var app = WebApplication.Create(args);

app.MapControllers();                         // endpoint mapped before auth
app.UseAuthorization();                       // never sees the user
app.UseAuthentication();                      // too late
app.Run();

// Controllers catch exceptions individually
[HttpPost]
public async Task<IActionResult> Create(CreateOrderRequest req)
{
    try
    {
        var id = await _orders.PlaceAsync(req);
        return Ok(new { id });                // wrong status code (should be 201)
    }
    catch (ValidationException ex)
    {
        return BadRequest(ex.Message);        // not Problem Details
    }
    catch (Exception ex)
    {
        return StatusCode(500, ex.ToString()); // leaks stack trace
    }
}
```

Problems:
- Auth is broken — every authenticated user is treated as anonymous.
- No correlation ID, no structured logging, no HSTS, no HTTPS redirect.
- Each controller invents its own error format → inconsistent contract for clients.
- Exception messages leak internals.

### After: Canonical Pipeline, Centralized Concerns

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddAuthentication("Bearer").AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddCors(o => o.AddPolicy("PartnerPortal",
    p => p.WithOrigins("https://partner.contoso.com").AllowAnyHeader().AllowAnyMethod()));
builder.Services.AddRateLimiter(_ => { /* policies */ });
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler();
app.UseHsts();
app.UseHttpsRedirection();
app.UseCorrelationId();
app.UseRouting();
app.UseCors("PartnerPortal");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers();
app.MapHealthChecks("/health/ready");

app.Run();

// Controller — no try/catch, no manual error mapping
[ApiController]
[Route("api/orders")]
[Authorize]
public class OrdersController(IOrderService orders) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest req, CancellationToken ct)
    {
        var id = await orders.PlaceAsync(req, ct);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDto>> GetById(Guid id, CancellationToken ct)
        => await orders.GetAsync(id, ct) is { } o ? Ok(o) : NotFound();
}
```

Now the pipeline owns: HTTPS, HSTS, CORS, AuthN, AuthZ, rate limiting, correlation ID, and exception → Problem Details translation. The controller owns only the use case.

## Interview Questions and Answers

### 1. Walk me through the order you'd wire middleware in `Program.cs` for a production API and why.

**Why this matters**: Production outages routinely come from incorrect middleware ordering. The interviewer is checking whether you know the order and can justify it.

**Answer**: I follow Microsoft's canonical order: `UseExceptionHandler → UseHsts → UseHttpsRedirection → UseStaticFiles → UseRouting → UseCors → UseAuthentication → UseAuthorization → UseRateLimiter → MapControllers`. Exception handling goes first so it catches everything below it. HSTS + HTTPS redirect protect every request. `UseRouting` must come before `UseAuthorization` because endpoint-level policies (`[Authorize(Policy = "X")]`) are only known after routing matches. `UseCors` sits between routing and auth so preflight OPTIONS requests succeed for partners. Auth before AuthZ because AuthZ reads `HttpContext.User`. Rate limiter just before endpoints so it can still see the matched endpoint's policy.

**Trade-off**: Some teams put `UseCors` before `UseRouting` — that works for global policies but breaks per-endpoint CORS attributes. I prefer the canonical order unless there's a specific reason.

**Real project**: On a payments API, swapping the order so `UseAuthentication` ran after `UseAuthorization` shipped to staging and every `[Authorize]` endpoint returned 401. Caught in integration tests on the build pipeline because we have `WebApplicationFactory` tests that assert specific endpoints return 200 with a valid JWT.

### 2. Convention-based middleware vs `IMiddleware` — when do you pick which?

**Why this matters**: Picking convention-based middleware that needs scoped services is one of the most common captive-dependency bugs in ASP.NET Core.

**Answer**: Convention-based (a class with a `RequestDelegate` constructor and `InvokeAsync`) is effectively a singleton — the constructor runs once. Anything injected into the constructor is captured for the app's lifetime. I use it when the middleware only needs singletons (a logger, an `IOptionsMonitor<T>`).

`IMiddleware` is registered in DI with whatever lifetime you want (usually scoped). Use it when the middleware needs scoped dependencies like `DbContext`, `ITenantContext`, or a request-scoped audit service. The alternative is to keep the convention-based pattern but inject scoped services into `InvokeAsync` parameters instead of the constructor.

**Trade-off**: `IMiddleware` requires an explicit DI registration (one extra line) and is slightly slower (per-request resolution). Convention-based is leaner but more error-prone.

**Real project**: A tenant-resolution middleware that looked up the tenant from EF Core was originally convention-based — it captured `AppDbContext` and started returning stale data after the first request per host. Rewriting it as `IMiddleware` registered as scoped fixed it.

### 3. How do you propagate a correlation ID through the pipeline and into logs?

**Why this matters**: Distributed tracing is non-negotiable in microservices. Interviewers want to know you've actually wired this up, not just read about it.

**Answer**: I add a `CorrelationIdMiddleware` early in the pipeline (right after `UseExceptionHandler`). It reads `X-Correlation-Id` from the request or generates a new GUID, writes it back to the response, and opens an `ILogger` scope with the value:

```csharp
using (logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = id }))
{
    await _next(context);
}
```

With Serilog or Application Insights, every log line inside the scope automatically carries the correlation ID. Outbound `HttpClient` calls add the same header via a `DelegatingHandler` so the ID flows to downstream services. In .NET 6+ I also rely on `Activity.Current` and W3C `traceparent` for end-to-end tracing in Application Insights.

**Trade-off**: Custom correlation IDs duplicate `traceparent` somewhat. I keep both because partner systems that don't speak W3C still expect `X-Correlation-Id`.

**Real project**: A SaaS Orders API + Payments worker on Azure Service Bus. Same correlation ID flows from the partner's HTTP call → Orders → message → Payments worker → Application Insights, making end-to-end debugging a one-query job.

### 4. A middleware reads `Request.Body` and the controller now sees an empty body. What happened and how do you fix it?

**Why this matters**: Tests whether the candidate understands that HTTP request bodies are streams and are consumed once.

**Answer**: Request bodies are forward-only `Stream`s — once you read to the end, the controller's model binder finds nothing. The fix is to enable buffering and rewind:

```csharp
ctx.Request.EnableBuffering();
using var reader = new StreamReader(ctx.Request.Body, leaveOpen: true);
var body = await reader.ReadToEndAsync();
ctx.Request.Body.Position = 0;          // rewind so the next middleware can read it
await next(ctx);
```

`EnableBuffering` switches the body to a `FileBufferingReadStream` that supports seeking.

**Trade-off**: Buffering the body costs memory (or temp disk for large bodies). For request logging, prefer logging metadata (path, method, content-length) instead of the full body unless you really need it.

**Real project**: A request-logging middleware on an Orders API silently dropped 50% of POST bodies because it forgot to rewind — debugging took half a day until someone reproduced it locally.

### 5. Where should you put global exception handling, and how does `IExceptionHandler` change the picture in .NET 8?

**Why this matters**: Many teams still wrap every controller method in try/catch. Senior candidates should know the modern alternative.

**Answer**: Global exception handling belongs in middleware via `UseExceptionHandler`, placed at the very top of the pipeline. In .NET 8 you implement `IExceptionHandler` for each exception family (validation, not-found, conflict, infrastructure) and register them:

```csharp
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

Each handler returns `true` if it handled the exception, otherwise the next handler tries. The final fallback returns a generic `500` ProblemDetails. Controllers no longer need try/catch.

**Trade-off**: Centralized handling makes per-endpoint context less obvious. Mitigate with structured logging that includes the endpoint route.

**Real project**: Consolidating six controllers' try/catch blocks into three `IExceptionHandler` implementations cut ~150 lines of boilerplate and standardized the error shape returned to mobile clients.

### 6. When would you reach for an endpoint filter instead of middleware?

**Why this matters**: Endpoint filters are newer (.NET 7+) and many teams default to middleware out of habit.

**Answer**: Middleware applies to every request that reaches it (or to a `Map`/`UseWhen` branch). Endpoint filters apply only to the endpoints you attach them to and run **after** routing — so they know the matched endpoint, route values, and metadata. Use endpoint filters when:

- The behavior is per-endpoint (e.g., validating a specific request type with FluentValidation).
- You only need it on minimal APIs (filters chain cleanly with `AddEndpointFilter`).
- You want compile-time grouping via `MapGroup(...).AddEndpointFilter(...)`.

Use middleware when the behavior is truly global (HTTPS redirect, exception handling, correlation IDs).

**Trade-off**: Endpoint filters don't exist for MVC controllers (those use action filters instead). Mixing both styles in one app costs readability.

**Real project**: A minimal API for an internal admin tool used a single `RequireApiKey` endpoint filter on a `MapGroup("/admin")` — much cleaner than branching the pipeline with `MapWhen`.

### 7. How do you test that middleware ordering is correct without booting Kestrel manually?

**Answer**: I use `WebApplicationFactory<TEntryPoint>` from `Microsoft.AspNetCore.Mvc.Testing`. It boots the full pipeline in-process and gives me an `HttpClient` that goes through Kestrel-equivalent code:

```csharp
public class PipelineOrderTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Anonymous_request_to_protected_endpoint_is_401()
    {
        var response = await _client.GetAsync("/api/orders/123");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
}
```

I cover at minimum: anonymous request to a `[Authorize]` endpoint (asserts AuthN/AuthZ wired), a valid JWT call (asserts the same), a request that triggers an exception (asserts ProblemDetails shape), and a health check call (asserts the terminal endpoint).

**Trade-off**: `WebApplicationFactory` is slower than unit tests (~hundreds of ms per test class). Keep it focused on integration concerns.

**Real project**: A nightly contract test suite of 30 such tests catches middleware regressions before they hit staging — including the time someone re-ordered `UseRouting` and broke per-endpoint policies.

### 8. You see `Headers are read-only, response has already started` in logs. What does it mean and how do you avoid it?

**Why this matters**: This is a classic post-`next` write bug that confuses everyone the first time.

**Answer**: ASP.NET Core sends response headers the moment the first byte of the body is written (or `Response.StartAsync` is called). After that, modifying `Response.Headers` throws. The bug is almost always a middleware that does `await next(ctx); ctx.Response.Headers["X-Whatever"] = ...;` — the endpoint already wrote the body.

The fix is `HttpContext.Response.OnStarting`, which registers a callback that runs **just before** headers flush:

```csharp
ctx.Response.OnStarting(() =>
{
    ctx.Response.Headers["X-Trace-Id"] = traceId;
    return Task.CompletedTask;
});
await next(ctx);
```

**Trade-off**: `OnStarting` callbacks run synchronously inside the response-start path; keep them tiny.

**Real project**: A response-time middleware that wrote `X-Elapsed-Ms` after `await next` worked fine in dev (small JSON responses, headers not flushed yet) and failed in production where streamed responses had already started. Switched to `OnStarting` and the bug disappeared.

## Summary Checklist

- [ ] I can list the canonical middleware order and justify each position.
- [ ] I can write a custom middleware as a class and register it with `UseMiddleware<T>`.
- [ ] I can choose between convention-based middleware and `IMiddleware` based on dependency lifetimes.
- [ ] I can explain why `UseRouting` must run before `UseAuthorization`.
- [ ] I can wire a correlation ID through middleware, logging scopes, and outbound HTTP calls.
- [ ] I can describe when to use endpoint filters instead of middleware.
- [ ] I can fix the "request body already read" and "headers are read-only" bugs.
- [ ] I can configure `UseExceptionHandler` + `IExceptionHandler` to produce Problem Details.
- [ ] I can integration-test the pipeline with `WebApplicationFactory<Program>`.
- [ ] I can adapt the same pipeline pattern to Azure Functions, microservices, and multi-tenant SaaS.
