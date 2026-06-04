# Entity Framework Core

## What It Is

Entity Framework Core (EF Core) is the official object–relational mapper (ORM) for .NET. It lets you query and persist domain objects — `Order`, `Customer`, `Invoice` — against a relational database (SQL Server, Azure SQL, PostgreSQL, SQLite, MySQL) or a document store (Cosmos DB) using LINQ instead of hand-written SQL. EF Core is the cross-platform successor to the original Entity Framework 6.

Three things are happening under the hood:

1. **Object–relational mapping** — your C# `Order` class is mapped to a `dbo.Orders` table, navigation properties become foreign keys, and `decimal` properties get `precision`/`scale` from the model configuration.
2. **LINQ translation** — `db.Orders.Where(o => o.Status == OrderStatus.Paid).ToListAsync()` is translated to parameterized SQL (`SELECT ... FROM Orders WHERE Status = @p0`) by an expression-tree pipeline.
3. **Change tracking and unit of work** — `DbContext` keeps a snapshot of every entity it loads. Calling `SaveChangesAsync` diffs the snapshots, generates `INSERT`/`UPDATE`/`DELETE` statements, and runs them inside a single transaction.

```csharp
// Domain class
public sealed class Order
{
    public Guid Id { get; init; }
    public Guid CustomerId { get; set; }
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
    public DateTimeOffset PlacedAt { get; set; }
    public byte[] RowVersion { get; set; } = [];
    public List<OrderLine> Lines { get; set; } = [];
}

// DbContext (the unit of work + query gateway)
public sealed class SalesDbContext(DbContextOptions<SalesDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
}
```

## Why It Exists

Before ORMs, .NET developers wrote `SqlCommand`, mapped `SqlDataReader` rows into objects by hand, and managed transactions, parameters, and connection pools manually. Each new column meant editing five places — SQL, mapper, DTO, validation, tests — and a single missed null check could corrupt production data.

EF Core was built to:

- Eliminate hand-written mapping code while still giving you a way to drop down to raw SQL when needed (`FromSqlInterpolated`, `ExecuteSqlInterpolatedAsync`).
- Provide a **strongly-typed, refactor-safe query language** (LINQ) instead of stringly-typed SQL embedded in C# files.
- Manage **schema evolution** through migrations versioned alongside code, so deployments can update the database deterministically.
- Centralize **change tracking and transactions** in a single unit of work, so business operations are atomic by default.
- Be **cross-platform** (runs on Linux, macOS, Windows; SQL Server, PostgreSQL, SQLite, Cosmos DB) and **modular** — you only install the providers you need.

## When To Use It

**Use EF Core for:**

- CRUD-heavy line-of-business APIs (order management, billing, CRM) where the schema and the object model evolve together.
- Aggregate-style write models in DDD where you load a small graph (`Order` + `OrderLines`), mutate it, and save in one transaction.
- Multi-database projects where you want one programming model across SQL Server, PostgreSQL, and SQLite (for tests).
- Code-first schema design with versioned migrations as the source of truth.
- Read scenarios where projections to DTOs and pagination are sufficient.

**Do not use EF Core for:**

- Bulk ETL (millions of rows per minute) — use `SqlBulkCopy`, Dapper, or `EFCore.BulkExtensions`.
- Reporting queries with complex window functions, recursive CTEs, or PIVOT — write SQL views or stored procedures.
- Schemas you cannot change (legacy databases with no migration story) — Dapper or raw ADO.NET is often cleaner.
- Workloads that are 100% read-only and latency-critical — micro-ORMs like Dapper avoid the change-tracker overhead.
- Cosmos DB write models that need fine-grained control over partition keys and request units — the SDK gives you more control than the EF Core Cosmos provider.

## Why It Is Important

EF Core directly drives four production-critical properties:

1. **Developer velocity** — adding a new column or entity is a one-line property change plus a migration, not a five-file refactor.
2. **Data correctness** — `SaveChangesAsync` is always one transaction. The change tracker prevents you from sending partial updates that would leave the database in a half-written state.
3. **Refactor safety** — renaming `Order.Total` to `Order.GrandTotal` is a Roslyn rename; LINQ queries follow. Stringly-typed SQL would silently break.
4. **Operational visibility** — EF Core integrates with `ILogger`, OpenTelemetry, and Application Insights so every query is captured with its SQL text, parameters, duration, and row count.

