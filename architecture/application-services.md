# Application Services

## What It Is

An **application service** is a class that orchestrates one **use case** of your system — *"place an order," "cancel a subscription," "refund a payment."* It sits between the API surface (controllers, Azure Function triggers, Service Bus handlers) and the **domain** + **infrastructure** layers. Its job is to:

1. Accept a command or request DTO from the outside.
2. Load the necessary aggregates through repositories.
3. Invoke domain methods that enforce business invariants.
4. Coordinate infrastructure calls (payment gateway, email, message bus) through interfaces.
5. Commit the Unit of Work.
6. Return a result (DTO, `Result<T>`, or void).

Application services hold **no business rules of their own**. They are stage directors. If you find domain logic inside one (`if (order.Total > 1000) order.Status = ...`), the rule belongs *inside* the `Order` aggregate.

```csharp
public sealed class ConfirmOrderHandler : IRequestHandler<ConfirmOrderCommand, Result>
{
    private readonly IOrderRepository _orders;
    private readonly IPaymentGateway _payments;
    private readonly IUnitOfWork _uow;

    public ConfirmOrderHandler(IOrderRepository orders, IPaymentGateway payments, IUnitOfWork uow)
    {
        _orders = orders;
        _payments = payments;
        _uow = uow;
    }

    public async Task<Result> Handle(ConfirmOrderCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();

        var intent = await _payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!intent.Succeeded) return Result.PaymentFailed(intent.Error);

        var outcome = order.Confirm(intent);   // domain enforces invariants
        if (!outcome.IsSuccess) return outcome;

        await _uow.SaveChangesAsync(ct);       // commit atomically
        return Result.Ok();
    }
}
```

In the .NET ecosystem, application services are typically expressed as **MediatR command handlers** (`IRequestHandler<TCommand, TResult>`), but the pattern existed long before MediatR — any class whose methods correspond 1:1 with use cases qualifies.

## Why It Exists

In Eric Evans' DDD terminology, the application layer was introduced to answer one question: *"where does the code that connects an HTTP request to a domain rule live?"*

The historic alternatives were both painful:

- **Fat controllers.** Business logic, EF queries, payment SDK calls, and email sending all stuffed into a single ASP.NET MVC action. Untestable, untraceable, prone to copy-paste duplication across endpoints.
- **Smart entities ("Active Record").** Domain objects that knew how to load themselves from the database and call external services. They violated the Single Responsibility Principle and were impossible to unit-test in isolation.

Application services exist to give use-case orchestration its **own home** — a thin, focused class per business operation. That separation drives three things:

1. The domain stays pure (no EF Core, no `HttpClient`, no `IConfiguration`).
2. The transport layer (controllers, Functions) becomes trivial — it only translates HTTP into a command.
3. Cross-cutting concerns (validation, logging, transactions, retries) live in one place and can be applied uniformly via MediatR pipeline behaviors.

## When To Use It

**Use application services when:**

- The system has more than a handful of use cases and you want each one to be a discoverable, testable unit.
- You apply **DDD / Clean Architecture**: the domain layer must not know about HTTP or storage.
- You use **MediatR** or a similar dispatcher, and naturally express each command/query as a handler.
- You need a place to apply **cross-cutting behaviors** (validation, logging, retries, transactions) via decorators or pipeline behaviors.

**Skip them when:**

- You are building a small CRUD admin tool. Minimal API endpoints calling `DbContext` directly are perfectly fine.
- The "use case" is a single-line query (`GET /api/products/{id}`). Use a controller or a read-only query service.
- The team is two engineers and one service. The ceremony exceeds the benefit.

## Why It Is Important

In production .NET / Azure systems, application services drive five concrete outcomes:

