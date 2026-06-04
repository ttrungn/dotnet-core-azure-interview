# Error Handling and Problem Details

## What It Is

Error handling is the discipline of converting failures — invalid input, business rule violations, infrastructure outages, unhandled exceptions — into **predictable HTTP responses** that clients can parse and act on. **Problem Details** ([RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) / updated [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) is the standard JSON shape for those error responses, with fields `type`, `title`, `status`, `detail`, `instance`, plus extensions.

```csharp
// Bad — leaks internals, no shape, no contract.
catch (Exception ex)
{
    return new ContentResult { Content = ex.ToString(), StatusCode = 500 };
}

// Good — consistent, parseable, client-friendly.
return Problem(
    title: "Card declined",
    detail: "Issuer rejected the capture",
    statusCode: StatusCodes.Status422UnprocessableEntity,
    type: "https://api.contoso.com/errors/card-declined");
```

## Why It Exists

Before Problem Details, every API invented its own error envelope: `{ "error": "..." }`, `{ "errors": [...] }`, `{ "success": false, "msg": "..." }`. Mobile clients had to write per-endpoint parsers. RFC 7807 standardized the shape so any HTTP client can show the right error UI based on `status` and `title` without per-endpoint custom code. ASP.NET Core added `ProblemDetails`, `AddProblemDetails()`, and (in .NET 8) `IExceptionHandler` to make this the default.

## When To Use It

**Always return `ProblemDetails` for:**
- Every `4xx` and `5xx` response from a JSON HTTP API.
- Validation errors (`ValidationProblemDetails` — extends with `errors` dict).
- Business-rule violations (`409`, `422`).
- Unhandled exceptions (via `IExceptionHandler` or `UseExceptionHandler`).

**Do not use `ProblemDetails` for:**
- `2xx` responses — those have their own shape.
- HTML browser pages — use `app.UseExceptionHandler("/error")` view instead.
- Non-HTTP transports (gRPC has its own `Status`, Service Bus uses dead-letter properties).

## Why It Is Important

A consistent error contract is a **production reliability multiplier**:

- **Client teams** stop writing per-endpoint error parsers; mobile crash rates drop.
- **API gateway / front-door** can act on `status` (retry `5xx`, throttle by `type`).
- **Observability** — `type` URIs become alerting categories ("alert when `card-declined` rate > 5%").
- **Security** — a single funnel ensures no stack traces, SQL fragments, or PII ever leak.
- **Audit / compliance** — every failure produces a structured record with `instance` (request ID) tying logs to user complaints.

## How It's Used in C# / .NET

| Feature | API | Since |
| --- | --- | --- |
| `ProblemDetails` type | `Microsoft.AspNetCore.Mvc.ProblemDetails` | .NET Core 2.1 |
| Auto 400 on model errors | `[ApiController]` + `ValidationProblemDetails` | .NET Core 2.1 |
| Service registration | `builder.Services.AddProblemDetails()` | .NET 7 |
| Global exception middleware | `app.UseExceptionHandler()` | .NET Core 2.1 |
| Status-code pages | `app.UseStatusCodePages()` | .NET Core 2.1 |
| `IExceptionHandler` (clean DI-based) | `Microsoft.AspNetCore.Diagnostics.IExceptionHandler` | **.NET 8** |
| `IProblemDetailsService` (customize shape) | `Microsoft.AspNetCore.Http.IProblemDetailsService` | .NET 7 |
| Minimal API helpers | `Results.Problem(...)`, `TypedResults.Problem(...)` | .NET 6/7 |

### Minimal Wire-Up (.NET 8+)

```csharp
// Program.cs
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Instance = ctx.HttpContext.Request.Path;
        ctx.ProblemDetails.Extensions["traceId"] = Activity.Current?.Id
            ?? ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["requestId"] = ctx.HttpContext.TraceIdentifier;
    };
});

builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddExceptionHandler<FallbackExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler();   // delegates to registered IExceptionHandlers
app.UseStatusCodePages();    // converts bare 404/405 into ProblemDetails too
```

### IExceptionHandler — .NET 8 Style

```csharp
public sealed class DomainExceptionHandler(IProblemDetailsService problems)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (status, title, type) = exception switch
        {
            OrderNotFoundException        => (404, "Order not found", "https://api.contoso.com/errors/order-not-found"),
            DuplicateIdempotencyException => (409, "Duplicate request", "https://api.contoso.com/errors/duplicate-request"),
            CardDeclinedException         => (422, "Card declined",   "https://api.contoso.com/errors/card-declined"),
            ConcurrencyConflictException  => (412, "Version conflict", "https://api.contoso.com/errors/version-conflict"),
            _ => (0, null!, null!)
        };

        if (status == 0) return false; // let next handler try

        httpContext.Response.StatusCode = status;
        return await problems.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = new ProblemDetails
            {
                Type = type,
                Title = title,
                Status = status,
                Detail = exception.Message
            },
            Exception = exception
        });
    }
}
```

### Validation Problem Details

`[ApiController]` automatically produces this for model-binding/`[Required]` failures:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Amount":   ["The Amount field must be greater than 0."],
    "Currency": ["The Currency field is required."]
  },
  "traceId": "00-abc..."
}
```

Or build it manually:

```csharp
var errors = new Dictionary<string, string[]>
{
    ["Amount"]   = ["Must be greater than 0"],
    ["Currency"] = ["Must be a valid ISO-4217 code"]
};
return ValidationProblem(new ValidationProblemDetails(errors)
{
    Status = StatusCodes.Status400BadRequest,
    Title  = "Invalid order payload"
});
```

### Exception → Status Code Mapping Table

| Exception | Status | `type` slug |
| --- | --- | --- |
| `ValidationException` | 400 | `validation-failed` |
| `UnauthorizedAccessException` | 401 | `unauthorized` |
| `ForbiddenException` | 403 | `forbidden` |
| `NotFoundException` / `KeyNotFoundException` | 404 | `not-found` |
| `MethodNotAllowedException` | 405 | `method-not-allowed` |
| `ConcurrencyConflictException` | 412 | `version-conflict` |
| `DuplicateIdempotencyException` | 409 | `duplicate-request` |
| `BusinessRuleException` | 422 | `business-rule-violation` |
| `RateLimitExceededException` | 429 | `rate-limit-exceeded` |
| Anything else | 500 | `internal-error` |

## Advantages

- **One contract everywhere** — clients have one error parser.
- **Built into .NET** — no third-party library; `AddProblemDetails()` is one line.
- **Extensible** — add `traceId`, `correlationId`, `errors`, `retryAfter` via `Extensions`.
- **Discoverable** — `type` URIs can link to docs explaining the error.
- **Test-friendly** — assert on `ProblemDetails.Status` and `Title`, not free-text scraping.

## Disadvantages

- **Verbose for trivial errors** — five fields when a string would do.
- **`type` URIs require discipline** — teams often leave them blank, losing half the value.
- **Schema drift** — adding `Extensions` keys without coordination breaks clients.
- **Browser unfriendly** — a human-friendly error page still needs separate handling.
- **Not for gRPC** — different transport, different convention.

## Common Mistakes

### 1. Leaking exception details in `detail`

```csharp
// Bad — stack trace, SQL fragments, internal types reach the client
return Problem(detail: ex.ToString(), statusCode: 500);
```

```csharp
// Fix — log internally, return a safe message
_logger.LogError(ex, "Capture failed {InvoiceId}", invoiceId);
return Problem(
    title: "Unexpected error",
    detail: "An unexpected error occurred. Reference: " + HttpContext.TraceIdentifier,
    statusCode: StatusCodes.Status500InternalServerError);
