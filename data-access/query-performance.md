# Query Performance

## What It Is

**Query performance** in EF Core is the discipline of making sure each LINQ query translates into the *right* SQL, ships the *right* amount of data over the wire, and is supported by the *right* indexes on the database side. It covers everything from the shape of your `Where`/`Include`/`Select` chain to whether the change tracker is on, to whether the database has a covering index on `(CustomerId, PlacedAt)`.

A good EF Core query has four properties:

1. **Filtered server-side** — `WHERE` clauses run on the database, not in C#.
2. **Projected to the minimum columns** the API actually returns.
3. **Tracked only if you intend to mutate** — read-only queries use `AsNoTracking`.
4. **Backed by an index** matching the predicate and sort columns.

When all four are present, an EF Core query is indistinguishable from hand-tuned SQL — fast, predictable, and inexpensive. When any one is missing, you ship N+1 queries, megabyte responses, captive change-tracker memory, or full table scans.

```csharp
// All four properties: filter in SQL, projection, no tracking, supported by IX_Orders_Customer_PlacedAt
var orders = await _db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Paid)
    .OrderByDescending(o => o.PlacedAt)
    .Skip(page * size).Take(size)
    .Select(o => new OrderSummaryDto(o.Id, o.CustomerName, o.Total, o.PlacedAt))
    .ToListAsync(ct);
```

## Why It Exists

EF Core is a productivity layer; it does not guarantee fast queries. The LINQ-to-SQL translator turns *correct* LINQ into *correct* SQL but not always into *fast* SQL. The hardest-to-detect performance bugs are not crashes — they're an endpoint that runs in 15 ms on dev with 200 rows and 2,800 ms on production with 200 million rows.

Engineers need an explicit query-performance discipline because:

- The change tracker is convenient but **expensive** when read paths use it by default.
- `Include` looks innocent until you discover it produced a 50-million-row Cartesian explosion.
- Lazy loading silently triggers an extra round trip per navigation property accessed in a `foreach`.
- A missing `OrderBy` makes `Skip/Take` non-deterministic across pages.
- Indexes are easy to forget when the only place the predicate appears is C#.

The cost of fixing these in production (after an incident) is 10–100× the cost of catching them in PR review.

## When To Use It

This entire topic applies to **every EF Core query** you ship to production. Specifically:

- **Always apply** the four properties (filter, project, track only if mutating, index) to read-heavy endpoints (search, list, dashboard).
- **Always benchmark** queries that run on large tables, in tight loops, or in time-sensitive paths (checkout, login, payment).
- **Always design** indexes alongside queries — not after performance incidents.
- **Consider compiled queries** for queries on the hot path of high-RPS services.

**Do not over-optimize:**
- One-off admin queries that run once a day with a small result set — readability wins.
- Background jobs where wall-clock time is not on the critical path.
- Prototype code that hasn't been validated by product.

## Why It Is Important

A single slow query on a high-traffic endpoint can:

1. **Cascade into pool exhaustion** — every concurrent request holds a SQL connection until its query returns; slow queries pile up; new requests time out waiting on `MaxPoolSize`.
2. **Drive Azure SQL DTU/vCore upgrades** — teams routinely 4× their Azure SQL bill because they didn't add an index.
3. **Burn the request budget** — 400 ms on the database leaves 100 ms for everything else if your SLA is 500 ms p95.
4. **Hide N+1 bugs** in dependency telemetry — if you don't look at App Insights' "1 HTTP request → 50 SQL dependencies" view, the bug stays in production for months.
5. **Make scale impossible** — a query that's 50 ms at 10 requests/sec might be 5,000 ms at 1,000 requests/sec because of lock contention.

Query performance is also the most common topic in senior interviews because it tests whether you understand *what EF Core is doing* — not just whether you can write LINQ.

## How It's Used in C# / .NET

### 1. Project to DTOs — don't return entities

