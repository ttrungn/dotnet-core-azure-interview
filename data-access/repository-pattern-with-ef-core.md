# Repository Pattern with EF Core

## What It Is

The **Repository Pattern** is a design pattern that puts a domain-shaped interface in front of persistence. Application code calls `_orders.GetByIdAsync(id)` instead of `_db.Orders.Include(...).FirstOrDefaultAsync(...)`. The repository hides what database you're using, what queries it runs, and how it loads aggregates.

When the persistence library is **EF Core**, the picture changes: `DbContext` + `DbSet<T>` is already a Unit of Work + repository implementation. `DbSet<Order>` is `IRepository<Order>` for free, with `LINQ`, tracking, and transactions built in. So the honest answer is: *you only add another `IOrderRepository` interface on top of EF Core when you have a specific reason*.

There are three valid postures:

1. **No repository.** Inject `DbContext` directly into application services. Honest, fast, no ceremony.
2. **Aggregate-shaped repository.** `IOrderRepository.GetByIdAsync(id)`, `Add(order)`, `Remove(order)`. Mirrors DDD aggregates. Useful when the domain boundary is real.
3. **Generic `IRepository<T>`.** A wrapper that exposes `Add`, `Get`, `Find`, `Query` for every entity. Tempting, almost always wrong with EF Core — it hides projections, includes, tracking, splits, transactions, and forces you to leak `IQueryable` (defeating the point) or fall back to read models.

```csharp
// (1) No repository — DbContext IS the repository
public sealed class OrderStatusService(OrdersDbContext db, ILogger<OrderStatusService> log)
{
    public async Task ShipAsync(Guid orderId, CancellationToken ct)
    {
        var order = await db.Orders.FirstAsync(o => o.Id == orderId, ct);
        order.MarkShipped();
        await db.SaveChangesAsync(ct);
    }
}

// (2) Aggregate-shaped repository — protects a DDD boundary
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    void Add(Order order);
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// (3) Generic — almost always an anti-pattern with EF Core
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
    IQueryable<T> Query(); // leaks EF abstractions, defeats the pattern
    void Add(T entity);
    void Remove(T entity);
}
```

## Why It Exists

Repository was popularized when applications talked to **untyped data access** (ADO.NET, raw SQL strings, stored procedures called by string name). The pattern moved that mess behind a clean interface so domain code wouldn't be smeared with `SqlConnection` and `IDataReader`. It also enabled in-memory implementations for testing.

EF Core (and EF6 before it) absorbed most of that value into the ORM itself:

- `DbContext` is the Unit of Work.
- `DbSet<T>` is a repository with a query language.
- `IDbContextFactory<T>` gives you proper lifetime control.
- `EF.CompileQuery` solves the compiled-query story repositories used to provide.

So the modern question is no longer *"do I need a repository?"* but *"do I need an **additional** repository on top of EF Core?"*. The answer is yes only when one of these is true:

- You have a real domain boundary (DDD aggregates) that should not leak `IQueryable` to the application layer.
- You need to swap stores (SQL → Cosmos, or test substitution beyond what `InMemoryDbContext` gives you).
- You want a single place to centralize tenant filters, soft-delete predicates, or audit hooks.
- You're following Hexagonal / Clean Architecture and the domain project cannot reference EF Core.

## When To Use It

**Use an explicit repository when:**

- DDD aggregates with strong invariants — the repository loads/saves the **aggregate root** as a unit, never partial entities.
- Multi-store scenarios (e.g., write to SQL, read from Cosmos) where the application doesn't care which backs each call.
- Hexagonal architecture where the domain project must not depend on EF Core.
- You need to enforce cross-cutting query filters (tenant id, soft-delete) more strictly than `HasQueryFilter` provides.
- Centralized read-model query objects (CQRS read side) where each query is a class implementing `ISpecification<T>` or a `IXxxQuery` interface.

**Do not use a repository when:**

- You're building a CRUD service and `DbContext` already does the job. Adding `IFooRepository` is pure ceremony.
- You only want it for testing — modern testing strategies (Testcontainers, SQLite, mocking at the service boundary) work fine without one.
- You'd implement it as a generic `IRepository<T>` that exposes `IQueryable<T>` — that's a leaky abstraction with worse ergonomics than EF Core.
- You'd write a separate repository per simple entity that contains only `Get` + `Add` + `Update` + `Delete` — that's the EF DbSet API renamed.

