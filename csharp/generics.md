# Generics

## What It Is

Generics are C#'s mechanism for writing code that operates on a **type parameter** (`T`) rather than a concrete type, while still being **type-safe** at compile time. The CLR specializes the implementation per value-type argument (`List<int>` and `List<long>` are distinct machine code) and shares a single implementation across reference-type arguments — there is no boxing and no runtime cast.

A generic type or method declares one or more parameters in angle brackets and may add **constraints** that restrict what types are accepted (`where T : class`, `where T : IAggregateRoot, new()`).

```csharp
// Generic interface used across an order/payment/invoice codebase
public interface IRepository<TEntity, TId>
    where TEntity : class, IAggregateRoot
    where TId    : struct
{
    Task<TEntity?> GetAsync(TId id, CancellationToken ct);
    Task AddAsync(TEntity entity, CancellationToken ct);
}

// Specialized implementations — same shape, different aggregate
public sealed class OrderRepository   : EfRepository<Order,   Guid> { /* ... */ }
public sealed class InvoiceRepository : EfRepository<Invoice, Guid> { /* ... */ }
```

Generics also power most of the BCL (`List<T>`, `Dictionary<TKey, TValue>`, `Task<T>`, `IEnumerable<T>`, `IOptions<T>`, `ILogger<T>`).

## Why It Exists

Before C# 2.0 / .NET 2.0 (2005), collections were untyped (`System.Collections.ArrayList`, `Hashtable`). Working with them meant:

```csharp
ArrayList orders = new ArrayList();
orders.Add(new Order(...));      // boxing if value type
Order o = (Order)orders[0];      // runtime cast — InvalidCastException if wrong
```

Three real problems:

1. **No compile-time safety** — you could `Add` anything; mistakes surfaced at runtime.
2. **Boxing / unboxing** — value types were heap-allocated to fit `object`, killing performance.
3. **Hand-rolled "generic" code** — every team wrote a `StronglyTypedCollection<T>` by code-gen, T4 templates, or copy-paste.

Generics were added to put **type parameters into the type system itself**, so `List<Order>.Add(customer)` is a compile error and `List<int>` stores raw ints without boxing. They also enabled rich abstractions (`Task<T>`, `Func<T,TResult>`, `IObservable<T>`) that would be unusable with `object`.

## When To Use It

**Use generics when:**

- You have a shape that genuinely applies to multiple types — collections, results, pipelines, validators, handlers.
- You need to preserve type safety across a layer: `IValidator<TRequest>`, `IRequestHandler<TRequest, TResponse>` (MediatR), `IOptions<TOptions>`.
- You want to avoid boxing for value types in performance-critical code (numeric algorithms, caching layers).
- You are designing infrastructure: repositories per aggregate, message handlers per event type, factories per product.

**Do NOT use generics when:**

- The "shared" code actually represents distinct business operations (`Process<T>` for `Order`, `Refund`, `Shipment` — three different workflows; give each its own name).
- You only have one concrete type in mind — generics for "future flexibility" usually create premature abstraction.
- The constraints would have to be so broad (`where T : class`) that the generic body can do nothing useful.
- Inheritance or composition models the relationship better than parametric polymorphism.

## Why It Is Important

Generics are how modern .NET avoids both **duplication** and **boxing** — two problems that dominate large codebases without them. A few production properties they directly enable:

- **`HttpClient` + typed clients**: `IHttpClientFactory.CreateClient<TClient>()` registers per-client policies.
- **`IOptions<TOptions>`**: strongly-typed configuration binding with validation (`ValidateOnStart`).
- **`ILogger<T>`**: the `T` becomes the log source category, automatically surfaced in App Insights' `customDimensions.SourceContext`.
- **EF Core `DbSet<TEntity>`**: each entity gets a typed query root without runtime casts.
- **MediatR / MassTransit**: `IRequestHandler<TRequest, TResponse>`, `IConsumer<TMessage>` — handlers are discovered by their generic interface and invoked safely.
- **`Result<T>` / `Option<T>`**: explicit success/failure shapes without a base class hierarchy.
- **JSON serialization**: `JsonSerializer.Deserialize<T>(stream, ct)` returns the right type.