1. **A clear, discoverable list of business operations.** Open `src/Orders.Application/Commands/` and you see every action the system can perform: one folder, one file per use case. Onboarding a new engineer takes hours, not days.
2. **Testability without a web server.** A handler is a plain class. You inject fakes for the repository, payment gateway, and UoW, and exercise every branch (declined card, idempotent replay, concurrency conflict) as fast unit tests.
3. **Uniform cross-cutting behavior.** Validation, logging, authorization, retries, and transactions are added via MediatR pipeline behaviors — one place, every handler. No "I forgot to validate that one endpoint" bugs.
4. **Reusable across transports.** The same `ConfirmOrderHandler` runs from an HTTP controller, an Azure Function Service Bus trigger, and a Hangfire job. The handler does not know who called it.
5. **A predictable place for the Unit of Work boundary.** Exactly one `SaveChangesAsync` per handler. Atomicity becomes obvious and easy to review.

## How It's Used in C# / .NET

### 1. Project layout (Clean Architecture)

```
src/
  Orders.Domain/           // Order, OrderLine, IOrderRepository (no EF Core)
  Orders.Application/      // Commands, queries, handlers, validators
    Commands/
      PlaceOrder/
        PlaceOrderCommand.cs
        PlaceOrderHandler.cs
        PlaceOrderValidator.cs
      ConfirmOrder/
      CancelOrder/
    Queries/
  Orders.Infrastructure/   // OrdersDbContext, repositories, Stripe adapter
  Orders.Api/              // Controllers, Program.cs
```

`Orders.Application` references `Orders.Domain` and `MediatR` only. It compiles without EF Core or ASP.NET Core.

### 2. Define the command

```csharp
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines,
    string IdempotencyKey
) : IRequest<Result<OrderId>>;
```

Commands are immutable records carrying the data needed to execute the use case. They never contain behavior.

### 3. The handler

```csharp
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Result<OrderId>>
{
    private readonly ICustomerRepository _customers;
    private readonly IInventoryRepository _inventory;
    private readonly IOrderRepository _orders;
    private readonly IIdempotencyStore _idempotency;
    private readonly IOutbox _outbox;
    private readonly IUnitOfWork _uow;
    private readonly ILogger<PlaceOrderHandler> _logger;

    public PlaceOrderHandler(
        ICustomerRepository customers,
        IInventoryRepository inventory,
        IOrderRepository orders,
        IIdempotencyStore idempotency,
        IOutbox outbox,
        IUnitOfWork uow,
        ILogger<PlaceOrderHandler> logger)
    {
        _customers = customers;
        _inventory = inventory;
        _orders = orders;
        _idempotency = idempotency;
        _outbox = outbox;
        _uow = uow;
        _logger = logger;
    }

    public async Task<Result<OrderId>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        if (await _idempotency.TryGetExistingAsync<OrderId>(cmd.IdempotencyKey, ct) is { } existing)
            return Result<OrderId>.Ok(existing);

        var customer = await _customers.GetByIdAsync(new CustomerId(cmd.CustomerId), ct);
        if (customer is null) return Result<OrderId>.NotFound("Customer not found");

        var inventory = await _inventory.GetForSkusAsync(cmd.Lines.Select(l => l.Sku).ToArray(), ct);
        var reservation = inventory.TryReserve(cmd.Lines);
        if (!reservation.IsSuccess) return Result<OrderId>.Failure(reservation.Error);

        var order = Order.Create(customer.Id, cmd.Lines);  // domain factory enforces invariants
        await _orders.AddAsync(order, ct);

        _outbox.Enqueue(new OrderPlacedEvent(order.Id.Value, customer.Id.Value, order.Total.Amount));

        await _idempotency.RecordAsync(cmd.IdempotencyKey, order.Id, ct);
        await _uow.SaveChangesAsync(ct);

        _logger.LogInformation("Order {OrderId} placed for customer {CustomerId}", order.Id, customer.Id);
        return Result<OrderId>.Ok(order.Id);
    }
}
```