In an Azure context, EF Core is what lets your ASP.NET Core service talk to Azure SQL behind a Managed Identity, retry transient failures via `EnableRetryOnFailure`, and ship JSON-column features that map directly to Azure SQL's JSON support.

## How It's Used in C# / .NET

### 1. Install the providers you need

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design   # for migrations
dotnet add package Microsoft.EntityFrameworkCore.Tools    # for Package Manager Console
```

For other databases: `Npgsql.EntityFrameworkCore.PostgreSQL`, `Microsoft.EntityFrameworkCore.Cosmos`, `Microsoft.EntityFrameworkCore.Sqlite`.

### 2. Register `DbContext` in DI

```csharp
// Program.cs
builder.Services.AddDbContext<SalesDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Sales"),
        sql =>
        {
            sql.EnableRetryOnFailure(maxRetryCount: 5);          // Azure SQL transient errors
            sql.CommandTimeout(30);
            sql.MigrationsAssembly("Sales.Infrastructure");
        });

    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();   // logs parameter values — never in prod
        options.EnableDetailedErrors();
    }
});
```

For high-throughput services, prefer **context pooling**:

```csharp
builder.Services.AddDbContextPool<SalesDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sales")));
```

See [dbcontext-lifetime.md](dbcontext-lifetime.md) for why scoped vs pooled matters.

### 3. Configure the model with `OnModelCreating` or `IEntityTypeConfiguration<T>`

Keep configuration out of the entity class — one configuration per file scales better:

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> e)
    {
        e.ToTable("Orders", schema: "sales");
        e.HasKey(o => o.Id);
        e.Property(o => o.Total).HasPrecision(18, 2);
        e.Property(o => o.Status).HasConversion<string>().HasMaxLength(32);
        e.Property(o => o.RowVersion).IsRowVersion();

        e.HasMany(o => o.Lines)
            .WithOne()
            .HasForeignKey(l => l.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        e.HasIndex(o => new { o.CustomerId, o.PlacedAt })
            .HasDatabaseName("IX_Orders_Customer_PlacedAt");
    }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(SalesDbContext).Assembly);
}
```

### 4. Query with LINQ — and project to DTOs

```csharp
public sealed record OrderSummaryDto(Guid Id, string CustomerName, decimal Total, DateTimeOffset PlacedAt);

public async Task<IReadOnlyList<OrderSummaryDto>> GetRecentOrdersAsync(
    Guid customerId, int page, int size, CancellationToken ct)
{
    return await _db.Orders
        .AsNoTracking()                                         // no change tracking for reads
        .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Paid)
        .OrderByDescending(o => o.PlacedAt)
        .Skip(page * size).Take(size)
        .Select(o => new OrderSummaryDto(                        // project — don't load entities
            o.Id,
            o.Customer.DisplayName,
            o.Total,
            o.PlacedAt))
        .ToListAsync(ct);
}
```

### 5. Load related data — `Include`, `ThenInclude`, `AsSplitQuery`

```csharp
var order = await _db.Orders
    .Include(o => o.Lines).ThenInclude(l => l.Product)
    .Include(o => o.Customer)
    .AsSplitQuery()                  // avoid Cartesian explosion across multiple collections
    .FirstOrDefaultAsync(o => o.Id == orderId, ct);
```

Prefer **`Select` projection** over `Include` when you don't need the full graph — it's faster and ships less data.

### 6. Insert, update, delete via the change tracker

```csharp
var order = new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Pending };
order.Lines.Add(new OrderLine { ProductId = pid, Quantity = 2, UnitPrice = 19.99m });

_db.Orders.Add(order);             // marks Added; lines added with it via navigation
await _db.SaveChangesAsync(ct);    // one transaction: INSERT Orders + INSERT OrderLines
```

For bulk writes without loading entities (.NET 7+):

