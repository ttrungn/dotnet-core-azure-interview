# Optimistic Concurrency

## What It Is

Optimistic concurrency is a strategy where the database does **not** hold a long-running lock on a row while a user reads and edits it. Instead, every row carries a **concurrency token** (typically a `rowversion`/`timestamp` column). When you save changes, EF Core adds `WHERE Id = @id AND RowVersion = @originalRowVersion` to the `UPDATE`. If another transaction has changed the row in the meantime, the `UPDATE` affects zero rows, and EF Core throws `DbUpdateConcurrencyException`.

The mental model: *"I assume nobody else will touch this row while I edit it. If I'm wrong, the database will tell me at SaveChanges and I will decide what to do."*

Contrast with **pessimistic concurrency** — `SELECT ... FOR UPDATE` or `UPDLOCK, HOLDLOCK` hints — which physically blocks other readers/writers until your transaction commits. Pessimistic locking kills throughput on hot rows (order status, account balance, inventory quantity) and is rarely acceptable in a web/API workload where a request might hold an open page for minutes.

```csharp
// Order entity with a rowversion concurrency token
public sealed class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public byte[] RowVersion { get; private set; } = default!; // SQL Server rowversion
}

// EF Core configuration
modelBuilder.Entity<Order>()
    .Property(o => o.RowVersion)
    .IsRowVersion(); // maps to SQL Server 'rowversion' / 'timestamp' type
```

The generated `UPDATE` looks like this:

```sql
UPDATE [Orders] SET [Status] = @p0
OUTPUT INSERTED.[RowVersion]
WHERE [Id] = @p1 AND [RowVersion] = @p2;
SELECT @@ROWCOUNT;
```

If `@@ROWCOUNT = 0`, EF Core raises `DbUpdateConcurrencyException`.

## Why It Exists

In multi-user systems, two people (or two services, or the same user in two tabs) routinely load the same record and edit it. Without conflict detection you get **lost updates**: the second `SaveChanges` silently overwrites the first one's work.

Historical alternatives:

- **Pessimistic locks** — Worked in the 1990s green-screen era when one user held one row. Fails under web latency (a user reads a page, goes to lunch, comes back, saves — the lock would have to be held for an hour).
- **Last-write-wins** — Default behavior with no token. The most recent `UPDATE` always succeeds, silently destroying earlier edits. Customers see their data "disappear".
- **Application-level mutexes** — Doesn't scale across multiple API instances behind a load balancer.

Optimistic concurrency solved this by piggy-backing the check on the `UPDATE` itself: zero locks held between read and write, but a guaranteed safety net at save time.

In distributed databases (Cosmos DB, DynamoDB, Cassandra) the same idea uses an **ETag**. The principle is identical: include the version you read, and the write fails if the server has a newer one.

## When To Use It

**Use optimistic concurrency for:**

- User-editable records where two people might edit the same row (order notes, customer profile, ticket status, product catalogue).
- REST/GraphQL APIs that expose an `If-Match` / `ETag` workflow — the HTTP spec is literally designed around it.
- Long-running edit screens where users may take minutes between load and save.
- Cosmos DB / DynamoDB writes — ETag is the only safe option.
- Idempotent or compensatable workflows where a retry on conflict is cheap.

**Do not use it for:**

- Hot append-only counters (page views, totals). Use `UPDATE ... SET Counter = Counter + 1` server-side instead — there is no "read" to conflict with.
- Insert-only audit/event tables. There's nothing to overwrite.
- Cases where **every** write must succeed without retry logic (e.g., regulated trade execution requiring serializable transaction with `UPDLOCK`).
- Sub-millisecond hot paths where the cost of `DbUpdateConcurrencyException` handling dominates (use pessimistic locking or queue-based serialization).

## Why It Is Important

In production, lost updates are the kind of bug that doesn't show up in tests, doesn't crash, and is discovered weeks later when a customer escalates that "my refund note vanished". They are silent data corruption.

Concrete production properties optimistic concurrency drives:

- **Correctness under concurrency** — No silent overwrites when two writers race.
- **Throughput** — No row locks held across the HTTP roundtrip, so hot rows don't serialize the whole API.
- **Auditability** — A `409 Conflict` is observable. You can log it, alert on it, and tune the workflow. Silent overwrites are not observable.
- **Cloud-friendly** — In a horizontally scaled Azure App Service / AKS deployment, you cannot rely on in-process locks. The database is the only safe arbiter.
- **HTTP contract clarity** — `ETag` + `If-Match` gives clients a well-defined way to handle conflicts (refresh, merge, abort).

## How It's Used in C# / .NET

### 1. `IsRowVersion()` — preferred for SQL Server

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.RowVersion)
    .IsRowVersion();
```

- Maps to SQL Server `rowversion` (8-byte binary).
- **Automatically incremented by the database** on every `UPDATE`. You never set it yourself.
- Becomes the concurrency token automatically. No need to also call `IsConcurrencyToken()`.

### 2. `IsConcurrencyToken()` — for any column you choose

```csharp
modelBuilder.Entity<Customer>()
    .Property(c => c.Email)
    .IsConcurrencyToken();
```

- Marks **any** column as part of the `WHERE` clause for concurrency checks.
- The application is responsible for changing the value (or the database via triggers).
- Useful for: `LastModifiedUtc` + version number, business-meaningful columns (e.g., balance), or providers without `rowversion`.

### 3. Catching `DbUpdateConcurrencyException`

```csharp
try
{
    await _db.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var databaseValues = await entry.GetDatabaseValuesAsync(ct);
        if (databaseValues is null)
        {
            // Row was deleted by another transaction
            throw new OrderDeletedException(entry.Entity);
        }
        // Apply your resolution strategy (reject, merge, last-write-wins)
    }
}
```

`ex.Entries` gives you each conflicting tracked entry, and `GetDatabaseValuesAsync` fetches the current row from the database so you can compare/merge.

### 4. Cosmos DB ETag

```csharp
modelBuilder.Entity<Order>()
    .ToContainer("orders")
    .Property<string>("_etag")
    .IsETagConcurrency();
```

The Cosmos provider sends `If-Match: <etag>` on every `Replace`/`Upsert`. A mismatch surfaces as `DbUpdateConcurrencyException` (HTTP 412 underneath).

### 5. NuGet packages

- `Microsoft.EntityFrameworkCore` (8.0+)
- `Microsoft.EntityFrameworkCore.SqlServer`
- `Microsoft.EntityFrameworkCore.Cosmos`
- `Polly` — for retry policies around transient conflicts.

## Advantages

- **High throughput** — no row locks between read and write.
- **Safe by construction** — once tokens are configured, conflicts cannot pass silently.
- **Works across processes and machines** — the database is the single arbiter.
- **Maps cleanly to HTTP** — `ETag` / `If-Match` / `412 Precondition Failed` is a standard idiom.
- **Cheap to add** — one column + one fluent-API line.

## Disadvantages

- **Requires retry or reject logic** — the application must handle `DbUpdateConcurrencyException`. There is no automatic resolution.
- **Hostile to wide aggregates** — if any field in the aggregate changes, all editors of that aggregate conflict, even on independent fields.
- **Schema cost** — adds a column to every concurrency-protected table.
- **Misleading on bulk operations** — `ExecuteUpdate`/`ExecuteDelete` bypass the change tracker and do **not** check concurrency tokens.
- **Confusing for newcomers** — "Why did my save randomly fail?" is a real support ticket category.

## Common Mistakes

### 1. Forgetting `IsRowVersion()` and assuming EF Core "just knows"

```csharp
// BAD — no concurrency token configured
public sealed class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public byte[] RowVersion { get; set; } = default!;
}

modelBuilder.Entity<Order>(); // <-- no IsRowVersion()
```

EF Core treats `RowVersion` as a plain `varbinary(max)` column. No conflict detection happens.

```csharp
// GOOD
modelBuilder.Entity<Order>()
    .Property(o => o.RowVersion)
    .IsRowVersion();
