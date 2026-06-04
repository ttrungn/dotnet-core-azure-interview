# Transactions

## What It Is

A transaction is a unit of work that the database treats as **all-or-nothing**. Either every change inside it commits together, or every change rolls back together, leaving the database as if none of them happened. The classic guarantees are summarized as **ACID**:

- **Atomicity** — all statements succeed or none do.
- **Consistency** — constraints, triggers, and invariants hold before and after.
- **Isolation** — concurrent transactions don't see each other's uncommitted state (degree depends on isolation level).
- **Durability** — once committed, the change survives a crash.

In .NET, you typically interact with transactions in three ways:

1. **Implicit per-`SaveChangesAsync()`** — EF Core wraps a single `SaveChanges` call in a transaction automatically.
2. **Explicit `IDbContextTransaction`** — when one logical operation spans multiple `SaveChanges` calls, raw SQL, or stored procedures.
3. **Ambient `TransactionScope`** — historical .NET pattern; still useful in narrow cases, but pitfalls with async and distributed coordination.

```csharp
// EF Core implicit transaction — already atomic
order.MarkPaid();
order.Lines.Add(new OrderLine(...));
await db.SaveChangesAsync(ct); // single transaction, all changes commit or none

// EF Core explicit transaction — multi-step workflow
await using var tx = await db.Database.BeginTransactionAsync(ct);
await db.SaveChangesAsync(ct);
await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Inventory SET Reserved = Reserved + {qty} WHERE Sku = {sku}", ct);
await tx.CommitAsync(ct);
```

## Why It Exists

Without transactions, partial failures corrupt data. Imagine a bank transfer: debit `$100` from account A, credit `$100` to account B. If the process crashes between the two updates, the money has disappeared. ACID transactions guarantee that either both updates happen or neither does — the invariant "total money in the system is constant" is preserved.

Before relational databases standardized this, applications hand-rolled error recovery with flag columns, journal tables, and manual cleanup scripts. Most of them got it subtly wrong. Transactions became the universal primitive because correctness under failure is too hard to reinvent per project.

In modern cloud workloads the pressure shifted:

- **Single-database transactions** are still cheap and the right answer for most operations within one service.
- **Distributed transactions** (across multiple databases, queues, services) using two-phase commit / MSDTC turned out to be too slow, too fragile, and incompatible with cloud-managed services. They've been replaced by patterns like the **outbox pattern** and **sagas** that achieve *eventual* consistency rather than ACID across services.

## When To Use It

**Use an explicit transaction when:**

- You need multiple `SaveChangesAsync` calls to commit together (e.g., header save → run domain logic → save lines).
- You combine EF Core changes with raw SQL or `ExecuteUpdate` that must be atomic with them.
- You're implementing the outbox pattern — `SaveChanges` for business data + outbox table must be one transaction.
- You need to elevate the isolation level (snapshot, serializable) for one operation.
- You need savepoints for partial rollback inside a longer workflow.

**Do not use an explicit transaction when:**

