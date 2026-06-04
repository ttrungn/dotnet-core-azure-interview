# Unit of Work

## What It Is

A **Unit of Work (UoW)** tracks every change made to a set of business objects during a single logical operation and then commits — or rolls back — all of those changes as one atomic database transaction. It is the bookkeeper for "what changed and when do we save it."

In .NET, you almost never need to write a Unit of Work from scratch, because **EF Core's `DbContext` *is* one**. Every entity attached to the context is tracked; calling `SaveChangesAsync` writes all pending inserts, updates, and deletes inside a single transaction.

```csharp
// One use case = one DbContext scope = one Unit of Work
public async Task<Result> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Create(cmd.CustomerId, cmd.Lines);
    await _orders.AddAsync(order, ct);              // staged

    var inventory = await _inventory.GetForSkusAsync(cmd.Skus, ct);
    inventory.Reserve(cmd.Skus);                    // staged (mutations tracked)

    _outbox.Enqueue(new OrderPlacedEvent(order.Id)); // staged

    await _uow.SaveChangesAsync(ct);                // ONE transaction, all-or-nothing
    return Result.Ok(order.Id);
}
```

If `SaveChangesAsync` fails, *nothing* is persisted: no order row, no inventory decrement, no outbox event. That guarantee — "all together or not at all" — is the entire point of the pattern.

## Why It Exists

Before .NET had EF Core (and back when teams used raw ADO.NET, NHibernate v1, or Linq2SQL), developers regularly shipped bugs like:

- The `Order` row was inserted, the `OrderLine` insert failed, and the customer got charged for an empty order.
- The customer's loyalty points were debited, the order was saved, but the email confirmation was queued *outside* the transaction and never fired because the process crashed.
- Two repositories each opened their own `SqlConnection` and their own implicit transaction — so partial writes leaked into production.

Martin Fowler formalized the pattern in *Patterns of Enterprise Application Architecture* (2002): a Unit of Work *"maintains a list of objects affected by a business transaction and coordinates the writing out of changes and the resolution of concurrency problems."* The pattern exists to make atomicity a first-class concept instead of something you stitched together by hand for every endpoint.

EF Core later built the pattern directly into the framework. That changed what teams need to *write*, but not what they need to *understand*.

## When To Use It

**Use a Unit of Work when:**

- A single business operation touches **multiple aggregates or tables** that must succeed or fail together (place order + reserve inventory + write outbox).
- You need a **clear transactional boundary** that matches a use case (one HTTP request, one Service Bus message, one Hangfire job).
- You want repositories to be **purely staging** — they add and remove entities; the UoW commits.

**Do not introduce an explicit `IUnitOfWork` interface when:**

- You are already using EF Core and your repositories share a `DbContext` per scope. `DbContext.SaveChangesAsync` *is* the UoW; wrapping it in another interface adds zero value.
- The operation is a single insert/update with no cross-aggregate consistency requirement.
- You only need a database transaction across *two queries*. Use `IDbContextTransaction` (`await _db.Database.BeginTransactionAsync(ct)`) directly.
- You are integrating with **distributed systems** (Cosmos DB partitions, Service Bus, external HTTP APIs). The UoW does not extend across processes — use the **Outbox pattern** instead.

## Why It Is Important

In production .NET / Azure systems, a correctly applied Unit of Work drives five outcomes:

1. **Atomicity of business operations.** A confirmed order, a debited inventory, and an outbox event all commit together or none of them do. Half-states are impossible.
2. **Consistent error semantics.** When `SaveChangesAsync` throws (constraint violation, deadlock, concurrency conflict), the caller can retry the entire use case knowing nothing was partially applied.
3. **Predictable transaction scope.** Database locks are held only for the duration of `SaveChangesAsync`, not scattered through the handler. That keeps SQL Server happy under load.
4. **Cheap testing.** Handlers can use an in-memory fake UoW and verify "did SaveChanges get called?" without a real database.
5. **A natural seam for the Outbox pattern.** Because everything is one transaction, you can write outbound integration events to an `OutboxMessages` table *in the same transaction* as the business change — guaranteeing at-least-once delivery without distributed transactions.