Notice what the handler *does not* do: it does not validate request shape (FluentValidation does that), it does not log the start of the request (a pipeline behavior does that), it does not retry on transient errors (Polly + behavior does that). It is small and linear.

### 4. The controller becomes trivial

```csharp
[ApiController, Route("api/orders")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Place(
        [FromBody] PlaceOrderRequest req,
        [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
        CancellationToken ct)
    {
        var cmd = new PlaceOrderCommand(req.CustomerId, req.Lines, idempotencyKey);
        var result = await mediator.Send(cmd, ct);
        return result.ToActionResult();
    }
}
```

### 5. MediatR pipeline behaviors for cross-cutting concerns

```csharp
// Validation
public sealed class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        foreach (var v in _validators)
        {
            var result = await v.ValidateAsync(request, ct);
            if (!result.IsValid) throw new ValidationException(result.Errors);
        }
        return await next();
    }
}

// Logging + telemetry
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<TRequest> logger) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        logger.LogInformation("Handling {Request}", typeof(TRequest).Name);
        try { return await next(); }
        finally { logger.LogInformation("{Request} completed in {Elapsed}ms", typeof(TRequest).Name, sw.ElapsedMilliseconds); }
    }
}

// Registration
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly));
builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
```

Add a new handler and these behaviors apply automatically. Add a new behavior and every handler gets it.

### 6. FluentValidation for request shape

```csharp
public sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.IdempotencyKey).NotEmpty().Length(8, 128);
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.Sku).NotEmpty();
            line.RuleFor(l => l.Quantity).GreaterThan(0);
        });
    }
}
```

Shape validation lives outside the handler. The handler only sees commands that are structurally valid.

## Advantages

- **Clear use-case granularity** — one class, one operation.
- **Trivially testable** — no HTTP, no DB, just inject fakes.
- **Cross-cutting concerns applied uniformly** via pipeline behaviors.
- **Transport-agnostic** — same handler runs from a controller, a Function, or a Worker.
- **Predictable transaction boundary** — one handler = one Unit of Work.
- **Onboarding accelerator** — new engineers learn the system by browsing one folder.

## Disadvantages

- **Ceremony for small projects** — five files for a one-line endpoint is wasteful.
- **MediatR is an indirection** — stack traces are longer; performance overhead is small but nonzero.
- **Risk of "anemic" handlers** — teams can drift into putting logic that belongs in the domain inside handlers.
- **Cross-handler workflows** are awkward — a handler calling another handler couples them; sagas are usually the right answer.

## Common Mistakes

### 1. Business rules inside the handler

**Problem:** discount logic, status transitions, and pricing rules live in the handler. The `Order` aggregate is a dumb data bag.

```csharp
// ❌
public async Task<Result> Handle(ConfirmOrderCommand cmd, CancellationToken ct)
{
    var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
    if (order.Status != OrderStatus.Pending) return Result.Conflict(); // rule here
    if (order.Lines.Count == 0) return Result.Invalid("Empty");        // rule here
    order.Status = OrderStatus.Confirmed;                              // rule here
    order.ConfirmedAt = DateTimeOffset.UtcNow;                         // rule here
    ...
}
```

**Fix:** push every rule into `Order.Confirm()`. The handler just calls it.

```csharp
// ✅
var outcome = order.Confirm(intent);
if (!outcome.IsSuccess) return outcome;
```

### 2. Handlers calling other handlers via MediatR

**Problem:** `PlaceOrderHandler` sends a `ReserveInventoryCommand` to MediatR. Each handler opens its own scope and Unit of Work. Atomicity is lost; observability is muddled.

**Fix:** if two handlers genuinely share work, extract a **domain service** that both call. For long-running multi-step workflows, use a **saga** (MassTransit, NServiceBus). MediatR is for in-process dispatch within one use case.

### 3. Returning EF Core entities from handlers

**Problem:** the handler returns an `Order` entity with lazy navigation properties; the controller serializes it and triggers more queries during JSON serialization (`N+1`).

