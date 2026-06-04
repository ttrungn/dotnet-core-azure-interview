# CQRS

## What It Is

**CQRS** (Command Query Responsibility Segregation) is an architectural pattern where the model used to **change** data (the *command* side) is separated from the model used to **read** data (the *query* side). Each side is free to use the data structures, queries, and storage that best fit its job — instead of forcing one ORM-mapped entity to serve both purposes.

In a typical CRUD ASP.NET Core service, a single `Order` class is used for inserts, updates, list pages, detail pages, and reports. As the system grows, that class becomes a kitchen sink: 40 navigation properties, lazy-loading exceptions, complex projections, and queries that load far more data than the screen actually needs. CQRS splits this in two:

- **Command side**: rich domain models that enforce invariants. Commands are imperative (`PlaceOrderCommand`, `CancelOrderCommand`, `CapturePaymentCommand`) and return either `void`, an ID, or a success/failure result.
- **Query side**: thin, read-optimized DTOs shaped exactly for a screen, report, or API contract. Queries are explicit (`GetOrderSummaryQuery`, `SearchPendingShipmentsQuery`) and return data.

```csharp
// Command — changes state, enforces rules
public sealed record CancelOrderCommand(Guid OrderId, string Reason) : IRequest;

// Query — returns a read-shaped DTO
public sealed record GetOrderSummaryQuery(Guid OrderId) : IRequest<OrderSummaryDto?>;
```

CQRS does **not** require two databases, event sourcing, or eventual consistency. The lightest form is "one database, two handler folders." Heavier variants use a denormalized read store (e.g., Cosmos DB, Elasticsearch) updated from domain events, but that is a deployment decision, not the pattern itself.

## Why It Exists

CRUD-style services age badly when:

- The same entity is reused for writes (`Order` with validation, business rules) and reads (`Order` projected to a JSON list endpoint with 200 rows).
- Read endpoints accidentally trigger lazy-loading or `N+1` queries because the entity was designed for the write path.
- Business rules end up scattered across controllers, services, and EF Core `SaveChanges` overrides because there is no clear "this changes state" boundary.
- Performance tuning the read side (caching, denormalization, indexes) breaks the write side, and vice versa.

CQRS was popularized by Greg Young around 2010, building on Bertrand Meyer's older **Command-Query Separation** principle ("methods should either do something or return something, never both"). It exists to:

1. Give the **write side** a place to own invariants, transactions, and consistency boundaries.
2. Give the **read side** freedom to be denormalized, cached, paginated, and shaped per use case.
3. Make every state change discoverable — every command is a named class, not an `UPDATE` buried in a service method.

## When To Use It

**Use CQRS for:**

- Services where read and write workloads have very different shapes — e.g., a marketplace where reads dominate (search, browse) and writes are complex (place order, refund).
- Domains with rich business rules — payments, fulfillment, inventory, subscription billing — where the write model needs to enforce invariants the read model does not care about.
- Systems that already use the **Mediator pattern** (MediatR) — CQRS handlers slot in naturally.
- Microservices that publish **integration events** after commands — pairing CQRS with the [outbox pattern](outbox-pattern.md) is a common combo.
- Reporting endpoints that need projections (joins, aggregations) the write model can't express cleanly.

**Do not use CQRS for:**

- Simple CRUD admin panels, internal tools, or a "table editor" UI. The overhead of command/query handlers, DTOs, and DI registration will exceed the value.
- Systems with one developer and a deadline measured in days.
- Anywhere a developer just needs to bump a status column. Don't write `MarkOrderShippedCommandHandler` if it's literally `UPDATE Orders SET Status=...`.
- Read-only services (BI dashboards) — there is no write side to separate from.

## Why It Is Important

CQRS gives you four things that compound as the system grows:

1. **Clear consistency boundaries.** Every command represents one atomic unit of work. The handler is the natural transaction scope and the natural place to enforce invariants (e.g., "cannot cancel a shipped order").
2. **Read performance freedom.** The query side can use Dapper for raw SQL, EF Core `AsNoTracking()` projections, or a separate read store. The write side keeps EF Core change tracking and domain logic.
3. **Discoverability.** A folder of `*Command.cs` files is a complete inventory of every state change in the service. New engineers learn the system by reading commands instead of reverse-engineering controllers.
4. **Pipeline behaviors.** With MediatR, cross-cutting concerns (validation, logging, transactions, retries, authorization) become reusable `IPipelineBehavior<TRequest, TResponse>` decorators applied to every command/query, not copy-pasted into each handler.

In a microservices / cloud context, CQRS pairs naturally with messaging. A command handler writes to its own DB and emits a domain event; downstream services subscribe via [Azure Service Bus](../azure/azure-service-bus.md). The read side may be eventually consistent — and that becomes a deliberate, visible design decision rather than an accident.

## How It's Used in C# / .NET

### 1. The standard stack: MediatR + ASP.NET Core + EF Core

[MediatR](https://github.com/jbogard/MediatR) is the de facto library for CQRS in .NET. It defines `IRequest<TResponse>`, `IRequestHandler<TRequest, TResponse>`, and an `IMediator` that dispatches requests to handlers.

```csharp
// Program.cs
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<Program>());

builder.Services.AddDbContext<OrdersDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sql")));

builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
```

### 2. A command and its handler

```csharp
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines,
    string IdempotencyKey) : IRequest<Guid>;

public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IInventoryService _inventory;
    private readonly IUnitOfWork _uow;
    private readonly ILogger<PlaceOrderHandler> _logger;

    public PlaceOrderHandler(
        IOrderRepository orders,
        IInventoryService inventory,
        IUnitOfWork uow,
        ILogger<PlaceOrderHandler> logger)
    {
        _orders = orders;
        _inventory = inventory;
        _uow = uow;
        _logger = logger;
    }

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        // 1. Idempotency check
        var existing = await _orders.FindByIdempotencyKeyAsync(cmd.IdempotencyKey, ct);
        if (existing is not null) return existing.Id;

        // 2. Domain invariants
        await _inventory.ReserveAsync(cmd.Lines, ct);
        var order = Order.Place(cmd.CustomerId, cmd.Lines, cmd.IdempotencyKey);

        // 3. Persist + emit domain event (handled by SaveChanges interceptor)
        _orders.Add(order);
        await _uow.SaveChangesAsync(ct);

        _logger.LogInformation("Placed order {OrderId} for customer {CustomerId}",
            order.Id, cmd.CustomerId);
        return order.Id;
    }
}
```

### 3. A query handler that bypasses the write model

```csharp
public sealed record GetOrderSummaryQuery(Guid OrderId) : IRequest<OrderSummaryDto?>;

public sealed class GetOrderSummaryHandler : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto?>
{
    private readonly OrdersDbContext _db;
    public GetOrderSummaryHandler(OrdersDbContext db) => _db = db;

    public Task<OrderSummaryDto?> Handle(GetOrderSummaryQuery q, CancellationToken ct) =>
        _db.Orders
            .AsNoTracking()
            .Where(o => o.Id == q.OrderId)
            .Select(o => new OrderSummaryDto(
                o.Id,
                o.CustomerName,
                o.Status.ToString(),
                o.Lines.Count,
                o.Total.Amount,
                o.Total.Currency))
            .SingleOrDefaultAsync(ct);
}
```

The query handler uses `AsNoTracking()` and a `Select` projection that loads only the columns the DTO needs. It never materializes the full `Order` aggregate.

### 4. Thin controllers / minimal APIs

```csharp
app.MapPost("/orders", async (PlaceOrderCommand cmd, IMediator mediator, CancellationToken ct) =>
{
    var id = await mediator.Send(cmd, ct);
    return Results.Created($"/orders/{id}", new { id });
});

app.MapGet("/orders/{id:guid}", async (Guid id, IMediator mediator, CancellationToken ct) =>
{
    var summary = await mediator.Send(new GetOrderSummaryQuery(id), ct);
    return summary is null ? Results.NotFound() : Results.Ok(summary);
});
```

Controllers translate HTTP to commands/queries and back. No business logic.

### 5. Pipeline behaviors for cross-cutting concerns

```csharp
public sealed class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0) throw new ValidationException(failures);
        return await next();
    }
}

builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
```

Every command now flows through validation → logging → transaction → handler without each handler knowing.

### 6. Heavier CQRS: separate read store

For a marketplace search endpoint, the write side stores normalized `Product`, `Inventory`, `Pricing` in SQL Server. After every write, a domain event is published via the [outbox pattern](outbox-pattern.md) to [Azure Service Bus](../azure/azure-service-bus.md). A projector reads events and updates a denormalized `ProductView` document in Cosmos DB. Search queries hit Cosmos DB directly — eventually consistent, but milliseconds of latency at p99.

### 7. CQRS vs Event Sourcing

CQRS and Event Sourcing are independent. You can do CQRS with a plain SQL Server, and you can do Event Sourcing without CQRS (though they pair well). Event Sourcing replaces "store current state" with "store the sequence of events"; CQRS just separates the read and write paths. Adopt CQRS first; only add Event Sourcing if you genuinely need a full audit history or temporal queries.

### 8. Quick reference

| Concept           | Type                       | Library                  | Returns                |
|-------------------|----------------------------|--------------------------|------------------------|
| Command           | `IRequest` / `IRequest<T>` | MediatR                  | void, ID, or Result    |
| Command handler   | `IRequestHandler<TReq,TRes>` | MediatR                | Persists changes       |
| Query             | `IRequest<TDto>`           | MediatR                  | Read-shaped DTO        |
| Query handler     | `IRequestHandler<TReq,TDto>` | MediatR                | Projection from DB     |
| Cross-cutting     | `IPipelineBehavior<TReq,TRes>` | MediatR              | Validation, logging, etc. |
| Validation        | `AbstractValidator<T>`     | FluentValidation         | Validation errors      |
| Read store        | Cosmos DB / Elasticsearch  | Azure SDK                | Denormalized documents |
| Eventing          | `INotification`            | MediatR (in-process) or Azure Service Bus | Domain events |

## Advantages

- **Single Responsibility per handler** — each handler does one thing, easy to test and reason about.
- **Performance** — query side can be tuned (raw SQL, projections, caching, read replicas) without touching the write model.
- **Discoverability** — `Commands/` folder is the full state-change inventory of the service.
- **Composable cross-cutting** — pipeline behaviors apply validation, logging, transactions, retries, telemetry to every request uniformly.
- **Pairs with messaging** — commands naturally produce [domain events](domain-events.md) and integration events.
- **Scalability options** — read and write sides can scale independently when split into separate stores.
- **Testability** — handlers are plain classes with constructor-injected dependencies; no HTTP, no DI container needed.

## Disadvantages

- **Boilerplate** — every operation needs a command/query class, handler, validator, and registration. Painful for trivial CRUD.
- **Indirection** — clicking "find references" on a command shows MediatR, not the handler. New developers must learn the pattern.
- **Two models to maintain** — when a field changes, you update both the write entity and the read DTO. Easy to forget one side.
- **Eventual consistency surprises** — if you split into read/write stores, reads can lag writes by milliseconds to seconds. UI must handle "I just saved this, why don't I see it?"
- **Risk of cargo-culting** — teams add CQRS to a CRUD app, then complain about boilerplate without getting any of the benefit.
- **Transaction scoping is less obvious** — a command that publishes events needs the [outbox pattern](outbox-pattern.md) to keep DB writes and events atomic.

## Common Mistakes

### 1. Treating Commands as DTOs and reusing them across endpoints

```csharp
// BUG: One "OrderCommand" used for both create and update — fields become nullable, intent is lost
public record OrderCommand(Guid? Id, Guid CustomerId, IReadOnlyList<OrderLine>? Lines, string? Status);
```

**Fix**: Each use case gets its own command. Names are imperative and specific:

```csharp
public sealed record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<OrderLine> Lines) : IRequest<Guid>;
public sealed record CancelOrderCommand(Guid OrderId, string Reason) : IRequest;
public sealed record MarkOrderShippedCommand(Guid OrderId, string TrackingNumber) : IRequest;
```

### 2. Returning entire domain entities from query handlers

```csharp
// BUG: returns the full aggregate, including navigation properties used only by the write side
public Task<Order> Handle(GetOrderQuery q, CancellationToken ct) =>
    _db.Orders.Include(o => o.Lines).Include(o => o.Customer).FirstOrDefaultAsync(o => o.Id == q.Id, ct);
```

**Fix**: Return a flat DTO with only the fields the caller needs.

```csharp
public Task<OrderSummaryDto?> Handle(GetOrderQuery q, CancellationToken ct) =>
    _db.Orders.AsNoTracking()
       .Where(o => o.Id == q.Id)
       .Select(o => new OrderSummaryDto(o.Id, o.Status.ToString(), o.Total.Amount))
       .SingleOrDefaultAsync(ct);
```

### 3. Doing both read and write in one handler

```csharp
// BUG: This is a query handler that also writes — violates Command-Query Separation
public async Task<OrderDto> Handle(GetOrcCachedQuery q, CancellationToken ct)
{
    var order = await _db.Orders.FindAsync(q.Id, ct);
    order!.LastViewedAt = _clock.GetUtcNow(); // mutation in a query!
    await _db.SaveChangesAsync(ct);
    return new OrderDto(order);
}
```

**Fix**: Either issue a separate `RecordOrderViewedCommand`, or emit a domain event and let a separate handler update the timestamp.

### 4. Putting business rules in query handlers

Query handlers should be pure projections. If you need to enforce a rule ("only show orders the user owns"), do it as an explicit `WHERE` filter or in a pipeline behavior, not as business logic mixed into the projection.

### 5. Skipping pipeline behaviors and copy-pasting validation/logging into every handler

```csharp
// BUG: every handler repeats the same validation/logging boilerplate
public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    if (cmd.CustomerId == Guid.Empty) throw new ValidationException(...);
    _logger.LogInformation("Handling PlaceOrder");
    // actual logic
}
```

**Fix**: Move validation to a `ValidationBehavior` and logging to a `LoggingBehavior` — both registered as `IPipelineBehavior<,>`. Handlers become focused.

### 6. Mixing CQRS jargon with simple CRUD endpoints

Don't write `GetUserByIdQuery` + `GetUserByIdHandler` for a single-table read. The MediatR layer adds nothing — a direct `_db.Users.FindAsync(id)` in a controller is shorter, clearer, and equivalent.

### 7. Forgetting transactional consistency when emitting events

```csharp
// BUG: DB commits, then Service Bus send throws — event is lost
await _db.SaveChangesAsync(ct);
await _serviceBus.SendAsync(new OrderPlaced(order.Id), ct); // may fail
```

**Fix**: Use the [outbox pattern](outbox-pattern.md) — write the event to an `Outbox` table in the same transaction as the order, and let a publisher background worker drain it.

## Best Practices

- **Name commands as imperatives, queries as questions.** `PlaceOrderCommand`, `GetOrderSummaryQuery`.
- **One handler per command/query.** Never share a handler across two commands.
- **Keep handlers thin.** Orchestration only: load → call domain method → save. Business rules live in the aggregate, not in the handler.
- **Use FluentValidation + a `ValidationBehavior`** for input validation. Reject early with structured `ProblemDetails`.
- **Project in EF Core with `Select`** for query handlers — never materialize the full aggregate.
- **Idempotency keys on commands** that may be retried (HTTP, queue triggers). Store the key in the database with a unique index.
- **Combine with the outbox pattern** when commands emit integration events.
- **Add a `TransactionBehavior`** for write-side commands so the unit of work boundary is consistent.
- **Don't use CQRS as a synonym for MediatR.** You can do CQRS with hand-written `IXxxCommandHandler` interfaces if MediatR is overkill.
- **Skip CQRS for trivial CRUD.** Use CQRS where the business value is real (rich domain, performance differentiation, audit, eventing).

## Related Concepts

- **Mediator pattern** — the dispatcher that routes commands/queries to handlers; MediatR is the standard implementation.
- **[Domain events](domain-events.md)** — in-process events raised by aggregates, dispatched on `SaveChanges`.
- **[Outbox pattern](outbox-pattern.md)** — atomic publication of integration events alongside commands.
- **Event Sourcing** — store the sequence of events instead of current state; orthogonal to but commonly paired with CQRS.
- **[Domain-driven design](domain-driven-design.md)** — CQRS fits naturally with aggregates, value objects, and bounded contexts.
- **[Unit of Work](unit-of-work.md)** — each command handler defines a unit of work boundary.
- **[Repositories](repositories.md)** — the write-side abstraction over persistence; query handlers often bypass them for performance.
- **[Application services](application-services.md)** — commands and queries are a structured form of application service.
- **[Aggregates](aggregates.md)** — the write side; queries usually don't load full aggregates.

## Real-World Usage

### E-commerce checkout (ASP.NET Core + MediatR + EF Core + Service Bus)

- Commands: `PlaceOrderCommand`, `CapturePaymentCommand`, `RefundOrderCommand`, `CancelOrderCommand`, `MarkOrderShippedCommand`.
- Queries: `GetOrderSummaryQuery`, `GetCustomerOrderHistoryQuery`, `SearchOrdersForOpsDashboardQuery`.
- Handlers register via `AddMediatR(...)`. Pipeline behaviors apply validation (FluentValidation), logging, and an `IUnitOfWork` transaction.
- After `PlaceOrderHandler` commits, a domain event `OrderPlaced` is appended to the outbox table. A `BackgroundService` reads outbox rows and sends `OrderPlacedIntegrationEvent` to a Service Bus topic.

### Banking / payments

The command side enforces double-entry bookkeeping invariants (debits = credits, no overdraft on a savings account). The query side projects to a denormalized `AccountStatementView` updated from `MoneyTransferred` events. Statements load in milliseconds even though the write model is heavily normalized.

### Insurance claims

Each claim transition is a named command (`SubmitClaimCommand`, `AssignAdjusterCommand`, `ApproveSettlementCommand`). Auditors can list every command class to verify which transitions exist. Read endpoints feed agent dashboards via Dapper queries against denormalized views.

### Healthcare booking

Write side: `BookAppointmentCommand` enforces "no two appointments overlap for the same doctor." Read side: a calendar UI loads 1,000 appointments per week through a single projection query — never through the write model.

## Code Example — Before and After

### Before: CRUD controller doing everything

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly OrdersDbContext _db;
    private readonly IEmailSender _email;

    public OrdersController(OrdersDbContext db, IEmailSender email)
    {
        _db = db;
        _email = email;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Order order)
    {
        if (order.Lines == null || order.Lines.Count == 0)
            return BadRequest("No lines");

        foreach (var line in order.Lines)
        {
            var product = await _db.Products.FindAsync(line.ProductId);
            if (product == null) return BadRequest($"Unknown product {line.ProductId}");
            if (product.Stock < line.Quantity) return BadRequest("Out of stock");
            product.Stock -= line.Quantity;
        }

        order.Status = "Placed";
        order.CreatedAt = DateTime.UtcNow;
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();

        await _email.SendAsync(order.CustomerEmail, "Order placed", $"Order {order.Id} placed");
        return CreatedAtAction(nameof(Get), new { id = order.Id }, order);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var order = await _db.Orders
            .Include(o => o.Lines)
            .Include(o => o.Customer)
            .FirstOrDefaultAsync(o => o.Id == id);
        return order is null ? NotFound() : Ok(order);
    }
}
```

Problems:
- Business rules (stock check, status assignment) are in the controller.
- The `GET` loads the entire aggregate with navigation properties, then serializes everything (incl. customer PII).
- Email send is in the request critical path — if SMTP is slow, the user waits.
- Reusing the `Order` entity as both API input and DB entity allows clients to set `Status`, `CreatedAt` directly.

### After: CQRS with MediatR

```csharp
// ---- Command ----
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines,
    string IdempotencyKey) : IRequest<Guid>;