- A single `SaveChangesAsync` already covers the work (it's already transactional).
- You'd be calling external services (HTTP, Service Bus, payment gateway) inside the transaction — the lock will be held across network latency. Use the outbox pattern instead.
- You're tempted to span two databases — use sagas or the outbox.
- You're using EF Core retry strategy (`EnableRetryOnFailure`) without wrapping in an `IExecutionStrategy.ExecuteAsync` block. This **will throw** at runtime.
- You're doing a read-only query — open transactions on reads hold shared locks unnecessarily.

## Why It Is Important

In production, the absence of correct transaction handling is one of the highest-impact bug categories:

- **Lost money / inventory / orders** — partial commits become customer-visible data corruption.
- **Phantom records** — order header saved, lines failed, customer sees an order with zero items.
- **Hard-to-reproduce bugs** — depends on which step crashed; only happens under load.
- **Audit failure** — auditors require provable atomicity for financial flows; a missing transaction is a compliance issue.

Properly used transactions deliver:

- **Correctness across multi-step operations.**
- **Predictable failure mode** — either fully applied or fully reverted, never half.
- **Compatibility with EF Core retry strategies via `IExecutionStrategy`** so transient Azure SQL errors don't break workflows.
- **Composability with optimistic concurrency** — a `DbUpdateConcurrencyException` rolls the whole transaction back.

## How It's Used in C# / .NET

### 1. Implicit transaction via `SaveChangesAsync`

EF Core wraps every `SaveChanges` call in a transaction automatically. Multiple inserts/updates/deletes accumulated in the change tracker commit atomically.

```csharp
order.MarkPaid();
foreach (var line in newLines) order.Lines.Add(line);
await db.SaveChangesAsync(ct); // atomic
```

### 2. Explicit `IDbContextTransaction`

```csharp
await using var tx = await db.Database.BeginTransactionAsync(
    IsolationLevel.ReadCommitted, ct);
try
{
    await db.SaveChangesAsync(ct);
    await db.Database.ExecuteSqlInterpolatedAsync(
        $"UPDATE Inventory SET Reserved = Reserved + {qty} WHERE Sku = {sku}", ct);
    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

### 3. Isolation levels (SQL Server / Azure SQL)

| Level                  | Reads see uncommitted? | Phantoms? | Use when                                         |
| ---------------------- | ---------------------- | --------- | ------------------------------------------------ |
| **Read Uncommitted**   | Yes                    | Yes       | Almost never. Reporting on stale data only.      |
| **Read Committed**     | No                     | Yes       | Default. Most OLTP workloads.                    |
| **Snapshot**           | No (versioned)         | No        | Reads should not block writes (read-heavy APIs). |
| **Serializable**       | No                     | No        | Strict invariants (account balances, inventory). |

Enable snapshot isolation per database:

```sql
ALTER DATABASE OrdersDb SET ALLOW_SNAPSHOT_ISOLATION ON;
```

Then use it:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(
    IsolationLevel.Snapshot, ct);
```

### 4. `TransactionScope` (ambient)

```csharp
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled); // <-- required for async

await db1.SaveChangesAsync(ct);
await db2.SaveChangesAsync(ct);
scope.Complete();
```

**Critical:** without `TransactionScopeAsyncFlowOption.Enabled`, the scope does **not** flow across `await` boundaries — you get random "transaction has aborted" errors.

### 5. `IExecutionStrategy` — required with retry-on-failure

When you enable `EnableRetryOnFailure()` (recommended on Azure SQL), EF Core refuses to let you start a transaction directly because a retry mid-transaction would commit nonsense. You must wrap in the execution strategy:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    await db.SaveChangesAsync(ct);
    await db.Database.ExecuteSqlInterpolatedAsync($"UPDATE ...", ct);
    await tx.CommitAsync(ct);
});
```

Without this wrapper you get: *"The configured execution strategy 'SqlServerRetryingExecutionStrategy' does not support user-initiated transactions."*

### 6. Savepoints

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
await db.SaveChangesAsync(ct);

await tx.CreateSavepointAsync("AfterHeader", ct);
try
{
    await ProcessLinesAsync(ct);
}
catch (LineValidationException)
{
    await tx.RollbackToSavepointAsync("AfterHeader", ct); // header still applied
}
await tx.CommitAsync(ct);
```

### 7. NuGet packages

- `Microsoft.EntityFrameworkCore` (8.0+)
- `Microsoft.EntityFrameworkCore.SqlServer`
- `System.Transactions.Local` (for `TransactionScope`)
- `Polly` — supplementary retry; prefer `EnableRetryOnFailure` + `IExecutionStrategy` for DB transients.

## Advantages

- **All-or-nothing correctness** — partial state cannot leak out.
- **Composes with optimistic concurrency** — conflicts roll back the entire unit of work.
- **Battle-tested isolation levels** for the cases where read consistency matters.
- **Cheap within a single database** — modern engines optimize tiny short transactions.
- **Supports savepoints** for fine-grained partial rollback.

## Disadvantages

- **Locks are real** — long transactions block other writers and hurt throughput.
- **Distributed transactions are essentially unusable in cloud** — MSDTC is unsupported on Azure SQL Managed Instance for most cross-DB scenarios.
- **Easy to misuse with async** — `TransactionScope` without `AsyncFlowOption.Enabled` is a footgun.
- **`EnableRetryOnFailure` adds ceremony** — manual `IExecutionStrategy` wrapping is required.
- **Deadlock potential** — multiple concurrent transactions touching the same rows in different orders.

## Common Mistakes

### 1. Holding a transaction across a network call

