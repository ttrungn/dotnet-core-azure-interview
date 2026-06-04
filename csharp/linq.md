# LINQ

## What It Is

**LINQ (Language-Integrated Query)** is C#'s built-in query syntax for collections, databases, XML, JSON, and any source that implements `IEnumerable<T>` or `IQueryable<T>`. It gives you SQL-like operations — `Where`, `Select`, `GroupBy`, `Join`, `Aggregate` — that compose into pipelines and run either in memory (LINQ to Objects) or get translated to another language such as SQL (LINQ to Entities via EF Core).

```csharp
var lateInvoices = invoices
    .Where(i => i.Status == InvoiceStatus.Pending && i.DueDate < today)
    .OrderBy(i => i.DueDate)
    .Select(i => new InvoiceSummary(i.Id, i.CustomerName, i.Amount))
    .ToList();
```

That snippet runs against an `IEnumerable<Invoice>` in memory. Change `invoices` to a `DbSet<Invoice>` and EF Core translates the same code to parameterised T-SQL. **Same syntax, completely different execution model** — and that distinction is the source of every interesting LINQ pitfall in production.

## Why It Exists

Before LINQ (.NET 3.5, 2007) developers wrote either:

- Tight, type-safe but verbose `foreach` loops with `if/else` and `Dictionary` accumulators.
- Raw ADO.NET / `SqlCommand` strings, with no compile-time type checking, no IntelliSense, and SQL-injection risk.

LINQ unified those two worlds: one strongly-typed query language that works against in-memory collections, EF Core, MongoDB drivers, Cosmos DB, and many other providers. The compiler validates queries against the source's element type, refactoring tools rename across them, and providers translate the same expression tree to the target backend.

## When To Use It

**Use LINQ for:**

- Filtering, projecting, ordering, grouping, and aggregating collections.
- Composing query logic from reusable, type-safe building blocks.
- Database access through EF Core or other LINQ providers.
- Streaming transformations with `IAsyncEnumerable<T>` from Cosmos DB or Service Bus.
- Set operations (`Union`, `Except`, `Intersect`) on sets of identifiers.

**Avoid LINQ when:**

- The transformation needs side effects (logging per row, sending events). Use a `foreach` — `ToList().ForEach` exists, but it hides intent.
- The hot path runs millions of iterations and you've measured allocation cost (the LINQ enumerators allocate). For those, a `for` over an array or `Span<T>` can be measurably faster.
- The query mixes provider-translated and in-memory logic — pull the translated part with `.ToListAsync()`, then continue with LINQ to Objects.

## Why It Is Important

LINQ is one of the most-used language features in modern .NET backends. Almost every controller action, query handler, and reporting job is built from LINQ pipelines. Two senior-level concerns matter most:

1. **Deferred vs immediate execution** — a `Where(...).Select(...)` expression does nothing until you enumerate it. Forgetting this causes "the query runs five times" bugs.
2. **`IQueryable` vs `IEnumerable`** — the boundary where the database stops and your process starts. Cross it accidentally and your query loads a million rows into memory.

Mastering these two distinctions is the difference between a 50 ms query and a 5 s timeout in a real system.

## How It's Used in C# / .NET

### 1. Method syntax vs query syntax

```csharp
// Method (preferred — composes better)
var ids = orders.Where(o => o.Total > 100).Select(o => o.Id).ToList();

// Query syntax
var ids2 = (from o in orders where o.Total > 100 select o.Id).ToList();
```

Query syntax can be clearer for joins and grouping; method syntax wins for short pipelines. Mix freely.

### 2. Deferred execution

```csharp
IEnumerable<Order> query = orders.Where(o => o.Status == OrderStatus.Pending);

// Nothing has run yet. The predicate is captured in an iterator.
foreach (var o in query) { /* runs the Where on each enumeration */ }
foreach (var o in query) { /* runs it AGAIN */ }
var count = query.Count();   // and AGAIN
```

Materialize once with `.ToList()`, `.ToArray()`, `.ToHashSet()`, `.ToDictionary(...)`, or async equivalents.

### 3. `IEnumerable<T>` vs `IQueryable<T>`

