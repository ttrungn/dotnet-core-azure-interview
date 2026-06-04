# Exception Handling

## What It Is

Exception handling is the mechanism C# uses to propagate, catch, and translate unexpected failures. An **exception** is an object (deriving from `System.Exception`) thrown at the point a method cannot complete its contract. The runtime unwinds the call stack, runs `finally` blocks for cleanup, and looks for a matching `catch` block — eventually reaching the host process if nothing handles it.

In a backend service, exception handling is not "wrap everything in try/catch." It is a **layered policy**:

- **Domain layer** throws specific exceptions (`PaymentDeclinedException`, `InsufficientStockException`) when business invariants are violated.
- **Application layer** may catch and translate to a result, or let them flow.
- **API edge** (ASP.NET Core exception middleware) translates uncaught exceptions to safe HTTP responses (RFC 7807 `ProblemDetails`).
- **Infrastructure adapters** wrap provider-specific exceptions (`SqlException`, `RequestFailedException` from Azure SDK) into domain-meaningful ones.

```csharp
// Throw at the boundary of "this cannot continue"
if (order.Status != OrderStatus.Pending)
    throw new InvalidOrderStateException(order.Id, order.Status);

// Catch where you can act — middleware, retry policy, message handler
try { await _payments.AuthorizeAsync(order.Id, order.Total, ct); }
catch (PaymentDeclinedException ex)
{
    _logger.LogWarning(ex, "Payment declined for {OrderId}", order.Id);
    return Problem("Payment declined", statusCode: 402);
}
```

## Why It Exists

Before structured exceptions (and before .NET), failure was communicated via **return codes** (`HRESULT`, `errno`). Every call required `if (result != 0) { ... }` boilerplate. Programmers forgot to check, errors were silently swallowed, and stack context was lost. Wrapping low-level failures up to a meaningful business message meant manual propagation through every layer.

Exceptions solved this by making failure a **first-class** control flow:

- The compiler does not let you forget — uncaught exceptions tear down the call stack.
- Each frame can add context (`throw new InvoiceGenerationException(invoiceId, ex)`).
- Cleanup runs deterministically in `finally` / `using`.
- The original stack trace is preserved (with `throw;`).

The trade-off the .NET team made explicit: **exceptions are for exceptional situations**, not for normal validation flow. Throwing/catching has measurable cost (microseconds, plus a heap allocation, plus stack-walk). A request that validates 50 fields by throwing 50 times is doing it wrong.

## When To Use It

**Throw an exception when:**

- A method cannot fulfill its contract (`order is null` in a method that requires one).
- A business invariant is violated and the operation must abort (`Cannot ship an unpaid order`).
- An infrastructure call failed and the layer above must decide what to do (`SQL timeout`, `Stripe 5xx`).
- An argument is invalid in a way the caller could have avoided (`ArgumentNullException`).

**Do NOT use exceptions when:**

- The condition is expected and frequent — use a return value (`bool TryParse`, `Result<T>`, `Option<T>`).
- Input validation against user data — return `ValidationProblemDetails` (HTTP 400) instead.
- Controlling normal flow (`if (NotFound) throw; ... catch (NotFoundException) { return null; }` — just return null).

**Catch an exception when:**

- You can **recover** (retry with Polly, fall back to cache).
- You can **translate** it to a more meaningful exception or response (`SqlException → ConcurrencyConflictException`).
- You are at a **boundary** that must not crash (message handler, middleware, top-level background loop).

**Do NOT catch when:**

- You can't do anything useful — let it propagate to a layer that can.
- You would just log and rethrow at every layer — log **once**, at the boundary.

## Why It Is Important

In production:

- A swallowed `NullReferenceException` becomes a silent data-corruption bug noticed a week later in customer support tickets.
- An unhandled exception in an Azure Function or `BackgroundService` can crash the host process — instances flap, the queue backs up, SLAs break.
- A leaked stack trace in an HTTP response exposes internal class names, library versions, connection strings, and file paths. That is a real security finding in OWASP A05:2021 (Security Misconfiguration).
- Catching too broadly (`catch (Exception)`) hides bugs from telemetry. Application Insights' "failure rate" graph is only useful if exceptions actually reach it.
- A retry loop that re-throws transient `SqlException`s but treats `UniqueConstraintViolation` as transient will retry forever.

Good exception policy is one of the strongest signals of seniority in a code review — it shows whether the engineer understands the operational surface.