**Fix:** handlers return **DTOs** or `Result<TDto>`. The mapping from entity to DTO is explicit and lives in the application layer.

### 4. Putting validation in the handler

**Problem:** `if (string.IsNullOrEmpty(cmd.Email)) return Result.Invalid(...)` scattered across every handler.

**Fix:** request shape validation goes to **FluentValidation** plus a `ValidationBehavior`. Domain invariants stay in aggregates. The handler does neither.

### 5. Forgetting `CancellationToken`

**Problem:** the handler ignores `ct`. A client disconnects; the server keeps charging Stripe and writing to the database.

**Fix:** pass `ct` to every async call — repositories, gateway, `SaveChangesAsync`. Azure Functions and ASP.NET Core both cancel `ct` on shutdown or client disconnect.

### 6. Multiple Units of Work in one handler

**Problem:** the handler calls `_uow.SaveChangesAsync` after each step "to commit progressively." Partial failures leave the database inconsistent.

**Fix:** one `SaveChangesAsync` at the end. If you genuinely need progressive commits (e.g., a long batch job), split the work into separate handlers and use a saga or outbox to coordinate them.

## Best Practices

- **One handler per use case.** Name it after the verb: `PlaceOrderHandler`, `CancelSubscriptionHandler`.
- **Handlers orchestrate; aggregates enforce rules.** If a handler grows past ~50 lines of logic, look for a domain method that wants to exist.
- **Inject interfaces only.** No `DbContext`, no `HttpClient`, no `ILogger<DbContext>` — only domain and application abstractions.
- **One `SaveChangesAsync` per handler.** The handler defines the transactional boundary.
- **Pass `CancellationToken` everywhere.** It is free; forgetting it costs production money.
- **Use FluentValidation for request shape; aggregates for domain rules.** Never both for the same rule.
- **Return `Result<T>`, not exceptions, for expected failures** (validation, not-found, conflict). Reserve exceptions for genuinely exceptional cases.
- **Apply cross-cutting concerns through pipeline behaviors**: validation, logging, telemetry, retries, transactions.
- **Keep handlers stateless and short-lived.** Register as `Scoped` to share `DbContext` with repositories in the same request.
- **Test handlers in isolation** with in-memory fakes. Reserve integration tests for repositories and the HTTP edge.

## Related Concepts

- [architecture/clean-architecture.md](clean-architecture.md) — application services live in the application layer.
- [architecture/cqrs.md](cqrs.md) — commands and queries are application service variants.
- [architecture/domain-driven-design.md](domain-driven-design.md) — the source of the application/domain split.
- [architecture/repositories.md](repositories.md) — handlers depend on repository interfaces.
- [architecture/unit-of-work.md](unit-of-work.md) — exactly one `SaveChangesAsync` per handler.
- [architecture/saga-pattern.md](saga-pattern.md) — for workflows that span multiple use cases.
- [aspnet-core/controllers-and-minimal-apis.md](../aspnet-core/controllers-and-minimal-apis.md) — the transport that hands requests to handlers.
- [aspnet-core/input-validation.md](../aspnet-core/input-validation.md) — FluentValidation + pipeline behaviors.

## Real-World Usage

### ASP.NET Core ordering API

Controllers are 3-line shells: bind the request, send the command via MediatR, translate `Result<T>` to an HTTP response. All business orchestration lives in `Orders.Application`. The `OrdersController` has no `DbContext`, no `IStripeClient`, no business knowledge — and is easy to keep that way during code review.

### Azure Functions worker for asynchronous workflows

```csharp
public sealed class OrderShippedFunction(IMediator mediator)
{
    [Function(nameof(OrderShippedFunction))]
    public Task Run(
        [ServiceBusTrigger("order-shipped", Connection = "ServiceBus")] OrderShippedMessage msg,
        CancellationToken ct) =>
        mediator.Send(new MarkOrderShippedCommand(msg.OrderId, msg.TrackingNumber, msg.ShippedAt), ct);
}
```

