# Repositories

## What It Is

A **repository** is an abstraction that exposes a domain aggregate (or persistence model) as if it were an in-memory collection. The application layer asks the repository for an `Order` by id, mutates it, and asks the repository to save it — without knowing whether the data lives in Azure SQL, Cosmos DB, an in-memory store for tests, or behind an HTTP call to a legacy mainframe.

The contract usually lives in the **domain or application layer**; the implementation lives in **infrastructure**. That single piece of indirection is what lets you keep business rules free of `SqlConnection`, `DbContext`, or `CosmosClient`.

```csharp
// Domain / application layer — no EF Core, no SQL
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task<IReadOnlyList<Order>> ListPendingForCustomerAsync(CustomerId customerId, CancellationToken ct);
}

// Infrastructure layer — knows about EF Core
public sealed class OrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _db;
    public OrderRepository(OrdersDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
        _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await _db.Orders.AddAsync(order, ct);

    public Task<IReadOnlyList<Order>> ListPendingForCustomerAsync(CustomerId customerId, CancellationToken ct) =>
        _db.Orders.Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Pending)
                  .ToListAsync(ct)
                  .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);
}
```

A repository is **not** a generic CRUD wrapper. It is a *collection of aggregates*, and the methods it exposes should read like things the business actually asks for: *"give me the pending orders for this customer," "save this confirmed order."*

## Why It Exists

Before repositories, ASP.NET / .NET Framework codebases typically looked like this:

- Controllers opened `SqlConnection`, wrote inline SQL, mapped `DataReader` rows by hand.
- Business rules (`if (order.Total > customer.CreditLimit) throw`) were tangled with `SELECT` statements.
- Swapping SQL Server for Cosmos DB, or adding a read replica, meant editing dozens of controllers.
- Unit tests required a real database, so most teams skipped them and relied on manual QA.

