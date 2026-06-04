# DbContext Lifetime

## What It Is

The **`DbContext` lifetime** is the rule that decides how long a single `DbContext` instance lives, when it is created, and when it is disposed. In ASP.NET Core, `AddDbContext<T>()` registers the context as **Scoped** by default — one instance per HTTP request — and the framework disposes it automatically when the request completes.

A `DbContext` is three things at once:

1. **A connection facade** — it owns a `DbConnection` from the ADO.NET connection pool while it has work to do.
2. **A unit of work** — it accumulates Added/Modified/Deleted entities and commits them atomically in `SaveChangesAsync`.
3. **An identity map** — within one instance, the same primary key always materializes the same .NET object.

Choosing the wrong lifetime breaks all three: connections leak, transactions span unrelated work, and the change tracker either hoards memory forever or fights itself across threads.

```csharp
// Default for web apps — Scoped: one DbContext per HTTP request
builder.Services.AddDbContext<SalesDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sales")));

// High-throughput web apps — pooled Scoped instances reused across requests
builder.Services.AddDbContextPool<SalesDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sales")));

// Background services, Blazor Server, parallel work — factory creates per-operation instances
builder.Services.AddDbContextFactory<SalesDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sales")));
```

## Why It Exists

EF Core's `DbContext` is **not thread-safe** and was designed around the unit-of-work pattern. The framework needs to know *when* to start a fresh tracker and *when* to throw the current one away. Without lifetime rules:

- A long-lived context would accumulate entities indefinitely and leak memory.
- Two HTTP requests sharing one context could trigger `InvalidOperationException: A second operation was started on this context before a previous operation completed.`
- Background services that hold a context for hours would hold a SQL connection out of the pool for the same duration.

By tying the default lifetime to the HTTP request scope, ASP.NET Core gives you a natural transaction boundary: one request loads, mutates, saves, and the framework disposes the context — releasing the connection back to the pool and clearing the tracker.

## When To Use It

- **Use Scoped (`AddDbContext`)** for the vast majority of ASP.NET Core APIs and MVC apps. One context per HTTP request gives a clean unit of work and lifetime aligned with the request.
- **Use Pooled Scoped (`AddDbContextPool`)** when profiling shows context construction is a hot path (very high RPS APIs, ~5k+ requests/sec per instance). The pool keeps a small number of pre-built contexts and resets state between requests.
- **Use `IDbContextFactory<T>`** in:
  - `BackgroundService` / `IHostedService` workers — they live as Singletons, so they must create a scope (or call `CreateDbContextAsync`) per unit of work.
  - **Blazor Server** — each component has its own lifecycle; sharing a scoped context across a circuit causes concurrency exceptions.
  - **Parallel queries** — running `Task.WhenAll(query1, query2)` on the same context throws.
  - **Console apps and Azure Functions** where the host lifetime model doesn't align with HTTP request scopes.

**Do not use:**

- **`Singleton`** for `DbContext` — ever. It captures one connection and one tracker for the life of the process; concurrent requests collide; memory grows unbounded.
- **`Transient`** for `DbContext` — multiple repositories within the same request each get a different context, breaking the unit of work; `SaveChangesAsync` in one doesn't see changes staged in another.
- Manually `new SalesDbContext()` inside a service — bypasses DI, breaks configuration, leaks connections.

## Why It Is Important

The wrong lifetime produces bugs that are silent in development and catastrophic in production:

1. **Captive dependency** — a Singleton service holding a Scoped `DbContext` pins one tracker for the life of the process. Memory grows, tracker state goes stale, and every request collides with the previous one.
2. **Thread-safety violations** — under load, two requests touching the same context throw `InvalidOperationException`. The error rate looks random; the bug is structural.
3. **Connection-pool exhaustion** — a context held for minutes (because it was captured) holds a connection slot, so other requests time out waiting on `MaxPoolSize`.
4. **Lost atomicity** — Transient contexts across repositories make `SaveChangesAsync` partial — only the entities added to *that* context get saved.
5. **Subtle data leaks** — pooled contexts that don't reset custom state (event handlers, properties) carry data between requests.