## Why It Is Important

Repository done **well** (aggregate-shaped, domain-driven) gives you:

- **Domain isolation.** Business logic never sees `DbContext`, change tracking, or `IQueryable`.
- **Stable contract.** The repository interface lives in the domain project and is mockable in unit tests.
- **A single place for invariants** like "loading an `Order` always includes `Lines`" or "every query filters by `TenantId`".
- **CQRS friendliness.** Write side uses aggregate repositories; read side uses dedicated query objects returning DTOs.

Repository done **badly** (generic, leaks `IQueryable`, exists only for testing) gives you:

- Worse performance — projections, splits, tracking control hidden.
- Worse testability — mocking `IQueryable<T>` is a nightmare; mocked LINQ doesn't match EF translation.
- More code, same behavior, more bugs.

Senior interviewers usually probe whether you know *when* the pattern adds value, not whether you can write `interface IRepository<T>`.

## How It's Used in C# / .NET

### 1. Aggregate-shaped repository (preferred when used)

```csharp
// Domain project — no EF Core reference
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetPendingByCustomerAsync(CustomerId id, CancellationToken ct);
    void Add(Order order);
    void Remove(Order order);
}

// Infrastructure project — references EF Core
public sealed class OrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _db;
    public OrderRepository(OrdersDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
        _db.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task<IReadOnlyList<Order>> GetPendingByCustomerAsync(
        CustomerId id, CancellationToken ct) =>
        _db.Orders
            .Where(o => o.CustomerId == id && o.Status == OrderStatus.Pending)
            .Include(o => o.Lines)
            .ToListAsync(ct)
            .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);

    public void Add(Order order) => _db.Orders.Add(order);
    public void Remove(Order order) => _db.Orders.Remove(order);
}
```

The application service calls `_orders.Add(order); await _uow.SaveChangesAsync(ct);` — never sees EF.

### 2. Specification pattern (composable queries without leaking `IQueryable`)

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    int? Take { get; }
    int? Skip { get; }
}

public sealed class PendingOrdersForCustomer : ISpecification<Order>
{
    public PendingOrdersForCustomer(CustomerId customerId, int page, int pageSize)
    {
        Criteria = o => o.CustomerId == customerId && o.Status == OrderStatus.Pending;
        Includes.Add(o => o.Lines);
        Skip = (page - 1) * pageSize;
        Take = pageSize;
    }
    public Expression<Func<Order, bool>> Criteria { get; }
    public List<Expression<Func<Order, object>>> Includes { get; } = new();
    public int? Take { get; }
    public int? Skip { get; }
}

// Repository applies the specification
public async Task<IReadOnlyList<Order>> ListAsync(
    ISpecification<Order> spec, CancellationToken ct)
{
    IQueryable<Order> q = _db.Orders.Where(spec.Criteria);
    foreach (var include in spec.Includes) q = q.Include(include);
    if (spec.Skip is not null) q = q.Skip(spec.Skip.Value);
    if (spec.Take is not null) q = q.Take(spec.Take.Value);
    return await q.ToListAsync(ct);
}
```

### 3. Query objects for read models (CQRS read side)

```csharp
public interface IOrderSummaryQuery
{
    Task<PagedResult<OrderSummaryDto>> RunAsync(
        Guid customerId, int page, int pageSize, CancellationToken ct);
}

public sealed class OrderSummaryQuery : IOrderSummaryQuery
{
    private readonly OrdersDbContext _db;
    public OrderSummaryQuery(OrdersDbContext db) => _db = db;