The repository pattern (formalized in Eric Evans' *Domain-Driven Design*, 2003) was the answer to four concrete pains:

1. **Persistence ignorance** — the domain model should not know it is being persisted.
2. **Testability** — application code should be exercisable without spinning up SQL Server.
3. **Encapsulation of queries** — the same "find overdue invoices" rule should not be re-implemented in three controllers.
4. **A clear seam for change** — when storage technology evolves, only the infrastructure layer changes.

With EF Core, the `DbContext` itself already implements the Unit-of-Work + Repository pattern (`DbSet<T>` *is* a repository). That changes the conversation: today, repositories exist not because EF Core needs them, but because **the domain layer should not depend on EF Core at all**.

## When To Use It

**Use a repository when:**

- You have a real **domain model** (rich aggregates with invariants), and you want the domain project to have **zero** dependencies on EF Core, Dapper, or Cosmos SDKs.
- The same query is needed from multiple use cases and you want a single place to enforce it (e.g., `GetActiveSubscriptionsForTenantAsync` honoring soft-delete + tenant filter).
- You need to swap storage in tests — an in-memory `FakeOrderRepository` is faster and more reliable than `UseInMemoryDatabase` for behavioral tests.
- You are implementing **CQRS** and want the command side to load aggregates through repositories while the query side uses raw SQL / Dapper / projections.

**Do not use a repository when:**

- You are building a thin CRUD service, a small admin tool, or a script. `DbContext` directly is shorter, faster to write, and easier to read.
- You would just be re-exposing every `DbSet<T>` method (`Where`, `Include`, `OrderBy`). That is not abstraction — it is friction with no boundary.
- The repository would leak `IQueryable<T>` upward. The moment a controller writes `_repo.Query().Where(o => o.Foo).Include(...)`, the abstraction is gone and you have only paid the indirection cost.
- The read side of a CQRS system needs flexible projections. Use a query service that returns DTOs, not a repository that loads aggregates.

## Why It Is Important

In production .NET / Azure systems, repositories drive five concrete properties:

1. **Domain purity.** The `Orders.Domain` project references nothing but `System.*`. Domain rules can be moved between Azure Functions, ASP.NET Core, and a Worker Service with zero changes.
2. **Testability without a database.** A `FakeOrderRepository : IOrderRepository` backed by a `Dictionary<OrderId, Order>` lets you write 200 unit tests for `PlaceOrderHandler` that run in under a second on CI.
3. **Query consistency.** Multi-tenant filters (`WHERE TenantId = @tenant`) and soft-delete filters (`WHERE DeletedAt IS NULL`) live in one place. A junior dev cannot accidentally bypass them.
4. **Bounded blast radius for storage migrations.** When the team moves the order history table from Azure SQL to Cosmos DB for cost reasons, only `OrderRepository` changes — handlers, controllers, and tests stay the same.
5. **A defensible boundary in code review.** "Why is `CosmosClient` being injected into a controller?" is a much easier conversation to have when the architecture rule is "infrastructure types only live behind repositories."

The pattern is *not* important because it ships with .NET. It is important because it gives the team a contract they can defend.

## How It's Used in C# / .NET

### 1. The contract lives in the domain or application layer

```csharp
// src/Orders.Domain/Repositories/IOrderRepository.cs
namespace Orders.Domain.Repositories;

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task<Order?> GetByIdForUpdateAsync(OrderId id, CancellationToken ct); // pessimistic lock variant
    Task AddAsync(Order order, CancellationToken ct);
    void Remove(Order order); // mark for deletion; SaveChanges flushes
}
```

Note what is **not** here: no `IQueryable<T>`, no `Expression<Func<Order, bool>>`, no `Include`. The contract speaks the business's language.

### 2. The implementation lives in infrastructure and uses EF Core

```csharp
// src/Orders.Infrastructure/Persistence/OrderRepository.cs
namespace Orders.Infrastructure.Persistence;

internal sealed class OrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _db;

    public OrderRepository(OrdersDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
        _db.Orders
           .Include(o => o.Lines)
           .Include(o => o.ShippingAddress)
           .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<Order?> GetByIdForUpdateAsync(OrderId id, CancellationToken ct)
    {
        // Read with UPDLOCK to prevent concurrent confirmations
        var sql = "SELECT * FROM Orders WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}";
        return await _db.Orders.FromSqlRaw(sql, id.Value)
                               .Include(o => o.Lines)
                               .FirstOrDefaultAsync(ct);
    }

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await _db.Orders.AddAsync(order, ct);

    public void Remove(Order order) => _db.Orders.Remove(order);
}
```

### 3. Registration via DI

```csharp
// Program.cs
builder.Services.AddDbContext<OrdersDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Orders")));

builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
```

Scoped lifetime matches the `DbContext` — one repository instance per HTTP request or per message-handling scope.

### 4. The application service / MediatR handler consumes it

```csharp
public sealed class ConfirmOrderHandler : IRequestHandler<ConfirmOrderCommand, Result>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly ILogger<ConfirmOrderHandler> _logger;

    public ConfirmOrderHandler(IOrderRepository orders, IUnitOfWork uow, ILogger<ConfirmOrderHandler> logger)
    {
        _orders = orders;
        _uow = uow;
        _logger = logger;
    }

    public async Task<Result> Handle(ConfirmOrderCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();

        order.Confirm(); // domain method enforces invariants

        await _uow.SaveChangesAsync(ct);
        _logger.LogInformation("Order {OrderId} confirmed", cmd.OrderId);
        return Result.Ok();
    }
}
```

### 5. Specification pattern — when query criteria multiply

When you have ten variants of "list orders" (by status, by date range, by customer, paged, sorted), avoid ten repository methods. Use a **specification**:

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    int? Take { get; }
    int? Skip { get; }
}

public sealed class PendingOrdersOlderThanSpec : ISpecification<Order>
{
    public PendingOrdersOlderThanSpec(DateTimeOffset cutoff, int page, int pageSize)
    {
        Criteria = o => o.Status == OrderStatus.Pending && o.CreatedAt < cutoff;
        Includes = new() { o => o.Customer };
        Take = pageSize;
        Skip = (page - 1) * pageSize;
    }
    public Expression<Func<Order, bool>> Criteria { get; }
    public List<Expression<Func<Order, object>>> Includes { get; }
    public int? Take { get; }
    public int? Skip { get; }
}

