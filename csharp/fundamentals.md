# C# Fundamentals for Backend Development

## What It Is

C# fundamentals are the everyday building blocks every backend engineer touches before any framework or architectural pattern enters the picture: the **type system** (value vs reference types, primitives, `decimal`, `Guid`, `DateTimeOffset`), **nullability**, **collections** (`List<T>`, `Dictionary<TKey,TValue>`, `IReadOnlyList<T>`), **access modifiers** (`public`/`internal`/`private`/`protected`/`file`), **control flow and pattern matching**, **resource management with `using`**, and a working grasp of **records** as concise data carriers. They look small, but they decide whether a payment is off by a cent, whether a customer id can collide, whether a timestamp is wrong by seven hours, and whether an `Order` can be put into an impossible state.

```csharp
// Sloppy fundamentals — three production bugs hiding in four lines
public class Order
{
    public double Total;                // 0.1 + 0.2 != 0.3 — wrong for money
    public int Id;                      // collides under load — should be Guid
    public DateTime CreatedAt;          // ambiguous time zone — should be DateTimeOffset
}

// Solid fundamentals
public sealed class Order
{
    public required Guid Id { get; init; }
    public required decimal Total { get; init; }
    public required DateTimeOffset CreatedAt { get; init; }
}
```

## Why It Exists

Backends process money, identities, time, and concurrent state. Languages without a careful type system (PHP arrays, dynamic JavaScript, untyped Python) leave every one of those concerns up to the developer's discipline. C# was designed so the **compiler** rejects the worst categories of bugs:

- A `decimal` exists because `double` cannot represent `0.10` exactly — using `double` for money produces cents that disappear at scale.
- `Guid` exists because auto-increment `int` ids collide across services, leak record counts to attackers, and break sharding.
- `DateTimeOffset` exists because `DateTime` carries no time-zone information, leading to "the report is off by 7 hours" tickets every quarter.
- Nullable reference types exist because `NullReferenceException` was the single most common runtime error in pre-C# 8 codebases.
- `using` statements exist because forgetting to dispose a `SqlConnection` exhausts the connection pool and brings production down.

Fundamentals exist so the compiler and the standard library do the work that would otherwise live in code reviews and incident retrospectives.

## When To Use It

You always use the fundamentals — there is no "skip" option. The real question is *which* primitive to use:

**Use for:**

- **Money and any exact arithmetic** → `decimal` (never `double`/`float`).
- **Public identifiers** → `Guid` (or `Ulid`/sequential GUIDs for index locality).
- **All timestamps** → `DateTimeOffset` (or `DateOnly`/`TimeOnly` for calendar/wall-clock values).
- **Collections crossing API boundaries** → expose `IReadOnlyList<T>`, `IReadOnlyDictionary<,>`, or `IEnumerable<T>`.
- **Resource holders** (DB connections, HTTP responses, streams) → wrap with `using`/`await using`.
- **Branching on shape** → pattern matching (`switch` expressions, property patterns).

**Do not use for:**

- `double` for currency, tax, or anything billable.
- `int` for primary keys in distributed systems.
- `DateTime.Now` anywhere — it returns local time and breaks across regions.
- `string` for known-shape values like emails, ISO country codes, currency codes — wrap them in value objects.
- `public` fields — always use properties.
- `var` when the right-hand side does not make the type obvious (`var x = GetThing();` hides intent).

## Why It Is Important

These choices compound across thousands of lines and dozens of services. A poorly-chosen `double` for `OrderLine.Price` does not crash — it silently rounds, and three years later finance discovers the company has under-charged $400,000. A missing `using` on `HttpClient` leaks sockets until the App Service hits the SNAT port limit and every outbound call times out. A `DateTime.Now` in a containerized worker on UTC infrastructure makes every scheduled job fire seven hours late.

In cloud systems (Azure App Service, Functions, AKS, Container Apps) you cannot rely on a single machine's clock, locale, or hostname. The fundamentals are the contract between your code and the chaos of distributed execution. Good fundamentals are also the cheapest form of reliability engineering — they cost nothing at runtime but eliminate whole classes of incidents.