Without generics, every one of these would either require object-typed APIs with casts or per-type code generation.

## How It's Used in C# / .NET

| Feature | Example |
| --- | --- |
| **Generic class** | `public sealed class PagedResult<T> { ... }` |
| **Generic interface** | `public interface IValidator<T> { Task<ValidationResult> ValidateAsync(T, CancellationToken); }` |
| **Generic method** | `public T Deserialize<T>(string json) ...` |
| **Type constraints** | `where T : class`, `where T : struct`, `where T : new()`, `where T : IAggregateRoot`, `where T : notnull`, `where T : unmanaged` |
| **Multiple constraints** | `where TEntity : class, IAggregateRoot, new()` |
| **`default(T)`** | Safe default value (`null` for ref, `0` / `default` for value). |
| **Open vs closed generic** | `IRepository<,>` (open, for DI) vs `IRepository<Order, Guid>` (closed). |
| **Covariance / contravariance** | `IEnumerable<out T>`, `Action<in T>`, `IComparer<in T>`. |
| **Generic math (.NET 7+)** | `static T Sum<T>(IEnumerable<T> items) where T : INumber<T>` |
| **`Type.MakeGenericType`** | Build closed generic at runtime (rare, used by frameworks). |

ASP.NET Core DI supports **open generic registration**:

```csharp
// Register the generic shape once; the container closes it on demand
services.AddScoped(typeof(IRepository<,>),  typeof(EfRepository<,>));
services.AddScoped(typeof(IValidator<>),     typeof(FluentValidator<>));

// Anywhere — the container synthesizes IRepository<Order, Guid>
public OrdersController(IRepository<Order, Guid> orders) { ... }
```

`IOptions<T>` pattern:

```csharp
public sealed class StripeOptions
{
    [Required] public string ApiKey { get; init; } = default!;
    [Url]      public string Endpoint { get; init; } = default!;
}

builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

public sealed class StripeClient
{
    public StripeClient(IOptions<StripeOptions> opts) { _opts = opts.Value; }
}
```

## Advantages

- **Compile-time type safety** — no runtime casts, no `InvalidCastException`.
- **No boxing for value types** — `List<int>` stores raw ints; performance-critical paths stay allocation-free.
- **Reusable infrastructure** with no per-type duplication (repositories, handlers, validators).
- **First-class support across the BCL and ecosystem** — collections, async, LINQ, EF Core, DI, options.
- **Open-generic DI** registers the abstraction once for all closed types.

## Disadvantages

- **Cognitive load** — `IRequestHandler<TRequest, TResponse> where TRequest : IRequest<TResponse>` is intimidating to juniors.
- **Constraint explosion** — multiple constraints (`where T : class, IAggregateRoot, new()`) make signatures noisy.
- **Hides domain meaning** — a `Repository<T>.GetAll()` does the same thing for `Order` and `Invoice`, but those are very different real-world queries.
- **Specialization cost** — every value-type closed generic gets its own JITted code (small per-type CPU + memory cost).
- **Reflection over generics is awkward** — `MakeGenericType` is needed to construct closed generics at runtime.

## Common Mistakes

### 1. Forcing a generic repository onto every aggregate

```csharp
// WRONG — hides the real queries, encourages anti-patterns
public interface IRepository<T> where T : class
{
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct); // loads ALL rows
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
}
```

```csharp
// RIGHT — repositories that express domain intent
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetPendingForCustomerAsync(Guid customerId, CancellationToken ct);
    Task<int> CountOverdueAsync(DateTime asOf, CancellationToken ct);
}
```

### 2. Using `object` instead of `T`