    public async Task<PagedResult<OrderSummaryDto>> RunAsync(
        Guid customerId, int page, int pageSize, CancellationToken ct)
    {
        var query = _db.Orders.AsNoTracking()
            .Where(o => o.CustomerId == customerId);

        var total = await query.CountAsync(ct);
        var items = await query
            .OrderByDescending(o => o.CreatedAt)
            .Skip((page - 1) * pageSize).Take(pageSize)
            .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Status, o.CreatedAt))
            .ToListAsync(ct);

        return new PagedResult<OrderSummaryDto>(items, total, page, pageSize);
    }
}
```

This is **not** a repository. It's a single-use query object. Combine with aggregate repositories on the write side for clean CQRS.

### 4. Generic repository (when it makes sense — rarely)

Generic repositories work when entities truly share behavior and you don't need EF features per type. In practice this is uncommon in business systems but might fit infrastructure types:

```csharp
public interface IAuditLogRepository
{
    Task AppendAsync(AuditEntry entry, CancellationToken ct);
    Task<IReadOnlyList<AuditEntry>> GetForCorrelationAsync(Guid id, CancellationToken ct);
}
```

A truly generic `IRepository<T>` exposing `IQueryable<T>` provides no value — callers still need to know it's EF, and they lose the readable intent.

### 5. NuGet packages

- `Microsoft.EntityFrameworkCore` (8.0+)
- `Ardalis.Specification` — specification pattern with EF Core support, well-maintained.
- `MediatR` — pair with query objects for clean CQRS reads.
- `Testcontainers.MsSql` — real SQL Server for repository integration tests.

## Advantages

- **Clean domain boundary** — domain code knows nothing about persistence.
- **Aggregate consistency** — a repository is the natural place to enforce "always load with children".
- **Test mockability at the right level** — mock `IOrderRepository`, not `DbContext`.
- **Centralized cross-cutting concerns** — tenant filter, soft-delete, audit hooks in one place.
- **Store-swap potential** — when the abstraction is honest, switching stores becomes feasible.

## Disadvantages

- **Ceremony tax** — interface + impl + DI registration for what could be `_db.Orders`.
- **Loss of EF features** — projections, splits, `AsSplitQuery`, `IncludeFilteredAsync` may be awkward to expose.
- **Leaky abstraction risk** — exposing `IQueryable<T>` means LINQ-to-Provider quirks leak through.
- **Slower for read models** — repositories load aggregates; reads usually want projected DTOs.
- **Misleads juniors** — they implement `IRepository<T>` everywhere assuming it's a best practice.

## Common Mistakes

### 1. Generic `IRepository<T>` over every entity

```csharp
// BAD — repository per entity, no domain value
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id);
    Task<IEnumerable<T>> GetAllAsync();
    void Add(T entity);
    void Update(T entity);
    void Remove(T entity);
}

services.AddScoped<IRepository<Order>, EfRepository<Order>>();
services.AddScoped<IRepository<Customer>, EfRepository<Customer>>();
services.AddScoped<IRepository<Invoice>, EfRepository<Invoice>>();
```

`GetAllAsync()` on `Orders` is a production landmine. None of this provides anything `DbSet<T>` doesn't.

```csharp
// GOOD — inject DbContext directly, or build aggregate repositories with intent
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetPendingByCustomerAsync(CustomerId id, CancellationToken ct);
    void Add(Order order);
}
```

### 2. Exposing `IQueryable<T>` from the repository

```csharp
// BAD — every caller can write any query
public interface IOrderRepository
{
    IQueryable<Order> Query();
}

// caller:
var ids = _orders.Query().Where(...).Select(o => o.Id).ToListAsync();
```

The abstraction is leaky — callers must know they're using EF, must understand translation rules, and the repository can't enforce filters reliably.

```csharp
// GOOD — explicit methods or specifications
Task<IReadOnlyList<OrderId>> GetIdsForBatchProcessingAsync(
    DateTime cutoff, CancellationToken ct);
```

### 3. Loading aggregates without children

```csharp
// BAD — Order without Lines means business logic can't compute totals
public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
    _db.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
```

```csharp
// GOOD — aggregate is loaded whole
public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
    _db.Orders
        .Include(o => o.Lines)
        .AsSplitQuery() // avoid cartesian explosion on multiple collections
        .FirstOrDefaultAsync(o => o.Id == id, ct);
```

### 4. Using `Microsoft.EntityFrameworkCore.InMemory` for repository tests

```csharp
// BAD — InMemory provider does not implement relational semantics
var options = new DbContextOptionsBuilder<OrdersDbContext>()
    .UseInMemoryDatabase("test").Options;
```

The in-memory provider doesn't enforce uniqueness, doesn't support transactions correctly, doesn't translate raw SQL, and case-sensitive string comparisons differ from SQL Server. Tests pass against InMemory and fail in production.

```csharp
// GOOD — SQLite for fast tests, Testcontainers SQL Server for high-fidelity
var options = new DbContextOptionsBuilder<OrdersDbContext>()
    .UseSqlite("DataSource=:memory:").Options;