## How It's Used in C# / .NET

### 1. Value types vs reference types

```csharp
// Value types — copied on assignment, live on the stack (typically), cannot be null without ?
int count = 5;
decimal price = 19.99m;
DateTimeOffset now = DateTimeOffset.UtcNow;
Guid id = Guid.NewGuid();
(string Code, decimal Amount) money = ("USD", 49.99m);

// Reference types — variable holds a pointer to the heap, default is null (or annotated as not-null)
string email = "user@contoso.com";
Order order = new();
List<OrderLine> lines = new();
```

Pass a `record struct` for hot-path value types (e.g., `Money`, `Coordinate`) to avoid heap allocations. Use `readonly struct` to make value types immutable.

### 2. Money: always `decimal`

```csharp
public readonly record struct Money(decimal Amount, string CurrencyCode)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.CurrencyCode != b.CurrencyCode)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(a.Amount + b.Amount, a.CurrencyCode);
    }
}

// EF Core mapping
modelBuilder.Entity<Order>()
    .Property(o => o.Total)
    .HasPrecision(18, 2);   // explicit precision/scale for the SQL column
```

### 3. Identifiers: `Guid` over `int`

```csharp
// Sequential GUIDs (good index locality on SQL Server)
public static class Ids
{
    public static Guid NewSequential() => Guid.CreateVersion7(); // .NET 9+

    // Pre-.NET 9: NewSequentialId via System.Data or a UUIDv7 polyfill
}

public sealed class Customer
{
    public required Guid Id { get; init; } = Ids.NewSequential();
}
```

### 4. Time: `DateTimeOffset` and `TimeProvider`

```csharp
public sealed class InvoiceService(TimeProvider time, IInvoiceRepository repo)
{
    public async Task IssueAsync(Invoice invoice, CancellationToken ct)
    {
        invoice.IssuedAt = time.GetUtcNow();          // testable
        invoice.DueAt    = invoice.IssuedAt.AddDays(30);
        await repo.SaveAsync(invoice, ct);
    }
}

// Registration
builder.Services.AddSingleton(TimeProvider.System);
```

Use `DateOnly` for birthdays and invoice dates; `TimeOnly` for business-hour cutoffs; `DateTimeOffset` for everything else.

### 5. Collections — pick the right contract

| Type                              | When to use                                                  |
|-----------------------------------|--------------------------------------------------------------|
| `List<T>`                         | Internal storage when you need `Add`, `Remove`, indexing      |
| `T[]`                             | Fixed-size, hot-path, interop                                 |
| `Dictionary<TKey, TValue>`        | Lookup by key inside a single thread/scope                    |
| `ConcurrentDictionary<TKey, TValue>` | Lookup by key from multiple threads (caches in singletons) |
| `HashSet<T>`                      | Membership tests, deduping                                    |
| `IReadOnlyList<T>` / `IReadOnlyDictionary<,>` | **Public** API surfaces — prevents callers from mutating |
| `IEnumerable<T>`                  | Lazy/streamed sequences, LINQ pipelines                       |
| `ImmutableArray<T>` / `ImmutableList<T>` | Shared state in singletons, snapshot semantics         |
| `IAsyncEnumerable<T>`             | Async streaming (Cosmos DB pages, file lines)                 |

```csharp
public IReadOnlyList<OrderLine> GetLines() => _lines.AsReadOnly();
```

### 6. Access modifiers

```csharp
public sealed class StripePaymentGateway : IPaymentGateway   // public surface
{
    private readonly HttpClient _http;                       // private implementation
    internal const string ApiVersion = "2024-06-01";         // visible to the assembly
    protected virtual TimeSpan Timeout => TimeSpan.FromSeconds(30); // for inheritors
}

file static class StripeJsonOptions { /* visible only in this file */ }
```

Default to `internal sealed` for implementations. Only mark a class `public` when it crosses an assembly boundary.