```csharp
// IQueryable — predicate translated to SQL, executed on the database
var q1 = dbContext.Orders.Where(o => o.Total > 100).ToListAsync(ct);

// IEnumerable — predicate runs in process, after loading the full table
var q2 = dbContext.Orders.AsEnumerable().Where(o => o.Total > 100).ToList();
```

The second form loads every row from the `Orders` table before filtering. On a million-row table that's a 30 s query that becomes a 30 ms one when you remove `.AsEnumerable()`.

Always know whether you're holding `IQueryable<T>` or `IEnumerable<T>` — the static type tells you.

### 4. Projection to DTOs (`.Select`)

```csharp
public async Task<List<InvoiceSummary>> GetSummariesAsync(Guid customerId, CancellationToken ct)
    => await dbContext.Invoices
        .Where(i => i.CustomerId == customerId)
        .OrderByDescending(i => i.IssuedAt)
        .Select(i => new InvoiceSummary(i.Id, i.Amount, i.IssuedAt, i.Status))
        .ToListAsync(ct);
```

Projecting straight to the DTO means EF Core selects only the four columns it needs — no entity tracking, no unused columns, no navigation property eager-loading.

### 5. `AsNoTracking()` for read paths

```csharp
var invoices = await dbContext.Invoices
    .AsNoTracking()
    .Where(i => i.Status == InvoiceStatus.Paid)
    .ToListAsync(ct);
```

When you're not going to mutate the entities, `AsNoTracking()` skips the change tracker — faster, less memory, and avoids stale-state confusion.

### 6. Aggregations

```csharp
public sealed record RevenueRow(string Currency, decimal Total, int Count);

var revenue = await dbContext.Payments
    .Where(p => p.PaidAt >= since)
    .GroupBy(p => p.Currency)
    .Select(g => new RevenueRow(g.Key, g.Sum(p => p.Amount), g.Count()))
    .ToListAsync(ct);
```

This translates to one SQL statement with `GROUP BY` and `SUM`. Do it in memory by loading all payments first and you've created a heap problem.

### 7. Pagination — server side

```csharp
public async Task<PagedResult<Invoice>> PageAsync(int page, int size, CancellationToken ct)
{
    var query = dbContext.Invoices.AsNoTracking().OrderByDescending(i => i.IssuedAt);
    var total = await query.CountAsync(ct);
    var items = await query.Skip((page - 1) * size).Take(size).ToListAsync(ct);
    return new(items, total);
}
```

`Skip`/`Take` on an `IQueryable` becomes `OFFSET`/`FETCH NEXT` — paging happens in the database.

### 8. Joining

```csharp
var rows = await (from inv in dbContext.Invoices
                  join cust in dbContext.Customers on inv.CustomerId equals cust.Id
                  where inv.Status == InvoiceStatus.Pending
                  select new { inv.Id, inv.Amount, cust.Email })
                 .ToListAsync(ct);
```

For navigation properties prefer `Include` / `Select` projections — joins are most useful when there's no navigation.

### 9. `Any` vs `Count() > 0`

```csharp
// Translates to EXISTS (SELECT 1 ... ) — stops at the first row
if (await dbContext.Orders.AnyAsync(o => o.CustomerId == id, ct)) { … }

// Translates to SELECT COUNT(*) FROM Orders WHERE ... — scans every matching row
if (await dbContext.Orders.CountAsync(o => o.CustomerId == id, ct) > 0) { … }
```

`Any` is always cheaper when you only care about existence.

### 10. `IAsyncEnumerable<T>` for streaming

```csharp
public async IAsyncEnumerable<OrderEvent> StreamAsync(
    DateTimeOffset since,
    [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var e in cosmos.GetItemQueryIterator<OrderEvent>(
            $"SELECT * FROM c WHERE c.occurredAt > '{since:O}'").AsAsyncEnumerable(ct))
    {
        yield return e;
    }
}
```

EF Core 8+ supports `AsAsyncEnumerable()` on queries — pages the result without buffering everything in memory.

## Advantages