```csharp
public sealed record OrderSummaryDto(Guid Id, string CustomerName, decimal Total, DateTimeOffset PlacedAt);

// GOOD: ships 4 columns, no tracking, no lazy loading
var dtos = await _db.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Paid)
    .Select(o => new OrderSummaryDto(o.Id, o.Customer.DisplayName, o.Total, o.PlacedAt))
    .ToListAsync(ct);

// BAD: loads every column of Order, plus every column of Customer via Include
var entities = await _db.Orders
    .Include(o => o.Customer)
    .Where(o => o.Status == OrderStatus.Paid)
    .ToListAsync(ct);
```

A projection over a 60-column `Orders` table can be 10× smaller on the wire than a full `Include`.

### 2. `AsNoTracking` for every read path

```csharp
// Set it as the default for a read-only context
public sealed class SalesReadDbContext(DbContextOptions<SalesReadDbContext> opts) : DbContext(opts)
{
    protected override void OnConfiguring(DbContextOptionsBuilder o) =>
        o.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}

// Or per-query
var products = await _db.Products.AsNoTracking().Where(p => p.IsActive).ToListAsync(ct);

// Use AsNoTrackingWithIdentityResolution when graph reads contain duplicate references
var orders = await _db.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)
    .ToListAsync(ct);
```

### 3. Eager-load with `Include` + `AsSplitQuery` (when needed)

```csharp
// GOOD: split queries avoid the Cartesian explosion across multiple collections
var order = await _db.Orders
    .Include(o => o.Lines).ThenInclude(l => l.Product)
    .Include(o => o.Payments)
    .Include(o => o.Shipments)
    .AsSplitQuery()
    .FirstOrDefaultAsync(o => o.Id == orderId, ct);
```

Rule of thumb: any query that includes more than **one collection** should consider `AsSplitQuery`. The trade-off is one round trip becomes N round trips — usually still cheaper than transferring an N×M×P matrix of rows.

### 4. Pagination — prefer keyset over offset for deep pages

```csharp
// Offset pagination — fine for first ~10 pages, gets slow as the offset grows
var page = await _db.Orders
    .AsNoTracking()
    .OrderByDescending(o => o.PlacedAt)
    .Skip((pageIndex - 1) * pageSize).Take(pageSize)
    .ToListAsync(ct);

// Keyset (seek) pagination — constant time regardless of depth
var page = await _db.Orders
    .AsNoTracking()
    .Where(o => o.PlacedAt < lastSeenPlacedAt || (o.PlacedAt == lastSeenPlacedAt && o.Id < lastSeenId))
    .OrderByDescending(o => o.PlacedAt).ThenByDescending(o => o.Id)
    .Take(pageSize)
    .ToListAsync(ct);
```

Offset pagination requires SQL Server to walk through the skipped rows even though it doesn't return them — at page 500 with 100 rows/page, that's a 50,000-row scan per request. Keyset pagination uses the index and is O(log N).

### 5. Inspect generated SQL

```csharp
// In dev — log every SQL command
builder.Services.AddDbContext<SalesDbContext>(o =>
    o.UseSqlServer(cs)
     .LogTo(Console.WriteLine, LogLevel.Information)
     .EnableSensitiveDataLogging());

// In prod — capture via OpenTelemetry / Application Insights
builder.Services.AddOpenTelemetry().WithTracing(t => t
    .AddSource("Microsoft.EntityFrameworkCore")
    .AddAzureMonitorTraceExporter());
```

In a SQL profiler or SSMS, prefix the query with `SET STATISTICS IO ON; SET STATISTICS TIME ON;` to see logical reads, scan counts, and CPU per query. Use `EXPLAIN` on PostgreSQL or `Include Actual Execution Plan` (`Ctrl+M`) in SSMS to find table scans.

