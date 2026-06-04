# Input Validation

## What It Is

Input validation is the discipline of rejecting malformed, missing, or untrusted request data **at the edge** of your service — before it reaches the application layer, the database, or any downstream system. In ASP.NET Core, validation runs during model binding (shape and type checks) and again inside endpoint filters, action filters, or handlers (business rules).

There are two distinct layers:

1. **Shape / contract validation** — "Is `email` present? Is `quantity` a positive integer? Is `currency` a 3-letter ISO code?" This is purely about the structure of the incoming DTO.
2. **Business / invariant validation** — "Does this customer exist? Is the SKU in stock? Is the refund amount less than the original charge?" This requires data the application owns.

```csharp
// Shape validation — runs during model binding / endpoint filter
public sealed record CreateOrderRequest(
    [Required] Guid CustomerId,
    [MinLength(1)] IReadOnlyList<OrderLineDto> Lines);

// Business validation — runs inside the command handler
if (!await _customers.ExistsAsync(request.CustomerId, ct))
    return Result.Fail("CUSTOMER_NOT_FOUND");
```

Skipping either layer leads to corrupted state, noisy 500s, or — worse — silent data quality bugs that surface weeks later in invoices and reports.

## Why It Exists

Before structured validation, .NET APIs typically scattered `if (string.IsNullOrEmpty(...))` checks throughout controllers and services. That style failed for three reasons:

- **Inconsistent error responses** — one endpoint returned 400 with a string, another returned 500 with a stack trace, a third silently inserted bad data.
- **Duplicated rules** — the same "email must be valid" check existed in the controller, the service, and the JavaScript front end, and they drifted out of sync.
- **Untestable** — validation logic was buried inside HTTP handlers, so unit tests had to spin up the whole pipeline to exercise a single rule.

DataAnnotations (2008) and later FluentValidation (2009) emerged to centralize rules in a declarative way, decouple them from the transport layer, and produce a single, machine-readable error contract — eventually standardized by [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807) and shipped as `ValidationProblemDetails` in ASP.NET Core.

## When To Use It

**Use validation for:**

- Every public HTTP endpoint, gRPC method, or message handler accepting external data.
- Inbound integration events from Azure Service Bus, Event Grid, or Kafka — never trust the producer.
- Configuration loaded from `appsettings.json` or Key Vault (use `IValidateOptions<T>` or `ValidateDataAnnotations()`).
- File uploads — verify MIME type, size, and magic bytes, not just the extension.

**Do not use validation as a substitute for:**

- **Authorization** — "Can this user perform this action?" belongs in policies and handlers (see [authentication-and-authorization.md](authentication-and-authorization.md)).
- **Database constraints** — unique indexes, foreign keys, and check constraints are your last line of defense; validation is the first.
- **Domain invariants** — rules like "an order total cannot be negative" belong inside the aggregate constructor, not in a DTO validator (see [../architecture/aggregates.md](../architecture/aggregates.md)).

## Why It Is Important

Validation drives four production properties:

1. **Security** — Rejecting oversized strings, malformed JSON, and SQL-shaped input shrinks the attack surface for injection, denial-of-service, and deserialization exploits.
2. **Reliability** — A 400 with a clear error is infinitely better than a 500 from a `NullReferenceException` deep in a payment handler.
3. **Client experience** — Front-end and partner teams can build retry logic and field-level error UIs only if your contract is consistent (`ValidationProblemDetails` with the `errors` dictionary).
4. **Operational cost** — Bad data that reaches your warehouse becomes a multi-day cleanup. Bad data rejected at the edge is a log line.

In an Azure-hosted microservice, validation also reduces noise in Application Insights, prevents Service Bus dead-letter queues from filling up with malformed messages, and keeps your SLO budget intact.

## How It's Used in C# / .NET

### 1. DataAnnotations — built-in, declarative

Lives in `System.ComponentModel.DataAnnotations`. Zero extra packages. Works automatically with `[ApiController]`.

