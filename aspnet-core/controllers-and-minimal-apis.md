# Controllers and Minimal APIs

## What It Is

ASP.NET Core gives you **two endpoint styles** for building HTTP APIs:

- **MVC Controllers** — classes that inherit from `ControllerBase`, decorated with attributes (`[ApiController]`, `[Route]`, `[HttpPost]`), with methods returning `ActionResult<T>`. This is the classic style introduced in ASP.NET Core 1.0.
- **Minimal APIs** — lambda or method-group endpoints registered directly on `WebApplication` via `MapGet`, `MapPost`, etc., introduced in .NET 6 and matured through .NET 7–9 with **endpoint filters**, **route groups**, **typed results**, and **`[AsParameters]`** binding.

Both produce the **same HTTP behavior**, share the **same routing**, the **same model binding**, the **same authentication and authorization pipeline**, and the **same OpenAPI tooling**. The choice is about ergonomics, structure, and team conventions — not capability.

```csharp
// MVC controller style
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orders;
    public OrdersController(IOrderService orders) => _orders = orders;

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDto>> GetById(Guid id, CancellationToken ct)
        => await _orders.GetAsync(id, ct) is { } o ? Ok(o) : NotFound();
}

// Minimal API style (equivalent)
app.MapGet("/api/orders/{id:guid}", async (Guid id, IOrderService orders, CancellationToken ct)
    => await orders.GetAsync(id, ct) is { } o ? Results.Ok(o) : Results.NotFound());
```

## Why It Exists

**Controllers** existed because ASP.NET MVC needed structured request handling: model binding, model validation, filters, content negotiation, and built-in conventions (an `Index` action under `/Home/`). They scale to dozens of endpoints per controller with a familiar class-based shape that maps cleanly to MediatR handlers, automated tests, and DI.

**Minimal APIs** were added in .NET 6 to solve a different set of problems:

- **Cold start matters in serverless and containers.** MVC pulls in `Microsoft.AspNetCore.Mvc.Core`, model validation, filters, content negotiation, and the entire action invoker pipeline. Minimal APIs skip what you don't use.
- **Microservices and webhooks rarely need 30 endpoints in one class.** A simple receiver, a health probe, or a webhook listener was a lot of ceremony for two lines of real code.
- **Modern C# (top-level statements, records, pattern matching, primary constructors) deserved an endpoint style that matched.** No more boilerplate `class`, `public override`, `[FromBody]` for everything.
- **Native AOT** in .NET 8+ favors source-generated, reflection-free endpoints — minimal APIs are AOT-friendly first.

The two styles **co-exist** in the same app. The framework converged them in .NET 7 (endpoint filters), .NET 8 (route groups, `IExceptionHandler`), and .NET 9 (built-in OpenAPI), so the gap is now mostly stylistic.

## When To Use It

**Use MVC Controllers for:**

- **Larger, structured APIs** (10+ endpoints per resource) where grouping by class and filters is helpful.
- **Codebases already using MVC conventions** (Razor, server-rendered views, filters, model binders).
- **Teams that rely on `[ApiController]` conventions**: automatic 400 on invalid model state, `[FromBody]` inferred for complex types, attribute routing required.
- **Apps that need MVC-only features**: model validation pipeline, `[ProducesResponseType]` attributes, action filters, custom model binders.

**Use Minimal APIs for:**

- **Small services, microservices, webhooks, health checks, and admin endpoints.**
- **Azure Functions–style work** where startup time and code size matter.
- **Native AOT scenarios** (.NET 8+) where reflection-heavy MVC is undesirable.
- **Modern .NET 8/9 codebases** that lean on `MapGroup`, endpoint filters, and `TypedResults`.

**Mix both** when convenient: MVC for the main resource APIs, minimal APIs for `/health`, `/version`, and webhooks. Both register the same way (`builder.Services.AddControllers()` + `app.MapControllers()` for MVC, just `app.MapGet/Post/...` for minimal).

## Why It Is Important

The endpoint layer is the **contract surface** of your service. Decisions here ripple to:

1. **API consumers** — partner mobile apps, frontends, and other services depend on stable status codes, response shapes, and validation behavior. Switching styles mid-flight can introduce subtle differences (default validation behavior, ProblemDetails shape).
2. **Operability** — controllers and minimal APIs both feed `app.MapHealthChecks`, OpenAPI generation, and route discovery for dashboards and Azure API Management. Inconsistent metadata produces inconsistent observability.
3. **Cold start and memory** — minimal APIs trim ~5–15 MB and several hundred milliseconds of startup on Azure App Service P1V3 in real measurements; meaningful for serverless and per-pod cost on AKS.
4. **Onboarding and review velocity** — picking one style as the default in your team's architecture decision records makes PR review faster and reduces mental context switching.

## How It's Used in C# / .NET

### 1. Setting up MVC controllers

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers()                              // MVC services
    .AddJsonOptions(o => o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase);
builder.Services.AddProblemDetails();                          // consistent error shape
builder.Services.AddEndpointsApiExplorer();                    // OpenAPI metadata
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseExceptionHandler();
app.MapControllers();                                          // attribute-routed
app.Run();
```

```csharp
[ApiController]
[Route("api/orders")]
[Produces("application/json")]
public sealed class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpGet("{id:guid}", Name = nameof(GetOrderById))]
    [ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> GetOrderById(Guid id, CancellationToken ct)
        => await _mediator.Send(new GetOrderQuery(id), ct) is { } dto ? Ok(dto) : NotFound();

    [HttpPost]
    [ProducesResponseType<OrderCreatedResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<OrderCreatedResponse>> Create(
        [FromBody] CreateOrderRequest request, CancellationToken ct)
    {
        var id = await _mediator.Send(new CreateOrderCommand(request), ct);
        return CreatedAtRoute(nameof(GetOrderById), new { id }, new OrderCreatedResponse(id));
    }
}
```

Key MVC conventions:

- **`[ApiController]`** enables: automatic 400 on invalid `ModelState`, inferred binding sources (`[FromBody]` for complex types, `[FromRoute]` for route values), and attribute routing requirement.
- **`ActionResult<T>`** lets you return both a typed body (`Ok(dto)`) and any `IActionResult` (`NotFound()`).
- **`[ProducesResponseType]`** improves OpenAPI; controllers don't infer response types like minimal APIs do.

### 2. Setting up minimal APIs

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddProblemDetails();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddOpenApi();                                  // .NET 9 native OpenAPI

var app = builder.Build();
app.UseExceptionHandler();
app.MapOpenApi();                                               // /openapi/v1.json

var orders = app.MapGroup("/api/orders").WithTags("Orders");    // route group

orders.MapGet("/{id:guid}", async (Guid id, IMediator mediator, CancellationToken ct)
        => await mediator.Send(new GetOrderQuery(id), ct) is { } dto
            ? Results.Ok(dto)
            : Results.NotFound())
    .WithName("GetOrderById")
    .Produces<OrderDto>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound);

orders.MapPost("/", async (CreateOrderRequest request, IMediator mediator, CancellationToken ct) =>
    {
        var id = await mediator.Send(new CreateOrderCommand(request), ct);
        return Results.CreatedAtRoute("GetOrderById", new { id }, new OrderCreatedResponse(id));
    })
    .Produces<OrderCreatedResponse>(StatusCodes.Status201Created)
    .ProducesValidationProblem();

app.Run();
```

### 3. `IResult` and `TypedResults` (minimal APIs)

`Results.Ok(dto)` returns `IResult`. `TypedResults.Ok(dto)` returns `Ok<T>` — the typed version is preferred in .NET 7+ because it enables OpenAPI inference and is testable without HTTP plumbing:

```csharp
orders.MapPost("/", async Task<Results<Created<OrderCreatedResponse>, ValidationProblem>>
    (CreateOrderRequest req, IMediator mediator, CancellationToken ct) =>
{
    var id = await mediator.Send(new CreateOrderCommand(req), ct);
    return TypedResults.Created($"/api/orders/{id}", new OrderCreatedResponse(id));
});
```

The `Results<...>` union type tells the framework (and OpenAPI) all possible response shapes.

### 4. Endpoint filters (.NET 7+) — middleware's per-endpoint cousin

```csharp
orders.MapPost("/", CreateOrder)
    .AddEndpointFilter<ValidationFilter<CreateOrderRequest>>()    // run FluentValidation
    .AddEndpointFilter(async (ctx, next) =>                       // inline timing
    {
        var sw = Stopwatch.StartNew();
        var result = await next(ctx);
        ctx.HttpContext.Response.Headers["X-Handler-Ms"] = sw.ElapsedMilliseconds.ToString();
        return result;
    });
```

`MapGroup` can attach filters that apply to all child endpoints:

```csharp
var admin = app.MapGroup("/admin")
    .RequireAuthorization("AdminPolicy")
    .AddEndpointFilter<AuditingFilter>();
```

### 5. Parameter binding sources

| Source                  | MVC attribute       | Minimal APIs                            |
|-------------------------|---------------------|-----------------------------------------|
| Route value             | `[FromRoute]`       | Inferred from route token (`{id}`)      |
| Query string            | `[FromQuery]`       | Inferred for primitives                 |
| Request body (JSON)     | `[FromBody]`        | Inferred for complex types              |
| Header                  | `[FromHeader]`      | `[FromHeader]`                          |
| Form                    | `[FromForm]`        | `[FromForm]` (.NET 7+)                  |
| DI service              | `[FromServices]`    | Inferred for registered services        |
| Aggregate of many       | (no equivalent)     | `[AsParameters]` on a record/class      |

```csharp
public record OrderQuery([FromQuery] int Page = 1, [FromQuery] int PageSize = 20, [FromQuery] string? Status = null);

app.MapGet("/api/orders", ([AsParameters] OrderQuery q, IOrderService svc, CancellationToken ct)
    => svc.SearchAsync(q.Page, q.PageSize, q.Status, ct));
```

### 6. Validation differences

- **MVC + `[ApiController]`** runs DataAnnotations on the model and automatically returns `400 ValidationProblemDetails` if `ModelState` is invalid. No manual code.
- **Minimal APIs** do **not** run model validation. You must either use FluentValidation in an endpoint filter, call validation yourself, or use the `MinimalApis.Extensions` community packages. See [input-validation.md](input-validation.md).

### 7. OpenAPI integration

| Version            | API                                                  |
|--------------------|------------------------------------------------------|
| .NET 6/7/8         | Swashbuckle (`AddSwaggerGen` / `UseSwaggerUI`)       |
| .NET 9+            | Built-in `AddOpenApi()` + `MapOpenApi()`             |

Both styles produce OpenAPI documents. Minimal APIs infer most metadata from `TypedResults` and binding; controllers benefit from `[ProducesResponseType]`. See [swagger-openapi.md](swagger-openapi.md).

### 8. Quick comparison

| Capability                          | MVC Controllers                | Minimal APIs                          |
|-------------------------------------|--------------------------------|---------------------------------------|
| Class-based structure               | Yes                            | No (lambdas / method groups)          |
| Filters                             | Action / resource / result filters | Endpoint filters (.NET 7+)         |
| Route grouping                      | `[Route]` on controller        | `MapGroup` (.NET 7+)                  |
| Automatic 400 on invalid model      | With `[ApiController]`         | Manual (FluentValidation / filter)    |
| Inferred binding sources            | With `[ApiController]`         | Native                                |
| Native AOT support                  | Partial                        | First-class (.NET 8+)                 |
| Cold start                          | Slower                         | Faster                                |
| Best for                            | Large structured APIs          | Microservices, webhooks, simple APIs  |

## Advantages

**Controllers:**

- Familiar class-based shape — easy onboarding for engineers from MVC, Java/Spring, or older ASP.NET.
- Rich filter pipeline (action / resource / result filters, exception filters) for cross-cutting concerns.
- Automatic model validation via `[ApiController]`.
- Per-controller routing/auth attributes scale to many endpoints cleanly.

**Minimal APIs:**

- Lower ceremony — endpoint is the lambda, not a class.
- Faster startup, smaller memory, AOT-friendly.
- Route groups + endpoint filters compose well.
- `TypedResults` give compile-time response shape and richer OpenAPI inference.
- Naturally encourages thin endpoint + MediatR/handler delegation.

## Disadvantages

**Controllers:**

- More files, more boilerplate for tiny services.
- Slower startup, larger memory footprint.
- Reflection-heavy invoker pipeline — less friendly to native AOT.
- Easy to drift into "service classes inside controller classes" if discipline slips.

**Minimal APIs:**

- No built-in model validation — easy to forget.
- Lambdas explode in size as endpoints grow; refactor to method groups early.
- No action filter pipeline — endpoint filters cover most cases but not all (no result filter, no resource filter).
- Less ecosystem tooling than controllers in some libraries (e.g., older versions of Hangfire dashboards).

## Common Mistakes

### 1. Putting business logic in the controller / endpoint

```csharp
// BUG: domain logic, DB access, and HTTP all in one place
[HttpPost]
public async Task<IActionResult> Capture(Guid invoiceId, decimal amount)
{
    var invoice = await _db.Invoices.FindAsync(invoiceId);
    if (invoice is null) return NotFound();
    var charge = await _stripe.ChargeAsync(invoice.CustomerId, amount);
    invoice.MarkPaid(charge.Id);
    await _db.SaveChangesAsync();
    return Ok();
}
```

**Fix**: Use a thin endpoint that delegates to MediatR or an application service:

```csharp
[HttpPost("{invoiceId:guid}/capture")]
public async Task<ActionResult<CaptureResponse>> Capture(Guid invoiceId, CaptureRequest req, IMediator mediator, CancellationToken ct)
    => Ok(await mediator.Send(new CaptureInvoiceCommand(invoiceId, req.Amount), ct));
```

### 2. Returning `IActionResult` everywhere instead of `ActionResult<T>`

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> Get(Guid id)             // ❌ no body type for OpenAPI
{
    var dto = await _svc.GetAsync(id);
    return dto is null ? NotFound() : Ok(dto);
}
```

**Fix**: `Task<ActionResult<OrderDto>>` gives strong typing and lets Swashbuckle infer the 200 response shape.

### 3. Forgetting `[ApiController]` and getting silent behavior

Without `[ApiController]`, invalid `ModelState` reaches your action and you return whatever you wrote — usually a 200 with no body. Add `[ApiController]` (or apply it project-wide in `Program.cs`).

### 4. Mixing endpoint styles inconsistently for the same resource

Half the `/api/orders` endpoints as MVC controllers, half as minimal APIs — auth, error handling, and OpenAPI conventions drift apart. Pick one style per resource.

### 5. Reading from the request body twice in a minimal API

```csharp
app.MapPost("/orders", async (HttpRequest req, CreateOrderRequest body) => { /* body is null */ });
```

Don't bind both `HttpRequest` and a body model — the framework consumes the stream first. Bind only the model.

### 6. Skipping `CancellationToken` in async endpoints

Endpoints should accept `CancellationToken` so client disconnects abort long-running work. ASP.NET Core binds the request's `HttpContext.RequestAborted` automatically.

### 7. Using `Results.Json(obj, statusCode: 201)` instead of `Results.Created`

`Results.Created("/api/orders/{id}", body)` sets the `Location` header — `Results.Json` does not. Mobile clients that read `Location` to navigate to the new resource will break.

### 8. Async void or missing `await`

```csharp
app.MapPost("/orders", (CreateOrderRequest req, IOrderService svc) =>
    svc.PlaceAsync(req));    // ❌ returns Task<Guid> raw — serialized awkwardly
```

**Fix**: `async (req, svc, ct) => Results.Ok(await svc.PlaceAsync(req, ct))`.

## Best Practices

- **Pick a default style per project** and document it in an ADR. Mix only with reason.
- **Keep endpoints thin** — bind, validate, delegate to a handler (MediatR, application service), return. Anything else belongs in a service.
- **Always include `CancellationToken`** as the last parameter on async endpoints.
- **Use `TypedResults` and `Results<TUnion>`** in minimal APIs for compile-time response types and richer OpenAPI.
- **Apply `[ApiController]`** to every API controller (or set `ApiBehaviorOptions` globally).
- **Group routes** with `MapGroup` (minimal) or controller `[Route]` (MVC) to apply auth and filters consistently.
- **Decorate with response types** (`[ProducesResponseType]` for MVC, `.Produces<T>()` for minimal) so OpenAPI is accurate.
- **Validate at the boundary** — DataAnnotations + `[ApiController]` for MVC, FluentValidation endpoint filter for minimal APIs.
- **Centralize error handling** via `UseExceptionHandler` + `IExceptionHandler` instead of try/catch in endpoints. See [error-handling-and-problem-details.md](error-handling-and-problem-details.md).
- **Test endpoints via `WebApplicationFactory<Program>`** — both styles boot identically.

## Related Concepts

- **Request pipeline** — endpoints execute at the end of the middleware pipeline. See [request-pipeline.md](request-pipeline.md).
- **REST design** — what makes a good resource URI, status code, and response. See [restful-api-design.md](restful-api-design.md).
- **Input validation** — DataAnnotations (MVC) and FluentValidation (minimal/MVC). See [input-validation.md](input-validation.md).
- **Error handling** — Problem Details + `IExceptionHandler`. See [error-handling-and-problem-details.md](error-handling-and-problem-details.md).
- **Authentication and authorization** — `[Authorize]` (MVC) and `.RequireAuthorization()` (minimal). See [authentication-and-authorization.md](authentication-and-authorization.md).
- **OpenAPI / Swagger** — both styles emit metadata. See [swagger-openapi.md](swagger-openapi.md).
- **API versioning** — both styles supported by `Asp.Versioning.Mvc`. See [api-versioning.md](api-versioning.md).
- **Dependency injection** — both bind services via `IServiceCollection`. See [../csharp/dependency-injection.md](../csharp/dependency-injection.md).
- **CQRS + MediatR** — endpoints become a routing layer over commands and queries. See [../architecture/cqrs.md](../architecture/cqrs.md).

## Real-World Usage

### ASP.NET Core API on Azure App Service

A typical Orders API on App Service uses **MVC controllers** because the team already had 30+ endpoints, custom action filters for auditing, and Razor pages for an internal admin UI:

```csharp
builder.Services.AddControllers(o => o.Filters.Add<AuditFilter>())
    .AddJsonOptions(o => o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase);
builder.Services.AddProblemDetails();
builder.Services.AddSwaggerGen();
// ...
app.MapControllers();
```

### Azure Functions (isolated worker) + minimal-API style HTTP triggers

Functions itself is closer to minimal APIs in spirit. HTTP triggers map cleanly:

```csharp
[Function("Webhook")]
public async Task<HttpResponseData> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "stripe")] HttpRequestData req)
{
    // delegate to a handler
}
```

For lightweight ASP.NET Core sidecars deployed alongside Functions, minimal APIs are the natural choice.

### Microservices

In a microservices estate (Orders, Payments, Notifications, Catalog), the team standardized on **minimal APIs + MapGroup + endpoint filters** for new services after .NET 8 — faster cold start in AKS, lower memory per pod, and AOT-ready. Legacy controller-based services stayed unchanged.

### Multi-tenant SaaS

Multi-tenancy works with both styles. A `MapGroup("/api/v{version:apiVersion}/orders").AddEndpointFilter<TenantResolutionFilter>()` keeps tenant resolution in one place. With controllers, the same is done via a `[ServiceFilter(typeof(TenantResolutionFilter))]` attribute on the controller class.

### Testing

```csharp
public class OrdersEndpointTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public OrdersEndpointTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task POST_orders_returns_201_with_Location_header()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new CreateOrderRequest(Guid.NewGuid(), Lines: new[] { new LineDto(Guid.NewGuid(), 1) }));

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