```csharp
// Mark all expired carts as abandoned — single UPDATE, no entity materialization
await _db.Carts
    .Where(c => c.LastActivity < DateTime.UtcNow.AddDays(-30))
    .ExecuteUpdateAsync(s => s.SetProperty(c => c.Status, CartStatus.Abandoned), ct);

// Hard-delete old audit rows
await _db.AuditLogs
    .Where(a => a.CreatedAt < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync(ct);
```

### 7. Migrations — code-first schema evolution

```bash
# Add a migration whenever the model changes
dotnet ef migrations add AddShippingAddressToOrders --project Sales.Infrastructure --startup-project Sales.Api

# Apply to local DB
dotnet ef database update --project Sales.Infrastructure --startup-project Sales.Api

# Generate an idempotent SQL script for CI/CD (preferred over auto-apply in production)
dotnet ef migrations script --idempotent --output sql/migrate.sql
```

See [migrations.md](migrations.md) for the deployment workflow.

### 8. Design-time `DbContext` factory for tooling

When your `DbContext` requires runtime config (Azure App Configuration, Key Vault), give the EF tools a way to build it:

```csharp
public class SalesDbContextFactory : IDesignTimeDbContextFactory<SalesDbContext>
{
    public SalesDbContext CreateDbContext(string[] args)
    {
        var options = new DbContextOptionsBuilder<SalesDbContext>()
            .UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=Sales-Design;Trusted_Connection=True")
            .Options;
        return new SalesDbContext(options);
    }
}
```

### 9. Modern features (.NET 8 / .NET 9)

- **JSON columns** — map a property of type `Address` directly to a JSON column in SQL Server / PostgreSQL:
  ```csharp
  e.OwnsOne(o => o.ShippingAddress, sa => sa.ToJson());
  ```
- **Primitive collections** — `List<string> Tags` is mapped to a JSON column natively (no join table).
- **`ExecuteUpdateAsync` / `ExecuteDeleteAsync`** — bulk operations without entity materialization.
- **Complex types** (EF 8+) — value-object semantics without owned-entity overhead.
- **Sentinel values, raw SQL composability, and `AsSplitQuery()` as default** for some shapes.

### 10. API quick reference

| API                                          | Use for                                                  |
|----------------------------------------------|----------------------------------------------------------|
| `AddDbContext<T>` / `AddDbContextPool<T>`    | Register context in DI                                   |
| `DbSet<T>.Add` / `Update` / `Remove`         | Stage changes for the next `SaveChangesAsync`            |
| `Include` / `ThenInclude`                    | Eager-load navigations                                   |
| `AsNoTracking` / `AsNoTrackingWithIdentityResolution` | Read-only queries                                |
| `AsSplitQuery`                               | Avoid Cartesian explosion across multiple `Include`s     |
| `Select` projection                          | Ship only the columns the API needs                      |
| `FromSqlInterpolated` / `SqlQuery<T>`        | Parameterized raw SQL                                    |
| `ExecuteUpdateAsync` / `ExecuteDeleteAsync`  | Server-side bulk operations (EF 7+)                      |
| `BeginTransactionAsync` / `IExecutionStrategy` | Multi-statement transactions with retry                |
| `[Timestamp]` / `IsRowVersion()`             | Optimistic concurrency token                             |
| `IDbContextFactory<T>`                       | Background services, Blazor, parallel work               |
| `dotnet ef migrations add` / `database update` / `migrations script` | Schema evolution                  |

## Advantages

- **Productivity** — write LINQ, get parameterized SQL; add a column, get a migration.
- **Refactor-safe queries** — rename a property, every query follows.
- **Unit of work + transactions for free** — `SaveChangesAsync` is atomic.
- **First-class testability** — projects target SQLite or Testcontainers-managed SQL Server for integration tests.
- **Cross-provider** — switch SQL Server ↔ PostgreSQL with a one-line change in `Program.cs`.
- **Mature ecosystem** — Pomelo, Npgsql, Oracle, Cosmos, SQLite providers; EF Power Tools; analyzers.
- **Telemetry integration** — Application Insights / OpenTelemetry capture every command.

## Disadvantages