```csharp
namespace OrdersApi.Contracts;

public sealed record CreateOrderRequest
{
    [Required]
    public Guid CustomerId { get; init; }

    [Required, StringLength(3, MinimumLength = 3)]
    [RegularExpression("^[A-Z]{3}$", ErrorMessage = "Currency must be ISO 4217.")]
    public string Currency { get; init; } = default!;

    [Required, MinLength(1), MaxLength(100)]
    public IReadOnlyList<OrderLineDto> Lines { get; init; } = [];
}

public sealed record OrderLineDto(
    [Required] string Sku,
    [Range(1, 1000)] int Quantity,
    [Range(0.01, 1_000_000)] decimal UnitPrice);
```

With `[ApiController]`, ASP.NET Core auto-returns a `400` with `ValidationProblemDetails` when `ModelState.IsValid == false`.

### 2. FluentValidation — fluent, testable, complex rules

NuGet: `FluentValidation.AspNetCore` (or `FluentValidation.DependencyInjectionExtensions` for manual wiring in .NET 8+).

```csharp
namespace OrdersApi.Validation;

public sealed class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator(ICurrencyCatalog currencyCatalog)
    {
        RuleFor(x => x.CustomerId).NotEmpty();

        RuleFor(x => x.Currency)
            .NotEmpty()
            .Length(3)
            .Must(currencyCatalog.IsSupported)
            .WithMessage("Currency '{PropertyValue}' is not supported.");

        RuleFor(x => x.Lines)
            .NotEmpty()
            .Must(lines => lines.Count <= 100)
            .WithMessage("Maximum 100 lines per order.");

        RuleForEach(x => x.Lines).SetValidator(new OrderLineValidator());
    }
}

public sealed class OrderLineValidator : AbstractValidator<OrderLineDto>
{
    public OrderLineValidator()
    {
        RuleFor(x => x.Sku).NotEmpty().MaximumLength(64);
        RuleFor(x => x.Quantity).GreaterThan(0).LessThanOrEqualTo(1_000);
        RuleFor(x => x.UnitPrice).GreaterThan(0m);
    }
}
```

Choose **FluentValidation** when rules are conditional, depend on injected services (catalog lookups, feature flags), or need to be unit-tested independently. Choose **DataAnnotations** for simple shape rules that travel with the DTO.

### 3. Minimal API endpoint filter (.NET 7+)

Minimal APIs do not run DataAnnotations or FluentValidation automatically. Wire them via an endpoint filter:

```csharp
public sealed class ValidationFilter<T>(IValidator<T> validator) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx,
        EndpointFilterDelegate next)
    {
        var argument = ctx.Arguments.OfType<T>().FirstOrDefault();
        if (argument is null)
            return Results.Problem("Request body required.", statusCode: 400);

        var result = await validator.ValidateAsync(argument, ctx.HttpContext.RequestAborted);
        if (!result.IsValid)
        {
            var errors = result.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());
            return Results.ValidationProblem(errors);
        }

        return await next(ctx);
    }
}

// Program.cs
app.MapPost("/api/v1/orders", CreateOrderAsync)
   .AddEndpointFilter<ValidationFilter<CreateOrderRequest>>();
```

### 4. Options validation

```csharp
builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // fail fast at startup, not on first request
```

## Advantages

- Fails fast — bad requests never reach business logic, persistence, or downstream APIs.
- Centralizes rules in one testable place per DTO.
- Produces a consistent, machine-readable error contract (`ValidationProblemDetails`).
- Documented automatically by Swagger / OpenAPI consumers.
- Reduces noise in logs and APM tools.

## Disadvantages

- DataAnnotations cannot easily express cross-field or async rules (e.g., "currency must exist in catalog").
- FluentValidation adds a dependency and a learning curve for juniors.
- Validators can grow into mini-business-logic if you let them — drawing the line between "shape" and "domain rule" requires discipline.
- Async validators (DB lookups) add latency to every request — cache when possible.