In Azure App Service, where each instance handles many requests per second, lifetime mistakes show up first as random 500s and end with database failovers. Getting this right is more important than picking the right index.

## How It's Used in C# / .NET

### 1. Default — `AddDbContext` (Scoped)

```csharp
// Program.cs
builder.Services.AddDbContext<SalesDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Sales"),
        sql => sql.EnableRetryOnFailure(5));
});

// Controller — DI provides one SalesDbContext per request
[ApiController]
[Route("api/orders")]
public sealed class OrdersController(SalesDbContext db, ILogger<OrdersController> logger) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id, CancellationToken ct)
    {
        var order = await db.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
        return order is null ? NotFound() : Ok(order);
    }
}
```

When the response is flushed, ASP.NET Core disposes the scope; the `SalesDbContext` is disposed; the underlying connection returns to the pool.

### 2. Context pooling — `AddDbContextPool`

```csharp
builder.Services.AddDbContextPool<SalesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Sales")),
    poolSize: 1024); // default is 1024
```

How it works: instead of `new SalesDbContext()` per request, the pool hands out a recycled instance, resets internal state (change tracker, query cache, current transaction), and returns it on dispose. Reduces allocations significantly on high-RPS workloads.

**Caveats**:

- Your `DbContext` must be **safe to reset**. Don't store request-specific state in fields, properties, or event handlers (`SavingChanges`, `SaveChangesFailed`) — they survive the pool unless you remove them.
- Constructor must accept only `DbContextOptions<T>` (or a primary constructor with no extra params) so the pool can rebuild internal state cleanly.

### 3. `IDbContextFactory<T>` — for non-request contexts

Register in DI:

```csharp
builder.Services.AddDbContextFactory<SalesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Sales")));
```

Use in a `BackgroundService`:

```csharp
public sealed class OutboxPublisher(
    IDbContextFactory<SalesDbContext> contextFactory,
    ServiceBusSender sender,
    ILogger<OutboxPublisher> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var db = await contextFactory.CreateDbContextAsync(stoppingToken);

            var pending = await db.OutboxMessages
                .Where(m => m.PublishedAt == null)
                .OrderBy(m => m.OccurredAt)
                .Take(100)
                .ToListAsync(stoppingToken);

            foreach (var msg in pending)
            {
                await sender.SendMessageAsync(msg.ToServiceBusMessage(), stoppingToken);
                msg.PublishedAt = DateTimeOffset.UtcNow;
            }

            await db.SaveChangesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(2), stoppingToken);
        }
    }
}
```

Each iteration gets a fresh context and disposes it — no captive state, no memory growth.

### 4. Parallel queries — never share a context

```csharp
// BUG: runs both queries on the SAME context concurrently → InvalidOperationException
var (orders, customers) = await TaskHelpers.WhenAll(
    db.Orders.ToListAsync(ct),
    db.Customers.ToListAsync(ct));

// FIX: use the factory to get one context per parallel branch
var (orders, customers) = await TaskHelpers.WhenAll(
    LoadAsync(contextFactory, c => c.Orders.ToListAsync(ct)),
    LoadAsync(contextFactory, c => c.Customers.ToListAsync(ct)));

static async Task<T> LoadAsync<T>(
    IDbContextFactory<SalesDbContext> factory,
    Func<SalesDbContext, Task<T>> query)
{
    await using var db = await factory.CreateDbContextAsync();
    return await query(db);
}
```

### 5. Blazor Server — factory per component

```csharp
// Program.cs
builder.Services.AddDbContextFactory<SalesDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sales")));

// OrdersList.razor.cs
public partial class OrdersList : ComponentBase, IAsyncDisposable
{
    [Inject] public IDbContextFactory<SalesDbContext> DbFactory { get; set; } = null!;
    private SalesDbContext _db = null!;

    protected override async Task OnInitializedAsync()
    {
        _db = await DbFactory.CreateDbContextAsync();
        Orders = await _db.Orders.AsNoTracking().Take(50).ToListAsync();
    }

    public async ValueTask DisposeAsync() => await _db.DisposeAsync();
}
```