- **Generated SQL can be poor** under careless LINQ (N+1, Cartesian explosion, client-side eval).
- **Change tracker overhead** is wasted on pure-read endpoints unless you opt out with `AsNoTracking`.
- **Migrations conflicts** are painful on big teams without a clear workflow (one merge of two parallel migrations can break history).
- **Provider parity gaps** — some features (e.g., JSON column queries) work differently or not at all across providers.
- **Black-box risk** — engineers who never look at the generated SQL ship slow queries to production.
- **Cold start cost** — first query in a process pays for model compilation (mitigated by `IModelCacheKeyFactory` and AOT in EF 8+).
- **Not ideal for bulk ETL** — use Dapper, `SqlBulkCopy`, or `EFCore.BulkExtensions` for million-row operations.

## Common Mistakes

### 1. The N+1 Query

```csharp
// BUG: 1 query for orders + N queries for each order's customer
var orders = await _db.Orders.Where(o => o.Status == OrderStatus.Paid).ToListAsync(ct);
foreach (var o in orders)
    Console.WriteLine($"{o.Id} - {o.Customer.DisplayName}"); // lazy load OR null after AsNoTracking
```

**Fix**: either `Include(o => o.Customer)` (eager) or, better, project once:

```csharp
var dtos = await _db.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Paid)
    .Select(o => new { o.Id, CustomerName = o.Customer.DisplayName })
    .ToListAsync(ct);
```

See [query-performance.md](query-performance.md).

### 2. Tracking Read-Only Queries

```csharp
// BUG: every order is snapshotted just to be returned over HTTP
var orders = await _db.Orders.Where(o => o.CustomerId == id).ToListAsync(ct);
```

**Fix**: add `AsNoTracking()` (or set `QueryTrackingBehavior.NoTracking` as the context default for read-only contexts).

### 3. `ToList()` Then `Where()` — Client-Side Filtering

```csharp
// BUG: pulls every order into memory, then filters in C#
var open = (await _db.Orders.ToListAsync(ct)).Where(o => o.Status == OrderStatus.Pending);
```

**Fix**: filter first, materialize second. `_db.Orders.Where(...).ToListAsync(ct)`.

### 4. Forgetting `CancellationToken`

```csharp
// BUG: HTTP request is canceled, but EF Core keeps running the query against SQL
public Task<List<Order>> GetAll() => _db.Orders.ToListAsync();
```

**Fix**: thread `CancellationToken` through every async EF call so canceled requests don't hold SQL connections.

### 5. Multiple `Include` Without `AsSplitQuery`

```csharp
// BUG: single SQL with Cartesian join blows up to millions of rows
var orders = await _db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .Include(o => o.Shipments)
    .ToListAsync(ct);
```

**Fix**: add `.AsSplitQuery()` (one query per collection) or project only what you need.

### 6. Mutating Detached Entities Without `Attach` / `Update`

```csharp
// BUG: nothing happens — entity isn't tracked
var order = orderDto.MapToEntity();
order.Total = newTotal;
await _db.SaveChangesAsync(ct);   // no-op
```

**Fix**: `_db.Orders.Update(order)` (marks all properties modified) or load + mutate + save.

### 7. Auto-Applying Migrations at Startup in Production

```csharp
// BUG: every replica races to migrate; one wins, others fail health checks
await db.Database.MigrateAsync();
```

**Fix**: generate an idempotent SQL script (`dotnet ef migrations script --idempotent`) and run it from your CI/CD pipeline before the new app version starts. See [migrations.md](migrations.md).

### 8. Injecting `DbContext` Into a Singleton

A scoped `DbContext` captured by a singleton is a classic captive dependency and produces concurrency exceptions under load. Use `IDbContextFactory<T>` from a background service. See [dbcontext-lifetime.md](dbcontext-lifetime.md).

## Best Practices