## Common Mistakes

### 1. Trusting model binding to catch missing required fields on `record` types

```csharp
// BAD — no [Required], no MinLength. Lines = null deserializes fine, then NRE later.
public sealed record CreateOrderRequest(Guid CustomerId, IReadOnlyList<OrderLineDto> Lines);
```

**Fix:** mark required members, or use FluentValidation, or enable nullable reference type validation:

```csharp
public sealed record CreateOrderRequest(
    [Required] Guid CustomerId,
    [Required, MinLength(1)] IReadOnlyList<OrderLineDto> Lines);
```

### 2. Mixing business rules into DTO validators

```csharp
// BAD — validator hits the database on every request, blocks pipeline, hard to test
RuleFor(x => x.CustomerId)
    .MustAsync(async (id, ct) => await _db.Customers.AnyAsync(c => c.Id == id, ct));
```

**Fix:** keep validators shape-only. Check existence inside the command handler where you have a transaction and can return a typed result (`Result.Fail("CUSTOMER_NOT_FOUND")`).

### 3. Returning custom error shapes instead of `ValidationProblemDetails`

```csharp
// BAD — every endpoint invents its own shape, clients can't parse uniformly
return BadRequest(new { error = "Email invalid" });
```

**Fix:** return `Results.ValidationProblem(errors)` or let `[ApiController]` handle it. One shape across the whole API.

### 4. Validating only on the controller, not on message handlers

```csharp
// BAD — Service Bus message handler trusts the payload
public async Task HandleAsync(OrderPlacedIntegrationEvent evt, CancellationToken ct)
{
    await _orders.CreateAsync(evt.CustomerId, evt.Lines, ct); // evt.Lines could be null
}
```

**Fix:** run the same validator at every boundary, including async messaging.

### 5. Allowing unbounded collections and strings

```csharp
public sealed record SearchRequest(string Query, List<string> Tags); // no caps anywhere
```

**Fix:** every string and collection needs `MaxLength`. Otherwise a single client can OOM your pod with a 50 MB array.

### 6. Forgetting to validate `IOptions<T>` at startup

A typo in `appsettings.Production.json` should crash the app at startup, not 30 minutes later on the first payment attempt. Use `ValidateOnStart()`.

## Best Practices

- One validator per DTO. Keep them shape-only.
- Always return `ValidationProblemDetails` (RFC 7807).
- Cap every string (`MaxLength`) and collection (`MinLength`/`MaxLength`).
- Validate at every trust boundary: HTTP, gRPC, message handlers, file uploads.
- Use `ValidateOnStart()` for configuration.
- Unit-test validators directly — they're plain classes, no HTTP needed.
- Log validation failures at `Information`, not `Error` — they're expected client mistakes.
- Pair with [error-handling-and-problem-details.md](error-handling-and-problem-details.md) for a coherent error story.

## Related Concepts

- [error-handling-and-problem-details.md](error-handling-and-problem-details.md) — the response shape for failed validation.
- [restful-api-design.md](restful-api-design.md) — status codes and contracts.
- [authentication-and-authorization.md](authentication-and-authorization.md) — validation is not authorization.
- [controllers-and-minimal-apis.md](controllers-and-minimal-apis.md) — where validators plug in.
- [../architecture/aggregates.md](../architecture/aggregates.md) — domain invariants live here, not in DTOs.
- [../testing-quality/unit-testing.md](../testing-quality/unit-testing.md) — validators are first-class unit-test targets.

## Real-World Usage

### ASP.NET Core Controllers

With `[ApiController]`, ModelState errors auto-return `400 ValidationProblemDetails`. Register FluentValidation alongside:

```csharp
builder.Services.AddControllers()
    .AddFluentValidation(cfg =>
        cfg.RegisterValidatorsFromAssemblyContaining<CreateOrderRequestValidator>());
```

### Azure Functions (isolated worker)

Functions do not get `[ApiController]` behavior. Validate explicitly inside the function or via a middleware:

```csharp
[Function("CreateOrder")]
public async Task<HttpResponseData> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
{
    var request = await req.ReadFromJsonAsync<CreateOrderRequest>();
    var result = await _validator.ValidateAsync(request!, req.FunctionContext.CancellationToken);
    if (!result.IsValid)
        return await req.WriteValidationProblemAsync(result.Errors);
    // ...
}
```

### Azure Service Bus message handlers

A poison message that fails validation should go to the dead-letter queue with a reason, not throw and retry forever:

```csharp
public async Task HandleAsync(ProcessMessageEventArgs args)
{
    var evt = args.Message.Body.ToObjectFromJson<OrderPlacedIntegrationEvent>();
    var result = await _validator.ValidateAsync(evt);
    if (!result.IsValid)
    {
        await args.DeadLetterMessageAsync(args.Message, "ValidationFailed",
            string.Join("; ", result.Errors.Select(e => e.ErrorMessage)));
        return;
    }
    // ...
}
```

### Multi-tenant SaaS

Tenant-specific rules (e.g., max line items per order varies by plan) belong in validators that take `ITenantContext` from DI, not hard-coded constants.

## Code Example — Before and After

### Before — inline checks, custom error shapes, business logic mixed in

```csharp
[HttpPost("/orders")]
public async Task<IActionResult> Create([FromBody] CreateOrderRequest request)
{
    if (request == null) return BadRequest("Body required");
    if (request.CustomerId == Guid.Empty) return BadRequest("Customer required");
    if (request.Lines == null || request.Lines.Count == 0)
        return BadRequest("At least one line required");

    foreach (var line in request.Lines)
    {
        if (string.IsNullOrEmpty(line.Sku)) return BadRequest("SKU required");
        if (line.Quantity <= 0) return BadRequest("Quantity must be positive");
    }

    // Business rule mixed with shape rule
    var customer = await _db.Customers.FindAsync(request.CustomerId);
    if (customer == null) return BadRequest("Unknown customer");

    var order = new Order { CustomerId = request.CustomerId };
    foreach (var l in request.Lines)
        order.AddLine(l.Sku, l.Quantity, l.UnitPrice);

    _db.Orders.Add(order);
    await _db.SaveChangesAsync();
    return Ok(order.Id);
}
```

Problems: inconsistent error shape, every rule duplicated per endpoint, business and shape concerns mixed, untestable without HTTP context.

### After — validator + handler separation

```csharp
namespace OrdersApi.Contracts;

public sealed record CreateOrderRequest(
    Guid CustomerId,
    string Currency,
    IReadOnlyList<OrderLineDto> Lines);

public sealed record OrderLineDto(string Sku, int Quantity, decimal UnitPrice);
```

```csharp
namespace OrdersApi.Validation;

public sealed class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Currency).NotEmpty().Length(3).Matches("^[A-Z]{3}$");
        RuleFor(x => x.Lines).NotEmpty().Must(l => l.Count <= 100);
        RuleForEach(x => x.Lines).SetValidator(new OrderLineValidator());
    }
}
```

```csharp
namespace OrdersApi.Endpoints;

public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/v1/orders", async (
            CreateOrderRequest request,
            IMediator mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(
                new CreateOrderCommand(request.CustomerId, request.Currency, request.Lines), ct);

            return result.IsSuccess
                ? Results.Created($"/api/v1/orders/{result.Value}", new { id = result.Value })
                : Results.Problem(result.Error.Message, statusCode: result.Error.StatusCode);
        })
        .AddEndpointFilter<ValidationFilter<CreateOrderRequest>>()
        .RequireAuthorization()
        .WithName("CreateOrder")
        .ProducesValidationProblem();
    }
}
```