```

### 2. Inventing a custom error shape next to ProblemDetails

```csharp
// Bad — two formats; clients now need two parsers
return BadRequest(new { code = "INVALID", msg = "Bad amount" });
```

```csharp
// Fix — always ProblemDetails (or ValidationProblemDetails)
return ValidationProblem(new ValidationProblemDetails(new Dictionary<string, string[]>
{
    ["Amount"] = ["Must be greater than 0"]
}));
```

### 3. Returning 500 for client mistakes

```csharp
// Bad — Polly will retry; alerts will fire
if (string.IsNullOrEmpty(req.Currency))
    throw new Exception("Currency missing"); // becomes 500
```

```csharp
// Fix — 400 for shape, 422 for rules
if (string.IsNullOrEmpty(req.Currency))
    return Problem(title: "Currency required", statusCode: 400);
```

### 4. Catch-all middleware that swallows status codes

```csharp
// Bad — overrides every response, even legitimate 404
app.Use(async (ctx, next) =>
{
    try { await next(); }
    catch { ctx.Response.StatusCode = 500; }
});
```

```csharp
// Fix — let UseExceptionHandler + IExceptionHandler do the work
app.UseExceptionHandler();
app.UseStatusCodePages();
```

### 5. Forgetting the `traceId` extension

```csharp
// Bad — client gets a 500 with no way to correlate to logs
return Problem(title: "Server error", statusCode: 500);
```

```csharp
// Fix — always include traceId so support can find the log entry
ctx.ProblemDetails.Extensions["traceId"] = Activity.Current?.Id
    ?? ctx.HttpContext.TraceIdentifier;