### 6. Indexes — design them with your queries

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> e)
    {
        // Supports WHERE CustomerId = ? ORDER BY PlacedAt DESC
        e.HasIndex(o => new { o.CustomerId, o.PlacedAt })
            .HasDatabaseName("IX_Orders_Customer_PlacedAt")
            .IsDescending(false, true);

        // Covering index — INCLUDE columns the projection needs
        e.HasIndex(o => o.Status)
            .HasDatabaseName("IX_Orders_Status_Covered")
            .IncludeProperties(o => new { o.Total, o.PlacedAt });

        // Filtered index for hot subset
        e.HasIndex(o => o.PlacedAt)
            .HasDatabaseName("IX_Orders_PlacedAt_Pending")
            .HasFilter("[Status] = 'Pending'");

        // Unique business identifier (not the PK)
        e.HasIndex(o => o.OrderNumber).IsUnique();
    }
}
```

### 7. `ExecuteUpdateAsync` / `ExecuteDeleteAsync` — server-side bulk ops

```csharp
// Mark stale carts abandoned — no entity materialization, single UPDATE
await _db.Carts
    .Where(c => c.LastActivity < DateTime.UtcNow.AddDays(-30))
    .ExecuteUpdateAsync(s => s
        .SetProperty(c => c.Status, CartStatus.Abandoned)
        .SetProperty(c => c.AbandonedAt, DateTimeOffset.UtcNow), ct);