```csharp
// BAD — Service Bus call inside the transaction
await using var tx = await db.Database.BeginTransactionAsync(ct);
order.MarkPaid();
await db.SaveChangesAsync(ct);
await _serviceBus.PublishAsync(new OrderPaid(order.Id), ct); // 200ms+ lock held
await tx.CommitAsync(ct);
```

If Service Bus is slow, your transaction holds locks for hundreds of milliseconds. If it fails, the database commit is rolled back but the message might already be sent (or vice versa).

```csharp
// GOOD — outbox pattern
await using var tx = await db.Database.BeginTransactionAsync(ct);
order.MarkPaid();
db.OutboxMessages.Add(new OutboxMessage(typeof(OrderPaid), order.Id));
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
// Separate background worker publishes from the outbox.
```

See [architecture/outbox-pattern.md](architecture/outbox-pattern.md).

### 2. Forgetting `TransactionScopeAsyncFlowOption.Enabled`

```csharp
// BAD — random "transaction has aborted" errors
using var scope = new TransactionScope();
await db.SaveChangesAsync(ct); // scope doesn't flow across await
scope.Complete();
```

```csharp
// GOOD
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    TransactionScopeAsyncFlowOption.Enabled);
await db.SaveChangesAsync(ct);
scope.Complete();
```

### 3. Starting a transaction with retry-on-failure enabled, no execution strategy

```csharp
services.AddDbContext<OrdersDbContext>(o =>
    o.UseSqlServer(cs, sql => sql.EnableRetryOnFailure()));

// BAD — throws at runtime
await using var tx = await db.Database.BeginTransactionAsync(ct);
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

```csharp
// GOOD
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
});
```

### 4. Using `Serializable` everywhere "to be safe"

```csharp
// BAD — serializable locks ranges, kills throughput
await using var tx = await db.Database.BeginTransactionAsync(
    IsolationLevel.Serializable, ct);
var orders = await db.Orders.Where(o => o.Status == OrderStatus.Pending).ToListAsync(ct);
// ... long workflow ...
await tx.CommitAsync(ct);
```

```csharp
// GOOD — Read Committed is the default for a reason
await using var tx = await db.Database.BeginTransactionAsync(
    IsolationLevel.ReadCommitted, ct);