- **Always project to DTOs** for read endpoints — never expose entities directly.
- **Always `AsNoTracking()`** read paths; consider setting `UseQueryTrackingBehavior(NoTracking)` for a read-only context.
- **Use `EnableRetryOnFailure`** on Azure SQL — transient failures are normal.
- **Apply migrations via idempotent SQL** in CI/CD; do not auto-migrate at app startup in production.
- **Keep `DbContext` scoped** for web requests; use `IDbContextFactory<T>` for background work, Blazor circuits, and parallel queries.
- **Always pass `CancellationToken`** into every async EF call.
- **Add indexes** that match your `Where` / `OrderBy` clauses — query plans are not magic.
- **Use `IEntityTypeConfiguration<T>`** to keep model config out of entity classes.
- **Log slow queries** via `ILogger` filters; set `EnableSensitiveDataLogging` only in development.
- **Inspect generated SQL** for every new endpoint — `LogTo(Console.WriteLine)` in dev, or capture via OpenTelemetry in prod.
- **Use Testcontainers (real SQL Server in Docker)** for integration tests, not the InMemory provider. See [../testing-quality/integration-testing.md](../testing-quality/integration-testing.md).

## Related Concepts

- **DbContext Lifetime** — Scoped vs Singleton vs pooled — see [dbcontext-lifetime.md](dbcontext-lifetime.md).
- **Migrations** — schema evolution workflow — see [migrations.md](migrations.md).
- **Query Performance** — N+1, projections, indexes — see [query-performance.md](query-performance.md).
- **Transactions** — `SaveChangesAsync`, `BeginTransactionAsync`, isolation — see [transactions.md](transactions.md).
- **Optimistic Concurrency** — `RowVersion`, `DbUpdateConcurrencyException` — see [optimistic-concurrency.md](optimistic-concurrency.md).
- **Repository Pattern** — when EF Core IS your repository — see [repository-pattern-with-ef-core.md](repository-pattern-with-ef-core.md).
- **Unit of Work** — `DbContext` is your unit of work — see [../architecture/unit-of-work.md](../architecture/unit-of-work.md).
- **Aggregates / Repositories (DDD)** — see [../architecture/aggregates.md](../architecture/aggregates.md), [../architecture/repositories.md](../architecture/repositories.md).
- **Outbox Pattern** — store domain events in the same transaction as state changes — see [../architecture/outbox-pattern.md](../architecture/outbox-pattern.md).
- **Azure SQL** — production hosting — see [../azure/azure-sql.md](../azure/azure-sql.md).
- **Dapper / micro-ORMs** — alternative for read-heavy workloads.

## Real-World Usage

### Order Management API on Azure App Service

A typical e-commerce service uses EF Core against Azure SQL with Managed Identity:

```csharp
builder.Services.AddDbContext<SalesDbContext>(options =>
{
    var conn = new SqlConnection(builder.Configuration.GetConnectionString("Sales"));
    conn.AccessToken = new DefaultAzureCredential()
        .GetToken(new TokenRequestContext(["https://database.windows.net/.default"]))
        .Token;
    options.UseSqlServer(conn, sql =>
    {
        sql.EnableRetryOnFailure(5);
        sql.CommandTimeout(30);
    });
});
```

`PlaceOrderHandler` loads the customer's cart, builds an `Order` aggregate, inserts a row into the outbox table, and calls `SaveChangesAsync` — one transaction covers the state change *and* the integration event, so a downstream Service Bus publisher can deliver it reliably.

### Background Worker — Outbox Publisher

A `BackgroundService` runs every 2 seconds, creates a new scope via `IDbContextFactory<SalesDbContext>`, claims a batch of unpublished outbox messages, publishes them to Azure Service Bus, and marks them as sent. See [../architecture/outbox-pattern.md](../architecture/outbox-pattern.md).

### Reporting Read Model on Cosmos DB

Sales summaries are projected to a Cosmos container via a separate `ReportingDbContext` using the Cosmos provider. Read latency drops from ~200 ms (SQL aggregate) to <10 ms (point read by `customerId` partition key).

## Code Example — Before and After

### Before: Hand-Written ADO.NET, Mutable State Everywhere