Identical test for both styles — `WebApplicationFactory` doesn't care.

## Code Example — Before and After

### Before: A 200-line controller doing everything

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IStripeClient _stripe;
    private readonly IEmailSender _email;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(AppDbContext db, IStripeClient stripe, IEmailSender email, ILogger<OrdersController> logger)
    { _db = db; _stripe = stripe; _email = email; _logger = logger; }

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest req)
    {
        if (req == null || req.Lines.Count == 0)
            return BadRequest("Order must have lines");

        try
        {
            var order = new Order(req.CustomerId);
            foreach (var l in req.Lines) order.AddLine(l.ProductId, l.Quantity);

            await _db.Orders.AddAsync(order);
            await _db.SaveChangesAsync();

            var charge = await _stripe.ChargeAsync(order.CustomerId, order.Total);
            order.MarkPaid(charge.Id);
            await _db.SaveChangesAsync();

            await _email.SendAsync(order.CustomerEmail, "Order confirmed",
                $"Thanks! Your order {order.Id} is on its way.");

            return Ok(new { id = order.Id });          // ❌ should be 201
        }
        catch (StripeException ex)
        {
            _logger.LogError(ex, "Charge failed");
            return StatusCode(500, ex.Message);        // ❌ leaks internals
        }
    }
}
```

Problems: business logic in the controller, no validation contract, wrong status code, internal errors leaked, no `CancellationToken`, mixed concerns.

### After: Thin endpoint + MediatR handler

```csharp
// Controller (MVC style)
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    [ProducesResponseType<OrderCreatedResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<OrderCreatedResponse>> Create(
        CreateOrderRequest request, CancellationToken ct)
    {
        var id = await _mediator.Send(new PlaceOrderCommand(request), ct);
        return CreatedAtAction(nameof(GetById), new { id }, new OrderCreatedResponse(id));
    }

    [HttpGet("{id:guid}", Name = nameof(GetById))]
    public async Task<ActionResult<OrderDto>> GetById(Guid id, CancellationToken ct)
        => await _mediator.Send(new GetOrderQuery(id), ct) is { } o ? Ok(o) : NotFound();
}