// IOrderRepository gains a single method:
Task<IReadOnlyList<Order>> ListAsync(ISpecification<Order> spec, CancellationToken ct);
```

Libraries like **Ardalis.Specification** package this pattern with EF Core support out of the box.

### 6. Generic repository — usually a smell

```csharp
// Avoid this in most domain-driven projects
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
    IQueryable<T> Query();
    Task AddAsync(T entity, CancellationToken ct);
    void Remove(T entity);
}
```

`IRepository<T>` is essentially `DbSet<T>` with a different name. It hides nothing, adds a layer of indirection, and tempts developers to leak `IQueryable<T>` upward. Reach for it only in genuinely CRUD-shaped services.

## Advantages

- **Domain purity** — business logic compiles without EF Core, SQL drivers, or Cosmos SDK.
- **Fast, deterministic tests** — fakes replace databases in unit tests.
- **Single source of truth for queries** — multi-tenant, soft-delete, and security filters are centralized.
- **Storage replaceability** — moving an aggregate from SQL to Cosmos is a one-class change.
- **Easier code review** — "where does this query live?" has one answer.
- **Aligns with DDD** — aggregates and repositories form a coherent design vocabulary.

## Disadvantages

- **Extra indirection** — for simple CRUD APIs, it adds files without adding value.
- **EF Core feature loss** — projections (`.Select(o => new OrderDto(...))`), `AsNoTracking`, split queries, and compiled queries are awkward to expose through a repository.
- **Method explosion** — without specifications, repositories grow `GetActiveByCustomerAndDateRangeOrderedByTotal...` methods.
- **Over-abstraction risk** — generic repositories often re-implement `DbSet<T>` with fewer features.
- **Read-side mismatch** — repositories return aggregates; reporting screens want flat DTOs.

## Common Mistakes

### 1. Exposing `IQueryable<T>` from the repository

**Problem:** the abstraction leaks. Controllers and handlers can write arbitrary EF queries, bypassing tenant filters and breaking the seam.

```csharp
// ❌ Anti-pattern
public interface IOrderRepository
{
    IQueryable<Order> Query();
}

// Caller writes:
var orders = _orders.Query().Where(o => o.Total > 1000).Include(o => o.Lines).ToList();
```

**Fix:** return materialized results (`IReadOnlyList<T>`, `Order`, or a specification result). Keep `IQueryable<T>` inside infrastructure.

```csharp
// ✅
public interface IOrderRepository
{
    Task<IReadOnlyList<Order>> ListAsync(ISpecification<Order> spec, CancellationToken ct);
}
```

### 2. One repository per `DbSet<T>` with generic CRUD

**Problem:** wrapping every entity in `Repository<Customer>`, `Repository<Order>`, `Repository<Invoice>` produces zero new behavior — just indirection.

```csharp
// ❌
public class Repository<T> where T : class
{
    private readonly DbContext _db;
    public Task<T?> GetByIdAsync(int id) => _db.Set<T>().FindAsync(id).AsTask();
}
```

**Fix:** create repositories only for **aggregate roots**, and only when there is real query encapsulation to do.

### 3. Calling `SaveChangesAsync` inside the repository

**Problem:** the repository commits its own changes, so a single use case touching three aggregates ends up in three separate transactions. Atomicity is lost.

```csharp
// ❌
public async Task AddAsync(Order order, CancellationToken ct)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct); // commits here — bad
}
```

**Fix:** repositories only stage changes. A separate **Unit of Work** (often the `DbContext` itself) commits at the end of the use case. See [architecture/unit-of-work.md](unit-of-work.md).

### 4. Loading whole aggregates just to read one field

**Problem:** `var name = (await _customers.GetByIdAsync(id)).Name;` pulls the full customer graph for a display string.

**Fix:** for reads, use a **query service** that returns DTOs or projections. Reserve repositories for the **command side** that needs full aggregates to enforce invariants.

```csharp
// ✅ Read side bypasses the repository
public sealed class CustomerQueries
{
    private readonly OrdersDbContext _db;
    public Task<CustomerSummaryDto?> GetSummaryAsync(CustomerId id, CancellationToken ct) =>
        _db.Customers.Where(c => c.Id == id)
                     .Select(c => new CustomerSummaryDto(c.Id, c.Name, c.Email))
                     .FirstOrDefaultAsync(ct);
}
```

### 5. Putting business rules inside the repository

**Problem:** `OrderRepository.ConfirmOrder(id)` mutates state, applies discounts, and writes audit rows. Now business logic depends on EF Core.

**Fix:** repositories only load and save. The `Order` aggregate enforces rules: `order.Confirm()`, `order.ApplyDiscount(code)`.

### 6. Mocking `DbContext` instead of using a repository

**Problem:** tests are full of `Mock<DbSet<Order>>` plumbing that breaks every time the query shape changes.

**Fix:** depend on `IOrderRepository`, mock the interface, and let the real repository be tested by a single integration test against SQL Server (in a container, via Testcontainers).

## Best Practices

- Define repository interfaces in the **domain** or **application** project; implementations in **infrastructure**.
- One repository per **aggregate root**, not per table.
- Method names should read like business intent: `ListUnpaidInvoicesForTenantAsync`, not `GetWhere`.
- Return materialized collections (`IReadOnlyList<T>`), never `IQueryable<T>`.
- Pair every async method with a `CancellationToken` — Azure-hosted services depend on cooperative cancellation for graceful shutdown.
- Keep mutation methods (`AddAsync`, `Remove`) side-effect-free w.r.t. the database. Let a Unit of Work commit.
- For complex query criteria, use the **specification pattern** rather than method explosion.
- Use a separate **query service / read model** for reporting and list screens — do not bend repositories into reporting tools.
- Scope repositories as **`Scoped`** in DI so they share the `DbContext` lifetime.
- Document the multi-tenant / soft-delete behavior inside the repository — make it impossible to bypass.

## Related Concepts

- [architecture/unit-of-work.md](unit-of-work.md) — repositories stage changes; UoW commits them.
- [architecture/aggregates.md](aggregates.md) — one repository per aggregate root.
- [architecture/domain-driven-design.md](domain-driven-design.md) — the design philosophy that gave rise to repositories.
- [architecture/cqrs.md](cqrs.md) — split write-side repositories from read-side query services.
- [data-access/repository-pattern-with-ef-core.md](../data-access/repository-pattern-with-ef-core.md) — concrete EF Core implementation details.
- [data-access/entity-framework-core.md](../data-access/entity-framework-core.md) — `DbContext` already implements UoW + Repository.
- [testing-quality/testable-code-design.md](../testing-quality/testable-code-design.md) — repositories as a seam for fakes.

## Real-World Usage

### ASP.NET Core ordering API

A typical `Orders.Api` project structure:

```
src/
  Orders.Domain/          // Order, OrderLine, IOrderRepository — no EF Core
  Orders.Application/     // ConfirmOrderHandler, PlaceOrderHandler (MediatR)
  Orders.Infrastructure/  // OrdersDbContext, OrderRepository, migrations
  Orders.Api/             // Controllers, Program.cs, DI wiring