The pattern matters because it makes a hard property of distributed enterprise software — "this either happened or it didn't" — boring and obvious.

## How It's Used in C# / .NET

### 1. `DbContext` is the Unit of Work

```csharp
public sealed class OrdersDbContext : DbContext
{
    public OrdersDbContext(DbContextOptions<OrdersDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Inventory> Inventory => Set<Inventory>();
    public DbSet<OutboxMessage> Outbox => Set<OutboxMessage>();
}

// Registration — Scoped lifetime = one DbContext per HTTP request / message scope
builder.Services.AddDbContext<OrdersDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Orders")));
```

Every repository injected with `OrdersDbContext` in the same scope shares the **same instance** and the **same change tracker**. That is what makes a single `SaveChangesAsync` commit *all* their changes together.

### 2. A thin `IUnitOfWork` abstraction (optional)

Some teams still want to hide `DbContext` from application code so the application layer does not reference EF Core:

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

internal sealed class EfUnitOfWork : IUnitOfWork
{
    private readonly OrdersDbContext _db;
    public EfUnitOfWork(OrdersDbContext db) => _db = db;
    public Task<int> SaveChangesAsync(CancellationToken ct) => _db.SaveChangesAsync(ct);
}

builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>();
```

The interface buys you one thing: the application project no longer references `Microsoft.EntityFrameworkCore`. That is genuinely useful only if you take Clean Architecture seriously.

### 3. Explicit transactions when you need more than `SaveChangesAsync`

If a use case spans **multiple `SaveChangesAsync` calls** — usually because you must re-read after a save — wrap them in an explicit transaction:

```csharp
public async Task<Result> Handle(RefundOrderCommand cmd, CancellationToken ct)
{
    await using var tx = await _db.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);
    try
    {
        var order = await _orders.GetByIdForUpdateAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();

        order.IssueRefund(cmd.Amount, cmd.Reason);
        await _db.SaveChangesAsync(ct);

        // Need the refund's generated ID for the outbox event
        _outbox.Enqueue(new RefundIssuedEvent(order.Id, order.LastRefund!.Id));
        await _db.SaveChangesAsync(ct);

        await tx.CommitAsync(ct);
        return Result.Ok();
    }
    catch
    {
        await tx.RollbackAsync(ct);
        throw;
    }
}
```

### 4. Concurrency and retries

EF Core throws `DbUpdateConcurrencyException` when an optimistic-concurrency token (`[Timestamp]` / `xmin`) mismatch is detected at `SaveChangesAsync`. The UoW boundary makes the retry strategy obvious: catch, reload, re-apply the command, save again.

```csharp
public async Task<Result> Handle(ChangeShippingAddressCommand cmd, CancellationToken ct)
{
    for (var attempt = 0; attempt < 3; attempt++)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();
        order.ChangeShippingAddress(cmd.NewAddress);
        try
        {
            await _uow.SaveChangesAsync(ct);
            return Result.Ok();
        }
        catch (DbUpdateConcurrencyException)
        {
            _db.ChangeTracker.Clear(); // discard and retry on a fresh state
        }
    }
    return Result.Conflict("Address change failed after retries");
}
```

### 5. Per-request scope in ASP.NET Core

ASP.NET Core's request pipeline opens a scope per HTTP request. Anything registered as `Scoped` — `DbContext`, repositories, `IUnitOfWork` — is shared within that request and disposed when the response is sent. That alignment is what makes "one request = one UoW" the default with **zero extra code**.

### 6. Per-message scope in Azure Functions / MassTransit

When consuming a Service Bus message, the host opens a scope per message. Each message is one Unit of Work. If the handler throws, the message is abandoned (or dead-lettered) and *no* database changes survive — exactly the semantics you want for at-least-once messaging combined with the Outbox.

## Advantages

- **Atomicity for free** — one method call commits the entire use case.
- **No leaked partial state** — failures roll back automatically.
- **Centralized commit point** — easier to reason about, easier to log.
- **Pairs cleanly with the Outbox pattern** — reliable event publishing without 2PC.
- **Cheap to test** — fakes can capture "save was called" without a real DB.
- **Scoped lifetime in DI mirrors the natural unit of work** (request/message).

## Disadvantages

- **Single-database scope only** — does not extend across microservices, Cosmos partitions, or external APIs.
- **Long-running transactions hurt** — holding locks for the duration of a slow handler kills throughput in SQL Server.
- **Hidden coupling through the change tracker** — bugs can be tricky (`Detached` entities, navigation properties resurrecting deleted rows).
- **`IUnitOfWork` abstraction is redundant** when you are happy referencing EF Core in the application layer.
- **Encourages "god transactions"** — a tempting handler that does ten things in one save will eventually deadlock production.

## Common Mistakes

### 1. Calling `SaveChangesAsync` inside repositories

**Problem:** each repository commits its own changes, breaking the transactional boundary.

```csharp
// ❌
public async Task AddAsync(Order order, CancellationToken ct)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct); // commits here, before inventory was reserved
}
```

If the next line in the handler fails (`inventory.Reserve(...)` throws), the order is already in the database. Now you have an inconsistency that requires a human to fix.

**Fix:** repositories only stage. The handler calls `SaveChangesAsync` exactly once at the end.

### 2. Injecting two different `DbContext` instances into the same handler

**Problem:** each `DbContext` has its own change tracker and its own transaction. `SaveChangesAsync` on one does not commit the other.

```csharp
// ❌
public ConfirmOrderHandler(OrdersDbContext orders, InventoryDbContext inventory) { ... }
```

**Fix:** if both aggregates live in the same database, use **one** `DbContext`. If they live in separate databases, you do *not* have a single Unit of Work — use the Outbox pattern and accept eventual consistency.

### 3. Holding the transaction open for an external HTTP call

**Problem:** inside a transaction, the handler calls Stripe over the network. The DB locks are held for hundreds of milliseconds. Under load the database melts.

```csharp
// ❌
order.Confirm();
var charge = await _stripe.ChargeAsync(...);  // 800ms while holding row locks
order.SetPaymentIntent(charge.Id);
await _db.SaveChangesAsync(ct);
```

**Fix:** call external services *outside* the transaction. Persist the intent first, react to the webhook, or use the Outbox to publish a "payment requested" event.

### 4. Multiple `SaveChangesAsync` calls in one use case "to commit progressively"

**Problem:** "I want the order saved even if the email fails later." That breaks atomicity. Now the system has confirmed orders with no notifications and no recovery story.

**Fix:** persist business state in one `SaveChangesAsync`, then publish an event from the Outbox. The event handler is responsible for sending email and can retry on its own schedule.

### 5. Treating `IUnitOfWork` as a transaction manager and calling `BeginTransaction` everywhere

**Problem:** `SaveChangesAsync` already opens and commits a transaction. Wrapping every call in a manual `BeginTransactionAsync` is noise — and frequently double-commits or leaks open transactions if exception handling is wrong.

**Fix:** trust `SaveChangesAsync`. Use `BeginTransactionAsync` only when you genuinely need multiple saves to commit atomically.

### 6. Forgetting that `DbContext` is not thread-safe

**Problem:** a handler spawns `Task.WhenAll(saveOrder, saveInventory)` against the same `DbContext`. EF Core throws "A second operation was started on this context instance" — or worse, corrupts the change tracker.

**Fix:** treat one `DbContext` as a single-threaded session. If you need parallelism, give each task its own scope (and its own UoW), or sequentialize the work.

## Best Practices

- Let the **`DbContext` be the Unit of Work** in EF Core projects. Do not invent a parallel abstraction without a reason.
- One `SaveChangesAsync` **per use case**. If you need more, document why in the handler.
- Register `DbContext` as **`Scoped`** so it aligns with the request / message lifetime.
- Keep all external I/O (HTTP, Service Bus publish, file uploads) **outside the transaction**. Use the Outbox to bridge.
- Implement **optimistic concurrency** (`[Timestamp]` / `RowVersion`) on aggregates and retry on `DbUpdateConcurrencyException` rather than locking pessimistically.
- For multi-step saves, use `BeginTransactionAsync` with `try/catch`/`RollbackAsync` — do not leave transactions open across `await` boundaries you do not control.
- Add an `IUnitOfWork` interface **only** if you must keep the application project free of EF Core references.
- Log the transaction outcome and elapsed time. Slow commits are an early warning of lock contention.
- For cross-service consistency, use **sagas** + **outbox**. Do not try to extend the UoW across processes.
- In Azure Functions / Worker Services, ensure each message processes in its own DI scope so each message is one UoW.

## Related Concepts

- [architecture/repositories.md](repositories.md) — repositories stage; UoW commits.
- [architecture/outbox-pattern.md](outbox-pattern.md) — pairs with the UoW for reliable event publishing.
- [architecture/saga-pattern.md](saga-pattern.md) — the cross-process replacement for a UoW.
- [architecture/aggregates.md](aggregates.md) — an aggregate is the unit of consistency *inside* a UoW.
- [data-access/transactions.md](../data-access/transactions.md) — isolation levels and explicit transactions.
- [data-access/dbcontext-lifetime.md](../data-access/dbcontext-lifetime.md) — Scoped lifetime mechanics.
- [data-access/optimistic-concurrency.md](../data-access/optimistic-concurrency.md) — how the UoW detects conflicts at commit time.

## Real-World Usage

### ASP.NET Core ordering API

`OrdersDbContext` is registered as `Scoped`. Every request handler injects repositories that share that context. A single `await _db.SaveChangesAsync(ct)` at the end of the handler commits the order, the inventory reservation, and the outbox row in one transaction. Middleware logs the elapsed transaction time to Application Insights so the team can spot lock-contention regressions.

### Azure Functions order processor

A Service Bus-triggered function consumes an `OrderShipped` message. The Functions host creates a DI scope per message; `OrdersDbContext` is `Scoped` so it lives only for that message. The handler loads the aggregate, mutates it, and saves once. If the save throws, the message is abandoned, retried, eventually dead-lettered — and the DB stays clean.

### Outbox pattern integration

Inside the same `SaveChangesAsync` that confirms the order, the handler writes an `OrderConfirmed` row to the `OutboxMessages` table. A background dispatcher (Hangfire job, separate Worker Service) reads `OutboxMessages` and publishes to Azure Service Bus, marking each row sent. Because the business change and the event row are in the same transaction, no event is ever lost and the system avoids distributed transactions.

### Multi-step saves with explicit transactions

A refund flow needs to (1) save the refund record to obtain its database-generated `Id`, then (2) write an outbox event that references that `Id`. Two `SaveChangesAsync` calls wrapped in `BeginTransactionAsync`. If either fails, `RollbackAsync` ensures the refund never went out and never published an event.

### Multi-tenant SaaS

`OrdersDbContext` overrides `SaveChangesAsync` to stamp `TenantId` on every added entity from `ITenantContext.Current`. The UoW becomes the chokepoint that enforces tenancy: no engineer can write a query path that bypasses tenant tagging on insert.

## Code Example — Before and After

### Before — fragmented saves, partial failures

```csharp
[HttpPost("api/orders")]
public async Task<IActionResult> PlaceOrder(PlaceOrderRequest req, CancellationToken ct)
{
    var order = new Order(req.CustomerId, req.Lines);
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct);   // commit #1

    foreach (var line in req.Lines)
    {
        var inv = await _db.Inventory.FindAsync(new object?[] { line.Sku }, ct);
        inv!.Reserve(line.Quantity);
        await _db.SaveChangesAsync(ct); // commit #2..N — per line!
    }

    await _bus.PublishAsync(new OrderPlacedEvent(order.Id), ct); // outside any transaction

    return Ok(order.Id);
}
```

Problems: the order can exist with only some inventory reserved if the loop crashes; the bus publish can fail silently leaving the database confirmed but downstream systems unaware. There is no atomic boundary.

### After — one Unit of Work, one transaction, outbox for reliable publishing

```csharp
public sealed record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<OrderLineDto> Lines) : IRequest<Result<OrderId>>;