### 7. Pattern matching

```csharp
public decimal ShippingFor(Order order) => order switch
{
    { Total: > 100m }                       => 0m,
    { CustomerTier: Tier.Premium }          => 0m,
    { Destination.Country: "US" or "CA" }   => 9.99m,
    { Destination.Country: "GB" or "FR" }   => 14.99m,
    _                                       => 24.99m
};

if (response is { StatusCode: 429, Headers.RetryAfter: { Delta: { } delta } })
{
    await Task.Delay(delta, ct);
}
```

Use property patterns to flatten deeply-nested checks; switch expressions to replace `if/else` ladders that return a value.

### 8. Resource management: `using` and `await using`

```csharp
public async Task<byte[]> DownloadAsync(Uri uri, CancellationToken ct)
{
    using var response = await _http.GetAsync(uri, ct);     // disposes HttpResponseMessage
    response.EnsureSuccessStatusCode();
    await using var stream = await response.Content.ReadAsStreamAsync(ct); // IAsyncDisposable
    using var ms = new MemoryStream();
    await stream.CopyToAsync(ms, ct);
    return ms.ToArray();
}
```

`using` declarations (no `{}`) dispose at the end of the enclosing scope. Use `await using` for `IAsyncDisposable` (most modern Azure SDK clients, EF `DbContext` since .NET 6).

### 9. Records as data carriers

```csharp
public sealed record CreateOrderRequest(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines,
    string CouponCode);

public sealed record OrderLineDto(Guid ProductId, int Quantity);
```

See [csharp/records-and-immutability.md](records-and-immutability.md) for the full treatment.

### 10. `string` essentials

```csharp
// Always specify culture for parsing/formatting numbers and dates
decimal amount = decimal.Parse(input, CultureInfo.InvariantCulture);
string log = $"{amount.ToString("F2", CultureInfo.InvariantCulture)} {currency}";

// Comparisons — be explicit about culture and case
if (input.Equals("admin", StringComparison.OrdinalIgnoreCase)) { /* ... */ }

// Building strings in a loop — StringBuilder, not +=
var sb = new StringBuilder();
foreach (var line in lines) sb.AppendLine(line.ToString());
```

## Advantages

- **Type safety** — most bugs are caught at compile time, not in production.
- **Performance** — value types, `Span<T>`, `Memory<T>`, and `stackalloc` enable zero-allocation code paths.
- **Tooling** — IDE refactorings, code analyzers, and Roslyn analyzers depend on a precise type system.
- **Interoperability** — `decimal`, `DateTimeOffset`, `Guid` map cleanly to SQL Server, Azure Cosmos DB, JSON.
- **Discoverability** — strong types make `Go to definition`, `Find usages`, and rename-refactors reliable.
- **Self-documenting APIs** — a method that takes `Money` and `DateTimeOffset` is unambiguous.

## Disadvantages

- **Verbosity** — declaring `IReadOnlyList<OrderLine>` everywhere is wordier than `[]` in JavaScript.
- **Learning curve** — value vs reference semantics, boxing, default values, and nullable annotations take time.
- **Primitive obsession risk** — engineers reach for `string`/`int` instead of building small value objects.
- **Type system noise** — over-generic code can become harder to read (`Func<IReadOnlyDictionary<string, IReadOnlyList<T>>, Task<Result<TResponse>>>`).
- **Cultural footguns** — `DateTime`, `string.Format` without culture, and implicit `decimal`-to-`double` conversions still compile.

## Common Mistakes

### 1. Using `double` for money

```csharp
// BUG — floating-point representation error
double total = 0.0;
foreach (var line in lines) total += line.Price; // accumulates rounding error
```

**Fix**: use `decimal`.

```csharp
decimal total = 0m;
foreach (var line in lines) total += line.Price;
```

### 2. `DateTime.Now` in cloud services

```csharp
// BUG — returns local time of the host machine; ambiguous and not testable
order.PlacedAt = DateTime.Now;
```