```csharp
// WRONG — runtime cast, no compile-time safety
public sealed class Cache
{
    private readonly ConcurrentDictionary<string, object> _items = new();
    public object Get(string key) => _items[key];
}

var order = (Order)cache.Get("order:42"); // can throw InvalidCastException
```

```csharp
// RIGHT — generic API preserves the type
public sealed class Cache<T>
{
    private readonly ConcurrentDictionary<string, T> _items = new();
    public T Get(string key) => _items[key];
}
```

### 3. Meaningless parameter names

```csharp
// WRONG — what is X? What is Y?
public interface IHandler<X, Y> { Task<Y> Handle(X x, CancellationToken ct); }
```

```csharp
// RIGHT — describe roles
public interface IRequestHandler<TRequest, TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken ct);
}
```

### 4. Constraints added "just in case"

```csharp
// WRONG — class and new() are not used inside the body; they restrict callers for no reason
public sealed class Wrapper<T> where T : class, new()
{
    public T? Value { get; set; }
}
```

```csharp
// RIGHT — only the constraints you actually need
public sealed class Wrapper<T> where T : notnull
{
    public T Value { get; init; } = default!;
}
```

### 5. Mixing variance directions

```csharp
// WRONG — IEnumerable<Order> cannot be assigned to IEnumerable<object>... actually it can.
//         But this signature breaks variance because T is used in both in/out positions.
public interface IBox<T> { T Get(); void Set(T value); }
// IBox<Order> is NOT IBox<IAggregateRoot>
```

```csharp
// RIGHT — split into covariant and contravariant interfaces if you need both
public interface IReader<out T> { T Get(); }    // covariant: IReader<Order> -> IReader<IAggregateRoot>
public interface IWriter<in T>  { void Set(T value); } // contravariant
```

### 6. Closing generics manually in DI

```csharp
// WRONG — must remember every type
services.AddScoped<IRepository<Order, Guid>, EfRepository<Order, Guid>>();
services.AddScoped<IRepository<Invoice, Guid>, EfRepository<Invoice, Guid>>();
services.AddScoped<IRepository<Payment, Guid>, EfRepository<Payment, Guid>>();
// ... 30 more
```

```csharp
// RIGHT — register the open generic once
services.AddScoped(typeof(IRepository<,>), typeof(EfRepository<,>));
```

### 7. Using `default(T)` and not handling null

```csharp
// WRONG — default(T) for a class is null; downstream NRE
public T GetOrDefault<T>(string key)
{
    return _items.TryGetValue(key, out var v) ? (T)v : default(T);
}
```

```csharp
// RIGHT — express nullability in the signature
public T? GetOrDefault<T>(string key) where T : class
    => _items.TryGetValue(key, out var v) ? (T)v : null;

public bool TryGet<T>(string key, [NotNullWhen(true)] out T? value) where T : class
{
    if (_items.TryGetValue(key, out var v)) { value = (T)v; return true; }
    value = null; return false;
}
```

## Best Practices

- **Name parameters descriptively** — `TEntity`, `TKey`, `TRequest`, `TResponse`, `TMessage` — not `T1`, `X`, `Y`.
- **Constrain only what the body needs**. Every constraint is a contract burden on callers.
- **Prefer open-generic DI registration** for cross-cutting infrastructure (`IValidator<>`, `IRequestHandler<,>`, `IRepository<,>`).
- **Split covariance and contravariance into separate interfaces** (`IReader<out T>` / `IWriter<in T>`) when both are needed.
- **Don't generic-ify domain behavior** — `IOrderRepository` is clearer than `IRepository<Order>` if it has order-specific methods.
- **Use `INumber<T>` (NET 7+)** for genuine math abstractions; otherwise stick to concrete numeric types.
- **Reach for MediatR / MassTransit generics** rather than building your own dispatcher framework.
- **Test the closed generic**, not the open shape — write tests for `Repository<Order>` behavior, not for the generic `EfRepository<TEntity>`.
- **Avoid generic methods on non-generic types when overloads work** — `void Log<T>(T value)` vs `void Log(string)` / `void Log(int)`.
- **Be explicit about nullable type parameters** with `T?` and `[NotNullWhen]` attributes.