```

### 6. Throwing exceptions for expected business outcomes

```csharp
// Bad — exceptions are expensive and noisy in logs
public Order Get(Guid id)
    => _db.Orders.Find(id) ?? throw new NotFoundException();
```

```csharp
// Fix — Result/Option pattern for expected absence; reserve exceptions for unexpected
public Order? Find(Guid id) => _db.Orders.Find(id);
// Controller decides: return NotFound() or Problem(404, ...).
```

## Best Practices

- Call `AddProblemDetails()` once in `Program.cs`; let the framework do the work.
- Use **`IExceptionHandler`** (.NET 8+) over old `IExceptionFilter` or custom middleware.
- Define **stable `type` URIs** (`https://api.contoso.com/errors/{slug}`) and document each in your OpenAPI/Swagger.
- Always attach **`traceId`** as a `ProblemDetails.Extensions["traceId"]`.
- Map exceptions to status codes **at the boundary**, not deep in the domain.
- Validate input with **FluentValidation** or `[Required]` → automatic `ValidationProblemDetails`.
- For `429` and `503`, include a `retryAfter` extension to mirror the HTTP header.
- **Log the exception** (`ILogger.LogError(ex, ...)`) before mapping — the response is sanitized; the log is not.
- Unit-test the mapping: send a known exception, assert `status == 422` and `type` URI.
- Keep **internal error codes** out of `title`; use `type` URIs or an `errorCode` extension.

## Related Concepts

- [aspnet-core/http-methods-and-status-codes.md](http-methods-and-status-codes.md)
- [aspnet-core/input-validation.md](input-validation.md)
- [aspnet-core/request-pipeline.md](request-pipeline.md)
- [aspnet-core/logging-and-monitoring.md](logging-and-monitoring.md)
- [csharp/exception-handling.md](../csharp/exception-handling.md)
- [architecture/reliability-design.md](../architecture/reliability-design.md)

## Real-World Usage

### ASP.NET Core APIs

Modern API templates (`dotnet new webapi`) wire `AddProblemDetails()` and `UseExceptionHandler()` by default in .NET 8+. Add one or two `IExceptionHandler` implementations and you're done.

### Azure Functions