// Use snapshot or row-locks only where invariants truly demand it.
```

### 5. Trying to span two databases with `TransactionScope` on Azure SQL

```csharp
// BAD — promotes to MSDTC, not supported on Azure SQL Database
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await ordersDb.SaveChangesAsync(ct);
await inventoryDb.SaveChangesAsync(ct); // boom
scope.Complete();
```

```csharp
// GOOD — saga or outbox between bounded contexts
await ordersDb.SaveChangesAsync(ct);
await _serviceBus.PublishAsync(new OrderPlaced(...), ct); // inventory service reacts
```

### 6. Catching the exception and committing anyway

```csharp
// BAD
try { await db.SaveChangesAsync(ct); }
catch { /* ignored */ }
await tx.CommitAsync(ct); // committing an inconsistent state
```

Always rollback on failure (or rely on `using` to rollback when `Commit` isn't called).

### 7. Forgetting `await using` on the transaction

```csharp
// BAD — synchronous disposal can deadlock
using var tx = db.Database.BeginTransaction();
```

```csharp
// GOOD
await using var tx = await db.Database.BeginTransactionAsync(ct);
```

## Best Practices

- **Default to letting `SaveChangesAsync` be your transaction.** Open an explicit transaction only when you genuinely need multiple steps.
- **Keep transactions short.** Measure: an OLTP transaction should be tens of milliseconds, not seconds.
- **Never call external services inside an open transaction.** Use the outbox pattern.
- **Always pair `BeginTransactionAsync` with `CreateExecutionStrategy()` when retry-on-failure is enabled.**
- **Use Read Committed unless you can articulate why you need stronger isolation.**
- **Use snapshot isolation for read-heavy APIs that conflict with writers.**
- **Use savepoints to localize partial rollback inside long workflows.**
- **Log transaction start, commit, and rollback** with correlation ids — they're invaluable in incident postmortems.
- **Test rollback paths** — inject failures between steps and assert the database is unchanged.
- **Never use distributed transactions across cloud-managed databases.** Replace with sagas or outbox.

## Related Concepts

- [data-access/optimistic-concurrency.md](data-access/optimistic-concurrency.md) — what to do when concurrent transactions conflict
- [data-access/unit-of-work.md](architecture/unit-of-work.md) — domain-level grouping that maps onto a transaction
- [architecture/outbox-pattern.md](architecture/outbox-pattern.md) — atomic "save + publish" without distributed transactions
- [architecture/saga-pattern.md](architecture/saga-pattern.md) — long-running multi-service workflows with compensations
- [data-access/repository-pattern-with-ef-core.md](data-access/repository-pattern-with-ef-core.md) — where to expose transaction boundaries
- [azure/azure-sql.md](azure/azure-sql.md) — Azure SQL transient errors and retry behavior
- [aspnet-core/error-handling-and-problem-details.md](aspnet-core/error-handling-and-problem-details.md) — surfacing transaction failures as HTTP responses

## Real-World Usage

### ASP.NET Core — order checkout

Checkout writes the order header, lines, payment record, and an outbox message in one transaction. If any step fails (validation, optimistic concurrency on inventory), everything rolls back.

```csharp
public async Task<CheckoutResult> CheckoutAsync(CheckoutRequest req, CancellationToken ct)
{
    var strategy = _db.Database.CreateExecutionStrategy();
    return await strategy.ExecuteAsync(async () =>
    {
        await using var tx = await _db.Database.BeginTransactionAsync(ct);

        var order = Order.Create(req.CustomerId, req.Lines);
        _db.Orders.Add(order);

        foreach (var line in order.Lines)
        {
            var rows = await _db.Inventory
                .Where(i => i.Sku == line.Sku && i.Available >= line.Quantity)
                .ExecuteUpdateAsync(s => s
                    .SetProperty(i => i.Available, i => i.Available - line.Quantity)
                    .SetProperty(i => i.Reserved, i => i.Reserved + line.Quantity), ct);

            if (rows == 0)
            {
                await tx.RollbackAsync(ct);
                return CheckoutResult.OutOfStock(line.Sku);
            }
        }

        _db.OutboxMessages.Add(OutboxMessage.For(new OrderPlaced(order.Id)));

        await _db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
        return CheckoutResult.Success(order.Id);
    });
}
```

### Azure SQL

Configure transient retry on the connection:

```csharp
services.AddDbContext<OrdersDbContext>(o =>
    o.UseSqlServer(cs, sql => sql
        .EnableRetryOnFailure(maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(10), null)));