// or:
await using var msSql = new MsSqlBuilder().Build();
await msSql.StartAsync();
var options = new DbContextOptionsBuilder<OrdersDbContext>()
    .UseSqlServer(msSql.GetConnectionString()).Options;
```

EF Core team [explicitly recommends against InMemory for testing](https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy#in-memory-as-a-database-fake).

### 5. Calling `SaveChangesAsync` inside the repository

```csharp
// BAD — repository owns transaction boundary, can't compose multiple repos
public async Task AddAsync(Order order, CancellationToken ct)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct);
}
```

Now an application service that calls `_orders.AddAsync` then `_inventory.ReserveAsync` gets two transactions and can't roll back atomically.

```csharp
// GOOD — repository tracks; Unit of Work commits
public void Add(Order order) => _db.Orders.Add(order);
// Application service:
_orders.Add(order);
_inventory.Reserve(order.Lines);
await _uow.SaveChangesAsync(ct); // one transaction
```

### 6. Mocking `DbSet<T>` for unit tests

```csharp
// BAD — mocking DbSet is brittle and tests nothing real
var mockSet = new Mock<DbSet<Order>>();
mockSet.As<IQueryable<Order>>().Setup(...);
```

LINQ-to-Objects vs LINQ-to-EF differ in null handling, string ops, navigation translation. Tests pass on mocks, fail on SQL.

```csharp
// GOOD — integration test against SQLite or Testcontainers
await using var db = new OrdersDbContext(InMemorySqliteOptions);
await db.Database.EnsureCreatedAsync();
var repo = new OrderRepository(db);
var order = Order.Create(...);
repo.Add(order);
await db.SaveChangesAsync();
var loaded = await repo.GetByIdAsync(order.Id, default);
Assert.NotNull(loaded);
```

## Best Practices

- **Default to no repository.** Use `DbContext` directly until a domain reason demands one.
- **If you add a repository, make it aggregate-shaped.** One repository per aggregate root, not per table.
- **Keep the interface in the domain project, the implementation in infrastructure.**
- **Never expose `IQueryable<T>`** from the public interface.
- **Don't call `SaveChangesAsync` inside the repository.** That's the Unit of Work's job.
- **Use the specification pattern (Ardalis.Specification) for composable queries** when needed.
- **For reads, prefer dedicated query objects returning DTOs** over loading aggregates and mapping.
- **Test with SQLite in-memory or Testcontainers SQL Server**, never `Microsoft.EntityFrameworkCore.InMemory`.
- **Centralize cross-cutting concerns** (tenant filter, soft-delete, audit) in the repository or via `HasQueryFilter`.
- **Use `AsSplitQuery()` when including multiple collections** to avoid cartesian explosion.

## Related Concepts

- [data-access/dbcontext-lifetime.md](data-access/dbcontext-lifetime.md) — DbContext IS the unit-of-work + repository
- [data-access/entity-framework-core.md](data-access/entity-framework-core.md) — what EF Core gives you out of the box
- [data-access/transactions.md](data-access/transactions.md) — repository + UoW + transactions
- [data-access/optimistic-concurrency.md](data-access/optimistic-concurrency.md) — concurrency tokens at the aggregate boundary
- [data-access/query-performance.md](data-access/query-performance.md) — projections, splits, no-tracking
- [architecture/aggregates.md](architecture/aggregates.md) — what a repository's "unit of load/save" should be
- [architecture/unit-of-work.md](architecture/unit-of-work.md) — the companion pattern
- [architecture/cqrs.md](architecture/cqrs.md) — separate write-side repositories from read-side query objects
- [testing-quality/integration-testing.md](testing-quality/integration-testing.md) — Testcontainers for repository tests

## Real-World Usage

### ASP.NET Core — DDD order management

Application service depends only on aggregate repositories and a Unit of Work:

```csharp
public sealed class PlaceOrderHandler
{
    private readonly IOrderRepository _orders;
    private readonly IInventoryRepository _inventory;
    private readonly IUnitOfWork _uow;
    private readonly ILogger<PlaceOrderHandler> _log;

    public PlaceOrderHandler(
        IOrderRepository orders, IInventoryRepository inventory,
        IUnitOfWork uow, ILogger<PlaceOrderHandler> log)
    {
        _orders = orders; _inventory = inventory; _uow = uow; _log = log;
    }