```csharp
public class OrderRepository
{
    private readonly string _connStr;
    public OrderRepository(string connStr) => _connStr = connStr;

    public List<Order> GetByCustomer(Guid customerId)
    {
        var orders = new List<Order>();
        using var conn = new SqlConnection(_connStr);
        conn.Open();
        // SQL injection risk and stringly typed
        using var cmd = new SqlCommand(
            $"SELECT Id, CustomerId, Status, Total FROM Orders WHERE CustomerId = '{customerId}'",
            conn);
        using var r = cmd.ExecuteReader();
        while (r.Read())
        {
            orders.Add(new Order
            {
                Id = r.GetGuid(0),
                CustomerId = r.GetGuid(1),
                Status = (OrderStatus)r.GetInt32(2),
                Total = r.GetDecimal(3)
            });
        }
        return orders;
    }

    public void AddLine(Guid orderId, Guid productId, int qty, decimal price)
    {
        using var conn = new SqlConnection(_connStr);
        conn.Open();
        using var cmd = new SqlCommand(
            "INSERT INTO OrderLines (OrderId, ProductId, Quantity, UnitPrice) VALUES (@o, @p, @q, @u)",
            conn);
        cmd.Parameters.AddWithValue("@o", orderId);
        cmd.Parameters.AddWithValue("@p", productId);
        cmd.Parameters.AddWithValue("@q", qty);
        cmd.Parameters.AddWithValue("@u", price);
        cmd.ExecuteNonQuery();
        // No transaction with the parent insert — partial writes possible
    }
}
```

Problems: SQL injection, blocking calls, no transaction, no concurrency token, no tests possible without a real database, every schema change touches multiple methods.

### After: EF Core, Async, Transactional, Testable

```csharp
public sealed class OrderRepository(SalesDbContext db) : IOrderRepository
{
    public Task<IReadOnlyList<OrderSummaryDto>> GetByCustomerAsync(Guid customerId, CancellationToken ct) =>
        db.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.PlacedAt)
            .Select(o => new OrderSummaryDto(o.Id, o.Customer.DisplayName, o.Total, o.PlacedAt))
            .ToListAsync(ct)
            .ContinueWith(t => (IReadOnlyList<OrderSummaryDto>)t.Result, ct);

    public Task<Order?> GetForUpdateAsync(Guid orderId, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == orderId, ct);
}

// Application layer — one transaction for state + outbox
public async Task<Guid> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Create(cmd.CustomerId, cmd.Lines);
    db.Orders.Add(order);
    db.OutboxMessages.Add(OutboxMessage.From(new OrderPlaced(order.Id, order.Total)));
    await db.SaveChangesAsync(ct);          // atomic: order + lines + outbox
    return order.Id;
}
```

Now the code is parameterized, async, cancellation-aware, atomic, and unit-testable with Testcontainers.

## Interview Questions and Answers

### 1. Walk me through what happens when you call `await db.SaveChangesAsync()`.

**Why this matters**: It separates candidates who treat EF Core as a black box from those who understand the change tracker and transaction model.

**Answer**: EF Core inspects every entity the `DbContext` is tracking, compares each property to the snapshot it captured at load time, and groups changes into `Added`, `Modified`, and `Deleted` sets. It opens a connection (from the pool), begins a transaction, and issues `INSERT` / `UPDATE` / `DELETE` statements in dependency order so foreign keys are satisfied. If anything fails — a unique-index violation, a row-version mismatch — it rolls back the transaction and throws (`DbUpdateException`, `DbUpdateConcurrencyException`). On success, it commits, refreshes generated values (identity keys, `RowVersion`), and accepts the changes so the next save only re-evaluates what's new.

**Trade-off**: The convenience hides cost — the change tracker runs over every loaded entity. For pure-read workloads, `AsNoTracking()` skips this entirely.

**Real project**: A 600 ms checkout endpoint dropped to 90 ms after we discovered the team was loading 50 reference entities for every request just to read display names. We projected to DTOs and tracking dropped from 50 entities to 0.

### 2. How do you avoid the N+1 query problem in EF Core?

**Why this matters**: N+1 is the single most common EF Core performance bug.