// Same in minimal API style
var orders = app.MapGroup("/api/orders").WithTags("Orders");

orders.MapPost("/", async Task<Results<CreatedAtRoute<OrderCreatedResponse>, ValidationProblem>>
    (CreateOrderRequest req, IMediator mediator, CancellationToken ct) =>
{
    var id = await mediator.Send(new PlaceOrderCommand(req), ct);
    return TypedResults.CreatedAtRoute(new OrderCreatedResponse(id), "GetOrderById", new { id });
})
.AddEndpointFilter<ValidationFilter<CreateOrderRequest>>();

orders.MapGet("/{id:guid}", async (Guid id, IMediator mediator, CancellationToken ct)
    => await mediator.Send(new GetOrderQuery(id), ct) is { } o ? Results.Ok(o) : Results.NotFound())
    .WithName("GetOrderById");
```

The endpoint now does **one thing**: bind, delegate, translate the handler's result to an HTTP response. Validation, persistence, payments, and notifications live in the handler and its collaborators — each unit-testable in isolation.

## Interview Questions and Answers

### 1. When would you pick MVC controllers over minimal APIs in a greenfield .NET 9 project?

**Why this matters**: Tests pragmatic decision-making, not religious preference.

**Answer**: I default to **minimal APIs for new services in .NET 8+** because of cold start, AOT readiness, and `TypedResults` + `MapGroup` ergonomics. I reach for **MVC controllers** when (a) the service has 20+ endpoints with strong grouping, (b) the team relies on action / resource / result filters that endpoint filters don't cover, (c) we already have Razor or model binders, or (d) onboarding new engineers from MVC-heavy backgrounds is a priority.

**Trade-off**: Mixing both styles increases cognitive load. I pin the default in an ADR; exceptions need a paragraph of justification.

**Real project**: An Orders microservice on AKS started as minimal APIs; the Admin API for the same domain — with auditing, paging, complex query DTOs, and 40+ endpoints — stayed on controllers because the action filter pipeline was central to its compliance story.

### 2. What does `[ApiController]` actually do?

**Why this matters**: Many developers add it from a template without understanding the behavior changes.

**Answer**: `[ApiController]` opts the controller into several conventions in MVC:

1. **Automatic 400 on invalid `ModelState`** — the framework short-circuits the action and returns `ValidationProblemDetails`.
2. **Binding source inference** — complex types bind from body, simple types from route/query, `IFormFile` from form, etc.
3. **Attribute routing required** — no convention-based routing fallback.
4. **`ProblemDetails` for error status codes** — `BadRequest()`, `NotFound()`, etc. produce RFC 7807 bodies automatically (when configured).
5. **Multipart form data inference** for parameters that need it.

You can configure these via `ApiBehaviorOptions` (e.g., disable the automatic 400 if you want to handle validation yourself).

**Trade-off**: The "automatic 400" hides where the validation runs — junior devs sometimes wonder why their `if (!ModelState.IsValid)` never executes. Document it in the project README.

**Real project**: Disabling automatic 400 to switch to FluentValidation + a custom `IActionFilter` standardized error shape across MVC and minimal endpoints.

### 3. Walk me through how a minimal API endpoint produces an OpenAPI document.

**Answer**: `WebApplication` registers an `EndpointDataSource` for every `MapXxx` call. With `AddEndpointsApiExplorer()` (or `AddOpenApi()` in .NET 9), the framework inspects each endpoint's metadata:

- **Route** + **HTTP method** from `MapGet/Post/...`.
- **Parameter binding** — request body, query, route, header — inferred from the delegate signature.
- **Response types** — from `TypedResults`, `Results<T1, T2>` union types, or `.Produces<T>()` calls.
- **Tags, summaries, descriptions** — from `.WithTags()`, `.WithSummary()`, `.WithDescription()`.
- **Auth** — from `.RequireAuthorization()`.

The OpenAPI generator (Swashbuckle or .NET 9 built-in) consumes this metadata to emit a JSON document at `/openapi/v1.json` (or `/swagger/v1/swagger.json` with Swashbuckle).

**Trade-off**: Minimal APIs infer a lot, but for response shape variants (multiple 4xx codes, custom error types) you still need explicit `.Produces` calls.

**Real project**: A partner Payments API generated a TypeScript SDK from the minimal-API OpenAPI document via NSwag in CI — partners consumed it directly from npm.

### 4. How do endpoint filters compare to action filters?

**Answer**:

| Aspect          | MVC action filters                | Endpoint filters (minimal APIs)      |
|-----------------|-----------------------------------|--------------------------------------|
| Scope           | Per-controller / per-action       | Per-endpoint or per-route-group      |
| Lifecycle hooks | `OnActionExecuting/Executed`, plus resource / result / exception filters | Single `InvokeAsync` wrapping the endpoint |
| Available on    | MVC controllers                   | Minimal APIs (and .NET 8+ minimal API controllers via `WithMetadata`) |
| Performance     | Reflection-driven                 | Source-friendly, slightly leaner     |
| DI              | Yes via `ServiceFilterAttribute`  | Yes via constructor injection        |

Endpoint filters handle the common case (wrap the call, modify result, short-circuit). For multi-stage MVC concerns (e.g., resource filters that run before model binding), action / resource filters are still more powerful.

**Real project**: Migrated a `RequestLoggingFilter` from MVC `IActionFilter` to a minimal-API endpoint filter when the team consolidated on minimal APIs — code shrank by 60% because the filter no longer needed `ActionExecutingContext` plumbing.

### 5. How do you avoid lambda explosion in minimal APIs?

**Why this matters**: Minimal APIs encourage inline lambdas; in a real service they balloon.

**Answer**: Three patterns:

1. **Method groups** — extract the handler to a static or instance method and pass it as a delegate:

   ```csharp
   orders.MapPost("/", OrdersEndpoints.Create);

   static async Task<IResult> Create(CreateOrderRequest req, IMediator m, CancellationToken ct)
       => Results.CreatedAtRoute("GetOrderById",
           new { id = await m.Send(new PlaceOrderCommand(req), ct) }, null);
   ```

2. **Endpoint classes** — group related endpoints with an `IEndpoint`-style interface and a `MapEndpoints` extension method per feature. Carter and FastEndpoints libraries automate this.
3. **`MapGroup`** — share route prefix, auth, and filters; the lambdas themselves stay tiny.

**Trade-off**: Method groups lose some compile-time generic inference around `TypedResults`. Most teams accept it.

**Real project**: A Payments service started with 30 inline lambdas in `Program.cs` (~600 lines). Refactored into `PaymentsEndpoints.MapPaymentEndpoints(app)` per feature — `Program.cs` shrank to ~80 lines.

### 6. How do you handle global validation in minimal APIs?

**Answer**: Minimal APIs don't run `ModelState` validation. Two patterns:

1. **FluentValidation in an endpoint filter** registered per endpoint or per group:

   ```csharp
   public class ValidationFilter<T> : IEndpointFilter where T : class
   {
       public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
       {
           var arg = ctx.Arguments.OfType<T>().FirstOrDefault();
           var validator = ctx.HttpContext.RequestServices.GetService<IValidator<T>>();
           if (arg is not null && validator is not null)
           {
               var result = await validator.ValidateAsync(arg, ctx.HttpContext.RequestAborted);
               if (!result.IsValid) return TypedResults.ValidationProblem(result.ToDictionary());
           }
           return await next(ctx);
       }
   }
   ```

2. **DataAnnotations + the `MinimalApis.Extensions` community package** for `.WithParameterValidation()`.

**Trade-off**: DataAnnotations is built-in but limited. FluentValidation has async, cross-field, and async rules, but adds a dependency.

**Real project**: A Notifications API used FluentValidation across both MVC and minimal endpoints with a shared `ValidationBehavior` (MediatR pipeline) for handler-level validation and an endpoint filter for boundary validation.

### 7. A teammate has a controller method returning `IActionResult`. You suggest `ActionResult<T>`. Why?

**Answer**:

- **Type safety** — `Ok(dto)` won't compile if `dto` isn't `T`.
- **OpenAPI** — Swashbuckle infers the 200 response type from `T`.
- **Tests** — `(await controller.GetAsync()).Value` gives a typed object; `IActionResult` requires `((OkObjectResult)result).Value` casting.
- **Documentation** — readers see the success shape in the signature without reading the body.

You still return `NotFound()`, `Conflict()`, etc. — they all flow through the `ActionResult<T>` implicit conversion.

**Trade-off**: For endpoints that genuinely return many shapes (rare in well-designed REST APIs), `IActionResult` is fine.

**Real project**: A code-review checklist item: "Public controller methods return `Task<ActionResult<T>>`." Caught dozens of typeless endpoints before they shipped.

### 8. How does native AOT affect your choice between controllers and minimal APIs in .NET 8/9?

**Why this matters**: AOT is increasingly relevant for cold-start-sensitive workloads (Container Apps, Functions, Lambda equivalents).

**Answer**: MVC controllers rely on runtime reflection for the action invoker, model binders, and filter pipeline. Native AOT trims unreferenced code aggressively and disallows runtime reflection on un-rooted members, which collides with MVC's pipeline. Microsoft's guidance is that **MVC is not (yet) fully AOT-compatible**; minimal APIs are first-class for AOT (with source generators emitting binding and OpenAPI metadata).

If AOT is on the roadmap (sub-100 ms cold start, ~30 MB binary), I start with minimal APIs. For traditional App Service or AKS workloads with no AOT goal, both are fine.

**Trade-off**: AOT also disables `System.Text.Json` features that require reflection (polymorphic serialization without source generator metadata). You may need `[JsonSerializable]` source-generator context classes.

**Real project**: A Payments webhook receiver migrated from controllers to minimal APIs with AOT, hitting ~80 ms cold start on Azure Container Apps vs ~600 ms before — meaningful when the same hook fires thousands of times per minute.

## Summary Checklist

- [ ] I can describe both controller and minimal API styles and produce a working example of each.
- [ ] I can explain what `[ApiController]` changes in MVC behavior.
- [ ] I can choose between MVC and minimal APIs based on size, cold start, AOT, and team conventions.
- [ ] I can use `TypedResults` and `Results<T1, T2>` to surface multiple response shapes.
- [ ] I can group endpoints with `MapGroup`, attach filters, and require authorization at the group level.
- [ ] I can keep endpoints thin by delegating to MediatR handlers or application services.
- [ ] I can wire validation in both styles (DataAnnotations + `[ApiController]` for MVC, FluentValidation endpoint filter for minimal).
- [ ] I can return correct status codes (`201 Created` + `Location`, `204 NoContent`, `409 Conflict`) in both styles.
- [ ] I can integrate OpenAPI for both styles and explain how metadata is inferred or declared.
- [ ] I can integration-test both styles with `WebApplicationFactory<Program>`.