## Related Concepts

- [csharp/interfaces-and-abstractions.md](csharp/interfaces-and-abstractions.md) — generics on interfaces, variance.
- [csharp/dependency-injection.md](csharp/dependency-injection.md) — open-generic registration, `ILogger<T>`, `IOptions<T>`.
- [csharp/nullable-reference-types.md](csharp/nullable-reference-types.md) — `T?` and `where T : notnull`.
- [data-access/repository-pattern-with-ef-core.md](data-access/repository-pattern-with-ef-core.md) — when generic vs domain-specific repos are right.
- [architecture/cqrs.md](architecture/cqrs.md) — MediatR `IRequestHandler<TRequest, TResponse>`.
- [aspnet-core/input-validation.md](aspnet-core/input-validation.md) — FluentValidation `IValidator<T>`.

## Real-World Usage

### ASP.NET Core APIs

`ILogger<T>` is injected into every controller — the generic parameter becomes the log source category (`SourceContext`) used for filtering in App Insights. `IOptions<TOptions>` binds strongly-typed configuration sections. Typed `HttpClient` clients via `IHttpClientFactory.CreateClient<TClient>()` attach per-client Polly policies.

### MediatR

The standard CQRS dispatcher in .NET: `IRequest<TResponse>` marks a request, `IRequestHandler<TRequest, TResponse>` handles it. The DI container resolves the right handler at runtime — adding a new command means writing one handler class, no wiring change.

### EF Core

`DbSet<TEntity>` is generic so `_db.Orders.Where(...)` returns `IQueryable<Order>` — full type safety through LINQ. Generic configuration via `IEntityTypeConfiguration<TEntity>`.

### Azure Service Bus + MassTransit

`IConsumer<TMessage>` defines a typed handler per message. The MassTransit pipeline routes messages to the right consumer by closed generic. Retry/redelivery policies are configured per message type via `e.UseMessageRetry(r => r.Interval(...))`.

### Testing

xUnit's `Theory` + `InlineData<T>` is generic. Builders like `AutoFixture`'s `fixture.Create<T>()` return strongly-typed test data. Moq's `Mock<T>` and FakeItEasy's `A.Fake<T>()` are both generic.

### Application Insights

`ITelemetryClient`-based logging is non-generic, but `ILogger<MyService>` flows the source category automatically. Custom `ITelemetryInitializer` implementations often use generic helpers.

## Code Example — Before and After

### Before — duplicated wrappers per type, runtime casts, no DI elegance

```csharp
public sealed class OrderResult
{
    public bool Succeeded { get; init; }
    public Order? Value { get; init; }
    public string? Error { get; init; }
}

public sealed class InvoiceResult
{
    public bool Succeeded { get; init; }
    public Invoice? Value { get; init; }
    public string? Error { get; init; }
}

public sealed class PaymentResult
{
    public bool Succeeded { get; init; }
    public Payment? Value { get; init; }
    public string? Error { get; init; }
}

// ... times every DTO. Repository registrations explode similarly.
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IInvoiceRepository, InvoiceRepository>();
services.AddScoped<IPaymentRepository, PaymentRepository>();
// Each repo class duplicates GetById/Add/SaveChanges logic.
```

### After — one generic shape, one open-generic registration, domain-specific methods where they matter