```

Controllers depend on MediatR. Handlers depend on `IOrderRepository`. Only `Orders.Infrastructure` references `Microsoft.EntityFrameworkCore.SqlServer`.

### Azure Functions order processor

```csharp
public sealed class OrderShippedFunction
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;

    public OrderShippedFunction(IOrderRepository orders, IUnitOfWork uow)
    {
        _orders = orders;
        _uow = uow;
    }

    [Function(nameof(OrderShippedFunction))]
    public async Task Run(
        [ServiceBusTrigger("order-shipped", Connection = "ServiceBus")] OrderShippedEvent evt,
        CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(new OrderId(evt.OrderId), ct);
        if (order is null) return; // idempotency: message arrived for deleted order

        order.MarkShipped(evt.TrackingNumber, evt.ShippedAt);
        await _uow.SaveChangesAsync(ct);
    }
}
```

The Function and the API share `IOrderRepository`. Only DI wiring differs.

### Multi-tenant SaaS

The repository enforces the tenant filter centrally so no caller can forget it:

```csharp
internal sealed class TenantAwareOrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _db;
    private readonly ITenantContext _tenant;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
        _db.Orders.Where(o => o.TenantId == _tenant.Current && o.Id == id)
                  .FirstOrDefaultAsync(ct);
}
```

A junior dev adding a new query method cannot accidentally leak data across tenants.

### Testing strategy

- **Unit tests** for handlers use `FakeOrderRepository : IOrderRepository` backed by `Dictionary<OrderId, Order>`. Sub-second feedback.
- **Integration tests** for `OrderRepository` itself run against SQL Server in a Testcontainers container. They verify `Include`, projections, and concurrency tokens.
- **End-to-end tests** hit the real API and assert HTTP responses. These remain few in number.

## Code Example — Before and After

### Before — controller talks to `DbContext` directly

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly OrdersDbContext _db;
    private readonly IStripeClient _stripe;

    public OrdersController(OrdersDbContext db, IStripeClient stripe)
    {
        _db = db;
        _stripe = stripe;
    }

    [HttpPost("{id:guid}/confirm")]
    public async Task<IActionResult> Confirm(Guid id, CancellationToken ct)
    {
        // Business rule mixed with EF Core query
        var order = await _db.Orders
                             .Include(o => o.Lines)
                             .FirstOrDefaultAsync(o => o.Id == id && o.TenantId == User.GetTenantId(), ct);
        if (order is null) return NotFound();
        if (order.Status != OrderStatus.Pending) return Conflict("Order already confirmed");
        if (order.Total <= 0) return BadRequest("Empty order");

        var charge = await _stripe.ChargeAsync(order.CustomerId.ToString(), order.Total, ct);
        if (!charge.Succeeded) return UnprocessableEntity();

        order.Status = OrderStatus.Confirmed;
        order.ConfirmedAt = DateTimeOffset.UtcNow;
        order.PaymentIntentId = charge.IntentId;

        await _db.SaveChangesAsync(ct);
        return Ok();
    }
}
```