```

### 2. Catching `DbUpdateConcurrencyException` and swallowing it

```csharp
// BAD — last-write-wins, defeats the whole point
try { await _db.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException)
{
    foreach (var entry in _db.ChangeTracker.Entries())
        entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync(ct));
    await _db.SaveChangesAsync(ct); // overwrite the other person's edits
}
```

```csharp
// GOOD — return 409 and let the client decide
catch (DbUpdateConcurrencyException)
{
    _logger.LogWarning("Concurrency conflict on Order {OrderId}", order.Id);
    return Results.Conflict(new ProblemDetails
    {
        Title = "The order was modified by another user.",
        Detail = "Reload the order and try again.",
        Status = StatusCodes.Status409Conflict
    });
}
```

### 3. Setting `RowVersion` manually

```csharp
// BAD — rowversion is database-generated, you must not assign it
order.RowVersion = Guid.NewGuid().ToByteArray();
await _db.SaveChangesAsync(ct);
```

```csharp
// GOOD — let the database set it. You only read it from the entity after SaveChanges.
order.UpdateStatus(OrderStatus.Shipped);
await _db.SaveChangesAsync(ct);
// order.RowVersion is now the new value
```

### 4. Using `ExecuteUpdate` and expecting concurrency checks

```csharp
// BAD — concurrency token ignored
await _db.Orders
    .Where(o => o.Id == orderId)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Cancelled), ct);
```

`ExecuteUpdate` runs raw SQL; it does **not** include the `RowVersion` comparison. If you need concurrency safety, either include it manually in the predicate, or load the entity and use `SaveChangesAsync`.

```csharp
// GOOD — explicit token in the predicate
var rows = await _db.Orders
    .Where(o => o.Id == orderId && o.RowVersion == expectedRowVersion)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Cancelled), ct);

if (rows == 0)
    throw new DbUpdateConcurrencyException("Order was modified.");
```

### 5. Retrying forever without a cap

```csharp
// BAD — infinite loop on a hot row
while (true)
{
    try { await _db.SaveChangesAsync(ct); break; }
    catch (DbUpdateConcurrencyException)
    {
        await entry.ReloadAsync(ct);
        order.UpdateStatus(newStatus); // reapply
    }
}
```

```csharp
// GOOD — bounded retry with Polly
var policy = Policy
    .Handle<DbUpdateConcurrencyException>()
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromMilliseconds(50 * attempt));