```csharp
namespace OrdersApi.Application;

public sealed class CreateOrderHandler(
    ICustomerRepository customers,
    IOrderRepository orders,
    IUnitOfWork uow,
    ILogger<CreateOrderHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        // Business validation lives here — needs the database
        if (!await customers.ExistsAsync(cmd.CustomerId, ct))
        {
            logger.LogInformation("Order rejected: unknown customer {CustomerId}", cmd.CustomerId);
            return Result.Fail<Guid>(Errors.CustomerNotFound);
        }

        var order = Order.Create(cmd.CustomerId, cmd.Currency, cmd.Lines);
        await orders.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);
        return Result.Ok(order.Id);
    }
}
```

Now shape validation lives in a testable validator class, business validation lives in the handler with the database, and the endpoint is three lines of glue. Clients get consistent `ValidationProblemDetails`.

## Interview Questions and Answers

### 1. Where should validation live — in the DTO, the endpoint, the handler, or the domain?

**Why this matters:** Junior engineers often pick one location and force everything in. Seniors layer it by responsibility.

**Answer:** Three layers. **Shape rules** (required, length, format) live in a validator attached to the DTO and run during the endpoint filter. **Business rules** (does this customer exist, is this SKU in stock) live in the command handler where the database is available. **Invariants** (an order total cannot be negative) live in the aggregate constructor and protect the domain even if every other layer is bypassed.

**Trade-off:** Putting business rules in validators (`MustAsync` with DB lookups) feels convenient but couples HTTP infrastructure to persistence and is hard to test in isolation.

**Real project:** On an orders API, `CreateOrderRequestValidator` only checks shape. The handler checks the customer exists, the SKU is active, and the price matches the catalog. The `Order` aggregate enforces "lines.Sum >= 0".

### 2. DataAnnotations or FluentValidation — which would you pick for a new service?

**Why this matters:** Tests team maturity around dependencies and testability.

**Answer:** **FluentValidation** for anything non-trivial. DataAnnotations is fine for simple DTOs with no conditional logic, but FluentValidation gives you DI, conditional rules (`When`/`Unless`), reusable child validators, and trivially testable validator classes. The two can coexist — DataAnnotations on the DTO for the OpenAPI document, FluentValidation in the pipeline for the actual checks.

**Trade-off:** Extra NuGet dependency, extra concept for new hires.

**Real project:** A payments API used FluentValidation because currency support and limits were tenant-specific — `ICurrencyCatalog` was injected into the validator.

### 3. How do you validate request bodies in Minimal APIs in .NET 8?

**Why this matters:** Minimal APIs do not auto-validate. Many teams ship endpoints with no validation at all.

**Answer:** Either (a) write an `IEndpointFilter<T>` that resolves `IValidator<T>` from DI and returns `Results.ValidationProblem` on failure, attached via `.AddEndpointFilter<ValidationFilter<CreateOrderRequest>>()`, or (b) use a package like `MinimalApis.Extensions` or `FluentValidation.AspNetCore` adapters. The filter approach is explicit and gives you a single place to log/instrument validation.

**Trade-off:** One extra line per endpoint vs. magic global behavior.

**Real project:** A team migrating from controllers to Minimal APIs added one `ValidationFilter<T>` and wired it on every `MapPost`/`MapPut`. Took an afternoon, fixed dozens of accidentally-unvalidated endpoints.

### 4. A Service Bus consumer keeps throwing `NullReferenceException` on a field that's supposed to be required. How do you fix it without losing messages?

**Why this matters:** Tests whether the candidate treats messaging boundaries with the same rigor as HTTP boundaries.

**Answer:** Run the same `IValidator<T>` against the deserialized message inside the handler. If it fails, call `DeadLetterMessageAsync` with the validation errors as the reason, then return successfully so the message is removed from the queue. Add an alert on dead-letter count. Never let invalid messages loop on retry — they'll exhaust delivery count and either get auto-DLQ'd with no context or block the queue.

**Trade-off:** DLQ messages need a human or a replay job to fix.

**Real project:** An order-fulfillment consumer DLQ'd ~30 malformed messages per week from a flaky partner integration. The DLQ reason told ops exactly which field was wrong; the partner fixed their producer.