    public async Task<Result<OrderId>> HandleAsync(PlaceOrder cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Lines);
        foreach (var line in order.Lines)
        {
            var reserved = await _inventory.TryReserveAsync(line.Sku, line.Quantity, ct);
            if (!reserved) return Result.OutOfStock(line.Sku);
        }
        _orders.Add(order);
        await _uow.SaveChangesAsync(ct);
        _log.LogInformation("Order {OrderId} placed", order.Id);
        return order.Id;
    }
}
```

Notice: zero EF references in the handler. Tests mock `IOrderRepository`, `IInventoryRepository`, `IUnitOfWork`.

### CQRS read side — query objects, no repository

```csharp
app.MapGet("/customers/{id:guid}/orders", async (
    Guid id, int page, int pageSize, IOrderSummaryQuery query, CancellationToken ct) =>
{
    var result = await query.RunAsync(id, page, pageSize, ct);
    return Results.Ok(result);
});
```

The query object talks to `DbContext` directly and projects to DTOs — fast, type-safe, no aggregate loading overhead.

### Azure SQL with retry strategy

Repositories run inside an `IExecutionStrategy.ExecuteAsync` block at the Unit of Work boundary. The repository methods themselves don't open transactions; the UoW does:

```csharp
public sealed class EfUnitOfWork : IUnitOfWork
{
    private readonly OrdersDbContext _db;
    public EfUnitOfWork(OrdersDbContext db) => _db = db;

    public Task<int> SaveChangesAsync(CancellationToken ct)
    {
        var strategy = _db.Database.CreateExecutionStrategy();
        return strategy.ExecuteAsync(() => _db.SaveChangesAsync(ct));
    }
}
```

### Multi-tenant SaaS

Repositories centralize the tenant filter so application code can't forget:

```csharp
public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct)
{
    var tenantId = _tenantAccessor.TenantId;
    return _db.Orders
        .Where(o => o.TenantId == tenantId)
        .Include(o => o.Lines)
        .FirstOrDefaultAsync(o => o.Id == id, ct);
}
```

Combined with `HasQueryFilter`, this is defense in depth.

### Testing

```csharp
public sealed class OrderRepositoryTests : IAsyncLifetime
{
    private MsSqlContainer _container = null!;
    private OrdersDbContext _db = null!;

    public async Task InitializeAsync()
    {
        _container = new MsSqlBuilder().Build();
        await _container.StartAsync();
        var options = new DbContextOptionsBuilder<OrdersDbContext>()
            .UseSqlServer(_container.GetConnectionString()).Options;
        _db = new OrdersDbContext(options);
        await _db.Database.MigrateAsync();
    }

    [Fact]
    public async Task GetByIdAsync_loads_order_with_lines()
    {
        var order = Order.Create(new CustomerId(Guid.NewGuid()),
            new[] { new OrderLine("SKU-1", 2, 19.99m) });
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();

        var repo = new OrderRepository(_db);
        var loaded = await repo.GetByIdAsync(order.Id, default);

        Assert.NotNull(loaded);
        Assert.Single(loaded!.Lines);
    }

    public async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await _container.DisposeAsync();
    }
}
```

### CI/CD

Run Testcontainers SQL Server in the pipeline. GitHub Actions has Docker available; just `services: mssql:` in the workflow. Tests are slow (~3s startup) but high-fidelity. For per-test isolation use transactions + rollback rather than recreating the database.

## Code Example — Before and After

### Before — generic repository over every entity

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id);
    Task<IEnumerable<T>> GetAllAsync();
    IQueryable<T> Query();
    void Add(T entity);
    void Update(T entity);
    void Remove(T entity);
    Task<int> SaveChangesAsync();
}

public sealed class EfRepository<T> : IRepository<T> where T : class
{
    private readonly DbContext _db;
    public EfRepository(DbContext db) => _db = db;
    public Task<T?> GetByIdAsync(object id) => _db.Set<T>().FindAsync(id).AsTask();
    public async Task<IEnumerable<T>> GetAllAsync() => await _db.Set<T>().ToListAsync();
    public IQueryable<T> Query() => _db.Set<T>();
    public void Add(T entity) => _db.Set<T>().Add(entity);
    public void Update(T entity) => _db.Set<T>().Update(entity);
    public void Remove(T entity) => _db.Set<T>().Remove(entity);
    public Task<int> SaveChangesAsync() => _db.SaveChangesAsync();
}

public sealed class OrderService
{
    private readonly IRepository<Order> _orders;
    public OrderService(IRepository<Order> orders) => _orders = orders;

    public async Task ShipAsync(Guid id)
    {
        // (1) GetByIdAsync doesn't include Lines — partial aggregate
        var order = await _orders.GetByIdAsync(id);

        // (2) caller must know to use Query() to filter — leaky
        var pending = await _orders.Query().Where(o => o.Status == OrderStatus.Pending).ToListAsync();

        // (3) GetAllAsync is a footgun nobody removed
        var all = await _orders.GetAllAsync();

        order!.MarkShipped(); // NullReferenceException waiting to happen
        await _orders.SaveChangesAsync();
    }
}
```