// Hard-delete audit logs older than 2 years
await _db.AuditLogs
    .Where(a => a.CreatedAt < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync(ct);
```

Avoid the loop pattern `foreach (var c in carts) { c.Status = ...; }` followed by `SaveChangesAsync` — that loads N entities, snapshots N entities, and writes N `UPDATE` statements.

### 8. Compiled queries for the hot path

```csharp
private static readonly Func<SalesDbContext, Guid, CancellationToken, Task<Order?>> GetOrderByIdAsync =
    EF.CompileAsyncQuery((SalesDbContext db, Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().FirstOrDefault(o => o.Id == id));

public Task<Order?> FindAsync(Guid id, CancellationToken ct) => GetOrderByIdAsync(_db, id, ct);
```

Skips the LINQ-to-SQL translation step on every call. Gains are modest (5–15%) but real for very high-RPS endpoints.

### 9. Raw SQL when LINQ can't express it

```csharp
// Parameterized — safe from SQL injection (FormattableString is escaped)
var topCustomers = await _db.Database
    .SqlQuery<TopCustomerDto>(
        $"SELECT TOP 10 CustomerId, SUM(Total) AS TotalSpent FROM sales.Orders WHERE PlacedAt >= {since} GROUP BY CustomerId ORDER BY TotalSpent DESC")
    .ToListAsync(ct);

// Or against an entity
var hotOrders = await _db.Orders
    .FromSqlInterpolated($"SELECT * FROM sales.Orders WITH (NOLOCK) WHERE Status = 'Pending'")
    .AsNoTracking()
    .ToListAsync(ct);
```

`WITH (NOLOCK)` is a SQL Server-specific hint that reads dirty rows; use it only for fast dashboards that can tolerate stale data.

### 10. API quick reference

| Technique                                | Use for                                                |
|------------------------------------------|--------------------------------------------------------|
| `AsNoTracking`                           | All read paths                                         |
| `AsNoTrackingWithIdentityResolution`     | Read paths that need consistent navigation references  |
| `Select(x => new Dto(...))`              | Ship only the columns the API returns                  |
| `Include` + `AsSplitQuery`               | Eager-load multiple collections without Cartesian blow-up |
| Keyset pagination                        | Deep pagination on hot tables                          |
| `ExecuteUpdateAsync` / `ExecuteDeleteAsync` | Bulk mutations without materialization              |
| `EF.CompileAsyncQuery`                   | Hot-path queries on high-RPS APIs                      |
| `FromSqlInterpolated` / `SqlQuery<T>`    | Parameterized raw SQL for cases LINQ can't express     |
| `HasIndex` + `IncludeProperties`         | Define covering indexes alongside the query            |
| `LogTo(Console.WriteLine)` / OpenTelemetry | Capture generated SQL for review                     |
| `SET STATISTICS IO ON` / `EXPLAIN`       | Measure scans, reads, plan choice                      |

## Advantages

- **Predictable latency** — projections + indexes turn unbounded scans into index seeks.
- **Lower SQL bill** — fewer DTU/RU consumed; Azure SQL stays on a cheaper tier longer.
- **Higher throughput** — connections released faster, pool stays free, more concurrent requests.
- **Better cancellation behavior** — `CancellationToken` + short queries means client disconnects release resources immediately.
- **Cleaner code** — projections force you to model the API contract explicitly instead of leaking entities.

## Disadvantages

- **Indexes cost write performance** — every `INSERT`/`UPDATE` updates every index. Don't index every column.
- **Keyset pagination is harder** — no "page 47 of 200" UI without a separate count query.
- **Compiled queries add boilerplate** — only worth it on the hot path.
- **Raw SQL bypasses model validation** — typos surface only at runtime.
- **`AsSplitQuery` causes multiple round trips** — on high-latency links (cross-region), one big query may still win.
- **`AsNoTracking` breaks if you forget and try to `SaveChanges`** — changes are silently lost.

## Common Mistakes

### 1. The N+1 Query

```csharp
// BUG: 1 query for the list + N queries inside the loop (lazy load OR re-query)
var orders = await _db.Orders.Where(o => o.CustomerId == id).ToListAsync(ct);
foreach (var o in orders)
{
    var customer = await _db.Customers.FirstAsync(c => c.Id == o.CustomerId, ct);
    Console.WriteLine($"{o.Id} - {customer.DisplayName}");
}
```

**Fix**: project once, with a join:

```csharp
var rows = await _db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == id)
    .Select(o => new { o.Id, CustomerName = o.Customer.DisplayName })
    .ToListAsync(ct);
```

### 2. `Include` Without `AsSplitQuery`

```csharp
// BUG: orders × lines × payments × shipments — millions of rows, gigabytes transferred
var orders = await _db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Include(o => o.Shipments)
    .Where(o => o.CustomerId == id)
    .ToListAsync(ct);
```

**Fix**: add `.AsSplitQuery()` or project narrowly:

```csharp
var orders = await _db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Include(o => o.Shipments)
    .Where(o => o.CustomerId == id)
    .AsSplitQuery()
    .ToListAsync(ct);
```

### 3. Client-Side Filtering

```csharp
// BUG: pulls every order into memory, filters in C#, ships unused rows over the wire
var open = (await _db.Orders.ToListAsync(ct)).Where(o => o.Status == OrderStatus.Pending);
```

**Fix**: filter first, materialize second. Since EF Core 3, the provider throws `InvalidOperationException` for predicates it can't translate — that's a good thing.

### 4. Offset Pagination on a Deep Page

```csharp
// BUG: at page 500 with 100/page, SQL Server scans 50,000 rows then discards them
var page = await _db.Orders.OrderBy(o => o.Id).Skip(50_000).Take(100).ToListAsync(ct);
```

**Fix**: keyset pagination keyed on `(PlacedAt, Id)` returning constant-time pages.

### 5. Missing Index on Predicate Columns

```csharp
// Query: WHERE CustomerId = ? AND Status = 'Paid' — needs an index
await _db.Orders.AsNoTracking()
    .Where(o => o.CustomerId == id && o.Status == OrderStatus.Paid)
    .ToListAsync(ct);
```

**Fix**: add a composite index in the entity config and ship it via migration:

```csharp
e.HasIndex(o => new { o.CustomerId, o.Status });
```

### 6. Tracking Reads

```csharp
// BUG: change tracker holds 10k entities just to render a CSV
var orders = await _db.Orders.ToListAsync(ct);
return BuildCsv(orders);
```

**Fix**: `_db.Orders.AsNoTracking().ToListAsync(ct)` (or set `QueryTrackingBehavior.NoTracking` as the default for a read-only context).

### 7. `Count()` Followed By a Loop

```csharp
// BUG: two queries — a COUNT and then a SELECT — when the result fits the page anyway
var total = await _db.Orders.CountAsync(ct);
var items = await _db.Orders.Skip(0).Take(20).ToListAsync(ct);
```

**Fix**: defer the `Count()` to a separate endpoint or load on demand; or use `Take(pageSize + 1)` and infer "has more" from the extra row.

### 8. Calling `SaveChangesAsync` Inside a Loop

```csharp
// BUG: 10k transactions, 10k round trips, 10k log writes
foreach (var c in carts)
{
    c.Status = CartStatus.Abandoned;
    await _db.SaveChangesAsync(ct);
}
```

**Fix**: use `ExecuteUpdateAsync` (no materialization) or save once after the loop.

### 9. Forgetting `CancellationToken`

A canceled HTTP request without a token still keeps the SQL query running, the connection pinned, and the response materialized — only to be thrown away. Always pass `CancellationToken` to every async EF call.

### 10. Using `Find` for a Query That Needs Includes

```csharp
// BUG: Find ignores Include — you'll get a tracked entity with empty navigations
var order = await _db.Orders.Include(o => o.Lines).FindAsync(id);
```

**Fix**: `await _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);`

## Best Practices

- **Project every read endpoint to a DTO.** Entities never leave the persistence layer.
- **Default to `AsNoTracking()`** on read paths; consider a read-only context with `QueryTrackingBehavior.NoTracking` globally.
- **Inspect generated SQL** for every new endpoint in dev — `LogTo(Console.WriteLine)` for one minute reveals N+1s and missing indexes faster than any review.
- **Define indexes in `IEntityTypeConfiguration<T>`** when you write the query — don't wait for a slow-query report.
- **Use keyset pagination** on hot, deep-paged endpoints (orders, audit logs, chats).
- **Use `ExecuteUpdateAsync` / `ExecuteDeleteAsync`** for bulk maintenance — never `foreach + SaveChanges`.
- **Use `AsSplitQuery()`** when including more than one collection.
- **Always pass `CancellationToken`** — client disconnects release SQL connections immediately.
- **Compile queries** on the hot path; benchmark before and after with BenchmarkDotNet.
- **Add Application Insights dependency telemetry** and alert on HTTP requests with more than ~10 SQL dependencies.
- **Re-run the same query under realistic data volume** (10× expected production rows) before shipping.
- **Use Testcontainers** for query benchmarks — InMemory provider doesn't reflect real query plans.

## Related Concepts

- **Entity Framework Core** — see [entity-framework-core.md](entity-framework-core.md).
- **DbContext Lifetime** — see [dbcontext-lifetime.md](dbcontext-lifetime.md). Long-lived contexts amplify tracking-overhead bugs.
- **Transactions** — see [transactions.md](transactions.md). Pagination interacts with isolation levels.
- **Optimistic Concurrency** — see [optimistic-concurrency.md](optimistic-concurrency.md).
- **Migrations** — see [migrations.md](migrations.md). Indexes are defined here.
- **Repository Pattern** — see [repository-pattern-with-ef-core.md](repository-pattern-with-ef-core.md). A leaky `IQueryable` repository hides query bugs.
- **Scalability Design** — see [../architecture/scalability-design.md](../architecture/scalability-design.md).
- **Azure SQL** — see [../azure/azure-sql.md](../azure/azure-sql.md). DTUs/vCores and Query Store.
- **Application Insights** — see [../azure/application-insights.md](../azure/application-insights.md). Dependency telemetry catches N+1.
- **Logging and Monitoring** — see [../aspnet-core/logging-and-monitoring.md](../aspnet-core/logging-and-monitoring.md).

## Real-World Usage

### Search Endpoint on Azure SQL

A product search endpoint handling 1,200 RPS was returning entities with 30 columns and four navigation properties via `Include`. p99 latency was 3.4 s; DTU was pinned at 95%. After:
- Switching to a projection of 6 columns (`Select(p => new ProductSearchDto(...))`).
- Adding a composite index `IX_Products_CategoryId_IsActive_Name`.
- Setting `QueryTrackingBehavior.NoTracking` globally for the read DbContext.

…p99 dropped to 120 ms and DTU fell to 28%. No infrastructure change.

### Audit Log Pagination

The admin UI paged through 80M audit log rows using offset pagination. Pages 1–5 loaded in <100 ms; page 200 took 38 s. Migrating to keyset pagination keyed on `(CreatedAt DESC, Id DESC)` returned page 200 in 35 ms. The UI was updated to "load more" instead of numbered pages.

### Nightly Job: Mark Expired Carts

```csharp
// Before: 4 hours, 400k entities, 400k UPDATE statements
foreach (var c in await _db.Carts.Where(c => c.LastActivity < cutoff).ToListAsync(ct))
{
    c.Status = CartStatus.Abandoned;
}
await _db.SaveChangesAsync(ct);