await policy.ExecuteAsync(async () =>
{
    await entry.ReloadAsync(ct);
    order.UpdateStatus(newStatus);
    await _db.SaveChangesAsync(ct);
});
```

### 6. Not exposing the version to the API client

```csharp
// BAD — client has no way to do If-Match
public record OrderDto(Guid Id, OrderStatus Status);
```

```csharp
// GOOD — return the version as an ETag header or a field
public record OrderDto(Guid Id, OrderStatus Status, string Version);
// Version = Convert.ToBase64String(order.RowVersion)
// Return as ETag: "abc123=="
```

## Best Practices

- **Use `IsRowVersion()` on SQL Server.** It's automatic, atomic, and 8 bytes.
- **Surface conflicts as HTTP 409 with `ProblemDetails`.** Never swallow.
- **Expose the row version as an `ETag`.** Clients send it back in `If-Match`.
- **Retry only when the operation is safe to replay** — status transitions yes, payment captures no.
- **Cap retries at 3 attempts with jitter.** Beyond that, return 409 and let the user decide.
- **Log every conflict with the entity id and user id.** Spikes indicate workflow design problems, not bugs.
- **Use `ReloadAsync` after a conflict** — the in-memory entity has stale `OriginalValues`.
- **Document which fields cause conflicts** — wide aggregates create false conflicts.
- **For Cosmos, always configure `_etag`.** The default is no concurrency control.

## Related Concepts

- [data-access/transactions.md](data-access/transactions.md) — what to do when conflicts span multiple rows
- [data-access/dbcontext-lifetime.md](data-access/dbcontext-lifetime.md) — long-lived DbContexts amplify stale-data risk
- [data-access/repository-pattern-with-ef-core.md](data-access/repository-pattern-with-ef-core.md) — where to centralize conflict handling
- [aspnet-core/error-handling-and-problem-details.md](aspnet-core/error-handling-and-problem-details.md) — returning 409 properly
- [aspnet-core/http-methods-and-status-codes.md](aspnet-core/http-methods-and-status-codes.md) — ETag/If-Match semantics
- [architecture/saga-pattern.md](architecture/saga-pattern.md) — compensating actions when retries are unsafe
- [architecture/outbox-pattern.md](architecture/outbox-pattern.md) — avoiding distributed transactions

## Real-World Usage

### ASP.NET Core APIs

A typical order-status `PATCH` endpoint:

```csharp
app.MapPatch("/orders/{id:guid}/status", async (
    Guid id, UpdateOrderStatusRequest req, OrdersDbContext db,
    HttpContext http, ILogger<Program> logger, CancellationToken ct) =>
{
    var order = await db.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
    if (order is null) return Results.NotFound();

    // Honour If-Match header (ETag-based optimistic concurrency)
    if (http.Request.Headers.IfMatch.Count > 0)
    {
        var clientVersion = http.Request.Headers.IfMatch.ToString().Trim('"');
        var serverVersion = Convert.ToBase64String(order.RowVersion);
        if (clientVersion != serverVersion)
            return Results.StatusCode(StatusCodes.Status412PreconditionFailed);
    }

    order.UpdateStatus(req.NewStatus);

    try
    {
        await db.SaveChangesAsync(ct);
    }
    catch (DbUpdateConcurrencyException)
    {
        logger.LogWarning("Concurrency conflict on Order {OrderId}", id);
        return Results.Conflict();
    }

    http.Response.Headers.ETag = $"\"{Convert.ToBase64String(order.RowVersion)}\"";
    return Results.NoContent();
});
```

### Azure SQL Database

Azure SQL fully supports `rowversion`. No extra configuration is needed beyond `IsRowVersion()`. Watch for:

- **Read-replicas** — secondary replicas can lag, so a client may read an old `RowVersion` then `If-Match` will fail on the primary. Pin writes to the primary.
- **Always Encrypted columns** cannot be used as concurrency tokens.

### Azure Cosmos DB

Every Cosmos item has a server-managed `_etag`. The EF Core Cosmos provider hooks this automatically when you configure `IsETagConcurrency()`. In the SDK, you pass `ItemRequestOptions { IfMatchEtag = etag }`.

### Multi-tenant SaaS

Each tenant's edits should conflict only within that tenant. Make sure the `WHERE` clause on updates also includes `TenantId` — concurrency tokens alone do not enforce tenant isolation.

### CI/CD

Add a test that simulates two concurrent `SaveChangesAsync` calls on the same row and asserts that the second one throws `DbUpdateConcurrencyException`. Without this test, a future change to the entity (e.g., dropping `IsRowVersion()` during a refactor) silently breaks safety.

## Code Example — Before and After

### Before — silent lost update

```csharp
public sealed class OrderService
{
    private readonly OrdersDbContext _db;
    public OrderService(OrdersDbContext db) => _db = db;

    public async Task UpdateStatusAsync(Guid orderId, OrderStatus newStatus, CancellationToken ct)
    {
        var order = await _db.Orders.FirstAsync(o => o.Id == orderId, ct);
        order.Status = newStatus;
        await _db.SaveChangesAsync(ct);
        // If two agents call this simultaneously, the second overwrites the first.
        // No exception. No log. The customer's earlier status change vanishes.
    }
}
```

### After — optimistic concurrency with bounded retry and proper conflict response

```csharp
public sealed class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public byte[] RowVersion { get; private set; } = default!;

    public void UpdateStatus(OrderStatus newStatus)
    {
        if (!IsValidTransition(Status, newStatus))
            throw new InvalidOperationException(
                $"Cannot transition from {Status} to {newStatus}.");
        Status = newStatus;
    }

    private static bool IsValidTransition(OrderStatus from, OrderStatus to) =>
        (from, to) switch
        {
            (OrderStatus.Pending, OrderStatus.Paid) => true,
            (OrderStatus.Paid, OrderStatus.Shipped) => true,
            (OrderStatus.Shipped, OrderStatus.Delivered) => true,
            _ => false
        };
}