**Fix**: inject `TimeProvider` and use `GetUtcNow()`.

```csharp
order.PlacedAt = _time.GetUtcNow(); // returns DateTimeOffset in UTC
```

### 3. `int` keys in distributed systems

```csharp
// BUG — concurrent inserts from multiple regions collide; predictable ids leak data
public int Id { get; set; }
```

**Fix**: use `Guid` (ideally UUIDv7/sequential).

```csharp
public Guid Id { get; init; } = Guid.CreateVersion7();
```

### 4. Forgetting `using` on disposables

```csharp
// BUG — connection pool exhaustion under load
public async Task<int> CountAsync()
{
    var conn = new SqlConnection(_cs);
    await conn.OpenAsync();
    return (int)await new SqlCommand("SELECT COUNT(*) FROM Orders", conn).ExecuteScalarAsync();
}
```

**Fix**:

```csharp
public async Task<int> CountAsync(CancellationToken ct)
{
    await using var conn = new SqlConnection(_cs);
    await conn.OpenAsync(ct);
    await using var cmd = new SqlCommand("SELECT COUNT(*) FROM Orders", conn);
    return (int)(await cmd.ExecuteScalarAsync(ct))!;
}
```

### 5. Public mutable collections

```csharp
public List<OrderLine> Lines { get; } = new(); // any caller can mutate the order
```

**Fix**: expose a read-only view and provide explicit methods to mutate.

```csharp
private readonly List<OrderLine> _lines = new();
public IReadOnlyList<OrderLine> Lines => _lines;
public void AddLine(OrderLine line) { /* invariants enforced here */ _lines.Add(line); }
```

### 6. String comparison without culture/case rules

```csharp
if (country == "us") // BUG — fails on "US", and depends on current culture
```

**Fix**:

```csharp
if (country.Equals("US", StringComparison.OrdinalIgnoreCase))
```

### 7. `var` that hides types

```csharp
var x = repo.Query();         // is this IQueryable<Order>? IEnumerable<Order>? a Task<>? Unclear.
```

**Fix**: use explicit types when the right-hand side is not obviously typed.

```csharp
IQueryable<Order> orders = repo.Query();
```

### 8. Mixing `DateTime` and `DateTimeOffset`

A `DateTime` with `Kind == Unspecified` deserialized from JSON, then compared to `DateTimeOffset.UtcNow`, produces silent off-by-one-day bugs across DST boundaries.

**Fix**: standardize on `DateTimeOffset` from the API boundary inward; convert at the edge.

## Best Practices