// After: 6 seconds, single UPDATE
await _db.Carts
    .Where(c => c.LastActivity < cutoff)
    .ExecuteUpdateAsync(s => s.SetProperty(c => c.Status, CartStatus.Abandoned), ct);
```

### Reporting Read Model in Cosmos DB

Sales summaries originally ran a 6-table aggregate query against Azure SQL (~220 ms). We projected the same data into a Cosmos container partitioned by `customerId` updated via the outbox pattern. Point reads dropped to <10 ms p99 and removed all DTU pressure from the OLTP database.

## Code Example — Before and After

### Before: N+1, Full Entity, Tracked, No Indexes

```csharp
public async Task<IActionResult> ListOrders(Guid customerId, int page, int size)
{
    // 1 query for orders, no AsNoTracking, no projection
    var orders = await _db.Orders
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.PlacedAt)
        .Skip(page * size).Take(size)
        .ToListAsync();

    var result = new List<object>();
    foreach (var o in orders)
    {
        // Lazy load (or re-query) for each customer name — N round trips
        var name = (await _db.Customers.FindAsync(o.CustomerId))?.DisplayName;
        result.Add(new { o.Id, CustomerName = name, o.Total, o.PlacedAt });
    }

    return Ok(result);
}
```

Symptoms: 51 SQL calls per request (1 + 50), p95 of 1.2 s, DTU at 90%, change tracker holding 50 orders + 50 customers per request.

### After: One Projection, Index-Backed, No Tracking, Cancellable

```csharp
public sealed record OrderSummaryDto(Guid Id, string CustomerName, decimal Total, DateTimeOffset PlacedAt);