public sealed class OrdersDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Order>().Property(o => o.RowVersion).IsRowVersion();
    }
}

public sealed class OrderStatusService
{
    private readonly OrdersDbContext _db;
    private readonly ILogger<OrderStatusService> _logger;
    private static readonly AsyncRetryPolicy RetryPolicy = Policy
        .Handle<DbUpdateConcurrencyException>()
        .WaitAndRetryAsync(3, attempt => TimeSpan.FromMilliseconds(50 * attempt));

    public OrderStatusService(OrdersDbContext db, ILogger<OrderStatusService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<UpdateResult> UpdateStatusAsync(
        Guid orderId, OrderStatus newStatus, CancellationToken ct)
    {
        try
        {
            await RetryPolicy.ExecuteAsync(async () =>
            {
                var order = await _db.Orders.FirstOrDefaultAsync(o => o.Id == orderId, ct)
                    ?? throw new OrderNotFoundException(orderId);

                order.UpdateStatus(newStatus); // throws if transition invalid
                await _db.SaveChangesAsync(ct);
            });
            return UpdateResult.Success;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            _logger.LogWarning(ex,
                "Concurrency conflict on Order {OrderId} after 3 retries", orderId);
            return UpdateResult.Conflict;
        }
        catch (InvalidOperationException ex)
        {
            _logger.LogInformation(ex,
                "Invalid status transition on Order {OrderId}", orderId);
            return UpdateResult.InvalidTransition;
        }
    }
}
```

The before version loses data silently. The after version detects the conflict, retries transient races, and returns a clear `Conflict` outcome that the API layer turns into HTTP 409.

## Interview Questions and Answers

### 1. Two support agents click "Mark as Shipped" on the same order at the same time. Walk me through what happens with and without optimistic concurrency.

**Why this matters:** Tests whether you understand the *silent corruption* failure mode, not just the API.

**Answer:** Without concurrency tokens, both `UPDATE` statements succeed; the second simply overwrites the first. Both agents see "success". If they had set different statuses, one is silently lost. With `IsRowVersion()`, the first `UPDATE` succeeds and increments `RowVersion`. The second sends the old `RowVersion` in its `WHERE` clause, matches zero rows, and EF Core throws `DbUpdateConcurrencyException`. We return HTTP 409 and the second agent reloads.

**Trade-off:** Optimistic costs one extra column and forces clients to handle 409. Pessimistic would block the second agent's request entirely.

**Real project:** On an order-management system, we discovered via audit logs that ~0.3% of status changes were being lost. Adding `IsRowVersion()` and a 409 handler eliminated the support tickets within a release.

### 2. What's the difference between `IsRowVersion()` and `IsConcurrencyToken()`?

**Why this matters:** Reveals depth on EF Core configuration.

**Answer:** `IsRowVersion()` is SQL Server-specific. It maps to the `rowversion`/`timestamp` type, the database auto-increments it on every `UPDATE`, and it becomes the concurrency token automatically. `IsConcurrencyToken()` is generic: it tells EF Core to include the column in the `WHERE` clause of `UPDATE`/`DELETE`, but **you** are responsible for changing the value (or a trigger is). Use `IsRowVersion()` whenever you can; fall back to `IsConcurrencyToken()` for non-SQL-Server providers or when the token is a meaningful business field like `LastModifiedUtc`.

**Real project:** On a PostgreSQL system we used `xmin` as a system column with `IsConcurrencyToken()` configured via `HasColumnName("xmin").HasColumnType("xid")` — same idea, different mechanism.

### 3. A junior developer wrapped `SaveChangesAsync` in `try/catch (DbUpdateConcurrencyException)` and reloaded + re-saved automatically. What's wrong with this?

**Why this matters:** Distinguishes "I caught the exception" from "I resolved it correctly".

**Answer:** It's silent last-write-wins. The point of the concurrency token was to detect that someone else changed the row — re-saving without re-evaluating the business decision throws that information away. The correct pattern is either: (a) return 409 to the caller and let them re-fetch and decide, or (b) re-load, re-apply the *business operation* (not the original field values), and retry only if the operation is idempotent and the conflict was likely a transient race.

**Trade-off:** Auto-retry feels user-friendly but can corrupt domain state. Returning 409 is honest.

### 4. How would you handle optimistic concurrency on an Azure Cosmos DB container?

**Why this matters:** Tests knowledge beyond SQL Server.

**Answer:** Cosmos items have a built-in server-managed `_etag` property. In EF Core Cosmos provider: `modelBuilder.Entity<Order>().Property<string>("_etag").IsETagConcurrency()`. The provider sends `If-Match: <etag>` on every replace/upsert; a mismatch surfaces as `DbUpdateConcurrencyException` (HTTP 412 Precondition Failed underneath). In the raw SDK, pass `new ItemRequestOptions { IfMatchEtag = item.ETag }`.

**Real project:** A multi-region Cosmos deployment with session consistency — every write went through the same ETag pattern. We surfaced the ETag in API responses as an `ETag` header so mobile clients could replay safely.

### 5. You're using `ExecuteUpdate` to bulk-cancel pending orders older than 30 days. Does optimistic concurrency apply?

**Why this matters:** A common production trap.

**Answer:** No. `ExecuteUpdate` and `ExecuteDelete` translate directly to SQL and bypass the EF change tracker entirely — the `RowVersion` comparison is not included. If concurrency safety matters for the bulk path, either include the version explicitly in the `WHERE` clause, or accept that bulk operations are "fire-and-forget" and use them only where last-writer-wins is acceptable (admin scripts, retention jobs).

**Trade-off:** Per-entity loads with concurrency checks are slow at scale; raw bulk update is fast but unchecked. Choose deliberately.

### 6. When would you *not* use optimistic concurrency?

**Why this matters:** Senior engineers know when a tool is wrong.

**Answer:** (1) Append-only / event-store tables — nothing to conflict with. (2) Pure counters — use `UPDATE ... SET x = x + 1` server-side; there is no read to race. (3) Regulated financial operations where every write must commit deterministically — pessimistic locks (`UPDLOCK, HOLDLOCK`) or queue-based serialization may be required to meet auditing requirements. (4) Very hot rows (single inventory counter under thousands of QPS) — retry storms dominate; switch to a queue, a sharded counter, or a different data structure.

### 7. How do you test optimistic concurrency code?

**Answer:** Spin up two `DbContext` instances, load the same entity in both, modify and save the first, then save the second and assert `DbUpdateConcurrencyException`. Use Testcontainers with a real SQL Server image — the in-memory provider doesn't simulate `rowversion`. Add a load test that hammers the same row from multiple threads and asserts the 409 rate matches expectations.

**Real project:** We had a regression where someone removed `IsRowVersion()` during a model cleanup; the unit test that did two concurrent saves caught it before it merged.

### 8. A customer reports that an edit they made "disappeared" yesterday. Walk me through diagnosis.

**Why this matters:** Tests observability thinking.

**Answer:** First, check whether concurrency tokens are configured on that entity — if not, lost updates are possible. Then check audit logs / change feed for two updates to the same row within seconds of each other. Look for `DbUpdateConcurrencyException` in logs at that timestamp; if none, the system was running without protection. If exceptions exist but were swallowed, look in the catch blocks. Add `ILogger` warnings on every conflict with `OrderId` and `UserId` so the next incident is one query away.

## Summary Checklist

- [ ] I can explain optimistic vs pessimistic concurrency with a concrete order-edit scenario.
- [ ] I know the difference between `IsRowVersion()` and `IsConcurrencyToken()` and when to use each.
- [ ] I handle `DbUpdateConcurrencyException` by returning 409, not by silently retrying.
- [ ] I cap retries with Polly and use jitter.
- [ ] I expose row versions as ETags and honour `If-Match` in PATCH/PUT endpoints.
- [ ] I know `ExecuteUpdate`/`ExecuteDelete` bypass concurrency tokens.
- [ ] I can configure Cosmos DB ETag concurrency with `IsETagConcurrency()`.
- [ ] I write a concurrent-save integration test for every concurrency-protected entity.
- [ ] I log every conflict with entity and user id for observability.
- [ ] I can defend choosing optimistic over pessimistic in a web/API workload.