**Answer**: Three strategies, in order of preference:
1. **Project with `Select`**: load only the columns you need; navigation accesses translate into joins.
2. **`Include` + `ThenInclude`** when you really need the entity graph; add `AsSplitQuery()` when multiple collections would cause a Cartesian explosion.
3. **Explicit loading** (`db.Entry(order).Collection(o => o.Lines).LoadAsync()`) for branches where the data is only conditionally needed.

Capture generated SQL in dev (`options.LogTo(Console.WriteLine)`), inspect it in PR review, and add Application Insights dependency telemetry that flags any HTTP request with more than ~5 SQL dependencies.

**Trade-off**: `AsSplitQuery` makes multiple round trips; on high-latency links to Azure SQL, fewer larger queries can be faster than many small ones.

**Real project**: A dashboard listing 50 orders made 51 SQL calls because of a lazy-loaded `Customer` navigation. Switching to a single projection brought it to 1 query and shaved 800 ms.

### 3. When would you NOT use EF Core?

**Why this matters**: Tests whether the candidate has shipped real workloads beyond CRUD.

**Answer**: Skip EF Core when:
- **Bulk ETL** — 5 million rows per minute. Use `SqlBulkCopy`, Dapper, or `EFCore.BulkExtensions`.
- **Complex reporting SQL** — window functions, recursive CTEs, PIVOT. Write a SQL view; query it via Dapper or `FromSqlInterpolated`.
- **Stored-procedure-heavy legacy databases** where the schema must not change.
- **Latency-critical read paths** under heavy load (e.g., feature flag lookups) — Dapper is leaner.
- **Cosmos DB write models** that require fine-grained partition-key and RU control — use the SDK directly.

**Trade-off**: Mixing ORMs ("EF Core for writes, Dapper for reads") is fine, but doubles your testing surface. Justify it.

**Real project**: We moved a nightly billing job (12 million invoices) from EF Core to `SqlBulkCopy` and cut the run from 3 hours to 12 minutes — the change tracker had been the bottleneck.

### 4. Explain `AsNoTracking` and when you'd use `AsNoTrackingWithIdentityResolution`.

**Answer**: `AsNoTracking()` tells EF Core to skip the change tracker for the query — entities are materialized but never snapshotted, so reads are faster and use less memory. The trade-off is that the same `Order` row loaded twice in the same query produces two distinct objects.

`AsNoTrackingWithIdentityResolution()` (EF 5+) is a middle ground: still no change tracking, but EF Core deduplicates entities by primary key during the query, so navigation properties that reference the same entity point to the same instance. Use it when you'd otherwise see duplicate `Customer` objects in a list of `Order` projections.

**Real project**: A reporting endpoint built a graph view that broke client expectations because the same `Customer` appeared as different instances. Switching to `AsNoTrackingWithIdentityResolution` fixed the client without re-enabling tracking.

### 5. How does EF Core translate LINQ to SQL, and what causes "client-side evaluation" warnings?

**Answer**: EF Core walks the expression tree of your LINQ query and matches subtrees to SQL fragments via the provider. Anything it can translate becomes SQL; anything it can't (a method call to your own code, complex string manipulation, custom value comparisons) historically ran on the client after materialization. Since EF Core 3.x, **the provider throws** rather than silently shipping the database down the wire — you opt in via `.AsEnumerable()` if you genuinely want client-side eval.

**Common triggers**: calling your own helper methods inside `Where`, using `Trim()` overloads not supported by the provider, calling `.ToString()` on enums, or using `string.Equals` with `StringComparison`.

**Real project**: A `Where(o => OrderHelpers.IsRecent(o))` predicate compiled fine but threw at runtime. We replaced the helper with `Where(o => o.PlacedAt >= DateTime.UtcNow.AddDays(-30))` so the comparison ran in SQL.

### 6. How do you handle migrations in a team of 8 engineers shipping multiple features per day?

**Why this matters**: Real-world EF Core failures cluster around migration conflicts.