Problems: business rules, persistence, and payment integration are tangled in the controller. The tenant filter is repeated in every endpoint. Unit testing is impossible without SQL Server and the Stripe SDK.

### After — repository, aggregate, and handler

```csharp
// Domain
public sealed class Order
{
    public OrderId Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public DateTimeOffset? ConfirmedAt { get; private set; }
    public string? PaymentIntentId { get; private set; }
    // ...

    public Result Confirm(PaymentIntent intent)
    {
        if (Status != OrderStatus.Pending) return Result.Conflict("Already confirmed");
        if (Total.Amount <= 0) return Result.Invalid("Empty order");

        Status = OrderStatus.Confirmed;
        ConfirmedAt = DateTimeOffset.UtcNow;
        PaymentIntentId = intent.Id;
        return Result.Ok();
    }
}

// Application
public sealed record ConfirmOrderCommand(OrderId OrderId) : IRequest<Result>;

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
        if (!intent.Succeeded) return Result.PaymentFailed();

        var outcome = order.Confirm(intent);
        if (!outcome.IsSuccess) return outcome;

        await _uow.SaveChangesAsync(ct);
        return Result.Ok();
    }
}

// API
[HttpPost("{id:guid}/confirm")]
public Task<IActionResult> Confirm(Guid id, CancellationToken ct) =>
    _mediator.Send(new ConfirmOrderCommand(new OrderId(id)), ct).ToActionResult();
```

Now the controller is two lines. The handler is testable with a `FakeOrderRepository` and a `FakeStripeGateway`. Tenant filtering lives in `TenantAwareOrderRepository`. The aggregate owns the invariants. Swapping SQL Server for PostgreSQL changes one file.

## Interview Questions and Answers

### 1. EF Core already gives you `DbContext` and `DbSet<T>`. Why add a repository on top?

**Why this matters:** interviewers want to know whether you understand that "the pattern is already there" is a fair point and that you only add layers for concrete reasons.

**Answer:** `DbContext` is a Unit of Work and `DbSet<T>` is a repository, true. I add `IOrderRepository` on top only when I want the **domain project to compile without EF Core**, or when I need to **centralize cross-cutting query rules** like multi-tenant filters or soft-delete. For a small CRUD API I skip it — directly using `DbContext` is shorter and faster.

**Trade-off:** an extra interface and class per aggregate, in exchange for domain purity and a stable seam for tests and storage swaps.

**Real project:** on a multi-tenant invoicing platform, `IInvoiceRepository` enforced the tenant filter and a soft-delete filter. Removing them required editing one class, not the entire query layer.

### 2. Should the repository commit changes itself?

**Why this matters:** confusing repositories with units of work leads to fragmented transactions and lost atomicity.

**Answer:** No. Repositories stage changes via `AddAsync` / `Remove`; the **Unit of Work** (the `DbContext`) commits at the end of the use case. If a handler updates an `Order`, debits an `Inventory`, and writes an `OutboxMessage`, all three must commit in the same transaction. A repository that calls `SaveChangesAsync` internally breaks that.

**Real project:** I once inherited a service where each repository called `SaveChanges` on add. Half the orders were paid but never shipped because the `Shipment` insert failed silently after the `Order` had already committed. Removing those calls fixed the atomicity bug.

### 3. When is a generic `IRepository<T>` actually justified?

**Why this matters:** it tests whether you can resist patterns that look elegant but provide no boundary.