public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IInventoryService _inventory;
    private readonly IUnitOfWork _uow;

    public PlaceOrderHandler(IOrderRepository orders, IInventoryService inventory, IUnitOfWork uow)
    {
        _orders = orders;
        _inventory = inventory;
        _uow = uow;
    }

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var existing = await _orders.FindByIdempotencyKeyAsync(cmd.IdempotencyKey, ct);
        if (existing is not null) return existing.Id;

        await _inventory.ReserveAsync(cmd.Lines, ct);
        var order = Order.Place(cmd.CustomerId, cmd.Lines, cmd.IdempotencyKey);
        // Aggregate raises OrderPlaced domain event; outbox interceptor persists it.

        _orders.Add(order);
        await _uow.SaveChangesAsync(ct);
        return order.Id;
    }
}

public sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.ProductId).NotEmpty();
            line.RuleFor(l => l.Quantity).GreaterThan(0);
        });
        RuleFor(x => x.IdempotencyKey).NotEmpty().MaximumLength(64);
    }
}

// ---- Query ----
public sealed record GetOrderSummaryQuery(Guid OrderId) : IRequest<OrderSummaryDto?>;
public sealed record OrderSummaryDto(Guid Id, string Status, int LineCount, decimal Total, string Currency);

public sealed class GetOrderSummaryHandler : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto?>
{
    private readonly OrdersDbContext _db;
    public GetOrderSummaryHandler(OrdersDbContext db) => _db = db;