- **Type-safe, compile-checked queries** with refactoring support.
- **Composable** — query fragments combine without string concatenation.
- **Provider-agnostic** — the same operators work over in-memory, EF Core, Cosmos, MongoDB.
- **Readable** — `Where`/`Select` reads like the intent; the alternative `foreach`/`if`/`accumulator` is mechanical.
- **Lazy evaluation** lets you build queries up step by step without paying for intermediate materialization.
- **Async-aware variants** (`ToListAsync`, `FirstOrDefaultAsync`, `AnyAsync`) integrate cleanly with `CancellationToken`.

## Disadvantages

- **Deferred execution surprises** — enumerating twice runs the query twice; closures capture variables.
- **`IEnumerable` ↔ `IQueryable` traps** — accidental client-side evaluation kills performance.
- **Allocation overhead** — each `Select`/`Where` allocates an iterator; hot paths sometimes want a hand-rolled loop.
- **Provider limitations** — some methods (custom functions, `string.Format` overloads) cannot be translated to SQL and throw at runtime.
- **Debugging** — stepping through a chained pipeline is harder than through a `foreach`.
- **Implicit `null` behaviour** — `FirstOrDefault` returns `default(T)`; for reference types in non-nullable contexts that's `null`, which can surprise.

## Common Mistakes

### 1. Counting when existence is enough

```csharp
// BUG — full COUNT(*) scan over the table
if (dbContext.Orders.Count(o => o.CustomerId == id) > 0) { … }
```

**Fix**:

```csharp
if (await dbContext.Orders.AnyAsync(o => o.CustomerId == id, ct)) { … }
```

EXISTS short-circuits on the first row.

### 2. Forcing client-side evaluation with `.ToList()` early

```csharp
// BUG — pulls every order into memory, then filters in process
var orders = dbContext.Orders.ToList()
    .Where(o => o.Status == OrderStatus.Pending)
    .ToList();
```

**Fix**:

```csharp
var orders = await dbContext.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync(ct);
```

Filter on the server, materialize once.

### 3. Multiple enumeration

```csharp
IEnumerable<Order> pending = orders.Where(o => o.Status == OrderStatus.Pending);
var count = pending.Count();                    // first enumeration
foreach (var o in pending) { /* second */ }      // second enumeration
var totals = pending.Sum(o => o.Total);          // third
```

**Fix**: materialize once.

```csharp
var pending = orders.Where(o => o.Status == OrderStatus.Pending).ToList();
```

In EF Core the consequences are worse — three round-trips to the database.

### 4. N+1 from lazy loading or per-row queries

```csharp
// BUG — issues one query per order
foreach (var order in dbContext.Orders.AsNoTracking().ToList())
{
    var customer = dbContext.Customers.Single(c => c.Id == order.CustomerId);
    Console.WriteLine($"{customer.Email}: {order.Total}");
}
```

**Fix**: project or `Include`.

```csharp
var rows = await dbContext.Orders.AsNoTracking()
    .Select(o => new { o.Total, Email = o.Customer.Email })
    .ToListAsync(ct);
```

### 5. Selecting whole entities for read-only display

```csharp
// BUG — loads every column, tracks every entity
var orders = await dbContext.Orders.ToListAsync(ct);
return orders.Select(o => new OrderRow(o.Id, o.Total)).ToList();
```

**Fix**: project to the DTO in the query and add `AsNoTracking()`.

```csharp
return await dbContext.Orders.AsNoTracking()
    .Select(o => new OrderRow(o.Id, o.Total))
    .ToListAsync(ct);
```

### 6. Unbounded queries

```csharp
// BUG — returns everything in the table
return await dbContext.Invoices.ToListAsync(ct);
```

**Fix**: always paginate, filter, or stream.

```csharp
return await dbContext.Invoices
    .OrderByDescending(i => i.IssuedAt)
    .Take(100)
    .ToListAsync(ct);
```

### 7. Calling C# methods inside `IQueryable` predicates

```csharp
// BUG — EF Core throws at runtime: cannot translate `IsValidEmail`
var bad = await dbContext.Customers
    .Where(c => EmailHelpers.IsValidEmail(c.Email))
    .ToListAsync(ct);
```

**Fix**: either inline the logic in terms of SQL-translatable predicates, or materialize first and filter in memory (paying the cost knowingly).