**Answer:** Rarely. A generic repository tends to re-expose `DbSet<T>` with a different name. I justify it only in genuinely CRUD-shaped services (admin tools, reference-data APIs) where there is no domain model worth protecting. For DDD-style projects, one repository per aggregate root is clearer.

**Trade-off:** generic repos reduce code duplication for trivial cases but make it easy to leak `IQueryable<T>` and violate boundaries.

### 4. How do you handle complex queries without exploding the repository surface?

**Why this matters:** repositories with 40 methods are a known anti-pattern.

**Answer:** Use the **specification pattern**. A single `ListAsync(ISpecification<Order> spec, CancellationToken ct)` method accepts a specification object that encapsulates criteria, includes, paging, and sorting. Libraries like Ardalis.Specification translate these to EF Core queries. Alternatively, for read-heavy screens, I bypass the repository entirely and use a dedicated **query service** returning DTOs.

**Real project:** an order admin screen needed twelve filter combinations. Instead of `GetByStatusAsync`, `GetByDateRangeAsync`, etc., one `OrdersSpecification` builder cut the surface from twelve methods to one.

### 5. How do you unit-test code that depends on a repository?

**Why this matters:** the whole point of repositories is testability — if you cannot demonstrate it, you have missed the value.

**Answer:** Inject `IOrderRepository` and provide a fake — typically a thin class backed by a `Dictionary<OrderId, Order>`. The fake is faster and more reliable than EF Core's in-memory provider (which has subtle behavior differences from SQL Server). For the repository implementation itself, write **integration tests** against a real database in a Testcontainers SQL Server container.

**Trade-off:** fakes can drift from real EF Core behavior, so a smaller integration-test suite is still needed to verify queries actually run.

### 6. What is the relationship between a repository and an aggregate?

**Why this matters:** repositories per table is the most common DDD mistake.

**Answer:** One repository per **aggregate root** — never per table or per entity. The aggregate is the transactional consistency boundary; the repository is the gateway to load and save it. So I have `IOrderRepository` returning `Order` (with its `OrderLine` children eagerly loaded), but **no** `IOrderLineRepository`, because `OrderLine` only exists as part of `Order`.

**Real project:** an early version of our codebase had `IOrderLineRepository.UpdateQuantity`. It let two requests update lines on the same order concurrently and corrupted the total. Removing it and routing all changes through `Order.ChangeLineQuantity()` plus `IOrderRepository.GetByIdAsync` fixed the race.

### 7. How do repositories fit into CQRS?

**Why this matters:** mixing reads and writes through the same abstraction is a frequent design problem.

**Answer:** Repositories serve the **command side** — they load aggregates so the domain can enforce invariants, and they save them. The **query side** uses a separate `IOrderQueries` service that runs projections directly against the database (or a denormalized read model in Cosmos / Elasticsearch) and returns DTOs. Trying to use one repository for both forces either over-fetching for queries or under-fetching for commands.

### 8. When would you remove the repository pattern from a project?

**Why this matters:** maturity to undo patterns is as important as introducing them.

**Answer:** When the codebase is essentially CRUD with no rich domain model, when every repository method just forwards to `DbContext`, and when the team is spending time maintaining mappings without gaining testability or a real boundary. I would replace `IOrderRepository` with `DbContext` directly, accept the coupling, and reinvest the time saved into integration tests and code review rules.

**Real project:** a reporting microservice with 50 read-only endpoints had 50 repositories that did nothing but call `Where`. Deleting them and using `OrdersDbContext` directly removed 2,000 lines of code and made the service easier for the team to onboard.

## Summary Checklist

- [ ] I can explain why repositories exist even though `DbContext` already implements the pattern.
- [ ] I define repository interfaces in the domain layer and implementations in infrastructure.
- [ ] I create one repository per aggregate root, not per table.
- [ ] I never expose `IQueryable<T>` from a repository.
- [ ] I keep `SaveChangesAsync` out of repositories and delegate it to a Unit of Work.
- [ ] I use the specification pattern when query criteria multiply.
- [ ] I use query services / read models for reporting screens, not repositories.
- [ ] I can write a `FakeOrderRepository` to unit-test a handler in milliseconds.
- [ ] I know when to skip the pattern entirely (small CRUD services, reporting APIs).
- [ ] I enforce multi-tenant and soft-delete filters inside the repository so they cannot be bypassed.