```csharp
// Reusable result wrapper
public sealed record Result<T>(bool Succeeded, T? Value, string? Error)
{
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}

// Marker for aggregates that can live in the generic repo
public interface IAggregateRoot { }

public abstract class Entity<TId> where TId : struct
{
    public TId Id { get; protected set; }
}

public sealed class Order : Entity<Guid>, IAggregateRoot { /* ... */ }
public sealed class Invoice : Entity<Guid>, IAggregateRoot { /* ... */ }
public sealed class Payment : Entity<Guid>, IAggregateRoot { /* ... */ }

// Generic shape covers the boring 80%
public interface IRepository<TEntity, TId>
    where TEntity : Entity<TId>, IAggregateRoot
    where TId : struct
{
    Task<TEntity?> GetAsync(TId id, CancellationToken ct);
    Task AddAsync(TEntity entity, CancellationToken ct);
    Task<int> SaveAsync(CancellationToken ct);
}

public sealed class EfRepository<TEntity, TId> : IRepository<TEntity, TId>
    where TEntity : Entity<TId>, IAggregateRoot
    where TId : struct
{
    private readonly DbContext _db;
    public EfRepository(DbContext db) => _db = db;

    public Task<TEntity?> GetAsync(TId id, CancellationToken ct)
        => _db.Set<TEntity>().FindAsync([id], ct).AsTask();

    public async Task AddAsync(TEntity entity, CancellationToken ct)
        => await _db.Set<TEntity>().AddAsync(entity, ct);

    public Task<int> SaveAsync(CancellationToken ct)
        => _db.SaveChangesAsync(ct);
}

// Domain-specific repository for the parts that aren't boilerplate
public interface IOrderRepository : IRepository<Order, Guid>
{
    Task<IReadOnlyList<Order>> GetPendingForCustomerAsync(Guid customerId, CancellationToken ct);
    Task<int> CountOverdueAsync(DateTime asOf, CancellationToken ct);
}

public sealed class OrderRepository : EfRepository<Order, Guid>, IOrderRepository
{
    private readonly OrderingDbContext _db;
    public OrderRepository(OrderingDbContext db) : base(db) { _db = db; }

    public Task<IReadOnlyList<Order>> GetPendingForCustomerAsync(Guid customerId, CancellationToken ct)
        => _db.Orders.Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Pending)
                     .ToListAsync(ct)
                     .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);

    public Task<int> CountOverdueAsync(DateTime asOf, CancellationToken ct)
        => _db.Orders.CountAsync(o => o.DueDate < asOf && o.Status != OrderStatus.Paid, ct);
}

// One DI registration covers Order, Invoice, Payment, and anything added later
services.AddScoped(typeof(IRepository<,>), typeof(EfRepository<,>));
services.AddScoped<IOrderRepository, OrderRepository>();
```

The after version uses generics where it removes duplication (`Result<T>`, generic `IRepository<,>` for CRUD), and uses domain-specific interfaces where the queries are genuinely different (`IOrderRepository`). That's the right balance — generics for shape, named methods for intent.

## Interview Questions and Answers

### 1. Why are generics preferred over `object`-based APIs in .NET?

**Why this matters:** Tests understanding of the core motivation.

**Answer:** Three reasons. First, **type safety** — the compiler stops `orders.Add(customer)` instead of failing at runtime with `InvalidCastException`. Second, **no boxing** for value types — `List<int>` stores raw 32-bit ints, while `ArrayList` boxes each one into a heap-allocated `object`. Third, **API clarity** — `Task<Order>` tells the reader exactly what comes back; `Task<object>` requires a cast everywhere and hides intent.

**Trade-off:** Generics add a small amount of cognitive overhead for newcomers (`IRequestHandler<TRequest, TResponse> where TRequest : IRequest<TResponse>`), but the gains in safety and performance are decisive.

**Real project:** A legacy service stored cached items as `Dictionary<string, object>` and cast on retrieval. After a refactor renamed a DTO, the cast started failing at runtime in production — a generic `Cache<T>` would have surfaced it at compile time.

### 2. When is a generic repository the wrong choice?

**Why this matters:** Probes whether the candidate sees the abstraction's downsides.

**Answer:** It's wrong when (a) most aggregates need very different queries (`GetActiveSubscriptionsExpiringIn7Days` vs `GetOrdersWithOverdueInvoices` — there's no shared abstraction worth extracting), (b) it encourages anti-patterns like `GetAll()` which loads everything, (c) it hides EF Core's full query surface (`Include`, `AsSplitQuery`, `AsNoTracking`), or (d) the team uses it to bypass thinking about domain modeling. A good rule: generic for raw CRUD plumbing, domain-specific interface for actual business queries.