```

Every explicit transaction must now go through `CreateExecutionStrategy().ExecuteAsync(...)`. Otherwise EF throws.

### Azure Service Bus / Event Grid

Never publish inside a database transaction. Use the **outbox pattern**: write the outgoing message to an `OutboxMessages` table inside the business transaction; a background worker reads the outbox and publishes, marking the row as sent.

### Testing

- **Unit tests:** mock the repository or use SQLite with a real transaction so rollback behavior is observable.
- **Integration tests:** use Testcontainers with SQL Server. Open a transaction in the test, run the system under test, assert state, then rollback to keep tests isolated and fast.

```csharp
[Fact]
public async Task Checkout_rolls_back_when_inventory_insufficient()
{
    await using var tx = await Db.Database.BeginTransactionAsync();
    var result = await Service.CheckoutAsync(new CheckoutRequest(...));
    Assert.Equal(CheckoutOutcome.OutOfStock, result.Outcome);
    Assert.Empty(await Db.Orders.ToListAsync()); // nothing persisted
    await tx.RollbackAsync();
}
```

### CI/CD

Run integration tests against a SQL Server Linux container in the pipeline. Apply migrations, run tests, drop database. Without real transactions, you cannot test failure-then-rollback behavior at all.

## Code Example — Before and After

### Before — non-atomic checkout with a distributed-transaction smell

```csharp
public async Task<Guid> CheckoutAsync(CheckoutRequest req, CancellationToken ct)
{
    var order = Order.Create(req.CustomerId, req.Lines);
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct); // (1) order saved

    foreach (var line in order.Lines)
    {
        // (2) separate SaveChanges per line — partial state possible
        var inv = await _db.Inventory.FirstAsync(i => i.Sku == line.Sku, ct);
        inv.Available -= line.Quantity;
        await _db.SaveChangesAsync(ct);
    }

    // (3) external publish — if this throws, order + inventory already committed,
    //     downstream services never hear about the order
    await _serviceBus.PublishAsync(new OrderPlaced(order.Id), ct);

    return order.Id;
}
```

Problems: any failure between steps 1, 2, 3 leaves the system inconsistent. Inventory may be deducted for an order downstream services never see.

### After — single transaction + execution strategy + outbox

```csharp
public sealed class CheckoutService
{
    private readonly OrdersDbContext _db;
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(OrdersDbContext db, ILogger<CheckoutService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<CheckoutResult> CheckoutAsync(CheckoutRequest req, CancellationToken ct)
    {
        var strategy = _db.Database.CreateExecutionStrategy();
        return await strategy.ExecuteAsync(async () =>
        {
            await using var tx = await _db.Database.BeginTransactionAsync(
                IsolationLevel.ReadCommitted, ct);
            try
            {
                var order = Order.Create(req.CustomerId, req.Lines);
                _db.Orders.Add(order);

                foreach (var line in order.Lines)
                {
                    var rows = await _db.Inventory
                        .Where(i => i.Sku == line.Sku && i.Available >= line.Quantity)
                        .ExecuteUpdateAsync(s => s
                            .SetProperty(i => i.Available, i => i.Available - line.Quantity),
                            ct);
                    if (rows == 0)
                    {
                        await tx.RollbackAsync(ct);
                        _logger.LogWarning(
                            "Checkout failed: out of stock for {Sku}", line.Sku);
                        return CheckoutResult.OutOfStock(line.Sku);
                    }
                }

                _db.OutboxMessages.Add(OutboxMessage.For(new OrderPlaced(order.Id)));
                await _db.SaveChangesAsync(ct);
                await tx.CommitAsync(ct);

                _logger.LogInformation("Checkout {OrderId} committed", order.Id);
                return CheckoutResult.Success(order.Id);
            }
            catch (DbUpdateConcurrencyException ex)
            {
                _logger.LogWarning(ex, "Concurrency conflict during checkout");
                await tx.RollbackAsync(ct);
                return CheckoutResult.Conflict;
            }
        });
    }
}
```

The after version: one transaction, atomic across order + inventory + outbox, retries on transient SQL errors automatically via `IExecutionStrategy`, and never holds a lock across a Service Bus publish.

## Interview Questions and Answers

### 1. Walk me through what happens at the database when EF Core calls `SaveChangesAsync` with three inserts.

**Why this matters:** Tests whether you understand the implicit transaction.

**Answer:** EF Core opens a transaction, runs the three `INSERT` statements (batched into a single round-trip when possible via `MERGE`/`OUTPUT` on SQL Server), and commits. If any insert fails (constraint violation, deadlock, concurrency conflict), the transaction rolls back and EF throws — none of the three rows persist. You don't write `BeginTransaction` yourself for this; it's automatic.

**Trade-off:** This convenience only covers what's in the change tracker. Mixing with raw SQL or multiple `SaveChanges` calls requires an explicit transaction.

### 2. Why is `EnableRetryOnFailure()` incompatible with `BeginTransactionAsync`, and how do you fix it?

**Why this matters:** A real production gotcha that crashes apps on first deploy to Azure SQL.

**Answer:** The retry strategy may retry mid-transaction, which would silently commit half the work twice. EF Core blocks this by throwing. The fix is to wrap your entire transactional block in `db.Database.CreateExecutionStrategy().ExecuteAsync(async () => { ... })`. The strategy retries the **whole** block as a unit, so each retry starts a fresh transaction.

**Real project:** First Azure SQL deploy of a payments service threw on every checkout. Wrapping in `IExecutionStrategy` was a 5-line fix.

### 3. Should you ever use `TransactionScope` in modern .NET?

**Answer:** Rarely. It's still useful when you must compose multiple `DbContext` instances (different schemas, different connection strings to the same server) under one local transaction — `BeginTransactionAsync` only works on one context. You must enable `TransactionScopeAsyncFlowOption.Enabled`, and you must avoid anything that promotes to MSDTC because Azure SQL doesn't support distributed transactions there. Most teams replace `TransactionScope` with either a single shared `DbContext` or the outbox pattern.

### 4. Two API requests both try to debit the same account by `$50`. The account has `$60`. Walk through what happens at Read Committed vs Snapshot vs Serializable.

**Why this matters:** Forces concrete reasoning about isolation.

**Answer:**

- **Read Committed:** Both reads see `$60`. Both writes attempt `UPDATE Accounts SET Balance = Balance - 50`. SQL Server serializes the updates with row locks. The second update succeeds with `Balance = -40`. Lost-update problem unless you re-check the balance with a `WHERE Balance >= 50` predicate (and check `@@ROWCOUNT`).
- **Snapshot:** Both transactions read `$60` from versioned storage, both compute `$10`, both try to commit. The second commit raises a write-conflict error because the row was modified after the snapshot started.
- **Serializable:** The first transaction's `SELECT` holds a range lock; the second blocks until the first commits, then reads `$10` and refuses the debit.

**Trade-off:** Snapshot gives you optimistic concurrency at the transaction level for free, but you must handle the conflict. Serializable serializes and throttles throughput. Read Committed needs explicit conditional updates.

**Real project:** On a wallet service we used a `WHERE Balance >= @amount` predicate with optimistic concurrency rather than relying on isolation level. Faster and explicit.

### 5. Your checkout publishes a `OrderPlaced` message to Service Bus after `SaveChangesAsync`. What's the failure mode and how do you fix it?

**Why this matters:** This is the most common cross-system consistency bug.

**Answer:** If the database commit succeeds but the Service Bus publish throws, downstream services never hear about the order. If you reverse the order (publish first, then commit), the publish may succeed but the commit fail, telling downstream about an order that doesn't exist. The fix is the **outbox pattern**: write the outgoing message into an `OutboxMessages` table inside the same database transaction, and have a background dispatcher publish from that table and mark rows as sent. Atomicity is now within one database.

**Real project:** A payment-confirmation flow lost ~0.1% of webhook events because the API committed the payment row, then crashed before publishing. Outbox eliminated it.

### 6. When would you use a savepoint?

**Answer:** Inside a long transaction where you want to attempt an optional step and revert just that step on failure without losing the rest. Example: bulk-importing 1000 invoices in one transaction; per-invoice validation failure should rollback that invoice's lines but keep the previous 999. `CreateSavepointAsync("BeforeInvoice")` + `RollbackToSavepointAsync("BeforeInvoice")` on validation error. Rarely needed in OLTP web APIs; mostly in batch jobs.

### 7. How do you test that a transaction actually rolls back when a step fails?

**Answer:** Integration test against a real database (Testcontainers SQL Server). Inject a failure into the second step (e.g., a mock that throws), call the service, then query the database directly to assert no rows from any step exist. The EF Core in-memory provider is useless for this — it does not implement transactional semantics.

**Real project:** We had a regression where someone moved `SaveChanges` outside the transaction; a rollback test caught it before it hit production.

### 8. A workflow needs to span an order DB, an inventory DB, and a billing DB. How do you handle consistency?

**Why this matters:** Tests cloud-native design thinking.

**Answer:** You do not use a distributed transaction. You design a **saga**: each step is a local transaction in its own database, plus a compensating action for failures. Coordinate via durable messages (Service Bus) using the outbox pattern so each local transaction atomically writes its state + the next event. If billing fails, publish `BillingFailed`, inventory listens and runs `ReleaseReservation`, order listens and marks itself `Cancelled`. Eventual consistency, observable via correlation ids.

**Trade-off:** More moving parts than a 2PC, but cloud-managed databases don't reliably support 2PC and saga is the only realistic option.

## Summary Checklist

- [ ] I know `SaveChangesAsync` is already transactional and don't open an explicit transaction when I don't need one.
- [ ] I use `IDbContextTransaction` for multi-step operations and dispose with `await using`.
- [ ] I wrap explicit transactions in `IExecutionStrategy.ExecuteAsync` when retry-on-failure is enabled.
- [ ] I can articulate the difference between Read Committed, Snapshot, and Serializable, and when to use each.
- [ ] I never make external service calls (HTTP, Service Bus) inside an open transaction.
- [ ] I use the outbox pattern for atomic "save + publish".
- [ ] I never use distributed transactions across cloud-managed databases; I use sagas.
- [ ] I always set `TransactionScopeAsyncFlowOption.Enabled` when I use `TransactionScope`.
- [ ] I write integration tests with Testcontainers that prove rollback works.
- [ ] I log transaction commit/rollback with correlation ids for incident response.