### 8. Variable capture in deferred predicates

```csharp
var queries = new List<IEnumerable<Order>>();
for (var status = 0; status < 3; status++)
    queries.Add(orders.Where(o => (int)o.Status == status));   // BUG — captures `status` by reference

foreach (var q in queries) Console.WriteLine(q.Count());        // all three use status == 3
```

**Fix**: capture a local copy.

```csharp
for (var status = 0; status < 3; status++)
{
    var captured = status;
    queries.Add(orders.Where(o => (int)o.Status == captured));
}
```

C# 5+ fixed this for `foreach`, but `for` loops still bite.

## Best Practices

- **Project to DTOs** for read paths — never return tracked entities from query endpoints.
- **Always `AsNoTracking()`** for read-only queries.
- **Prefer `Any`** over `Count() > 0`.
- **Materialize once** — `.ToList()` / `.ToListAsync()` at the boundary, then operate on the list.
- **Push filtering to the database** — `Where` before `AsEnumerable` / `ToList`.
- **Use async variants** (`ToListAsync`, `FirstOrDefaultAsync`) and pass `CancellationToken`.
- **Paginate every list endpoint** — `Skip`/`Take` with a sensible upper bound.
- **Profile with `ToQueryString()` or EF Core logging** to confirm what SQL is generated.
- **Use `IAsyncEnumerable<T>`** for streaming large results.
- **Test queries against the real provider** (SQLite or Testcontainers) — LINQ to Objects in unit tests can pass code that EF Core rejects.

## Related Concepts

- **[data-access/entity-framework-core.md](../data-access/entity-framework-core.md)** — LINQ-to-SQL translation rules.
- **[data-access/query-performance.md](../data-access/query-performance.md)** — N+1 detection, execution plans.
- **[data-access/repository-pattern-with-ef-core.md](../data-access/repository-pattern-with-ef-core.md)** — where queries live.
- **[csharp/async-await.md](async-await.md)** — async LINQ operators.
- **[csharp/generics.md](generics.md)** — `IEnumerable<T>` / `IQueryable<T>` variance.
- **[csharp/fundamentals.md](fundamentals.md)** — collection types behind LINQ.
- **[architecture/cqrs.md](../architecture/cqrs.md)** — projection-heavy read models.

## Real-World Usage

### Reporting endpoint in ASP.NET Core

```csharp
[ApiController, Route("api/reports")]
public sealed class RevenueController(AppDbContext db) : ControllerBase
{
    public sealed record RevenueRow(string Currency, decimal Total, int Count);

    [HttpGet("revenue")]
    public async Task<ActionResult<IReadOnlyList<RevenueRow>>> GetRevenue(
        [FromQuery] DateTimeOffset since,
        CancellationToken ct)
    {
        var rows = await db.Payments
            .AsNoTracking()
            .Where(p => p.PaidAt >= since && p.Status == PaymentStatus.Captured)
            .GroupBy(p => p.Currency)
            .Select(g => new RevenueRow(g.Key, g.Sum(p => p.Amount), g.Count()))
            .ToListAsync(ct);

        return Ok(rows);
    }
}
```

The whole report is one SQL statement, no entities tracked, paged by currency.

### Streaming export from an Azure Function

```csharp
public sealed class InvoiceExportFunction(AppDbContext db, ILogger<InvoiceExportFunction> log)
{
    [Function("ExportInvoices")]
    public async Task RunAsync([TimerTrigger("0 0 * * * *")] TimerInfo timer, CancellationToken ct)
    {
        await using var stream = File.Create($"/tmp/invoices-{DateTime.UtcNow:yyyyMMddHH}.ndjson");
        await using var writer = new StreamWriter(stream);

        await foreach (var inv in db.Invoices
            .AsNoTracking()
            .Where(i => i.Status == InvoiceStatus.Paid)
            .OrderBy(i => i.Id)
            .Select(i => new { i.Id, i.Amount, i.Currency, i.IssuedAt })
            .AsAsyncEnumerable()
            .WithCancellation(ct))
        {
            await writer.WriteLineAsync(JsonSerializer.Serialize(inv));
        }

        log.LogInformation("Export complete");
    }
}
```