**Trade-off:** A generic base + per-aggregate sub-interface (as in the example above) often gives both wins.

**Real project:** An `IRepository<T>` exposed `GetAll()` for "convenience." Three years later, the `Orders` table had 80 million rows and `GetAll()` calls were OOM-ing pods on every startup. Removing the method forced the team to write the right query.

### 3. What's the difference between an open generic and a closed generic?

**Why this matters:** Common confusion; relevant to DI registration.

**Answer:** An **open generic** is the unspecialized shape: `List<>`, `IRepository<,>`. You cannot create an instance — `typeof(List<>)` returns the open type metadata, useful mostly for reflection and DI registration. A **closed generic** has all type parameters filled in: `List<Order>`, `IRepository<Order, Guid>`. You can instantiate it and call its members. ASP.NET Core DI's `services.AddScoped(typeof(IRepository<,>), typeof(EfRepository<,>))` registers the open generic; when something requests `IRepository<Order, Guid>`, the container synthesizes the closed form.

**Trade-off:** Open-generic registration is more flexible but requires that the implementation truly works for all closed forms — if `EfRepository<TEntity, TId>` assumes `TId == Guid`, it's a bug waiting to happen.

**Real project:** Adding a new aggregate used to mean adding three DI registrations. Switching to open-generic registration removed ~50 lines from `Program.cs` and made onboarding obvious.

### 4. What is covariance vs contravariance? Give a backend example.

**Why this matters:** Often misunderstood, but it appears whenever you assign `IEnumerable<DerivedDto>` to `IEnumerable<BaseDto>`.

**Answer:** **Covariance** (`out T`) lets you use a more derived type where a less derived one is expected — `IEnumerable<out T>` means `IEnumerable<Order>` is an `IEnumerable<IAggregateRoot>`. **Contravariance** (`in T`) lets you use a less derived type where a more derived one is expected — `IComparer<in T>` means an `IComparer<IAggregateRoot>` can be used as `IComparer<Order>`. Variance is only allowed on **interfaces and delegates**, not on classes, and `T` must appear only in the matching position (output for `out`, input for `in`).

**Trade-off:** Variance makes generic APIs more flexible but constrains how `T` can be used inside the interface.

**Real project:** A reporting service exposed `IReadOnlyList<OrderSummary>`. Because `IReadOnlyList<T>` is covariant, the same list could be consumed by code expecting `IReadOnlyList<ITransactionSummary>` — no LINQ projection or casts needed.

### 5. Why does `ILogger<T>` take a generic parameter when it doesn't seem to do anything with it?

**Why this matters:** Tests understanding of why .NET frameworks use generics for **metadata**, not just behavior.

**Answer:** The `T` is used as the **log category name**. When `ILogger<OrderService>` is resolved, the logger's category is `OrderService`'s full type name. This drives filtering rules (`"LogLevel": { "MyApp.OrderService": "Debug" }`), Application Insights' `customDimensions.SourceContext`, and structured queries (`traces | where customDimensions.SourceContext == "MyApp.OrderService"`). Without the generic, every class would have to pass its name as a string — tedious and refactor-unsafe.

**Trade-off:** It means moving `OrderService` to a new namespace can change the logger category, which can break log-search bookmarks. Most teams accept this.

**Real project:** A team had used a single `ILoggerFactory.CreateLogger("App")` everywhere. Migrating to `ILogger<T>` immediately made App Insights queries much more useful — no more `where message contains "OrderService"` hacks.

### 6. How does open-generic DI registration work, and when would it not?

**Why this matters:** Tests practical DI knowledge.