public async Task<IActionResult> ListOrders(
    Guid customerId, int page, int size, CancellationToken ct)
{
    var dtos = await _db.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.PlacedAt)
        .Skip(page * size).Take(size)
        .Select(o => new OrderSummaryDto(o.Id, o.Customer.DisplayName, o.Total, o.PlacedAt))
        .ToListAsync(ct);

    return Ok(dtos);
}

// In the entity configuration
e.HasIndex(o => new { o.CustomerId, o.PlacedAt })
    .HasDatabaseName("IX_Orders_Customer_PlacedAt")
    .IsDescending(false, true)
    .IncludeProperties(o => new { o.Total });
```

Result: 1 SQL call, p95 ~45 ms, DTU under 20%, zero tracker overhead.

## Interview Questions and Answers

### 1. How do you detect and fix an N+1 query in EF Core?

**Why this matters**: N+1 is the most common production-killing EF Core bug.

**Answer**: Detection: enable `LogTo(Console.WriteLine)` in dev and watch for one query followed by N similar queries inside a loop. In production, Application Insights' dependency telemetry surfaces it as "HTTP request → 50 SQL calls".

Fix: replace the loop with a single LINQ query that projects the joined data, or use `Include` if you really need the entity graph:

```csharp
// Single projection — 1 SQL query
var rows = await _db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == id)
    .Select(o => new { o.Id, CustomerName = o.Customer.DisplayName, o.Total })
    .ToListAsync(ct);