`AsAsyncEnumerable()` streams results page by page — memory footprint stays flat regardless of table size.

### Unit testing LINQ logic with EF Core in-memory or SQLite

```csharp
[Fact]
public async Task GetSummaries_ReturnsOrderedByIssuedAtDescending()
{
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseSqlite("DataSource=:memory:")
        .Options;
    await using var db = new AppDbContext(options);
    await db.Database.OpenConnectionAsync();
    await db.Database.EnsureCreatedAsync();

    db.Invoices.AddRange(
        new Invoice { Id = Guid.NewGuid(), IssuedAt = new DateTimeOffset(2025, 01, 01, 0, 0, 0, TimeSpan.Zero) },
        new Invoice { Id = Guid.NewGuid(), IssuedAt = new DateTimeOffset(2025, 02, 01, 0, 0, 0, TimeSpan.Zero) });
    await db.SaveChangesAsync();

    var sut = new InvoiceService(db);
    var summaries = await sut.GetSummariesAsync(default, CancellationToken.None);

    summaries.Select(s => s.IssuedAt).Should().BeInDescendingOrder();
}
```

Prefer SQLite or Testcontainers over EF Core's in-memory provider — the in-memory provider does not enforce SQL semantics and lets buggy queries pass.

## Code Example — Before and After

### Before — accidental client-side everything

```csharp
public List<OrderRow> GetRecentPaidOrders(Guid customerId)
{
    var all = dbContext.Orders.ToList();                                 // loads every order in the DB
    return all.Where(o => o.CustomerId == customerId
                       && o.Status == OrderStatus.Paid)                  // filtered in process
              .OrderByDescending(o => o.PaidAt)
              .Take(20)
              .Select(o => new OrderRow(o.Id, o.Total, o.PaidAt!.Value)) // tracked entities, all columns
              .ToList();
}
```

Problems:

- Loads the entire `Orders` table into memory.
- Tracks every entity (memory + CPU).
- Sync API in an async pipeline — blocks a thread.
- No `CancellationToken`.

### After — server-side, projected, async, cancellable

```csharp
public async Task<IReadOnlyList<OrderRow>> GetRecentPaidOrdersAsync(
    Guid customerId,
    CancellationToken ct)
    => await dbContext.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Paid)
        .OrderByDescending(o => o.PaidAt)
        .Take(20)
        .Select(o => new OrderRow(o.Id, o.Total, o.PaidAt!.Value))
        .ToListAsync(ct);
```

What improved:

- One SQL statement, indexed by `CustomerId`.
- Only three columns selected.
- No tracking; no allocation pressure on the change tracker.
- Async, cancellable, honors the request lifetime.

## Interview Questions and Answers

### 1. What's the difference between `IEnumerable<T>` and `IQueryable<T>`?

**Why this matters**: This is the single most important LINQ distinction in real applications.

**Answer**: `IEnumerable<T>` represents an in-memory sequence; operators run in the .NET process. `IQueryable<T>` represents a query against a provider (EF Core, Cosmos, etc.); operators build an expression tree that the provider translates into the target language (SQL, NoSQL, OData). The moment you call `AsEnumerable()` or `ToList()` on an `IQueryable`, you switch into in-memory mode and the database stops helping.

**Trade-off**: Sometimes you genuinely need to do work the provider cannot translate — then materialize the minimum subset first.

**Real project**: A junior wrote `.ToList().Where(...)` and turned a 30 ms query into a 30 s one on a 4 M-row table. Fix: move `Where` before `ToList`.

### 2. What is deferred execution and what bugs does it cause?

**Answer**: Most LINQ operators don't execute until you enumerate (`foreach`, `ToList`, `Count`, etc.). Bugs:

- Enumerating multiple times re-runs the whole pipeline — multiple DB round-trips.
- Closures capture variables by reference — `for` loops with deferred predicates use the final value.
- Disposed `DbContext` — the query runs after the `using` ends and throws.

**Fix**: materialize at the boundary, capture locals inside loops, ensure the `DbContext` lives until enumeration.

### 3. Why is `Any` faster than `Count() > 0`?