Why: Blazor Server's component lifecycle does **not** match a single DI scope; injecting a Scoped `DbContext` directly causes concurrency exceptions when components re-render.

### 6. Validate at startup

Catch lifetime bugs before the first request:

```csharp
builder.Host.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes = true;     // throw if a Scoped service is captured by a Singleton
    o.ValidateOnBuild = true;    // resolve every registered service at startup
});
```

### 7. Quick reference

| Registration                       | Lifetime                       | Use for                                  |
|------------------------------------|--------------------------------|------------------------------------------|
| `AddDbContext<T>`                  | Scoped (per HTTP request)      | Default for ASP.NET Core APIs / MVC      |
| `AddDbContextPool<T>`              | Pooled Scoped (recycled)       | Very high RPS, stateless contexts        |
| `AddDbContextFactory<T>`           | Singleton factory + transient contexts | Workers, Blazor, parallel queries, Functions |
| `AddPooledDbContextFactory<T>`     | Factory backed by a pool       | Both worker-style and high-RPS combined  |
| Manual `new SalesDbContext()`      | None                           | Never in production                      |

## Advantages

- **Automatic disposal** — ASP.NET Core scopes guarantee `DbContext` is disposed when the request ends, returning the connection to the pool.
- **Built-in unit of work** — one context per request means `SaveChangesAsync` is a single transaction across all repositories in that request.
- **Identity map** — one instance per request guarantees that two `db.Orders.Find(id)` calls return the same object.
- **Pooling reduces allocations** for high-RPS APIs without changing consumer code.
- **`IDbContextFactory`** gives a clean way to use EF Core outside HTTP requests without the captive-dependency trap.

## Disadvantages

- **Easy to misuse** — captive dependencies look fine in development and explode under load.
- **Pooled contexts must be carefully written** — any per-request state in fields or event handlers leaks across requests.
- **`IDbContextFactory` adds boilerplate** — every consumer must remember to `await using var db = ...` and dispose it.
- **Long-running requests hold a connection** — `MaxPoolSize` (default 100 for SQL Server) becomes the hard ceiling on concurrent requests. Pagination, streaming, and `CancellationToken` matter.
- **Blazor Server changes the rules** — engineers coming from MVC must relearn lifetime decisions or face concurrency exceptions.

## Common Mistakes

### 1. Injecting `DbContext` into a Singleton

```csharp
// BUG: AppCache (Singleton) captures one SalesDbContext forever
builder.Services.AddSingleton<AppCache>();
builder.Services.AddDbContext<SalesDbContext>(o => o.UseSqlServer(cs));

public sealed class AppCache(SalesDbContext db)
{
    public Task<List<Product>> AllProductsAsync() => db.Products.ToListAsync();
}
```

The first request initializes `AppCache` with a context tied to *that* request's scope. After the request ends, the scope is disposed — the context is now disposed too, but `AppCache` still holds it. Every subsequent call throws `ObjectDisposedException`.

**Fix**:

```csharp
public sealed class AppCache(IDbContextFactory<SalesDbContext> factory)
{
    public async Task<List<Product>> AllProductsAsync(CancellationToken ct)
    {
        await using var db = await factory.CreateDbContextAsync(ct);
        return await db.Products.AsNoTracking().ToListAsync(ct);
    }
}
```

### 2. Resolving from the root provider in a `BackgroundService`

```csharp
// BUG: GetRequiredService on the root provider — disposed when host shuts down,
// but the context is also leaked because no scope is ever disposed.
public sealed class BadWorker(IServiceProvider sp) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var db = sp.GetRequiredService<SalesDbContext>(); // captive
        while (!ct.IsCancellationRequested)
        {
            await db.SaveChangesAsync(ct);
            await Task.Delay(1000, ct);
        }
    }
}
```