HTTP-triggered Functions do not get `[ApiController]` magic — manually return `new ObjectResult(new ProblemDetails { ... }) { StatusCode = 422 }` or use the `Results.Problem` minimal helpers when running the ASP.NET Core integration model.

### Azure API Management

APIM policies can transform backend errors into a tenant-specific `ProblemDetails` shape (e.g., redact `type` URIs that expose internal slugs). Useful when fronting multiple services with different error conventions.

### Application Insights

Set `ProblemDetails.Extensions["traceId"]` to `Activity.Current?.Id`. Support engineers can paste that ID into App Insights → find the full trace, exception, dependency calls. See [aspnet-core/logging-and-monitoring.md](logging-and-monitoring.md).

### Multi-Tenant SaaS

Add `tenantId` and `region` to `ProblemDetails.Extensions` so per-tenant error dashboards can group failures. Never expose other tenants' identifiers.

### Public Partner APIs

Document each `type` URI in OpenAPI with a stable `operationId`. Partners write code against the URI, not the human-readable `title`, which you may localize.

## Code Example — Before and After

### Before — ad-hoc errors, leaked internals, inconsistent shapes

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly IOrderService _orders;

    public OrdersController(IOrderService orders) => _orders = orders;

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest req)
    {
        try
        {
            if (req.Lines == null || req.Lines.Count == 0)
                return BadRequest("No lines");

            var id = await _orders.CreateAsync(req);
            return Ok(new { id });
        }
        catch (DuplicateOrderException ex)
        {
            return StatusCode(409, new { error = ex.Message });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { error = ex.ToString() }); // leaks stack
        }
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var order = await _orders.FindAsync(id);
        if (order == null) return NotFound("Order not found"); // plain text
        return Ok(order);
    }
}
```

Problems: three different error shapes (`string`, `{ error }`, raw 404 string), stack traces leak, no `traceId`, no `type` URI, no validation problem details. Mobile team writes per-endpoint parsers.

### After — single contract, exception handler, validation, traceability

```csharp
// ----- Program.cs -----
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(o =>
    {
        o.InvalidModelStateResponseFactory = ctx =>
        {
            var details = new ValidationProblemDetails(ctx.ModelState)
            {
                Type   = "https://api.contoso.com/errors/validation-failed",
                Title  = "One or more validation errors occurred",
                Status = StatusCodes.Status400BadRequest,
                Instance = ctx.HttpContext.Request.Path
            };
            details.Extensions["traceId"] = Activity.Current?.Id
                ?? ctx.HttpContext.TraceIdentifier;
            return new BadRequestObjectResult(details);
        };
    });

builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Instance = ctx.HttpContext.Request.Path;
        ctx.ProblemDetails.Extensions["traceId"] = Activity.Current?.Id
            ?? ctx.HttpContext.TraceIdentifier;
    };
});

builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddExceptionHandler<FallbackExceptionHandler>();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapControllers();
app.Run();


// ----- Handlers -----
namespace Contoso.Orders.Api.Errors;

public sealed class DomainExceptionHandler(
    IProblemDetailsService problems,
    ILogger<DomainExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken ct)
    {
        var (status, title, type) = exception switch
        {
            OrderNotFoundException         => (404, "Order not found",
                "https://api.contoso.com/errors/order-not-found"),
            DuplicateOrderException        => (409, "Duplicate order",
                "https://api.contoso.com/errors/duplicate-order"),
            BusinessRuleException br       => (422, br.RuleTitle,
                $"https://api.contoso.com/errors/{br.RuleSlug}"),
            ConcurrencyConflictException   => (412, "Version conflict",
                "https://api.contoso.com/errors/version-conflict"),
            _ => (0, null!, null!)
        };

        if (status == 0) return false;

        logger.LogWarning(exception, "Handled domain exception → {Status} {Title}", status, title);

        httpContext.Response.StatusCode = status;
        return await problems.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext    = httpContext,
            ProblemDetails = new ProblemDetails
            {
                Type   = type,
                Title  = title,
                Status = status,
                Detail = exception.Message
            },
            Exception      = exception
        });
    }
}