**Answer**: 
1. **One pending migration per feature branch**; rebase before merging so the migration sits at the tip of `main`.
2. **Idempotent SQL scripts in CI/CD** (`dotnet ef migrations script --idempotent`) — never auto-apply at app startup.
3. **Expand–contract pattern**: add nullable columns and dual-write code first; backfill; then make non-nullable; then remove the old column. This keeps deployments forward- and backward-compatible.
4. **Block schema-breaking migrations behind a PR template** that requires a backfill plan and a rollback plan.
5. **Run the new migration in a staging slot** before swapping. See [../azure/deployment-to-azure.md](../azure/deployment-to-azure.md) and [../devops/blue-green-deployment.md](../devops/blue-green-deployment.md).

**Trade-off**: Idempotent scripts are larger; you trade build-time work for runtime safety.

**Real project**: A two-developer parallel merge produced two migrations targeting the same table. The CI build failed because the second migration's `Down()` couldn't run on a database that had only the first. We added a pre-merge gate that runs `dotnet ef migrations script --from <last-deployed>` and diffs it.

### 7. How would you write integration tests for EF Core code?

**Answer**: Use **Testcontainers** to spin up a real SQL Server (or Postgres) container per test class, apply migrations once, and run each test inside a transaction that's rolled back at teardown. Avoid the **InMemory provider** — it doesn't enforce constraints, transactions, or concurrency, so it lies about your code's behavior.

```csharp
public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();
    private SalesDbContext _db = null!;

    public async Task InitializeAsync()
    {
        await _sql.StartAsync();
        var options = new DbContextOptionsBuilder<SalesDbContext>()
            .UseSqlServer(_sql.GetConnectionString()).Options;
        _db = new SalesDbContext(options);
        await _db.Database.MigrateAsync();
    }

    public Task DisposeAsync() => _sql.DisposeAsync().AsTask();
}
```

**Trade-off**: Testcontainers is slower than InMemory (container startup ~5 s), but the bugs it catches (cascade deletes, indexes, unique constraints, concurrency) repay that 100×.

**Real project**: An InMemory-only test suite let a missing unique index ship to production; we got a duplicate-customer report within 2 hours. We migrated to Testcontainers the next week.

### 8. What's the difference between `Update`, `Attach`, and just modifying a tracked entity?

**Answer**:
- **Modifying a tracked entity**: load via the context, change properties, call `SaveChangesAsync`. EF Core detects only the columns that actually changed and writes a tight `UPDATE`.
- **`Attach(entity)`**: tells the context the entity already exists; state is `Unchanged`. Use when you have a detached entity (e.g., from an API request) and want to set a navigation FK without loading the whole graph.
- **`Update(entity)`**: attaches the entity and marks every property as `Modified`. Writes every column — even unchanged ones — and can overwrite concurrent updates if you don't use a row-version token.

**Trade-off**: `Update` is convenient for upserts but dangerous in concurrency-sensitive scenarios. Prefer load + mutate + save, or use `ExecuteUpdateAsync` for targeted single-column updates.

**Real project**: A REST `PUT /orders/{id}` endpoint used `Update`, which silently overwrote a `Status` change a background job had just made. We switched to load + mutate + save with `RowVersion` to detect the conflict. See [optimistic-concurrency.md](optimistic-concurrency.md).

## Summary Checklist

- [ ] I can explain what `DbContext`, `DbSet<T>`, and the change tracker do during `SaveChangesAsync`.
- [ ] I can register EF Core with `AddDbContext` / `AddDbContextPool` / `AddDbContextFactory` and explain when to use each.
- [ ] I can configure entities with `IEntityTypeConfiguration<T>`, including indexes, value converters, and `IsRowVersion`.
- [ ] I can identify and fix N+1 queries with `Include`, `Select` projection, or `AsSplitQuery`.
- [ ] I can use `AsNoTracking()` and explain when to add `WithIdentityResolution`.
- [ ] I can write a migration, generate an idempotent SQL script, and deploy it via CI/CD safely.
- [ ] I can use `ExecuteUpdateAsync` / `ExecuteDeleteAsync` for server-side bulk operations.
- [ ] I can write integration tests with Testcontainers instead of the InMemory provider.
- [ ] I can recognize when EF Core is the wrong tool (bulk ETL, complex reporting, latency-critical reads).
- [ ] I can inspect generated SQL and tie performance back to model configuration and indexes.