## How It's Used in C# / .NET

| Construct / API | Use |
| --- | --- |
| `throw new XxxException(...)` | Signal a failure at the call site. |
| `throw;` (inside catch) | Rethrow preserving the original stack trace. |
| `throw ex;` | **WRONG** — resets the stack trace. Almost never correct. |
| `ExceptionDispatchInfo.Capture(ex).Throw()` | Rethrow a captured exception from a different context (queues, scheduling). |
| `try { } catch (XxxException ex) when (filter)` | Exception filter — only catches if predicate is true. No stack-frame unwind on miss. |
| `try { } finally { }` | Guaranteed cleanup. Prefer `using` / `await using` for `IDisposable` / `IAsyncDisposable`. |
| `AggregateException.InnerExceptions` | From `Task.WhenAll` failures. |
| `app.UseExceptionHandler("/error")` | ASP.NET Core production exception middleware. |
| `app.UseDeveloperExceptionPage()` | Dev-only detailed page. **Never** in production. |
| `IExceptionHandler` (NET 8+) | Pluggable typed handler for the exception middleware. |
| `ProblemDetails` / `ValidationProblemDetails` | RFC 7807 standard error response shapes. |
| `Polly` retry/circuit-breaker | Handles transient exceptions declaratively. |
| `ILogger<T>.LogError(ex, "...")` | Structured logging with exception object. |

A typical exception-handling middleware:

```csharp
public sealed class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IHostEnvironment _env;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger,
        IHostEnvironment env)
    {
        _next = next; _logger = logger; _env = env;
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        try { await _next(ctx); }
        catch (NotFoundException ex)              { await Write(ctx, 404, "Not found", ex.Message); }
        catch (ValidationException ex)            { await Write(ctx, 400, "Validation failed", ex.Message); }
        catch (PaymentDeclinedException ex)       { await Write(ctx, 402, "Payment declined", ex.Reason); }
        catch (ConcurrencyConflictException ex)   { await Write(ctx, 409, "Conflict", ex.Message); }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception for {Path}", ctx.Request.Path);
            var detail = _env.IsDevelopment() ? ex.ToString() : "An unexpected error occurred.";
            await Write(ctx, 500, "Internal Server Error", detail);
        }
    }

    private static Task Write(HttpContext ctx, int status, string title, string detail)
    {
        ctx.Response.StatusCode = status;
        ctx.Response.ContentType = "application/problem+json";
        return ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status, Title = title, Detail = detail,
            Instance = ctx.Request.Path
        });
    }
}
```

## Advantages

- **Cannot be silently ignored** — uncaught exceptions tear down the call stack.
- **Stack trace + context** preserved for diagnostics.
- **Separation of concerns** — failure handling is centralized at boundaries, not sprinkled in every method.
- **Composable with Polly** for retry/circuit-breaker/timeout patterns.
- **Standard transport** — `ProblemDetails` plus exception middleware gives every endpoint consistent error shape.

## Disadvantages

- **Performance cost** — throwing is expensive (~10-100 µs + allocation + stack-walk). Not for hot paths or normal flow.
- **Hidden control flow** — readers may not realize a deep call can throw `DbUpdateConcurrencyException`.
- **Easy to abuse** — `catch (Exception)` hides bugs; `throw ex` destroys traces.
- **Cross-thread propagation is awkward** — fire-and-forget tasks swallow exceptions unless explicitly observed.
- **Stack traces are noisy** in async code (state machines).

## Common Mistakes

### 1. `throw ex;` — destroying the stack trace

```csharp
// WRONG — original stack frame is lost
catch (SqlException ex)
{
    _logger.LogError(ex, "DB failure");
    throw ex; // restart from here
}
```

```csharp
// RIGHT — preserve the original stack
catch (SqlException ex)
{
    _logger.LogError(ex, "DB failure");
    throw;
}
```

### 2. Catching `Exception` everywhere

```csharp
// WRONG — hides bugs, makes telemetry useless
try { await _repo.SaveAsync(order, ct); }
catch (Exception) { /* swallow */ }
```

```csharp
// RIGHT — catch specifically, or let it bubble
try { await _repo.SaveAsync(order, ct); }
catch (DbUpdateConcurrencyException ex)
{
    throw new ConcurrencyConflictException(order.Id, ex);
}
```

### 3. Using exceptions for normal validation flow