- **Money is `decimal`. Time is `DateTimeOffset`. Ids are `Guid`.** Memorize this.
- **Inject `TimeProvider`** instead of touching `DateTime.UtcNow`/`DateTime.Now` directly — see [csharp/dependency-injection.md](dependency-injection.md).
- **Make types `sealed` by default**; unseal only when you intend inheritance.
- **Prefer `record` for DTOs and value objects**; `class` for entities with behavior.
- **Expose read-only collection interfaces** on public APIs.
- **Use `required` properties** (C# 11+) for non-nullable construction-time data.
- **Always pass `CultureInfo.InvariantCulture`** when parsing/formatting numbers and dates for storage or wire format.
- **Always pass `StringComparison`** when comparing strings.
- **Wrap disposables in `using`/`await using`** — never store an `HttpResponseMessage` in a field.
- **Wrap "stringly-typed" values** (currency codes, country codes, emails) in tiny value objects to make invalid states unrepresentable.
- **Enable `<Nullable>enable</Nullable>`** in every project — see [csharp/nullable-reference-types.md](nullable-reference-types.md).

## Related Concepts

- **[csharp/nullable-reference-types.md](nullable-reference-types.md)** — fundamentals for null safety.
- **[csharp/records-and-immutability.md](records-and-immutability.md)** — concise, equality-by-value types built on the fundamentals.
- **[csharp/object-oriented-programming.md](object-oriented-programming.md)** — uses fundamentals to build behavior.
- **[csharp/linq.md](linq.md)** — operates on the collection contracts described here.
- **[csharp/exception-handling.md](exception-handling.md)** — pairs with `try`/`finally` and `using` for resource safety.
- **[architecture/domain-driven-design.md](../architecture/domain-driven-design.md)** — value objects are how DDD codifies "primitive obsession is bad".
- **[architecture/entities-and-value-objects.md](../architecture/entities-and-value-objects.md)** — the building blocks built on these fundamentals.
- **[data-access/entity-framework-core.md](../data-access/entity-framework-core.md)** — `decimal` precision, `Guid` keys, and `DateTimeOffset` are first-class in EF Core.

## Real-World Usage

### ASP.NET Core Web API on Azure App Service

```csharp
public sealed record CreateInvoiceRequest(
    Guid CustomerId,
    IReadOnlyList<InvoiceLineDto> Lines,
    string CurrencyCode);

[ApiController]
[Route("api/invoices")]
public sealed class InvoicesController(IInvoiceService svc) : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<InvoiceDto>> Create(
        CreateInvoiceRequest req,
        CancellationToken ct)
    {
        var dto = await svc.CreateAsync(req, ct);
        return CreatedAtAction(nameof(GetById), new { id = dto.Id }, dto);
    }
}
```

Notice the surface: `Guid`, `IReadOnlyList<T>`, `CancellationToken`, `string` for an ISO currency code (validated server-side). No raw `int` ids, no `List<T>` on the wire, no async method without a token.

### Azure Functions (isolated worker)

```csharp
[Function("AccrueInterest")]
public async Task RunAsync(
    [TimerTrigger("0 0 1 * * *")] TimerInfo timer,
    FunctionContext ctx)
{
    var ct = ctx.CancellationToken;
    foreach (var account in await _accounts.ActiveAsync(ct))
    {
        decimal interest = Math.Round(account.Balance * account.Rate / 12m, 2, MidpointRounding.ToEven);
        await _accounts.AddInterestAsync(account.Id, interest, _time.GetUtcNow(), ct);
    }
}
```

Banker's rounding (`MidpointRounding.ToEven`) is the standard for accruals — using the wrong mode causes audit failures.

### Testing fundamentals with `FakeTimeProvider`

```csharp
[Fact]
public async Task DueAt_IsThirtyDaysAfterIssueDate()
{
    var clock = new FakeTimeProvider(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));
    var sut = new InvoiceService(clock, new InMemoryInvoiceRepo());

    var invoice = await sut.IssueAsync(new Invoice { Amount = 100m, CurrencyCode = "USD" }, default);

    invoice.DueAt.Should().Be(new DateTimeOffset(2026, 1, 31, 0, 0, 0, TimeSpan.Zero));
}
```

Tests are deterministic because `TimeProvider` is injected — see [testing-quality/unit-testing.md](../testing-quality/unit-testing.md).

## Code Example — Before and After

### Before: primitive obsession, ambiguous types, fragile arithmetic

```csharp
public class OrderService
{
    public int CreateOrder(int customerId, string total, string when)
    {
        double amount = double.Parse(total);             // wrong type
        DateTime placed = DateTime.Parse(when);          // ambiguous TZ, default culture
        var id = new Random().Next();                    // collisions, not unique
        using (var conn = new SqlConnection("Server=..."))
        {
            conn.Open();
            new SqlCommand($"INSERT INTO Orders VALUES ({id}, {customerId}, {amount}, '{placed}')", conn)
                .ExecuteNonQuery();                      // SQL injection + culture bugs
        }
        return id;
    }
}
```

Problems:

- `double` for money — silent rounding errors.
- `DateTime.Parse` uses the host's culture; `'1/2/2026'` differs between US and EU.
- `Random.Next` ids collide and are predictable.
- Hard-coded connection string.
- SQL injection via string interpolation.

### After: correct types, value objects, DI, parameterized SQL

```csharp
public readonly record struct Money(decimal Amount, string CurrencyCode);

public sealed record CreateOrderRequest(
    Guid CustomerId,
    Money Total,
    DateTimeOffset PlacedAt);

public sealed class OrderService(
    AppDbContext db,
    TimeProvider time,
    ILogger<OrderService> logger)
{
    public async Task<Guid> CreateAsync(CreateOrderRequest req, CancellationToken ct)
    {
        var order = new Order
        {
            Id          = Guid.CreateVersion7(),
            CustomerId  = req.CustomerId,
            Total       = req.Total.Amount,
            Currency    = req.Total.CurrencyCode,
            PlacedAt    = req.PlacedAt == default ? time.GetUtcNow() : req.PlacedAt
        };

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        logger.LogInformation("Order {OrderId} created for {CustomerId}", order.Id, order.CustomerId);
        return order.Id;
    }
}
```

What improved:

- **`decimal` Money** — no rounding errors, currency captured.
- **`Guid` ids** — collision-free, non-guessable, distributed-system safe.
- **`DateTimeOffset`** — unambiguous time across regions, paired with `TimeProvider` for testability.
- **EF Core + parameterized queries** — SQL injection impossible.
- **DI** — `DbContext`, clock, and logger are injected (see [csharp/dependency-injection.md](dependency-injection.md)).
- **`record` request** — immutable, value-equality, easy to validate.
- **`CancellationToken`** — request cancellations propagate to the database.

## Interview Questions and Answers

### 1. Why is `decimal` required for money, and what goes wrong with `double`?

**Why this matters**: This is the single most common production-data bug in financial systems and the easiest to ship without noticing.

**Answer**: `double` is binary floating-point — it cannot represent `0.10` exactly. Summing prices yields values like `0.30000000000000004`. After a million transactions you have observable drift, audit failures, and reconciliation tickets. `decimal` is base-10 with 28–29 significant digits, exactly representing values you'd write on a check. EF Core maps `decimal` to `decimal(18,2)` (or a precision you specify) — a real numeric SQL type.

**Trade-off**: `decimal` is ~10× slower than `double`. For monetary code that's irrelevant; for scientific computing you use `double`.

**Real project**: A billing service used `double` for line totals. After 18 months, the trial-balance report was off by $1,200. Migrating to `decimal` was a two-week migration with downtime-free dual writes.

### 2. When would you use `record` vs `class` vs `struct` vs `record struct`?

**Answer**:

| Type           | Use for                                                         |
|----------------|-----------------------------------------------------------------|
| `class`        | Entities with identity and behavior (`Order`, `Customer`)        |
| `sealed record`| DTOs, requests/responses, immutable value objects               |
| `readonly struct` | Small (≤16 bytes) zero-allocation values (`Coordinate`)      |
| `readonly record struct` | Small value objects with value-equality (`Money`, `Range`) |

**Real project**: `Money(decimal, string)` is a `readonly record struct` — equality is value-based, allocation-free, and you can put millions in a `List<Money>` without GC pressure.

### 3. Why prefer `DateTimeOffset` over `DateTime` in backend code?

**Answer**: `DateTimeOffset` always carries a UTC offset. `DateTime` has a `Kind` (`Utc`, `Local`, `Unspecified`) that's frequently wrong after deserialization. Cloud services run in containers configured for UTC; a developer's laptop runs in local time. The same code behaves differently in dev and production unless time carries its zone with it.

**Trade-off**: `DateOnly`/`TimeOnly` (introduced in .NET 6) cover wall-clock concerns — invoice dates, business hours — without a meaningless time-of-day component.

**Real project**: A scheduling system that used `DateTime` produced "the meeting moved an hour" tickets every March and November. Replacing the type with `DateTimeOffset` (and storing as `datetimeoffset` in SQL) eliminated them.

### 4. How do you make a class's collection property safe to expose?

**Answer**: Keep the backing collection private, expose a read-only view, and provide methods that maintain invariants.

```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public void AddLine(OrderLine line)
    {
        if (_lines.Count >= 50) throw new InvalidOperationException("Order line limit reached");
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order is not editable");
        _lines.Add(line);
    }
}
```

**Why**: A public `List<T>` lets any caller call `.Clear()`, `.Add(invalid)`, or sort it — silently destroying the aggregate's invariants. Read-only interfaces preserve encapsulation while still allowing iteration.

### 5. What does `using` actually do, and when do you need `await using`?

**Answer**: `using` generates a `try`/`finally` that calls `Dispose()` (synchronous). `await using` calls `DisposeAsync()` and is required for `IAsyncDisposable` — `DbContext` (since .NET 6), `Azure.Storage.Blobs.BlobClient`, `ServiceBusClient`, file streams opened asynchronously.

**Pitfall**: Forgetting `await using` on `DbContext` doesn't crash — it falls back to `Dispose()`, which blocks the thread. Under load, this serializes async work and shows up as thread-pool starvation.

### 6. Why are `Guid` keys preferred over auto-increment `int` in cloud systems?

**Answer**:

- **No central coordinator** — each service generates ids independently; no `SELECT IDENT_CURRENT` round-trip.
- **No leak of business size** — `id=12345` tells competitors how many users you have; a `Guid` doesn't.
- **Safe to merge across databases** — multi-region writes don't collide.
- **Idempotent client retries** — the client can generate the id, retry on timeout, and the server uses `INSERT … IF NOT EXISTS` to dedupe.

**Trade-off**: GUIDs fragment B-tree indexes (insert in random order). Use sequential GUIDs (`Guid.CreateVersion7()` in .NET 9, or `NEWSEQUENTIALID()` in SQL Server) to preserve locality.

### 7. What is "primitive obsession" and how do you fix it?

**Why this matters**: Stringly-typed code (`string email`, `string countryCode`, `decimal amount` with no currency) is a top source of subtle bugs — methods get arguments in the wrong order, validation is repeated everywhere, refactors are dangerous.

**Answer**: Wrap recurring values in tiny `record struct` value objects with validation in the constructor:

```csharp
public readonly record struct Email
{
    public string Value { get; }
    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email", nameof(value));
        Value = value.ToLowerInvariant();
    }
    public override string ToString() => Value;
}
```

Now `SendInvoiceAsync(Email to)` cannot be called with `customerName`. The compiler enforces what code reviews would have caught.

### 8. What rounding strategy do you use for money calculations, and why?

**Answer**: For accruals and split calculations, use **banker's rounding** (`MidpointRounding.ToEven`):

```csharp
decimal interest = Math.Round(balance * rate / 12m, 2, MidpointRounding.ToEven);
```

Banker's rounding is unbiased over many transactions; default `.NET` rounding (away from zero) systematically over-collects on midpoints, which compounds into millions of dollars at bank scale. For VAT and tax, follow the regulator's rule (often `AwayFromZero`). Always store the rounded value in the database so totals reconcile exactly.

## Summary Checklist

- [ ] I always use `decimal` for money, `Guid` for ids, and `DateTimeOffset` for time.
- [ ] I inject `TimeProvider` instead of touching `DateTime.UtcNow` directly.
- [ ] I know when to use `class`, `sealed record`, `readonly struct`, and `readonly record struct`.
- [ ] I expose `IReadOnlyList<T>` / `IReadOnlyDictionary<,>` on public APIs, never `List<T>`.
- [ ] I wrap every `IDisposable` and `IAsyncDisposable` in `using` / `await using`.
- [ ] I pass `CultureInfo.InvariantCulture` for parse/format and `StringComparison` for comparisons.
- [ ] I use pattern matching to flatten nested conditionals and switch expressions to compute values.
- [ ] I prefer `internal sealed` and only mark types `public` at assembly boundaries.
- [ ] I wrap stringly-typed values (`Email`, `CurrencyCode`, `Money`) in value objects.
- [ ] I enable nullable reference types in every project and treat warnings as errors.