**Answer**: `Any` translates to `EXISTS (SELECT 1 ...)` which stops at the first match. `Count() > 0` translates to `SELECT COUNT(*) ...` which scans every matching row. Same semantic answer, very different cost — especially when there's an index that lets EXISTS short-circuit immediately.

### 4. How do you avoid N+1 queries with LINQ and EF Core?

**Answer**: Three tools:

1. **Projection** — `Select` a flat shape that pulls related data in the same SQL statement.
2. **Eager loading** — `Include` / `ThenInclude` to load navigations explicitly.
3. **Split queries** — `AsSplitQuery()` for large `Include` trees, which splits into multiple round-trips but avoids cartesian explosion.

Detect N+1 by enabling EF Core logging or running `MiniProfiler` in a dev environment; you'll see one query repeating per row.

**Real project**: A dashboard was firing 1 + 250 queries per request. Switching the outer query to a single `Select` with a sub-projection brought it to 1.

### 5. When would you choose `IAsyncEnumerable<T>` over `Task<List<T>>`?

**Answer**: When the result set is large enough that buffering it would hurt — exports, paged reports, long-running streams from Cosmos or Service Bus. `IAsyncEnumerable<T>` lets the producer page results and the consumer process one at a time with `await foreach`. Combined with `WithCancellation(ct)` it honors backpressure end to end.

**Trade-off**: Harder to test, no `Count` until enumerated, can't `.Where()` after collection without explicit `.ToListAsync()`.

### 6. What is the cost of `AsNoTracking()` and when do you skip it?

**Answer**: `AsNoTracking()` makes EF Core skip adding entities to the change tracker. For read endpoints that's pure win — less memory, faster materialization, no risk of accidental updates. Skip it when you actually intend to mutate the entities and call `SaveChangesAsync` — then you need tracking.

**Real project**: Adding `AsNoTracking()` to all read-only repositories cut a hot path's allocation rate by 35% in a load test.

### 7. How do you debug "this LINQ query throws at runtime but compiles fine"?

**Answer**: Almost always a provider translation failure. Steps:

- Inspect the generated SQL with `query.ToQueryString()` (EF Core 5+).
- Move the non-translatable part out of the predicate (e.g., compute a `DateTime` in C#, pass it as a variable).
- If still impossible, materialize early — `ToListAsync` first, then continue with LINQ to Objects.

### 8. How would you implement keyset (cursor) pagination with LINQ?

**Answer**: Pass the last seen `IssuedAt` and `Id` as the cursor; query with `WHERE (IssuedAt, Id) < (@lastDate, @lastId) ORDER BY IssuedAt DESC, Id DESC`. In LINQ:

```csharp
var page = await db.Invoices
    .AsNoTracking()
    .Where(i => i.IssuedAt < cursor.LastDate
             || (i.IssuedAt == cursor.LastDate && i.Id.CompareTo(cursor.LastId) < 0))
    .OrderByDescending(i => i.IssuedAt).ThenByDescending(i => i.Id)
    .Take(size)
    .ToListAsync(ct);
```

**Trade-off**: harder to compute total pages, but constant-time per page regardless of offset — `OFFSET 100000` is linear; cursors are constant.

**Real project**: switched an admin search page from `Skip(page * size)` to keyset pagination when the table grew past 5 M rows; latency dropped from 4 s to 80 ms on the deepest pages.

## Summary Checklist

- [ ] I know whether each query is `IEnumerable<T>` or `IQueryable<T>` at every line.
- [ ] Read endpoints use `AsNoTracking()` and project to DTOs.
- [ ] I use `AnyAsync` for existence checks, never `CountAsync(...) > 0`.
- [ ] Every async LINQ call carries a `CancellationToken`.
- [ ] List endpoints are paginated with sensible upper bounds.
- [ ] No `ToList()` between `Where` and the database call.
- [ ] I avoid multiple enumeration — materialize once, then iterate.
- [ ] I confirm generated SQL with `ToQueryString()` for any non-trivial query.
- [ ] Hot streaming paths use `IAsyncEnumerable<T>`.
- [ ] My LINQ tests run against SQLite or Testcontainers, not EF Core in-memory.