The same handler that powers an admin "Mark Shipped" button also runs from the Service Bus subscriber. One use case, one tested handler, two transports.

### Cross-cutting transaction behavior

```csharp
public sealed class TransactionBehavior<TRequest, TResponse>(OrdersDbContext db, ILogger<TRequest> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (db.Database.CurrentTransaction is not null) return await next();
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        try
        {
            var result = await next();
            await tx.CommitAsync(ct);
            return result;
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

Every command handler is wrapped in a transaction without needing to think about it.

### Telemetry and Application Insights

`LoggingBehavior` writes a structured log per request with `RequestName`, `ElapsedMs`, and `Outcome`. Application Insights picks this up and you get per-use-case latency and failure dashboards essentially for free.

### Multi-tenant SaaS

A `TenantContextBehavior` extracts the tenant from the JWT, stamps it on the `ITenantContext` scoped service, and ensures every handler runs with tenant filtering enforced. Handlers themselves remain unaware of tenancy.

## Code Example — Before and After

### Before — fat controller

```csharp
[ApiController, Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly OrdersDbContext _db;
    private readonly IStripeClient _stripe;
    private readonly IEmailSender _email;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(OrdersDbContext db, IStripeClient stripe, IEmailSender email, ILogger<OrdersController> logger)
    {
        _db = db; _stripe = stripe; _email = email; _logger = logger;
    }

    [HttpPost("{id:guid}/confirm")]
    public async Task<IActionResult> Confirm(Guid id, [FromHeader(Name = "Idempotency-Key")] string idem, CancellationToken ct)
    {
        if (string.IsNullOrEmpty(idem)) return BadRequest("Missing idempotency key");

        var existing = await _db.IdempotencyKeys.FindAsync(new object?[] { idem }, ct);
        if (existing is not null) return Ok(existing.OrderId);

        var order = await _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);
        if (order is null) return NotFound();
        if (order.Status != OrderStatus.Pending) return Conflict("Already confirmed");
        if (order.Lines.Count == 0) return BadRequest("Empty");

        var charge = await _stripe.ChargeAsync(order.CustomerId.ToString(), order.Total.Amount, ct);
        if (!charge.Succeeded) return UnprocessableEntity(charge.Error);

        order.Status = OrderStatus.Confirmed;
        order.ConfirmedAt = DateTimeOffset.UtcNow;
        order.PaymentIntentId = charge.IntentId;

        _db.IdempotencyKeys.Add(new IdempotencyKey(idem, order.Id));
        await _db.SaveChangesAsync(ct);

        await _email.SendOrderConfirmationAsync(order.CustomerId, order.Id, ct);
        _logger.LogInformation("Confirmed {Id}", id);
        return Ok();
    }
}
```

Problems: business rules in the controller, idempotency in the controller, email send *outside* the transaction (loss possible on crash), no validation seam, untestable without HTTP + SQL + Stripe.

### After — application service + behaviors + outbox

```csharp
// Application
public sealed record ConfirmOrderCommand(OrderId OrderId, string IdempotencyKey)
    : ICommand<Result<OrderId>>;