**Fix**: create a scope (or use `IDbContextFactory`) per iteration.

```csharp
public sealed class GoodWorker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<SalesDbContext>();
            // ... work ...
            await db.SaveChangesAsync(ct);
            await Task.Delay(1000, ct);
        }
    }
}
```

### 3. Running parallel queries on one context

```csharp
// BUG: InvalidOperationException — DbContext is not thread-safe
var (orders, invoices) = await TaskHelpers.WhenAll(
    db.Orders.Where(o => o.CustomerId == id).ToListAsync(ct),
    db.Invoices.Where(i => i.CustomerId == id).ToListAsync(ct));
```

**Fix**: await sequentially (almost always fast enough), or use `IDbContextFactory` to get one context per parallel branch.

### 4. Pooled context with mutable state

```csharp
public sealed class SalesDbContext : DbContext
{
    public string? CurrentTenantId { get; set; } // BUG: bleeds across requests in a pool

    public SalesDbContext(DbContextOptions<SalesDbContext> opts) : base(opts) { }
}
```

The pool resets EF Core's internal state but **not your properties**. Tenant A's data shows up in Tenant B's response.

**Fix**: don't store request-specific state on the context. Inject `ITenantContext` into the query path instead, or override `OnConfiguring` carefully with state captured at construction.

### 5. Manually instantiating `DbContext`

```csharp
// BUG: bypasses DI, no retry policy, no connection pooling integration, no scope
var db = new SalesDbContext(new DbContextOptionsBuilder<SalesDbContext>()
    .UseSqlServer("Server=...").Options);
```

**Fix**: always go through DI (`AddDbContext`) or `IDbContextFactory.CreateDbContextAsync()`.

### 6. Forgetting `await using` with the factory

```csharp
// BUG: context never disposed → connection leaked
var db = await factory.CreateDbContextAsync();
return await db.Orders.ToListAsync();
```

**Fix**: `await using var db = await factory.CreateDbContextAsync();`

### 7. Holding a `DbContext` across HTTP requests via a static or singleton cache

Aside from the captive-dependency bug, this also serializes the entire system on one connection. Avoid at all costs.

## Best Practices

- **Default to Scoped** for ASP.NET Core; only switch to pooled when profiling proves a need.
- **Use `IDbContextFactory<T>`** for all background services, Blazor Server, parallel queries, and Azure Functions.
- **Enable `ValidateScopes = true` and `ValidateOnBuild = true`** in every environment — they cost nothing and catch lifetime bugs at startup.
- **Keep the context constructor minimal** — `DbContextOptions<T>` only — so pooling works.
- **Don't store request state** on the context; inject collaborators that carry it.
- **Always `await using`** the context returned by a factory so disposal happens even on exceptions.
- **Cancel long-running queries** with `CancellationToken` so a slow client doesn't pin a connection.
- **Monitor connection-pool usage** in Application Insights (`Microsoft.Data.SqlClient.ConnectionAvailableCount`) to spot leaks early.
- **Write a DI smoke test** that resolves every registered service via the scoped provider — it catches captive dependencies before deployment.

## Related Concepts

- **Dependency Injection** — see [../csharp/dependency-injection.md](../csharp/dependency-injection.md). DbContext lifetime is the canonical DI lifetime question.
- **Unit of Work** — `DbContext` *is* your unit of work — see [../architecture/unit-of-work.md](../architecture/unit-of-work.md).
- **Entity Framework Core** — see [entity-framework-core.md](entity-framework-core.md).
- **Migrations** — design-time factories matter here — see [migrations.md](migrations.md).
- **Transactions** — context lifetime delimits the implicit transaction — see [transactions.md](transactions.md).
- **Repository Pattern** — repositories that share a scoped context share the unit of work — see [repository-pattern-with-ef-core.md](repository-pattern-with-ef-core.md).
- **ASP.NET Core Request Pipeline** — see [../aspnet-core/request-pipeline.md](../aspnet-core/request-pipeline.md).
- **Azure SQL Connection Pool** — see [../azure/azure-sql.md](../azure/azure-sql.md).
- **Background Services (`IHostedService`)** — they live as Singletons and need factories.