**Answer:** `services.AddScoped(typeof(IValidator<>), typeof(FluentValidator<>))` tells the container: "for any closed `IValidator<TFoo>`, synthesize `FluentValidator<TFoo>`." It works when the implementation is genuinely generic. It fails when the implementation has different per-type behavior — e.g., `OrderValidator` needs to call an `IOrderRepository`. In that case, register concrete validators per type (`services.AddScoped<IValidator<PlaceOrderRequest>, PlaceOrderRequestValidator>()`), or use scanning (`Scrutor`'s `Scan(...)`) to register them in bulk.

**Trade-off:** Open-generic registration is concise but limits where you can put per-type logic. Scanning + per-type registrations is more verbose but more honest.

**Real project:** We started with open-generic for FluentValidation, but every request needed custom validation — the "generic" registration was unused. Switching to `Scan(...).FromAssemblyOf<X>().AddClasses(c => c.AssignableTo(typeof(IValidator<>)))` registered all 40 validators with one line and let each be unique.

### 7. What does `where T : new()` mean and when is it useful?

**Why this matters:** Constraint use is a small but telling detail.

**Answer:** `where T : new()` requires `T` to have a **public parameterless constructor** — the generic method can do `new T()` inside. It's useful for factories (`var instance = new T()`) but in modern .NET it's increasingly replaced by DI (`IServiceProvider.GetRequiredService<T>()`) or by activator helpers. Using it forces every caller's type to expose a parameterless constructor, which is often a poor fit for domain entities that require validated creation (`Order.Create(customerId, items)`).

**Trade-off:** `new()` is simple but pushes a structural requirement onto unrelated types. For test fixtures and DTOs it's fine; for domain models, prefer factory methods.

**Real project:** A generic test builder used `where T : new()`. Adding a domain entity with no parameterless constructor (deliberately, to enforce invariants) broke the build. We replaced `new T()` with an `IFactory<T>` injected via DI and removed the constraint.

### 8. .NET 7 added "generic math" with `INumber<T>`. What problem does that solve?

**Why this matters:** Shows awareness of the modern surface.

**Answer:** Before .NET 7, you couldn't write `static T Sum<T>(IEnumerable<T> values) where T : ...` because there was no constraint that captured "supports `+` and `0`." You had to overload for `int`, `long`, `decimal`, `double` separately, or use `dynamic` (slow + unsafe). `INumber<T>` and its siblings (`IAdditionOperators<T,T,T>`, `INumberBase<T>`) let you write **one** generic implementation that works for every numeric type. It's especially useful for math-heavy libraries, statistics, and SIMD pipelines.

**Trade-off:** It's invasive to use in business code — most CRUD APIs deal with `decimal` for money and don't need it. Reach for it in genuine numeric infrastructure.

**Real project:** A pricing-engine library exposed separate `CalculateAverage(IEnumerable<decimal>)` and `CalculateAverage(IEnumerable<double>)`. After .NET 7 we collapsed them into `static T CalculateAverage<T>(IEnumerable<T>) where T : INumber<T>` — half the code, same behavior.

## Summary Checklist

- [ ] I can explain what a type parameter is and why it gives compile-time safety.
- [ ] I use descriptive parameter names (`TRequest`, `TEntity`) — not `T1`, `X`, `Y`.
- [ ] I add **only** the constraints my generic body actually needs.
- [ ] I know the difference between open (`IRepository<,>`) and closed (`IRepository<Order, Guid>`) generics.
- [ ] I register cross-cutting infrastructure with **open-generic DI** (`AddScoped(typeof(IRepository<,>), ...)`).
- [ ] I recognize when a generic repository is the right call — and when domain-specific interfaces are clearer.
- [ ] I can explain covariance (`out T`) and contravariance (`in T`) and where they appear in the BCL.
- [ ] I use `ILogger<T>`, `IOptions<T>`, `IRequestHandler<,>`, `IValidator<>` idiomatically.
- [ ] I use `T?` and nullable annotations on generic returns, not bare `default(T)`.
- [ ] I avoid hiding domain meaning behind vague generic abstractions.