```csharp
// WRONG — throws once per missing field; expensive and unclear
foreach (var field in required)
    if (string.IsNullOrEmpty(request.Get(field)))
        throw new ArgumentException($"{field} is required");
```

```csharp
// RIGHT — collect errors, return a 400 ProblemDetails
var errors = required
    .Where(f => string.IsNullOrEmpty(request.Get(f)))
    .ToDictionary(f => f, f => new[] { "Required" });

if (errors.Count > 0) return ValidationProblem(new ValidationProblemDetails(errors));
```

### 4. Leaking stack traces in production

```csharp
// WRONG — returns full stack trace to the caller
catch (Exception ex)
{
    return StatusCode(500, ex.ToString());
}
```

```csharp
// RIGHT — log internally, return a safe ProblemDetails externally
catch (Exception ex)
{
    _logger.LogError(ex, "Unhandled error");
    return Problem("An unexpected error occurred.", statusCode: 500);
}
```

### 5. Logging the same exception at every layer

```csharp
// WRONG — three entries in App Insights for one failure
public async Task<Order> GetAsync(...)
{
    try { return await _repo.GetAsync(...); }
    catch (Exception ex) { _logger.LogError(ex, "repo"); throw; }
}
// caller also logs, controller also logs
```

```csharp
// RIGHT — log once at the boundary (middleware), let it propagate cleanly
public Task<Order> GetAsync(...) => _repo.GetAsync(...);
```

### 6. Catching `Exception` to suppress cancellation

```csharp
// WRONG — eats OperationCanceledException; the host can't tell shutdown is honored
try { await _http.GetAsync(url, ct); }
catch (Exception ex) { _logger.LogError(ex, "HTTP failed"); }
```

```csharp
// RIGHT — let cancellation flow through
try { await _http.GetAsync(url, ct); }
catch (OperationCanceledException) when (ct.IsCancellationRequested) { throw; }
catch (HttpRequestException ex) { _logger.LogError(ex, "HTTP failed"); throw; }
```

### 7. Forgetting `IAsyncDisposable` cleanup on exception

```csharp
// WRONG — if SQL throws, the ServiceBus client is never disposed
var bus = new ServiceBusClient(connString);
await _db.SaveChangesAsync(ct);
await bus.DisposeAsync();
```

```csharp
// RIGHT — await using guarantees disposal
await using var bus = new ServiceBusClient(connString);
await _db.SaveChangesAsync(ct);
```

## Best Practices

- **Throw specific exceptions** (`PaymentDeclinedException`), not `Exception` or `ApplicationException`.
- **Use exception filters** (`catch (X) when (...)`) instead of `catch + if + rethrow` — no extra stack-unwind on the miss path.
- **Catch at boundaries**: middleware for HTTP, `try/catch` around the body of a `BackgroundService` loop, around each Service Bus message handler.
- **Log once, with structured properties** (`LogError(ex, "Order {OrderId} failed", id)`).
- **Never return stack traces to API clients** in production.
- **Map exceptions to HTTP**: 400 (validation), 401 (auth), 403 (forbidden), 404 (not found), 409 (concurrency), 422 (semantic), 500 (unexpected).
- **Re-wrap infrastructure exceptions** at adapter boundaries so the domain doesn't depend on `SqlException`.
- **Always rethrow `OperationCanceledException`** when the user cancellation token is the cause.
- **Use Polly** for retries/circuit-breakers — do not roll your own loop.
- **Test the unhappy paths** as carefully as the happy path.

## Related Concepts

- [aspnet-core/error-handling-and-problem-details.md](aspnet-core/error-handling-and-problem-details.md) — `ProblemDetails`, `IExceptionHandler`, middleware setup.
- [aspnet-core/input-validation.md](aspnet-core/input-validation.md) — when to use validation results instead of exceptions.
- [csharp/async-await.md](csharp/async-await.md) — how exceptions propagate through `Task` and `Task.WhenAll`.
- [architecture/reliability-design.md](architecture/reliability-design.md) — Polly retry/circuit-breaker.
- [azure/application-insights.md](azure/application-insights.md) — capturing and querying exception telemetry.
- [testing-quality/unit-testing.md](testing-quality/unit-testing.md) — `Assert.Throws<T>` patterns.

## Real-World Usage

### ASP.NET Core APIs