## Real-World Usage

### High-RPS Catalog API on Azure App Service

A product catalog API serving 8,000 RPS per instance was allocating ~3 MB/s just from `SalesDbContext` construction. Switching from `AddDbContext` to `AddDbContextPool` reduced allocations by 80% and dropped p99 latency from 22 ms to 14 ms. The team audited the context for mutable state, removed an `OnSavingChanges` lambda that captured a request-scoped logger, and added an integration test that runs 1,000 parallel requests to detect cross-request bleed.

### Outbox Worker on AKS

A `BackgroundService` polling an outbox table every two seconds was originally written with `services.GetRequiredService<SalesDbContext>()` from the root provider. After 12 hours, memory hit the pod limit and the pod was OOM-killed. Switching to `IDbContextFactory<SalesDbContext>` with `await using var db = ...` per iteration kept memory flat at 90 MB indefinitely.

### Blazor Server Admin Console

An admin dashboard for back-office users was throwing intermittent `InvalidOperationException` whenever a user opened two tabs. Replacing direct `SalesDbContext` injection with `IDbContextFactory<SalesDbContext>` and one short-lived context per component lifecycle eliminated the errors.

### Multi-Tenant SaaS

For a multi-tenant API, the tenant filter must not be stored on the pooled context. Instead, an `ITenantContext` is injected into each query method, and EF Core's global query filters read it via a delegate registered in `OnModelCreating`. The pooled context stays stateless.

## Code Example — Before and After

### Before: Singleton Service Holding a Scoped Context

```csharp
// Program.cs
builder.Services.AddDbContext<SalesDbContext>(o => o.UseSqlServer(cs));
builder.Services.AddSingleton<ProductCatalogCache>();

public sealed class ProductCatalogCache
{
    private readonly SalesDbContext _db;
    private Dictionary<Guid, Product>? _cache;

    public ProductCatalogCache(SalesDbContext db) => _db = db;

    public async Task<Product?> GetAsync(Guid id, CancellationToken ct)
    {
        _cache ??= await _db.Products.ToDictionaryAsync(p => p.Id, ct);
        return _cache.GetValueOrDefault(id);
    }
}
```

Symptoms in production:
- Second request throws `ObjectDisposedException` because the context was disposed with the first request's scope.
- Memory grows because the `_cache` is never refreshed.
- Connection-pool exhaustion as the connection is held forever (until the eventual exception).
- `ValidateScopes = true` rejects the registration at startup — only if it's enabled.

### After: Singleton Cache + Factory + Periodic Refresh