```

**Trade-off**: For very wide projections, sometimes splitting is faster than joining. Measure.

**Real project**: An order dashboard was making 51 SQL calls per request; the fix was a 2-line LINQ change and dropped p95 from 1.2 s to 80 ms.

### 2. When does `AsSplitQuery` help and when does it hurt?

**Answer**: It helps when a single query would produce a Cartesian explosion across multiple collections. If `Order` has 4 lines and 3 payments, the joined query returns 12 rows of duplicated `Order` data; with 50 orders it becomes 600 rows; with deeper graphs it explodes.

`AsSplitQuery` runs one query per collection, so you trade fewer rows for more round trips. It hurts when the round-trip latency is high (cross-region Azure SQL) or when the collections are small and the join wouldn't blow up.

Rule of thumb: any query with more than one `Include(... collection)` should use `AsSplitQuery`, and benchmark to confirm.

**Real project**: A dashboard `Include`ing 3 collections produced 4 GB of transfer for one user; switching to split queries dropped it to 40 MB.

### 3. Explain the difference between offset and keyset pagination, and when you'd use each.

**Answer**:
- **Offset pagination** (`Skip(n).Take(size)`): simple, supports "page N of M" UIs. SQL Server scans the skipped rows even though it doesn't return them, so cost grows linearly with page depth.
- **Keyset (seek) pagination**: uses a `WHERE` predicate on the last seen value (`Where(o => o.PlacedAt < lastSeen)`). Cost is O(log N) using the supporting index; constant time regardless of page depth.

Use offset when the dataset is small (< 10k rows) or pages are shallow. Use keyset for hot lists like audit logs, chats, orders, or feeds.

**Trade-off**: Keyset can't show "page 47 of 200" — it's "load more". For UX with deep page jumps, you'd combine keyset with separate count caches.

**Real project**: Page 200 of an audit-log dashboard went from 38 s (offset) to 35 ms (keyset).

### 4. How do you decide which indexes to add?

**Answer**: Look at every query that hits a table with > 100k rows and ask:
- What columns are in `Where`? Those go into the index.
- What's in `OrderBy`? Those go in the index in matching direction.
- What columns are in `Select` projection? Add them as `IncludeProperties(...)` for a covering index.

For SQL Server, use Query Store (built into Azure SQL) to surface the highest-impact missing indexes. For PostgreSQL, `EXPLAIN ANALYZE` and `pg_stat_statements`. Don't add indexes blindly — every index slows down inserts and updates.

Define indexes in `IEntityTypeConfiguration<T>` so they ship with migrations and live in source control:

```csharp
e.HasIndex(o => new { o.CustomerId, o.PlacedAt })
 .IncludeProperties(o => new { o.Total, o.Status });
```

**Real project**: Adding a covering index `(CustomerId, PlacedAt) INCLUDE (Total, Status)` cut a hot endpoint's DTU from 90% to 22% with zero code changes.

### 5. When would you use `ExecuteUpdateAsync` instead of loading entities and calling `SaveChangesAsync`?

**Answer**: Whenever you need to update or delete rows based on a predicate and don't need the change tracker, concurrency tokens, or domain events.

```csharp
await _db.Carts
    .Where(c => c.LastActivity < cutoff)
    .ExecuteUpdateAsync(s => s.SetProperty(c => c.Status, CartStatus.Abandoned), ct);