### 5. Should validation errors be logged?

**Why this matters:** Wrong logging here either floods your logs or hides real bugs.

**Answer:** Log at `Information`, not `Warning` or `Error`. Validation failures are expected — they're clients sending bad data. Logging them at `Error` will swamp Application Insights, page your on-call for nothing, and burn through your log ingestion budget. The exception is **systematic** failure (e.g., a specific tenant suddenly has 100% validation failure rate) — that should trigger a metric-based alert, not per-event logging.

**Trade-off:** You lose individual-event visibility, but you keep operational sanity.

**Real project:** A team dropped their App Insights bill by 40% by reclassifying validation errors from `Warning` to `Information` and adding a `ValidationFailureRate` metric.

### 6. How do you validate configuration so the app fails fast at deploy time?

**Why this matters:** Production outages from typos in `appsettings.json` are embarrassingly common.

**Answer:** Bind options with `AddOptions<T>().Bind(...).ValidateDataAnnotations().ValidateOnStart()`. Decorate the options class with `[Required]`, `[Url]`, `[Range]`. For complex rules, implement `IValidateOptions<T>`. `ValidateOnStart()` runs the validators during host startup — the app refuses to start if config is broken, so the deployment fails the health probe and Azure rolls back.

**Trade-off:** A typo means the new revision never serves traffic — which is exactly what you want.

**Real project:** A Stripe integration crashed every payment because `Stripe:WebhookSecret` was missing in production. Adding `ValidateOnStart()` caught the next occurrence at deploy time, before any user traffic hit the broken pod.

### 7. How would you validate a file upload?

**Why this matters:** File uploads are a classic injection and DoS vector.

**Answer:** Four checks: (1) `RequestSizeLimit` attribute or Kestrel `MaxRequestBodySize` to cap bytes; (2) `IFormFile.Length` check inside the action; (3) MIME type from `IFormFile.ContentType` and a magic-byte check against the file content (don't trust the client); (4) for documents stored in Blob Storage, generate a short-lived SAS, never expose the container directly. Optionally scan with Defender for Storage.

**Trade-off:** Magic-byte checks require reading the first few bytes synchronously — adds latency.

**Real project:** A document-management API rejected uploads whose declared MIME didn't match the magic bytes; this caught a partner that was sending HTML disguised as `.pdf`.

### 8. Validation passed but the database threw a unique-constraint violation. Whose fault is it?

**Why this matters:** Tests understanding of layered defense and concurrency.

**Answer:** It's not a fault — it's correct layered defense. Validation runs against the request shape at time T1; the database constraint enforces the invariant at commit time T2. Between T1 and T2 another request could insert the same value. Catch `DbUpdateException` (or check `SqlException.Number == 2627/2601` for SQL Server), translate to a `409 Conflict` `ProblemDetails`, and log at `Information`. Trying to prevent it with a pre-check is a TOCTOU race.

**Trade-off:** Slightly less friendly error message than a pre-check, but correct under concurrency.

**Real project:** A user-registration endpoint stopped trying to "check if email exists" before insert and instead caught the unique-constraint violation. Bug count dropped, code got simpler, race condition disappeared.

## Summary Checklist

- [ ] I separate shape validation (DTO) from business validation (handler) from invariants (aggregate).
- [ ] I return `ValidationProblemDetails` (RFC 7807) consistently across the API.
- [ ] I cap every string and collection with `MaxLength` / `MinLength`.
- [ ] I validate inbound messages at every boundary — HTTP, gRPC, Service Bus, file upload.
- [ ] I validate configuration with `ValidateOnStart()` so deploys fail fast.
- [ ] I log validation failures at `Information`, not `Error`.
- [ ] I unit-test validators directly without spinning up the HTTP pipeline.
- [ ] I dead-letter invalid messages with a clear reason instead of looping retries.
- [ ] I use FluentValidation for conditional and DI-dependent rules.
- [ ] I never use validation as a substitute for authorization or database constraints.