`app.UseExceptionHandler()` (with a `/error` endpoint or an `IExceptionHandler` in .NET 8+) is the standard production boundary. Combine with `app.UseStatusCodePages()` for 404/401 responses. Never enable `UseDeveloperExceptionPage` outside `Development`.

### Azure Functions

The Functions host catches unhandled exceptions per invocation. For queue/Service Bus triggers, a thrown exception causes the message to be retried (5 times by default) and then dead-lettered. Use this — but make sure your handler is idempotent.

### Azure Service Bus consumers

In a `BackgroundService` driving `ServiceBusProcessor`, supply both `ProcessMessageAsync` (success path) and `ProcessErrorAsync` (broker errors). Inside the message handler, wrap business logic in try/catch; on application failure, call `AbandonMessageAsync` to release the lock for retry, or `DeadLetterMessageAsync` for poison messages.

### Polly resilience

```csharp
services.AddHttpClient<IPaymentGateway, StripePaymentGateway>()
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()                                   // 5xx, 408
        .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(3, attempt => TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt))));
```

### Application Insights

`LogError(ex, ...)` flows through `ApplicationInsightsLoggerProvider` and shows up in the `exceptions` table. Kusto query: `exceptions | where timestamp > ago(1h) | summarize count() by type, outerMessage`.

### Testing

```csharp
[Fact]
public async Task PlaceOrder_with_invalid_total_throws()
{
    var act = () => _service.PlaceAsync(new(){ Total = -1m }, default);
    var ex = await Assert.ThrowsAsync<InvalidOrderException>(act);
    ex.Field.Should().Be(nameof(Order.Total));
}
```

## Code Example — Before and After

### Before — swallows errors, leaks stack traces, mixes validation with exceptions

```csharp
[HttpPost]
public IActionResult Place([FromBody] PlaceOrderRequest req)
{
    try
    {
        if (req == null) throw new Exception("Body required");
        if (req.Items == null || req.Items.Count == 0)
            throw new Exception("Need items");

        var order = _service.Place(req);
        return Ok(order);
    }
    catch (Exception ex)
    {
        // Leaks full stack trace to caller
        return StatusCode(500, ex.ToString());
    }
}
```

Problems: throws for validation, `catch (Exception)` hides bugs, `ex.ToString()` exposes internals, no logging, sync only.

### After — boundary middleware, typed exceptions, validation as data