```

This sends a single `UPDATE Carts SET Status = ... WHERE ...` statement. No entities materialized, no snapshots, no `SaveChangesAsync` overhead. For 400k rows it's 6 seconds vs 4 hours.

**Trade-off**:
- No tracking means no automatic concurrency check (`RowVersion`) — accept overwriting concurrent changes.
- Triggers domain events you'd raise in `SaveChangesAsync` are bypassed.
- Some side effects (audit columns) won't be applied unless you set them explicitly.

**Real project**: An overnight cart-cleanup job was the slowest part of the deploy verification window. Switching to `ExecuteUpdateAsync` made it instant.

### 6. How would you investigate a slow EF Core query in production?

**Answer**:
1. **Open Application Insights**, filter by the slow endpoint, look at the dependency waterfall — find the long SQL call.
2. **Grab the SQL text** (Application Insights captures it; never log parameter values in production unless you've configured PII redaction).
3. **Run it in SSMS / Azure Data Studio** with `SET STATISTICS IO ON; SET STATISTICS TIME ON;` and **Include Actual Execution Plan**.
4. Look for **scans, key lookups, sort operators with spills, hash joins on large tables, missing-index suggestions**.
5. **Map back to the LINQ** — is the SQL what you expected? If not, fix the LINQ (often a missed projection or an unintended `Include`).
6. **Add or adjust the index** in `IEntityTypeConfiguration<T>` and ship a migration.
7. **Verify with Query Store** that the new plan is being used.

**Trade-off**: Sometimes the right answer is a query rewrite (project less, split queries differently), not a new index. Indexes have a write-cost — don't add them as a reflex.

### 7. What's the difference between `Find`, `FirstOrDefault`, and `Single`?

**Answer**:
- **`Find(key)` / `FindAsync(key)`**: looks first in the change tracker, then queries the database if not found. Returns the tracked entity. Does **not** support `Include`, `Where`, or projections.
- **`FirstOrDefault(predicate)`**: always queries the database, returns the first matching row or `null`. Supports the full LINQ surface (`Include`, `Select`, etc.).
- **`Single(predicate)`**: queries the database, returns the single matching row, throws if zero or more than one. Useful when uniqueness is part of the invariant you're enforcing.

Use `Find` only for trivial PK lookups without navigation properties; use `FirstOrDefault` for everything else.

**Real project**: A developer used `Find(id)` after a `.Include(o => o.Lines)` and got an order with empty `Lines` — `Find` ignored the `Include`. The fix was to switch to `FirstOrDefaultAsync(o => o.Id == id, ct)`.

### 8. How do you stop a slow query from blocking the connection pool?

**Answer**:
- **Pass `CancellationToken` everywhere**, so client disconnects cancel the SQL command and release the connection immediately.
- **Set `CommandTimeout`** to a sensible value (e.g., 30 seconds for user-facing queries) so runaway queries don't pin the connection.
- **Use `AsNoTracking` and projections** to minimize materialization time.
- **Don't run long queries on the same DbContext as request work** — push reports to a separate read replica or a background job.
- **Monitor pool usage** with `Microsoft.Data.SqlClient.ConnectionAvailableCount` in Application Insights; alert when it trends down.
- **Set sensible `MaxPoolSize`** in the connection string but treat it as a ceiling, not a target.

**Real project**: A weekly report query without `CommandTimeout` pinned 8 connections for 12 minutes during a peak window and starved the API. Adding a 30-second timeout plus moving the report to a dedicated read-only context solved it.

## Summary Checklist

- [ ] I can identify N+1 queries from logs and Application Insights and fix them via projection.
- [ ] I can choose between `Include`, `Select`, and `AsSplitQuery` for graph reads.
- [ ] I can apply `AsNoTracking` (or set a no-tracking default for a read-only context).
- [ ] I can implement keyset pagination and explain why it beats offset for deep pages.
- [ ] I can design composite and covering indexes in `IEntityTypeConfiguration<T>`.
- [ ] I can use `ExecuteUpdateAsync` / `ExecuteDeleteAsync` for server-side bulk operations.
- [ ] I can compile a hot-path query with `EF.CompileAsyncQuery`.
- [ ] I can read a SQL Server execution plan and identify scans, key lookups, and missing-index suggestions.
- [ ] I can ensure every async EF call passes a `CancellationToken`.
- [ ] I can detect slow queries in Application Insights and tie them back to the LINQ source.