### After — aggregate-shaped repository + dedicated read-side query

```csharp
// Domain project — no EF reference
namespace Orders.Domain;

public sealed class Order
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public byte[] RowVersion { get; private set; } = default!;
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public static Order Create(CustomerId customerId, IEnumerable<OrderLine> lines) { /* ... */ }
    public void MarkShipped() { /* enforce invariants */ }
}

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    void Add(Order order);
}

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// Infrastructure project — EF Core
namespace Orders.Infrastructure;

public sealed class EfOrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _db;
    public EfOrderRepository(OrdersDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
        _db.Orders
            .Include(o => o.Lines)
            .AsSplitQuery()
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public void Add(Order order) => _db.Orders.Add(order);
}

// Application project — depends only on Orders.Domain
namespace Orders.Application;

public sealed class ShipOrderHandler
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly ILogger<ShipOrderHandler> _log;

    public ShipOrderHandler(
        IOrderRepository orders, IUnitOfWork uow, ILogger<ShipOrderHandler> log)
    {
        _orders = orders; _uow = uow; _log = log;
    }

    public async Task<Result> HandleAsync(ShipOrder cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();

        order.MarkShipped(); // invariants enforced inside aggregate
        await _uow.SaveChangesAsync(ct);

        _log.LogInformation("Order {OrderId} shipped", cmd.OrderId);
        return Result.Success;
    }
}

// Read side — query object, not a repository
public sealed class OrderListQuery
{
    private readonly OrdersDbContext _db;
    public OrderListQuery(OrdersDbContext db) => _db = db;

    public Task<List<OrderSummaryDto>> RunAsync(
        Guid customerId, int page, int pageSize, CancellationToken ct) =>
        _db.Orders.AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .Skip((page - 1) * pageSize).Take(pageSize)
            .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Status, o.CreatedAt))
            .ToListAsync(ct);
}
```

The after version: clean layers, domain ignorant of EF, aggregates loaded whole, read path projected for performance, every concern has a name.

## Interview Questions and Answers

### 1. EF Core's `DbContext` is already a Unit of Work and `DbSet<T>` is already a repository. So why would you ever add `IOrderRepository`?

**Why this matters:** Tests whether you can defend the pattern instead of cargo-culting it.

**Answer:** Three reasons that actually justify it: (1) your domain project must not reference EF Core (Hexagonal/Clean), so you need an interface in the domain layer; (2) you're modeling DDD aggregates and want a single method like `GetByIdAsync` that always loads the aggregate whole (with `Include` and `AsSplitQuery`); (3) you want a guaranteed place to enforce cross-cutting filters like tenant id or soft delete. If none of those apply, skip the abstraction — adding `IOrderRepository` that just delegates to `_db.Orders` is pure ceremony.

**Trade-off:** Pattern enforcement costs interface + impl + DI registration. Worth it when boundary is real; cargo cult otherwise.

### 2. What's wrong with a generic `IRepository<T>` over every entity?

**Answer:** It exposes operations that don't fit every entity (`GetAllAsync` on `Orders` is a landmine), forces you to leak `IQueryable<T>` to keep flexibility (defeating the abstraction), hides EF features like `Include`, `AsSplitQuery`, `AsNoTracking`, and provides nothing `DbSet<T>` doesn't. Aggregate-shaped repositories with explicit, named methods carry their intent.

**Real project:** Inherited a codebase with `IRepository<T>` everywhere. Replaced with two aggregate repos and direct `DbContext` use for simple CRUD. Lines of code dropped 30%, p99 latency dropped because we could finally project properly.