    public Task<OrderSummaryDto?> Handle(GetOrderSummaryQuery q, CancellationToken ct) =>
        _db.Orders.AsNoTracking()
            .Where(o => o.Id == q.OrderId)
            .Select(o => new OrderSummaryDto(
                o.Id, o.Status.ToString(), o.Lines.Count, o.Total.Amount, o.Total.Currency))
            .SingleOrDefaultAsync(ct);
}

// ---- Controller ----
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Place(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var id = await _mediator.Send(cmd, ct);
        return CreatedAtAction(nameof(Get), new { id }, new { id });
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id, CancellationToken ct)
    {
        var dto = await _mediator.Send(new GetOrderSummaryQuery(id), ct);
        return dto is null ? NotFound() : Ok(dto);
    }
}
```

Now:
- Business rules live in `Order.Place(...)` (the aggregate).
- The query handler returns exactly the DTO the screen needs.
- Validation is centralized in a `PlaceOrderValidator` and applied automatically by the pipeline behavior.
- The email send is triggered by an `OrderPlaced` domain event handler, not the controller — and pushed through the outbox so it survives crashes.
- Each handler can be unit-tested in isolation.

## Interview Questions and Answers

### 1. Walk me through CQRS and the problem it solves.

**Why this matters**: It tests whether you can articulate the *value* of the pattern rather than parrot a definition.

**Answer**: CQRS separates the model used to change state from the model used to read state. The write model owns invariants, transactions, and business rules. The read model is shaped per use case — usually flat DTOs projected directly from the database, sometimes from a denormalized read store. The problem it solves is that in a CRUD-style service, the same entity becomes both an over-validated write target and a slow, over-loaded read source. Splitting them lets each side evolve independently.

**Trade-off**: You add classes (command, handler, query, DTO) for every operation. For a five-table admin tool, that's pure overhead. For a payments service with rich rules, it pays off in test coverage and clarity within a quarter.

**Real project**: A subscription billing service where `ChargeSubscriptionCommand` enforces prorating rules, while `GetSubscriptionDashboardQuery` returns a denormalized DTO joining 6 tables. The write path is tested with unit tests against the aggregate; the read path is tested with integration tests against the SQL projection.

### 2. When is CQRS overkill?

**Why this matters**: A senior engineer knows when *not* to apply a pattern. Junior engineers often apply CQRS uniformly, then complain about boilerplate.

**Answer**: CQRS is overkill when:
- The domain is simple CRUD with no real business rules — a single-table editor doesn't need a command handler.
- The team is small and the deadline is short — the boilerplate slows initial delivery.
- Reads and writes have the same shape and same load profile — there's nothing to gain from separating them.
- The team doesn't have discipline to keep handlers thin — without that, you end up with a fatter version of the original mess.

**Trade-off**: Sometimes it's worth a small upfront cost in a new service even if today it's CRUD, because the service will grow. But always be specific about *why* — "because the domain will get complex" is not enough; you need a concrete reason.

**Real project**: An internal HR tool with 30 tables and 5 users — written as plain controllers + EF Core, shipped in a week. Adding MediatR + CQRS would have added 200 classes for no benefit.

### 3. Does CQRS require two databases?

**Why this matters**: A common misconception. Candidates who say "yes" reveal they've copied the diagram without understanding the principle.

**Answer**: No. CQRS is about separating the **models** for reads and writes. They can share a single database. The lightest form is one database with two folders of handlers — write handlers use the aggregate + EF Core change tracking, read handlers do `AsNoTracking()` projections. You only introduce a separate read store (Cosmos DB, Elasticsearch, Redis) when you need it for performance, scale, or different query patterns, and that decision is independent of CQRS itself.

**Trade-off**: A separate read store buys you faster reads and independent scaling, but costs you eventual consistency, projector code to keep the read store fresh, and double the operational surface.

**Real project**: An order service started with one SQL database. After 18 months, the search endpoint was 800ms p99 due to joins across 12 tables. Added a Cosmos DB read store fed by Service Bus events. Search dropped to 30ms p99; total infrastructure cost went up ~30%; business agreed because the conversion rate impact was much larger.

### 4. How does CQRS pair with the outbox pattern?

**Why this matters**: Tests whether the candidate connects CQRS to the messaging concerns that come with it.

**Answer**: A command often needs to publish an integration event after it commits — e.g., `PlaceOrderCommand` should result in an `OrderPlaced` event going to Service Bus so fulfillment, billing, and notifications can react. The naive approach (`SaveChanges` then `serviceBus.SendAsync`) loses events when the second call fails after the first. The [outbox pattern](outbox-pattern.md) solves this: the handler writes both the order rows and the outbox row in the same DB transaction, and a background worker drains the outbox and publishes to Service Bus. Combined with [domain events](domain-events.md), the aggregate can declaratively raise `OrderPlaced`, an EF Core `SaveChanges` interceptor maps it to an outbox row, and the rest is automatic.

**Trade-off**: The outbox adds a background worker, dedup logic on the consumer side, and a few milliseconds of publish latency. It's the price of correctness.

**Real project**: An order placement service emits 4 integration events per order (`OrderPlaced`, `InventoryReserved`, `PaymentRequested`, `CustomerNotificationRequested`). All four are written to the outbox in the same transaction. A `BackgroundService` reads outbox rows, sends to Service Bus topics, and marks them dispatched. Consumers are idempotent.

### 5. Walk me through a MediatR pipeline behavior for validation.

**Why this matters**: Tests whether the candidate understands how to apply cross-cutting concerns uniformly across all handlers.

**Answer**: A pipeline behavior is a decorator that wraps every `IRequestHandler`. MediatR builds a pipeline of behaviors around the handler at runtime. A validation behavior resolves all `IValidator<TRequest>` registered for the request type, runs them in parallel, collects errors, and throws if any failed before invoking `next()`. Registering it with `services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>))` makes it apply to every command and query automatically. The handler stays focused on business logic.

**Trade-off**: Pipeline behaviors are open generics — easy to register, but stack traces and "find references" become harder for new developers. Document the pipeline order somewhere visible.

**Real project**: A 200-handler service uses 5 behaviors: Logging → Validation → Authorization → Transaction → Handler. Adding a new behavior (e.g., automatic OpenTelemetry span creation) is one DI registration and applies to every handler.

### 6. How would you keep command handlers from doing too much?

**Why this matters**: Even with CQRS, handlers can balloon into 400-line god classes.

**Answer**: Keep three responsibilities per handler: (1) load the aggregate(s) the operation needs, (2) call a single domain method, (3) save and let the unit of work commit. If the handler is doing branching logic, calling multiple services, or shaping output, push that logic down into the aggregate (domain method) or up into the API layer (HTTP shaping). Cross-cutting concerns go into pipeline behaviors. If the handler has more than ~4 dependencies, split the command into smaller commands.

**Trade-off**: Pushing logic into aggregates requires real domain modeling. If the team writes anemic entities and rich services, the handler will always be fat. The fix is at the domain layer, not the handler.

**Real project**: A `PlaceOrderHandler` that grew to 300 lines was refactored: stock check moved into `Order.Place(...)`, payment authorization moved into `Payment` aggregate, notification became a domain event. The handler dropped to 25 lines.

### 7. What's the difference between a command, a domain event, and an integration event?

**Why this matters**: These three concepts are often conflated. Mixing them up causes coupling, lost messages, or distributed deadlocks.

**Answer**:
- A **command** is an instruction to do something — `PlaceOrderCommand`. It expects a single handler that may succeed or fail. Sent in-process via MediatR, or across services via a queue.
- A **[domain event](domain-events.md)** is something that *already happened* inside one aggregate — `OrderPlaced`. Raised by the aggregate, dispatched in-process (e.g., on `SaveChanges`) to zero or more handlers within the same service. Same transaction, same process.
- An **integration event** is something that happened, published *across* service boundaries — `OrderPlacedIntegrationEvent` sent to a Service Bus topic. Other services subscribe. Eventual consistency by design.

The transition from domain event to integration event usually happens at the outbox: a domain event handler maps the in-process event to an integration event row in the outbox.

**Trade-off**: Domain events keep coupling inside one service. Integration events allow cross-service decoupling but require schema versioning, idempotent consumers, and DLQ handling.

**Real project**: An order service raises `OrderPlaced` (domain event) → sends a thank-you email synchronously, then writes `OrderPlacedIntegrationEvent` to the outbox → published to Service Bus → consumed by fulfillment, billing, and the data warehouse.

### 8. How do you test CQRS handlers?

**Why this matters**: Tests reveal whether handlers are truly decoupled from infrastructure.

**Answer**: Command handlers get unit tests with fake repositories and unit-of-work, exercising business rule branches (declined payment, out-of-stock, idempotency hit). Query handlers get integration tests against a real database (SQLite in-memory or a test SQL Server instance), because the LINQ-to-SQL translation is the thing you're verifying. Pipeline behaviors get their own unit tests with a stubbed handler delegate. Don't write a single "end-to-end test through MediatR" for every handler — that's slow and doesn't add value beyond the controller integration test.

**Trade-off**: Query handler tests against a real DB are slower than pure unit tests. Run them in parallel against ephemeral databases or use SQLite in-memory for the fast feedback loop.

**Real project**: A 200-handler service has ~600 unit tests for command handlers (3-4 branches each), ~200 integration tests for query handlers, plus contract tests for HTTP shapes. The full suite runs in under 90 seconds.

## Summary Checklist

- [ ] I can define CQRS as separating write models from read models, not as "use MediatR."
- [ ] I can name when CQRS pays off (rich domain, performance differentiation) and when it's overkill (simple CRUD).
- [ ] I can write a `IRequest` command + `IRequestHandler` and explain the transaction boundary.
- [ ] I can write a query handler that uses `AsNoTracking()` + `Select` projection and never materializes the full aggregate.
- [ ] I can register and explain a MediatR `IPipelineBehavior` for validation, logging, or transactions.
- [ ] I can explain how CQRS pairs with the outbox pattern and domain events.
- [ ] I can distinguish a command, a domain event, and an integration event.
- [ ] I can recognize and refactor a god-handler with too many dependencies.
- [ ] I can decide between a single database vs. a separate read store, with concrete trade-offs.
- [ ] I can test command handlers as pure units and query handlers as DB integration tests.