public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Result<OrderId>>
{
    private readonly IOrderRepository _orders;
    private readonly IInventoryRepository _inventory;
    private readonly IOutbox _outbox;
    private readonly IUnitOfWork _uow;
    private readonly ILogger<PlaceOrderHandler> _logger;

    public PlaceOrderHandler(
        IOrderRepository orders,
        IInventoryRepository inventory,
        IOutbox outbox,
        IUnitOfWork uow,
        ILogger<PlaceOrderHandler> logger)
    {
        _orders = orders;
        _inventory = inventory;
        _outbox = outbox;
        _uow = uow;
        _logger = logger;
    }

    public async Task<Result<OrderId>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var inventory = await _inventory.GetForSkusAsync(cmd.Lines.Select(l => l.Sku).ToArray(), ct);
        var reservation = inventory.TryReserve(cmd.Lines);
        if (!reservation.IsSuccess) return Result<OrderId>.Failure(reservation.Error);

        var order = Order.Create(new CustomerId(cmd.CustomerId), cmd.Lines);
        await _orders.AddAsync(order, ct);

        _outbox.Enqueue(new OrderPlacedEvent(order.Id.Value, order.CustomerId.Value, order.Total.Amount));

        await _uow.SaveChangesAsync(ct);   // ONE commit. ONE transaction.
        _logger.LogInformation("Order {OrderId} placed; outbox event queued", order.Id);
        return Result<OrderId>.Ok(order.Id);
    }
}