```csharp
builder.Services.AddDbContextFactory<SalesDbContext>(o => o.UseSqlServer(cs));
builder.Services.AddSingleton<ProductCatalogCache>();
builder.Services.AddHostedService<ProductCatalogRefresher>();

public sealed class ProductCatalogCache
{
    private volatile Dictionary<Guid, Product> _snapshot = new();
    public Product? Get(Guid id) => _snapshot.GetValueOrDefault(id);
    internal void Replace(Dictionary<Guid, Product> next) => _snapshot = next;
}

public sealed class ProductCatalogRefresher(
    IDbContextFactory<SalesDbContext> factory,
    ProductCatalogCache cache,
    ILogger<ProductCatalogRefresher> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var db = await factory.CreateDbContextAsync(stoppingToken);
                var snapshot = await db.Products
                    .AsNoTracking()
                    .ToDictionaryAsync(p => p.Id, stoppingToken);
                cache.Replace(snapshot);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to refresh product catalog");
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

The cache is now a true Singleton with no captured context, the refresh runs in a worker with a per-iteration context, and the connection is released after every refresh.

## Interview Questions and Answers

### 1. Why is `DbContext` registered as Scoped by default in ASP.NET Core?

**Why this matters**: It tests whether the candidate understands the unit-of-work model behind EF Core.

**Answer**: `DbContext` is a unit of work and an identity map. Tying it to the HTTP request scope means one request gets one consistent change tracker, one transaction boundary, and one connection from the pool — and the framework guarantees it's disposed when the response is flushed. Scoped is also safe from a thread-safety standpoint because each request runs on a separate scope; the context is never shared across concurrent requests.

**Trade-off**: It assumes a unit of work matches a request. For Blazor Server, background workers, and parallel queries, this assumption breaks and you need `IDbContextFactory<T>`.

**Real project**: A REST API that processed multi-step checkouts within a single request found it could rely on `SaveChangesAsync` being atomic across multiple repositories because they shared one scoped context.

### 2. What happens if you inject a `DbContext` into a Singleton?

**Why this matters**: This is the single most common DI bug with EF Core.

**Answer**: The Singleton captures the context from the *first* scope that resolves it. After that scope is disposed, the context is disposed too. The next call throws `ObjectDisposedException`. Even before that, the context's change tracker accumulates state across all "requests" the singleton sees, leaking memory and causing data from one operation to bleed into the next.

The fix is to inject `IDbContextFactory<T>` (or `IServiceScopeFactory`) and create a fresh context per operation, wrapping it in `await using`.

If you enable `ValidateScopes = true` in `Program.cs`, the DI container will throw at startup instead of failing silently in production.

**Real project**: A team shipped a Singleton "cache warmer" that worked for the first request, then 500'd for everyone else. Two hours of incident time was spent before someone added scope validation.

### 3. When would you use `AddDbContextPool` instead of `AddDbContext`?

**Answer**: Use `AddDbContextPool` when you've profiled and found `DbContext` allocation/construction on the hot path of a high-RPS API. The pool keeps a small set of pre-built contexts and resets them between requests — fewer allocations, less GC pressure, slightly lower latency.

Requirements:
- The context must be safe to reset — no mutable state in fields/properties, no captured event handlers that survive disposal.
- Constructor must accept only `DbContextOptions<T>`.

**Trade-off**: A bug-prone optimization for low-RPS services. The framework's `Scoped` lifetime is already very fast; pooling only matters past a few thousand RPS per instance.

**Real project**: A product catalog API hitting 8,000 RPS/instance saw a 10–15% allocation reduction after switching to pooling. A back-office admin tool at 50 RPS would see no measurable difference.

### 4. How do you use `DbContext` correctly inside a `BackgroundService`?

**Answer**: Resolve `IDbContextFactory<T>` (or `IServiceScopeFactory`) in the constructor, then create a fresh context per iteration:

```csharp
public sealed class OutboxPublisher(IDbContextFactory<SalesDbContext> factory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var db = await factory.CreateDbContextAsync(ct);
            var batch = await db.OutboxMessages.Where(m => m.PublishedAt == null).Take(100).ToListAsync(ct);
            // publish, mutate, SaveChangesAsync
            await Task.Delay(TimeSpan.FromSeconds(2), ct);
        }
    }
}
```

Never resolve the context once and reuse it — the tracker would grow forever, the connection would never return to the pool, and one failure could poison every subsequent iteration.

**Real project**: An outbox publisher that reused one context OOM-killed its pod after 14 hours. Refactoring to per-iteration contexts kept memory flat for weeks.

### 5. Why can't you run two queries in parallel on the same `DbContext`?

**Answer**: `DbContext` is not thread-safe. It maintains a single underlying ADO.NET command/reader pipeline; running two operations concurrently throws `InvalidOperationException: A second operation was started on this context before a previous operation completed`. The change tracker also assumes single-threaded access — concurrent mutations can corrupt its internal dictionaries.

The fix is to use one context per parallel branch via `IDbContextFactory<T>`, or simply `await` queries sequentially (often fast enough, especially when SQL Server batches multiple round trips itself).

**Trade-off**: Parallelism has diminishing returns when each query takes ~5 ms and the database is the bottleneck. Sequential is usually fine.

**Real project**: A dashboard that loaded six widgets in parallel via `Task.WhenAll` threw `InvalidOperationException` under load testing. Switching to per-widget contexts via the factory fixed it; the latency was unchanged because SQL Server happily ran the connections in parallel itself.

### 6. How does `DbContext` lifetime interact with connection pooling?

**Answer**: A `DbContext` lazily opens a `DbConnection` (from the ADO.NET pool) when it first executes a command, then holds it until the context is disposed *or* the transaction closes — whichever comes later. With Scoped lifetime, the connection is returned to the pool when the HTTP request ends. With a leaked or captive context, the connection is held indefinitely, and at some point `MaxPoolSize` (default 100 for SQL Server) is reached and new requests throw `InvalidOperationException: Timeout expired ... waiting for an available connection`.

This is why `CancellationToken` matters: a canceled request without a token still pins the connection until the SQL query completes. Properly threaded cancellation releases connections immediately on client disconnect.

**Real project**: An Azure SQL service hit pool exhaustion every Black Friday until we audited every async EF call for `CancellationToken` and added pool-usage telemetry.

### 7. How would you use EF Core in a Blazor Server app?

**Answer**: Blazor Server keeps a single circuit per user, but a circuit hosts many components, each with its own lifecycle and rendering thread. Injecting a Scoped `DbContext` shares it across components, which causes concurrency exceptions whenever two components render queries at the same time.

The correct pattern is `AddDbContextFactory<T>` and a per-component context created in `OnInitializedAsync` and disposed in `DisposeAsync`:

```csharp
[Inject] public IDbContextFactory<SalesDbContext> Factory { get; set; } = null!;
private SalesDbContext _db = null!;