```csharp
// Domain exception
public sealed class InsufficientStockException : Exception
{
    public Guid ProductId { get; }
    public int Requested { get; }
    public int Available { get; }
    public InsufficientStockException(Guid productId, int requested, int available)
        : base($"Product {productId}: requested {requested}, available {available}")
    {
        ProductId = productId; Requested = requested; Available = available;
    }
}

// Controller — clean happy path; failures are typed
[ApiController, Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly IOrderService _service;
    private readonly IValidator<PlaceOrderRequest> _validator;

    public OrdersController(IOrderService service, IValidator<PlaceOrderRequest> validator)
    {
        _service = service; _validator = validator;
    }

    [HttpPost]
    public async Task<ActionResult<OrderDto>> PlaceAsync(
        [FromBody] PlaceOrderRequest req, CancellationToken ct)
    {
        var validation = await _validator.ValidateAsync(req, ct);
        if (!validation.IsValid)
            return ValidationProblem(validation.ToDictionary());

        var order = await _service.PlaceAsync(req, ct);
        return CreatedAtAction(nameof(GetAsync), new { id = order.Id }, order);
    }
}

// IExceptionHandler (.NET 8+) maps domain exceptions to ProblemDetails
public sealed class DomainExceptionHandler : IExceptionHandler
{
    private readonly ILogger<DomainExceptionHandler> _logger;
    public DomainExceptionHandler(ILogger<DomainExceptionHandler> l) => _logger = l;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var (status, title) = ex switch
        {
            InsufficientStockException     => (StatusCodes.Status409Conflict,  "Insufficient stock"),
            PaymentDeclinedException       => (StatusCodes.Status402PaymentRequired, "Payment declined"),
            ConcurrencyConflictException   => (StatusCodes.Status409Conflict,  "Concurrency conflict"),
            NotFoundException              => (StatusCodes.Status404NotFound,  "Not found"),
            _                              => (0, null!) // not handled, fall through to default handler
        };

        if (status == 0) return false;

        _logger.LogWarning(ex, "Handled domain exception {Type}", ex.GetType().Name);
        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status, Title = title, Detail = ex.Message, Instance = ctx.Request.Path
        }, cancellationToken: ct);
        return true;
    }
}

// Program.cs
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

The after version separates validation (returned as data) from exceptions (escalated to middleware). The controller is short and intent-revealing; the cross-cutting policy lives in one place.

## Interview Questions and Answers

### 1. When would you throw an exception vs. return a `Result<T>`?

**Why this matters:** Distinguishes principled exception use from "throw everything."

**Answer:** Throw when the situation is **exceptional** — a contract is broken, an invariant violated, infrastructure is failing. Return a `Result` (or specific response) when the failure is **expected and frequent** — user input validation, "item not found" for a search endpoint, "coupon expired." In a payment flow, `PaymentDeclinedException` is fine because it indicates rare provider-side rejection; "card number must be 16 digits" is validation and should be a 400 from `ValidationProblemDetails`.

**Trade-off:** Result types add ceremony at every call site. Exceptions add hidden control flow. Most .NET teams use exceptions for failures, results for input validation — a pragmatic mix.

**Real project:** A team converted all validation to exceptions for "consistency." Their p99 latency jumped because the high-traffic search endpoint was throwing/catching ~5 exceptions per request. Reverting to validation results dropped CPU 20%.

### 2. Why is `throw ex;` almost always wrong?

**Why this matters:** Subtle but very common bug.

**Answer:** `throw ex` resets the exception's stack-trace origin to the rethrow line. The original stack frame — where the problem actually occurred — is lost. Debugging from logs becomes guesswork. Use `throw;` (no operand) to preserve the original trace, or `throw new XxxException("context", ex)` to wrap while preserving the inner exception.

**Trade-off:** None. There is no scenario where `throw ex` is preferable to `throw;` except cross-thread captures where you should use `ExceptionDispatchInfo.Capture(ex).Throw()` instead.

**Real project:** Investigating an intermittent NullReferenceException, every log entry pointed at the rethrow line in a generic repository — useless. Fixing every `throw ex;` to `throw;` revealed the actual bug was 8 frames deeper.

### 3. How do you handle exceptions in a background `BackgroundService`?

**Why this matters:** A single unhandled exception in `ExecuteAsync` stops the service forever.

**Answer:** Wrap the body of the worker loop in try/catch. Catch `OperationCanceledException` separately (it's the normal stop signal). For other exceptions, log with context, optionally back off, and continue the loop. Never let `ExecuteAsync` itself throw — that terminates the hosted service silently in many configurations.

**Trade-off:** Catching too broadly inside the loop may mask a fatal misconfiguration (e.g., bad connection string) — combine with health checks and metrics so persistent failures are visible.

**Real project:** An invoice-generation worker silently stopped after one `SqlException` because the loop wasn't wrapped. We saw it three days later when finance asked where the invoices were. Now every worker has `try { await ProcessOneAsync(ct); } catch when (!ct.IsCancellationRequested) { log, delay, continue; }`.

### 4. A teammate writes `catch (Exception) { return null; }`. What do you say in code review?

**Why this matters:** Tests review skills around silent failures.

**Answer:** Three issues: (a) it hides every bug — `NullReferenceException`, OOM, configuration errors all turn into a benign-looking `null`. (b) Callers can't distinguish "not found" from "service is broken." (c) Telemetry has nothing to alert on. Replace with catching specific exceptions and either rethrowing, mapping to a typed `NotFoundException`, or returning a `Result`. At minimum, log with context and let the exception propagate.

**Trade-off:** There's one legitimate variation — top-level loops (worker services, message handlers) must catch broadly to stay alive, but they should always log first.

**Real project:** A `GetCustomer` helper caught all exceptions and returned null. A bad migration corrupted a foreign key — every customer lookup returned null, the UI showed empty profiles, no alerts fired. Diagnosed only after support tickets. Net effect: hours of customer impact for one line of "defensive" code.

### 5. How do you ensure `DbContext`, `HttpClient`, and Service Bus client are disposed on exceptions?

**Why this matters:** Resource leaks under failure are common.

**Answer:** Always use `using` (sync) or `await using` (async) for `IDisposable`/`IAsyncDisposable` — the compiler emits a `try/finally` that disposes even on exception. For DI-managed resources (most `DbContext` and typed `HttpClient`), the container disposes them at the end of the scope. For one-off `ServiceBusClient` or `BlobClient`, prefer DI registration over manual construction.

**Trade-off:** Inside a `using` block, exceptions still propagate after the dispose. `Dispose` itself may throw — wrap in a separate try if cleanup is best-effort.

**Real project:** A worker created `new SqlConnection(connString)` per message without `using`. Under failure it leaked a connection per error, eventually exhausting the pool, then every message failed. The fix was a one-line `using var conn = ...` — and adding a metric for `SqlClient.Connections.Open`.

### 6. Why is `catch (Exception ex) when (ex is SqlException sqlEx && sqlEx.Number == 1205)` preferred over `catch + if + throw;`?

**Why this matters:** Tests knowledge of exception filters (C# 6+).

**Answer:** Exception filters (`when` clauses) are evaluated **without unwinding the stack**. If the filter is false, the search for a handler continues with the stack still intact — better diagnostics and slightly cheaper. A `catch + if + throw;` actually unwinds, then re-throws, losing the original frame as the "active" one in some debuggers.

**Trade-off:** Filters that do non-trivial work run during exception search — keep them cheap and side-effect-free.

**Real project:** Our SQL retry policy used `catch + if` for deadlock detection (error 1205). After switching to a `when` filter, Application Insights showed the original SQL frame as the failure site instead of the retry-loop frame. Debugging time dropped meaningfully.

### 7. How should `Task.WhenAll` failures be handled?

**Why this matters:** A subtle pitfall — `await` on `WhenAll` only throws the **first** exception.

**Answer:** `await Task.WhenAll(tasks)` rethrows just the first faulted task's exception even if multiple failed. To inspect all, hold the combined task: `var all = Task.WhenAll(tasks); try { await all; } catch { foreach (var e in all.Exception!.InnerExceptions) log(e); throw; }`. For fan-out where partial success is acceptable, use `Task.WhenAll` with each task pre-wrapped in try/catch and return a result list.

**Trade-off:** Logging all inner exceptions inflates telemetry — pick a representative one for the alerting metric and log the rest as detail.

**Real project:** A reconciliation job fanning out 30 calls to Stripe occasionally had 3-4 simultaneous failures, but only one ever showed in App Insights. Switching to the `all.Exception.InnerExceptions` pattern revealed a pattern — they were all the same timeout window — leading us to a network MTU issue.

### 8. What HTTP status code should each domain exception map to?

**Why this matters:** REST hygiene; affects clients, monitoring, and SLA reporting.

**Answer:** Standard mapping:

| Exception | HTTP |
| --- | --- |
| `ValidationException` (bad input) | 400 |
| `UnauthorizedException` / missing JWT | 401 |
| `ForbiddenException` (authenticated but not allowed) | 403 |
| `NotFoundException` | 404 |
| `MethodNotAllowedException` | 405 |
| `ConflictException` (duplicate, version) | 409 |
| `UnprocessableEntityException` (valid shape, invalid business state) | 422 |
| `PaymentDeclinedException` | 402 |
| `TooManyRequestsException` | 429 |
| Anything else | 500 |

Return RFC 7807 `ProblemDetails` with `type` (URI for the error category), `title`, `status`, `detail`, `instance`.

**Trade-off:** Over-specific 4xx codes (418, 451) are cute but confuse clients and monitoring. Stick to the well-known ones.

**Real project:** An API mapped every business error to 500. Our SRE dashboard's "5xx error rate" alert fired constantly for normal validation failures, hiding real outages. Reclassifying validation/conflict to 4xx restored signal-to-noise on alerts.

## Summary Checklist

- [ ] I throw specific exception types (`PaymentDeclinedException`), never `Exception` or `ApplicationException`.
- [ ] I rethrow with `throw;` to preserve the stack trace — never `throw ex;`.
- [ ] I catch only at meaningful boundaries: middleware, worker loops, message handlers, infrastructure adapters.
- [ ] I use `IExceptionHandler` / `UseExceptionHandler` + `ProblemDetails` for HTTP error responses.
- [ ] I never expose stack traces or internal messages to clients in production.
- [ ] I let `OperationCanceledException` flow through when cancellation is requested.
- [ ] I use Polly for retries, timeouts, and circuit-breakers — not hand-rolled loops.
- [ ] I prefer validation results (`ValidationProblemDetails`) over exceptions for expected input errors.
- [ ] I use `using` / `await using` so resources are released even on exception.
- [ ] I log exceptions **once** with structured context, at the boundary.