[ApiController, Route("api/orders")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpPost]
    public Task<IActionResult> Place(PlaceOrderRequest req, CancellationToken ct) =>
        mediator.Send(new PlaceOrderCommand(req.CustomerId, req.Lines), ct).ToActionResult();
}
```

Now: order, inventory reservation, and outbox event commit together. If any step throws, *nothing* is persisted and the client gets a clean error to retry. A background dispatcher publishes the outbox row to Service Bus and marks it sent — reliably, without distributed transactions.

## Interview Questions and Answers

### 1. EF Core's `DbContext` already implements Unit of Work. Why do some teams still add an `IUnitOfWork` interface?

**Why this matters:** the interviewer wants to know whether you can resist patterns added "for symmetry" and whether you understand the only legitimate reason to add this one.

**Answer:** The only honest reason is **dependency direction**. If the application project must not reference `Microsoft.EntityFrameworkCore`, you expose `IUnitOfWork.SaveChangesAsync` and implement it in infrastructure. Otherwise the interface adds nothing — `DbContext` is already a UoW. I do not add `IUnitOfWork` on small services; I do add it in Clean-Architecture-style solutions where layer purity is a hard rule.

**Trade-off:** one more class, one more interface, and one more DI registration in exchange for a layering rule the compiler can enforce.

### 2. A handler updates an Order and reserves Inventory in the same use case. How do you make those atomic?

**Why this matters:** this is the textbook UoW scenario and the textbook place teams get it wrong.

**Answer:** Both repositories share the same `DbContext` because both are `Scoped`. The handler calls `await _uow.SaveChangesAsync(ct)` **once** at the end. EF Core wraps both inserts/updates in a single transaction. If `SaveChangesAsync` throws, neither change is persisted.

**Real project:** I once inherited a checkout where the team called `SaveChangesAsync` twice — once after the order insert, once after the inventory update. About 0.3% of orders were saved with no inventory reservation because the second save deadlocked. Collapsing to one save eliminated the inconsistency.

### 3. You need to publish a Service Bus event after saving an order. How do you avoid the "saved but never published" bug?

**Why this matters:** the most common production failure mode in distributed .NET systems.

**Answer:** Write the event to an **Outbox** table *inside the same `SaveChangesAsync`* as the order. A background dispatcher reads the outbox and publishes to Service Bus, marking each row sent. Because the business change and the event row commit together, you can never end up with an order that has no event. Service Bus's at-least-once semantics combined with idempotent consumers handle duplicates.

**Trade-off:** small added latency before the message hits the bus (typically seconds), and an extra table + worker to maintain. In exchange you avoid distributed transactions and ghost data.

### 4. When should you call `BeginTransactionAsync` explicitly instead of trusting `SaveChangesAsync`?

**Why this matters:** unnecessary explicit transactions are noise; missing them when needed causes real bugs.

**Answer:** Only when the use case needs **multiple** `SaveChangesAsync` calls that must commit together — usually because the second save depends on database-generated IDs from the first. Wrap them in `await using var tx = await _db.Database.BeginTransactionAsync(ct)` with explicit `CommitAsync` and `RollbackAsync` in `catch`.

### 5. A long-running handler holds a transaction open while calling Stripe. What goes wrong?

**Why this matters:** lock contention is the silent killer of SQL Server under load.

**Answer:** Database row locks are held for the entire network round-trip to Stripe — easily hundreds of milliseconds. Under load you get blocking chains, then deadlocks, then 500s. The fix is to keep the transaction tight: persist a "payment requested" record, commit, call Stripe outside the transaction, and update the order state asynchronously via webhook or outbox. The UoW boundary should never include an external HTTP call.

### 6. How do you handle `DbUpdateConcurrencyException` from `SaveChangesAsync`?

**Why this matters:** optimistic concurrency is the standard pattern; misunderstanding it causes data loss.

**Answer:** I catch it, clear the change tracker (`_db.ChangeTracker.Clear()`), reload the aggregate, re-apply the command, and call `SaveChangesAsync` again — usually with a small retry budget (2–3 attempts). If the command is naturally idempotent (e.g., "mark shipped") the retry is safe. If conflicting intents collide, the second one fails with a clear `409 Conflict`.

**Real project:** an inventory adjust endpoint without retries surfaced ~20 user-facing errors per day during a sale. Adding a 3-attempt retry eliminated all but two of them, and those two were genuine concurrent intents that deserved the 409.

### 7. How does the Unit of Work apply when consuming a Service Bus message in an Azure Function?

**Why this matters:** background processing is where atomicity and retry semantics get muddled.

**Answer:** The Functions host opens a DI scope per message. `OrdersDbContext` is `Scoped`, so each message gets its own UoW. The handler mutates aggregates, writes outbox rows for any downstream events, and calls `SaveChangesAsync` once. If the call throws, the host abandons the message; Service Bus redelivers; eventually the message goes to dead-letter. The database is never left in a partially applied state.

### 8. Can a Unit of Work span multiple microservices?

**Why this matters:** distinguishing local atomicity from distributed consistency is a senior-level marker.

**Answer:** No. A Unit of Work is bounded to a single database (single `DbContext`). Across microservices you do not have a UoW — you have a **saga** (orchestrated or choreographed) plus **outbox-driven events** for eventual consistency. Pretending otherwise leads to people reaching for `TransactionScope` over MSDTC, which is operationally a disaster in Azure-hosted environments.

## Summary Checklist

- [ ] I treat `DbContext` as the Unit of Work in EF Core projects.
- [ ] I call `SaveChangesAsync` exactly once per use case.
- [ ] I keep repositories side-effect-free; only the UoW commits.
- [ ] I register `DbContext` as `Scoped` so it aligns with the request / message lifetime.
- [ ] I never hold a transaction open across an external HTTP call.
- [ ] I pair the UoW with the Outbox pattern for reliable event publishing.
- [ ] I handle `DbUpdateConcurrencyException` with bounded retries.
- [ ] I only add an explicit `IUnitOfWork` interface when the application layer must be EF-Core-free.
- [ ] I use `BeginTransactionAsync` only when multiple saves must commit together.
- [ ] I know a UoW cannot span microservices — sagas and outboxes do that job.