public sealed class ConfirmOrderHandler(
    IOrderRepository orders,
    IPaymentGateway payments,
    IIdempotencyStore idempotency,
    IOutbox outbox,
    IUnitOfWork uow,
    ILogger<ConfirmOrderHandler> logger)
    : IRequestHandler<ConfirmOrderCommand, Result<OrderId>>
{
    public async Task<Result<OrderId>> Handle(ConfirmOrderCommand cmd, CancellationToken ct)
    {
        if (await idempotency.TryGetExistingAsync<OrderId>(cmd.IdempotencyKey, ct) is { } prior)
            return Result<OrderId>.Ok(prior);

        var order = await orders.GetByIdAsync(cmd.OrderId, ct);
        if (order is null) return Result<OrderId>.NotFound();

        var intent = await payments.ChargeAsync(order.CustomerId, order.Total, ct);
        if (!intent.Succeeded) return Result<OrderId>.PaymentFailed(intent.Error);

        var outcome = order.Confirm(intent);
        if (!outcome.IsSuccess) return Result<OrderId>.Failure(outcome.Error);

        outbox.Enqueue(new OrderConfirmedEvent(order.Id.Value, order.CustomerId.Value));
        await idempotency.RecordAsync(cmd.IdempotencyKey, order.Id, ct);
        await uow.SaveChangesAsync(ct);

        logger.LogInformation("Order {OrderId} confirmed", order.Id);
        return Result<OrderId>.Ok(order.Id);
    }
}

// Validation
public sealed class ConfirmOrderValidator : AbstractValidator<ConfirmOrderCommand>
{
    public ConfirmOrderValidator()
    {
        RuleFor(x => x.IdempotencyKey).NotEmpty().Length(8, 128);
        RuleFor(x => x.OrderId).NotEqual(default(OrderId));
    }
}