public sealed class FallbackExceptionHandler(
    IProblemDetailsService problems,
    ILogger<FallbackExceptionHandler> logger,
    IHostEnvironment env) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken ct)
    {
        logger.LogError(exception, "Unhandled exception for {Path}", httpContext.Request.Path);

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        return await problems.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext    = httpContext,
            ProblemDetails = new ProblemDetails
            {
                Type   = "https://api.contoso.com/errors/internal-error",
                Title  = "An unexpected error occurred",
                Status = StatusCodes.Status500InternalServerError,
                Detail = env.IsDevelopment() ? exception.Message : null
            },
            Exception      = exception
        });
    }
}


// ----- Controller — now lean -----
namespace Contoso.Orders.Api.Controllers;

[ApiController]
[Route("api/v1/orders")]
public sealed class OrdersController(IOrderService orders) : ControllerBase
{
    [HttpPost]
    [ProducesResponseType(typeof(OrderCreatedResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status422UnprocessableEntity)]
    public async Task<IActionResult> Create(
        CreateOrderRequest request, CancellationToken cancellationToken)
    {
        // Validation handled by [ApiController] → ValidationProblemDetails.
        // Domain exceptions caught by DomainExceptionHandler.
        var id = await orders.CreateAsync(request, cancellationToken);
        return CreatedAtAction(nameof(Get), new { id }, new OrderCreatedResponse(id));
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDetailsDto), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDetailsDto>> Get(
        Guid id, CancellationToken cancellationToken)
    {
        var order = await orders.FindAsync(id, cancellationToken);
        return order is null
            ? throw new OrderNotFoundException(id) // mapped to 404 by handler
            : Ok(order);
    }
}
```

Now: one consistent shape, validation auto-formatted, exceptions mapped centrally, `traceId` on every error, internal details only in `Development`.

## Interview Questions and Answers

### 1. What is RFC 7807 and why use it?

**Why this matters:** Tests knowledge of API standards rather than reinventing wheels.

**Answer:** RFC 7807 (updated by RFC 9457) defines the `application/problem+json` media type and the `ProblemDetails` JSON shape — `type`, `title`, `status`, `detail`, `instance`. It exists so any HTTP client can show a meaningful error UI from any RFC-compliant API without per-endpoint parsers. ASP.NET Core's `ProblemDetails` and `AddProblemDetails()` are first-class.

**Trade-off:** Adopting it requires retrofitting existing endpoints; partner APIs may need a versioned migration.

**Real project:** Migrating a B2B API to RFC 7807 reduced partner support tickets ~40% because errors became self-explanatory and linkable to docs via `type` URI.

### 2. What's the difference between `IExceptionHandler` and `IExceptionFilter`?

**Why this matters:** .NET 8 introduced `IExceptionHandler`; knowing why matters.

**Answer:** `IExceptionFilter` is MVC-specific, runs only for controller exceptions, after model binding. `IExceptionHandler` (.NET 8+) is middleware-level — it catches anything `UseExceptionHandler()` sees, including Minimal APIs, static files, and authentication failures. It's also DI-friendly, async, and can chain multiple handlers (first one to return `true` wins). Prefer `IExceptionHandler` for new code.

### 3. How do you map domain exceptions to HTTP status codes without polluting the domain layer?

**Why this matters:** Clean Architecture purists worry about leaking HTTP into the domain.

**Answer:** Define domain exceptions (`OrderNotFoundException`, `BusinessRuleException`) with no HTTP knowledge. The mapping lives in an `IExceptionHandler` at the API layer — a `switch` expression converting type to status. The domain stays transport-agnostic; the API decides the HTTP contract.

**Trade-off:** Throwing for expected outcomes (like "not found") is debated. Result/Option types are cleaner but require all callers to handle them.

### 4. How should validation errors differ from business-rule errors?

**Why this matters:** Same `4xx` family, different semantics.

**Answer:** **Validation** = "your input is structurally wrong" — missing field, wrong type, negative amount. Returns `400` with `ValidationProblemDetails` and a per-field `errors` dict. **Business rule** = "input is syntactically valid but violates a domain rule" — order total below minimum, cannot ship to embargoed country. Returns `422` with `ProblemDetails` and a clear `type` URI. The client treats them differently: validation maps to form-field highlights; business rule maps to a toast notification.

### 5. Why include `traceId` in every ProblemDetails response?

**Why this matters:** Operational efficiency — support tickets become resolvable.

**Answer:** The user copies "Reference: 00-abc..." from the error UI into a support ticket; the engineer pastes it into App Insights or Datadog and lands directly on the exception, dependencies, and surrounding logs. Without `traceId`, troubleshooting starts with "what time roughly?" and ends in despair. Use `Activity.Current?.Id` (W3C TraceContext) — see [aspnet-core/logging-and-monitoring.md](logging-and-monitoring.md).

**Real project:** Adding `traceId` to ProblemDetails cut average ticket resolution time from 90 min to 12 min.

### 6. When would you bypass ProblemDetails and return a raw response?

**Why this matters:** Senior judgment — knowing when not to follow the rule.

**Answer:** Three cases. **(1)** Browser-facing HTML errors — return an HTML view, not JSON. **(2)** Webhook acks where the sender (GitHub, Stripe) ignores bodies and only checks status. **(3)** File-download endpoints where the client expects bytes — return `404` with empty body rather than JSON. Outside these, always ProblemDetails.

### 7. How do you safely include details for developers but not for end users?

**Why this matters:** A common security mistake.

**Answer:** Use `IHostEnvironment.IsDevelopment()` in the fallback handler: only include `exception.Message` in `detail` in dev. In staging/prod, return generic `"An unexpected error occurred"` + `traceId`. The full exception always goes to `ILogger.LogError(ex, ...)` so engineers have it in App Insights. Never use `exception.ToString()` in any response — stack traces reveal internal classes, file paths, dependency versions.

### 8. How do you evolve the ProblemDetails shape without breaking clients?

**Why this matters:** Backward compatibility for a shared error contract.

**Answer:** ProblemDetails is **extensible by design** — clients are required to ignore unknown fields. Add new keys to `Extensions` (`retryAfter`, `errorCode`, `correlationId`) freely. Never remove or rename existing keys; never repurpose `type` URIs — they are contracts. Document each `type` URI in OpenAPI. For breaking changes (`status` changes, `type` removal), version the API (`/api/v2/...`) — see [aspnet-core/api-versioning.md](api-versioning.md).

**Real project:** A team renamed `type` from `card-declined` to `payment-declined`; mobile apps with switch-on-type logic broke. The fix was to keep both URIs aliased for two release cycles, then deprecate.

## Summary Checklist

- [ ] I call `AddProblemDetails()` and `UseExceptionHandler()` in `Program.cs`.
- [ ] I implement `IExceptionHandler` (.NET 8+) to map domain exceptions to status codes.
- [ ] I return `ValidationProblemDetails` (`400`) for input errors and `ProblemDetails` (`422`) for rule violations.
- [ ] I attach `traceId` from `Activity.Current?.Id` on every error response.
- [ ] I define stable `type` URIs (`https://api.contoso.com/errors/...`) and document them in OpenAPI.
- [ ] I never leak stack traces, SQL, or PII in `detail`.
- [ ] I always `LogError(ex, ...)` before sanitizing the response.
- [ ] I declare `[ProducesResponseType(typeof(ProblemDetails), ...)]` for every error path.
- [ ] I evolve the shape additively via `Extensions`; I version the API for breaking changes.
- [ ] I unit-test the exception-to-status mapping.