### 3. Why is `Microsoft.EntityFrameworkCore.InMemory` a bad choice for repository tests?

**Why this matters:** Major real-world testing trap.

**Answer:** The in-memory provider doesn't implement relational semantics — no transactions, no foreign key enforcement, no concurrency tokens, no `ExecuteUpdate`, case-insensitive string comparisons differ from SQL Server. Tests pass against InMemory then fail in production. Microsoft's own docs recommend against it. Use SQLite in-memory for fast tests (real SQL parsing, real constraints) or Testcontainers SQL Server for highest fidelity.

### 4. Where does `SaveChangesAsync` belong — in the repository or in the Unit of Work?

**Answer:** Unit of Work. If the repository commits, you can't compose multiple repository calls atomically. You'd save the order, then fail to save the inventory reservation, and have inconsistent state. Put `SaveChangesAsync` on `IUnitOfWork` (which is typically backed by the same `DbContext` and registered as Scoped). The repository's job is to track changes; the UoW's job is to commit them.

### 5. How would you implement composable filtering without exposing `IQueryable<T>`?

**Answer:** Specification pattern. Define `ISpecification<T>` with `Criteria`, `Includes`, `Skip`, `Take`. Callers create specs like `new PendingOrdersForCustomer(id, page, pageSize)`. Repository applies the spec to its `IQueryable<T>` internally and returns the result. The Ardalis.Specification library is a solid implementation. Composition lives in named, testable spec classes; EF stays inside the repository.

**Trade-off:** Specifications are wordy compared to inline LINQ. Use them when you have many similar queries, not for one-off needs.

### 6. You're building a CQRS system. How does the repository pattern fit?

**Answer:** Asymmetrically. **Write side**: aggregate repositories (`IOrderRepository.GetByIdAsync`, `Add`) load the whole aggregate, the handler mutates it, UoW saves. **Read side**: no repositories — dedicated query objects (`IOrderSummaryQuery.RunAsync`) project directly to DTOs using `AsNoTracking` and `Select`. Loading aggregates to map to read DTOs wastes I/O and CPU. Two different patterns for two different concerns.

**Real project:** On a reporting-heavy commerce platform, switching the read side from "load aggregate + map" to "project directly to DTO" cut p95 by 60% on the dashboard endpoint.

### 7. How do you test a repository properly?

**Answer:** Integration test against a real database. Testcontainers SQL Server is the gold standard — apply migrations, run the test, assert. Use a transaction-per-test pattern (`BeginTransactionAsync` in `IAsyncLifetime.InitializeAsync`, `RollbackAsync` in `DisposeAsync`) to keep tests isolated and fast. Don't mock `DbSet<T>` — LINQ-to-Objects vs LINQ-to-EF differ in subtle ways and the mocks lie. Don't use the InMemory provider — it lies differently.

### 8. A junior added an `IOrderRepository` interface but every method just delegates to `_db.Orders` with no extra logic. Is this useful?

**Why this matters:** Tests whether you can call out cargo culting.

**Answer:** No — it's ceremony with no value. The interface adds a layer that exists only to be mocked, but you can't unit-test EF translation against a mock anyway. Either give the repository real behavior (aggregate loading with includes, centralized filters, domain-shaped methods) or remove it and inject `DbContext`. Pattern compliance for its own sake is technical debt, not clean code.

**Real project:** Reviewing a PR where a new microservice had `ICustomerRepository`, `IInvoiceRepository`, `IPaymentRepository`, all empty wrappers. Removed them, deleted 800 lines, no behavior change.

## Summary Checklist

- [ ] I default to using `DbContext` directly and add `IXxxRepository` only with a real reason.
- [ ] When I add a repository, it's aggregate-shaped, not generic.
- [ ] My repository interface lives in the domain project, implementation in infrastructure.
- [ ] I never expose `IQueryable<T>` from a repository's public surface.
- [ ] `SaveChangesAsync` lives in the Unit of Work, not the repository.
- [ ] For CQRS reads, I use dedicated query objects projecting to DTOs, not aggregate repositories.
- [ ] I test repositories with SQLite in-memory or Testcontainers SQL Server, never the InMemory provider.
- [ ] I use `AsSplitQuery()` when including multiple collections.
- [ ] I use specifications (Ardalis.Specification) when query composition demands it.
- [ ] I can explain to a teammate when *not* to add a repository, and defend that recommendation.