protected override async Task OnInitializedAsync() =>
    _db = await Factory.CreateDbContextAsync();

public async ValueTask DisposeAsync() => await _db.DisposeAsync();
```

For very short queries, an even simpler pattern is `await using var db = await Factory.CreateDbContextAsync();` inside the method that needs it.

### 8. How would you detect and prevent captive dependencies before they reach production?

**Answer**:

1. **`ValidateScopes = true` and `ValidateOnBuild = true`** in `Program.cs`. The first one throws when a Scoped service is captured by a Singleton; the second one resolves every registered service at startup, so missing registrations fail fast.
2. **A DI integration test** that builds the real container and resolves every controller, hosted service, and handler via a scoped provider. Run it in CI on every commit.
3. **A Roslyn analyzer or architecture test** (e.g., NetArchTest) that forbids `DbContext` as a constructor parameter of any class registered as Singleton.
4. **Connection-pool telemetry** in Application Insights — a leak shows up as `ConnectionAvailableCount` trending down over hours.

**Trade-off**: Enabling validation in production adds a few milliseconds to startup but pays for itself the first time it catches a regression.

**Real project**: We added a 30-line xUnit test that walks every type implementing `IController` and resolves it from a scoped provider. It has caught three captive-dependency PRs in 18 months — each would have been a production incident.

## Summary Checklist

- [ ] I can explain why `DbContext` is Scoped by default in ASP.NET Core.
- [ ] I can describe the symptoms of a captive `DbContext` (Singleton holding Scoped).
- [ ] I can choose between `AddDbContext`, `AddDbContextPool`, and `AddDbContextFactory` for a given scenario.
- [ ] I can refactor a `BackgroundService` to use `IDbContextFactory<T>` correctly.
- [ ] I can explain why two parallel queries on one context throw and how to fix it.
- [ ] I can use `DbContext` correctly in Blazor Server via the factory pattern.
- [ ] I can enable `ValidateScopes` and `ValidateOnBuild` and write a DI smoke test.
- [ ] I can connect connection-pool exhaustion to leaked or captive contexts.
- [ ] I can write a pooled-context-safe `SalesDbContext` (no mutable fields, options-only constructor).
- [ ] I can audit a codebase for `new SalesDbContext()` calls and replace them with DI / factories.