// API
[ApiController, Route("api/orders")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpPost("{id:guid}/confirm")]
    public Task<IActionResult> Confirm(Guid id, [FromHeader(Name = "Idempotency-Key")] string idem, CancellationToken ct) =>
        mediator.Send(new ConfirmOrderCommand(new OrderId(id), idem), ct).ToActionResult();
}
```

Email sending now lives in a separate `OrderConfirmedEventHandler` subscribed to the outbox event. The confirmation is *atomic*; the email is *eventually reliable*. The handler is unit-testable in milliseconds.

## Interview Questions and Answers

### 1. What is the difference between an application service and a domain service?

**Why this matters:** confusing the two is one of the most common DDD mistakes and a frequent interview filter.

**Answer:** An **application service** orchestrates a use case — it loads aggregates, calls infrastructure, commits the UoW. It has no business rules of its own. A **domain service** lives *inside* the domain layer and encapsulates business logic that does not naturally fit on a single aggregate — for example, `TransferFunds(fromAccount, toAccount, amount)` which spans two `Account` aggregates. Application services call domain services; the reverse never happens.

**Real project:** a banking module had a "transfer funds" use case. The handler (`TransferFundsHandler`) loaded both accounts and called a `FundsTransferService.Transfer(from, to, amount)` domain service that enforced the business rule "no negative balances after transfer." The handler stayed thin; the rule stayed in the domain.

### 2. Why use MediatR for application services instead of just calling a service class from the controller?

**Why this matters:** MediatR is fashionable; interviewers want to know whether you can defend it on substance.

**Answer:** Three reasons. First, **pipeline behaviors** give me a single place to apply validation, logging, retries, and transactions to every handler. Second, the **discoverability** is better — every command/query lives in a folder named after the use case. Third, the **transport decoupling** is real — the same handler powers controllers, Functions, and background jobs without modification. If none of those apply (small project, single transport), I skip MediatR and inject a plain service class. The pattern is what matters, not the library.

**Trade-off:** longer stack traces, a small runtime indirection, and one more dependency in the application project.

### 3. Where does validation belong: handler, controller, or aggregate?

**Why this matters:** putting it in the wrong place produces either duplicated rules or rules that can be bypassed.

**Answer:** **Three layers, three concerns.** Request *shape* (required fields, length, format) belongs in **FluentValidation** running as a pipeline behavior — it rejects malformed input before it ever touches the domain. Use-case-specific authorization (does this user own this order?) lives in the handler or in an authorization behavior. **Business invariants** (an order with zero lines cannot be confirmed) live in the **aggregate** and are enforced by domain methods like `Order.Confirm()`. The handler never duplicates an invariant the aggregate already enforces.

### 4. A handler grows to 200 lines. What is going wrong and how do you fix it?

**Why this matters:** handler bloat is the most common signal that domain logic has leaked into the application layer.

**Answer:** Almost certainly business rules have crept in. I look for `if`s that gate state changes (`if (order.Status != ... )`), date math, pricing calculations, or coordination of multiple invariants — all of those belong on an aggregate or a domain service. I extract a domain method (`order.Confirm(intent)`, `order.ApplyDiscount(code)`), move the rules into it, and shrink the handler back to "load, call, save." If the use case genuinely spans multiple aggregates and steps, I split it into a saga.

**Real project:** a 300-line `RefundOrderHandler` was reduced to 40 lines by introducing `order.IssueRefund(amount, reason)` and `Order.CanIssueRefund` invariants on the aggregate. The handler kept the orchestration; the domain owned the rules.

### 5. How do you test an application service?

**Why this matters:** if you cannot describe a clean test, you have not really separated concerns.

**Answer:** Plain xUnit tests with **NSubstitute** (or Moq) fakes for every dependency: `IOrderRepository`, `IPaymentGateway`, `IUnitOfWork`. I drive the handler with realistic commands and assert on the returned `Result<T>` plus the calls made on the fakes (`await payments.Received(1).ChargeAsync(...)`). The aggregate's invariants are tested separately in domain unit tests. The full HTTP path is covered by a small number of integration tests using `WebApplicationFactory<T>` and a Testcontainers SQL Server.

### 6. How do application services compose for long-running workflows that span multiple steps and external systems?

**Why this matters:** distinguishing a use case from a workflow is a senior-level marker.

**Answer:** A single application service handles **one** atomic use case. For workflows that span multiple use cases — *"customer placed an order → reserve inventory → charge payment → ship → notify"* — I use a **saga** (MassTransit, NServiceBus) that coordinates by listening to integration events and dispatching new commands. Each step is still a handler, but the orchestration sits outside any single handler. Trying to do it with handlers calling handlers turns into spaghetti and breaks atomicity.

### 7. What goes wrong when a handler calls an external HTTP service inside its transaction?

**Why this matters:** the textbook "hot lock contention" production failure.

**Answer:** Database row locks are held for the duration of the HTTP round-trip — easily hundreds of milliseconds. Under load you see blocking chains, then deadlocks, then 500s. The fix is to keep external calls **outside** the transaction. Persist state, commit, then call the external service — or invert it: call the external service first, capture the intent (idempotently), then persist. For unreliable downstream systems, use the outbox so the message is published reliably after commit.

### 8. When would you NOT use the application service pattern at all?

**Why this matters:** maturity to recognize when ceremony costs more than it buys.

**Answer:** When the system is genuinely CRUD. A 5-endpoint admin tool over a reference-data table does not need MediatR, commands, handlers, and behaviors. A Minimal API endpoint calling `DbContext.Customers.Add(...)` and `SaveChangesAsync` is shorter, faster, and easier to maintain. I introduce the pattern when there are more than a handful of true use cases with non-trivial orchestration — not as a default.

## Summary Checklist

- [ ] I have one application service / handler per use case, named after the verb.
- [ ] Handlers contain no business rules — they orchestrate; aggregates enforce.
- [ ] Every handler commits exactly once via the Unit of Work.
- [ ] I use MediatR pipeline behaviors for validation, logging, retries, and transactions.
- [ ] Request shape validation lives in FluentValidation; domain invariants live in aggregates.
- [ ] Handlers depend only on interfaces (repositories, gateways, UoW) — no `DbContext` directly in application code unless I have chosen to allow it.
- [ ] I pass `CancellationToken` to every async call.
- [ ] Handlers return `Result<T>` for expected failures; exceptions for genuinely exceptional cases.
- [ ] External HTTP calls live outside the database transaction; reliable publishing uses the outbox.
- [ ] I can unit-test any handler in milliseconds without a database or web server.
