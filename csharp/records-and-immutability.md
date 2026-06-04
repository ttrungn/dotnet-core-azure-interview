# Records and Immutability

## What It Is

A **record** is a C# 9 reference type (and C# 10 `record struct`) designed for data-centric objects. Compared to a class, a record gives you:

- **Value-based equality** — two records with the same property values are `Equals` and have the same `GetHashCode`.
- **A copy/`with` expression** — produce a new instance with selected properties changed: `var v2 = v1 with { Amount = 100 };`.
- **A concise positional syntax** — `public sealed record Money(decimal Amount, string Currency);` declares the type, constructor, properties, deconstructor, and `ToString` override in one line.
- **An overridden `ToString`** that prints the property names and values.

**Immutability** is the design discipline of *not allowing state to change after construction*. Records make immutability easy (positional records have `init`-only properties) but do not enforce it — you can mutate a record's mutable members just like any object.

```csharp
public sealed record Money(decimal Amount, string Currency);

var price  = new Money(99.00m, "USD");
var bumped = price with { Amount = price.Amount + 10m }; // new instance
Console.WriteLine(price);   // Money { Amount = 99.00, Currency = USD }
Console.WriteLine(price == new Money(99.00m, "USD")); // True — value equality
```

Records are not a replacement for entities (objects with identity). They shine for **DTOs, value objects, integration events, and configuration objects** — types where two instances with identical data really should be considered the same thing.

## Why It Exists

Before records (C# 9, .NET 5, late 2020), writing an immutable data type was painful:

```csharp
public sealed class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }

    public override bool Equals(object? obj)
        => obj is Money m && m.Amount == Amount && m.Currency == Currency;
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public override string ToString() => $"Money {{ Amount = {Amount}, Currency = {Currency} }}";
    public Money WithAmount(decimal amount) => new(amount, Currency);
    public Money WithCurrency(string currency) => new(Amount, currency);
    public void Deconstruct(out decimal amount, out string currency) { amount = Amount; currency = Currency; }
}
```

Twenty lines for what is conceptually "a 2-tuple of amount and currency." Worse, every change required updating equality, hash code, `ToString`, and every `WithX` method — easy to forget, easy to get inconsistent.

Records collapse all of that boilerplate into a single declaration and **guarantee** the generated `Equals`/`GetHashCode`/`ToString`/`with` stay in sync with the property list. They also rehabilitate value semantics in a language that defaults to reference equality.

Immutability as a discipline existed long before records — F#, Scala, Rust, Erlang, and Clojure all rely on it. The promise: data that doesn't change is easier to reason about, safe to share across threads, simple to cache, and impossible to corrupt via accidental mutation.

## When To Use It

**Use a record for:**

- **DTOs** — request bodies, response bodies, integration events, message payloads.
- **Value objects** — `Money`, `Address`, `EmailAddress`, `DateRange`. Things defined entirely by their value, with no identity.
- **Domain events** — `OrderPlaced(Guid OrderId, DateTime At, decimal Total)` is a captured fact, immutable by definition.
- **Configuration option classes bound from `IConfiguration`** — `StripeOptions` snapshot doesn't change at runtime.
- **Result/Option/Either wrappers** — `Result<T>(bool Succeeded, T? Value, string? Error)`.

**Do NOT use a record for:**

- **Entities with identity and lifecycle** — `Order`, `Customer`, `Invoice`. These have an ID; two orders with the same totals are still distinct, and they mutate over time (status transitions, line items added).
- **EF Core entities** — value equality breaks change tracking; mutable navigation properties betray "immutable" intent.
- **Types that hold mutable collections you expose** — `record Cart(List<Item> Items)` looks immutable but `cart.Items.Add(...)` mutates state and breaks equality semantics.
- **Types with significant behavior** — a `PaymentProcessor` that has methods, dependencies, and orchestrates a workflow is a class.

## Why It Is Important

Records and immutability matter in production for concrete operational reasons:

- **Thread safety** — an immutable object is safe to share across threads without locks. A `Money` value passed through a parallel pricing pipeline cannot be torn or partially updated.
- **Cache safety** — putting a mutable object in `IMemoryCache` is dangerous: any consumer can mutate the cached instance, and now every other consumer sees the change. Immutable records are safe.
- **Equality-based deduplication** — `HashSet<OrderLineDto>` actually works as expected. Integration tests can `Assert.Equal(expectedEvent, actualEvent)` directly.
- **Message-bus payloads** — Service Bus / Kafka payloads are conceptually immutable facts. Modeling them as records prevents subtle bugs where a handler mutates the payload mid-pipeline.
- **Diffing and auditing** — calculating "what changed" between two versions of an object is straightforward with value equality.
- **Refactor safety** — adding a new property to a positional record automatically extends equality, copy, and `ToString` — no chance of forgetting one.

## How It's Used in C# / .NET

| Feature | Example |
| --- | --- |
| **Positional record** | `public sealed record Money(decimal Amount, string Currency);` |
| **Nominal record** | `public sealed record StripeOptions { public string ApiKey { get; init; } = default!; }` |
| **`record struct` (C# 10)** | `public readonly record struct Coordinate(double Lat, double Lng);` — value-type semantics + records' generated members. |
| **`record class`** | Explicit form of `record` (default). |
| **`with` expression** | `var modified = original with { Status = OrderStatus.Confirmed };` |
| **`init`-only setters** | `public string Name { get; init; }` — settable in object initializer, not later. |
| **`required` members (C# 11)** | `public required string Name { get; init; }` — caller must set it or get a compile error. |
| **Primary constructors (C# 12)** | `public sealed class StripeClient(IOptions<StripeOptions> opts, HttpClient http) { ... }` (works on classes too now). |
| **Deconstruction** | `var (amount, currency) = price;` |
| **Inheritance** | Records support inheritance with proper value equality; `sealed` is recommended. |
| **`System.Text.Json`** | Records serialize/deserialize naturally; positional records use constructor binding. |
| **`ImmutableArray<T>`, `ImmutableList<T>`, `ImmutableDictionary<TKey,TValue>`** | True immutable collections; mutation returns a new instance. |
| **`IReadOnlyList<T>` / `IReadOnlyCollection<T>`** | Read-only views — important for properties on records that hold collections. |

A representative DTO + event + value object trio:

```csharp
// Value object — entirely defined by amount + currency
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }
}

// API request DTO
public sealed record PlaceOrderRequest(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines,
    string? CouponCode);

public sealed record OrderLineDto(Guid ProductId, int Quantity, Money UnitPrice);

// Integration event published to Azure Service Bus
public sealed record OrderPlacedEvent(
    Guid OrderId,
    Guid CustomerId,
    Money Total,
    DateTime PlacedAtUtc);
```

## Advantages

- **Less boilerplate** — value equality, hash, `ToString`, `with`, deconstruct all generated.
- **Refactor-safe** — adding a property updates all generated members consistently.
- **Value semantics** — `==` and `Equals` behave by data, not by reference.
- **Thread-safe by default** (when properties are `init` and types are themselves immutable).
- **Great for tests** — `Assert.Equal(expected, actual)` on records compares data directly.
- **`with` enables functional update patterns** without manual builder code.

## Disadvantages

- **Easy to misuse for entities** — record equality on `Order` will say two orders with the same data are equal, which is almost never what you want for identity-bearing types.
- **Not deeply immutable** — a record property of type `List<T>` is still mutable; `record Order(List<Line> Lines)` is misleading.
- **Inheritance is supported but complicated** — value equality has surprising rules across derived types (`record A` vs `record B : A`).
- **Slightly larger generated code** than a plain `class` — usually negligible.
- **EF Core does not play well with records as entities** — change tracking depends on reference equality.
- **`==` on records is value-based**, which can surprise developers used to reference equality everywhere.

## Common Mistakes

### 1. Using records for entities with identity

```csharp
// WRONG — two distinct orders with same totals compare as equal
public sealed record Order(Guid Id, Guid CustomerId, OrderStatus Status, decimal Total);

var a = new Order(idA, custA, OrderStatus.Pending, 100m);
var b = new Order(idA, custA, OrderStatus.Pending, 100m);
// a == b → True. Confusing for entities; their identity is Id alone.
```

```csharp
// RIGHT — entities are classes with identity-based equality
public sealed class Order : Entity<Guid>
{
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; private set; }
}

public abstract class Entity<TId> where TId : struct
{
    public TId Id { get; protected set; }
    public override bool Equals(object? obj) => obj is Entity<TId> e && e.Id.Equals(Id);
    public override int GetHashCode() => Id.GetHashCode();
}
```

### 2. Putting a mutable collection inside a "immutable" record

```csharp
// WRONG — Lines is mutable; consumers can do dto.Lines.Add(...)
public sealed record CartDto(Guid CustomerId, List<CartLineDto> Lines);
```

```csharp
// RIGHT — expose IReadOnlyList<T> and use ImmutableArray<T> internally
public sealed record CartDto(Guid CustomerId, ImmutableArray<CartLineDto> Lines);
// or
public sealed record CartDto(Guid CustomerId, IReadOnlyList<CartLineDto> Lines);
```

### 3. Using `with` to bypass invariants

```csharp
public sealed record Money(decimal Amount, string Currency);

var price = new Money(99m, "USD");
var bad = price with { Amount = -1m };
// 'with' doesn't run a constructor body, so any throw-in-ctor validation is skipped
```

```csharp
// RIGHT — validate in the primary constructor's body OR use a nominal record with init validators
public sealed record Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount));
        if (string.IsNullOrWhiteSpace(currency)) throw new ArgumentException("required", nameof(currency));
        Amount = amount;
        Currency = currency;
    }
}
// 'with' DOES run the constructor when you use positional records — see below
```

For positional records, the compiler emits a deconstruct-and-reconstruct sequence; validation in the primary constructor body **does** run. The mistake above applies when you have an `init` property without going through the constructor.

### 4. Inheriting records casually

```csharp
// WRONG — equality is surprising across derived types
public record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);

var a = new Animal("Rex");
var d = new Dog("Rex", "Husky");
// a.Equals(d) -> false because EqualityContract differs — but it's still subtle
```

```csharp
// RIGHT — seal record hierarchies, or avoid inheritance entirely for data types
public sealed record OrderPlacedEvent(Guid OrderId);
public sealed record OrderShippedEvent(Guid OrderId, DateTime At);
```

### 5. Choosing record for syntax brevity alone

```csharp
// WRONG — PaymentProcessor has behavior, dependencies, lifetime; it's not a data bag
public sealed record PaymentProcessor(IPaymentGateway Gateway, ILogger<PaymentProcessor> Logger)
{
    public Task ChargeAsync(Guid orderId, decimal amount, CancellationToken ct) => ...;
}
```

```csharp
// RIGHT — use a class (with primary constructor if you like brevity)
public sealed class PaymentProcessor(IPaymentGateway gateway, ILogger<PaymentProcessor> logger)
{
    public Task ChargeAsync(Guid orderId, decimal amount, CancellationToken ct) => ...;
}
```

### 6. Forgetting that record fields are nullable-aware

```csharp
// WRONG — Currency is non-nullable but deserializer can put null there
public sealed record Money(decimal Amount, string Currency);
```

```csharp
// RIGHT — validate at deserialization boundary OR use [Required] / FluentValidation
public sealed record Money
{
    public decimal Amount { get; init; }
    public required string Currency { get; init; } // C# 11 — compile error if not set

    public Money() { } // for serializer
}
```

### 7. Comparing reference equality unexpectedly

```csharp
var m1 = new Money(100m, "USD");
var m2 = new Money(100m, "USD");
ReferenceEquals(m1, m2); // false
m1 == m2;                // TRUE — value equality
m1.Equals(m2);           // TRUE
```

If you actually need reference equality on a record, use `ReferenceEquals` explicitly — or stop and ask why you have a record at all.

## Best Practices

- **`sealed record`** by default — avoids accidental inheritance equality gotchas.
- **Use positional records** for small DTOs and value objects (≤ 5 properties); switch to nominal (`record { ... }`) when validation or property-level docs matter.
- **Use `IReadOnlyList<T>` or `ImmutableArray<T>`** for collection properties.
- **Validate invariants in the primary constructor body** so `with` expressions also validate.
- **Use `required` members (C# 11)** when a property must be set — guards against partial initialization.
- **Reserve records for data; use classes for behavior.**
- **Do not use records for EF Core entities.**
- **Treat integration events and message payloads as records** — they are facts, not mutable state.
- **For deep immutability**, all properties (recursively) must themselves be immutable.
- **Use `record struct`** when avoiding heap allocations matters (very hot paths, > 100k/sec).
- **Override `ToString`** explicitly if records contain secrets (API keys, PII) — the default `ToString` includes every property.

## Related Concepts

- [csharp/object-oriented-programming.md](csharp/object-oriented-programming.md) — class vs record, when to use which.
- [csharp/nullable-reference-types.md](csharp/nullable-reference-types.md) — `required` and nullable annotations on record properties.
- [architecture/entities-and-value-objects.md](architecture/entities-and-value-objects.md) — DDD's identity vs value distinction maps to class vs record.
- [architecture/domain-events.md](architecture/domain-events.md) — domain events as records.
- [aspnet-core/restful-api-design.md](aspnet-core/restful-api-design.md) — records as request/response DTOs.
- [data-access/entity-framework-core.md](data-access/entity-framework-core.md) — why entities should not be records.

## Real-World Usage

### ASP.NET Core APIs

Request and response models are textbook record use. Positional records keep DTO files short, `[ApiController]` + FluentValidation enforces invariants at the boundary, and OpenAPI/Swashbuckle picks up the constructor parameters automatically.

### Azure Service Bus / Event Hubs

Integration events are immutable facts — perfect records. A producer creates `new OrderPlacedEvent(...)` and `await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(evt)))`. The consumer deserializes into the same record; with `[JsonConstructor]` on a positional record, `System.Text.Json` populates via the constructor (running any validation).

### Azure Functions / MediatR

Function inputs and MediatR `IRequest<TResponse>` types are typically records — they carry data into the handler with value semantics that simplify testing.

### `IOptions<TOptions>` configuration

Configuration types bound from `appsettings.json` are good records (or sealed classes with `init`). Use `required` for non-optional settings and `ValidateDataAnnotations().ValidateOnStart()` for runtime guarantees.

### Testing

`Assert.Equal(expected, actual)` on records compares property-by-property. With xUnit + FluentAssertions, `actual.Should().BeEquivalentTo(expected)` works without extra setup. Builders (`TestBuilders.Order()`) compose well with `with` to produce minor variations.

### Cache and parallel pipelines

Immutable records can be safely placed in `IMemoryCache`, `IDistributedCache`, `Channel<T>`, or `BlockingCollection<T>`. No defensive copies, no race conditions on consumer-side mutation.

## Code Example — Before and After

### Before — mutable DTO, hand-rolled equality, leaky mutation in cache

```csharp
public class OrderDto
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderLineDto> Lines { get; set; } = new();
    public string Status { get; set; } = "Pending";
    public decimal Total { get; set; }
}

public class OrderLineDto
{
    public Guid ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class OrderService
{
    private readonly IMemoryCache _cache;

    public OrderDto GetOrder(Guid id)
    {
        return _cache.GetOrCreate($"order:{id}", entry => LoadFromDb(id))!;
    }
}

// Somewhere far away:
var o = orderService.GetOrder(id);
o.Status = "Confirmed";        // MUTATES the cached instance!
// Next caller of GetOrder sees Status = "Confirmed" with stale data
```

### After — immutable record, value equality, cache-safe

```csharp
public sealed record OrderDto(
    Guid Id,
    Guid CustomerId,
    ImmutableArray<OrderLineDto> Lines,
    OrderStatus Status,
    Money Total);

public sealed record OrderLineDto(Guid ProductId, int Quantity, Money UnitPrice);

public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return this with { Amount = Amount + other.Amount };
    }
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }

public sealed class OrderQueryService
{
    private readonly IMemoryCache _cache;
    private readonly OrderingDbContext _db;
    private readonly ILogger<OrderQueryService> _logger;

    public OrderQueryService(IMemoryCache cache, OrderingDbContext db, ILogger<OrderQueryService> logger)
    {
        _cache = cache; _db = db; _logger = logger;
    }

    public async Task<OrderDto?> GetAsync(Guid id, CancellationToken ct)
    {
        if (_cache.TryGetValue<OrderDto>($"order:{id}", out var cached))
            return cached;

        var entity = await _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);
        if (entity is null) return null;

        var dto = new OrderDto(
            entity.Id,
            entity.CustomerId,
            entity.Lines.Select(l => new OrderLineDto(l.ProductId, l.Quantity, new Money(l.UnitPrice, l.Currency))).ToImmutableArray(),
            entity.Status,
            new Money(entity.Total, entity.Currency));

        _cache.Set($"order:{id}", dto, TimeSpan.FromMinutes(5));
        return dto;
    }
}

// Consumers cannot mutate the cached instance — types are immutable
var o = await orderQueryService.GetAsync(id, ct);
var confirmed = o! with { Status = OrderStatus.Confirmed }; // new instance, cache untouched
```

The after version makes accidental cache poisoning impossible, equality meaningful (`OrderDto a == OrderDto b` does what you expect), and updates explicit via `with`.

## Interview Questions and Answers

### 1. When would you choose a `record` over a `class`?

**Why this matters:** Foundational decision; affects equality, identity, and mutation semantics.

**Answer:** Choose a record when the type is **defined by its data** — DTOs, value objects, integration events, configuration. Choose a class when the type has **identity, lifecycle, or behavior** — entities, services, repositories, controllers. The deciding question: "If two instances have all the same property values, are they the same thing?" Yes → record. No → class. A `Money(100, "USD")` is interchangeable with any other `Money(100, "USD")`, so it's a record. Two `Order`s with the same totals are still different orders, so `Order` is a class.

**Trade-off:** Records have a tiny generated-code overhead and value equality may surprise developers expecting reference equality. The clarity of intent outweighs both in DTO code.

**Real project:** A team modeled `Customer` as a record; two customers with the same name and email compared as equal, which broke a deduplication step in the import pipeline. Reverting `Customer` to a class with `Id`-based equality fixed it immediately.

### 2. Are records truly immutable?

**Why this matters:** Common misconception.

**Answer:** Not by themselves — they're only as immutable as their properties. A positional record has `init`-only properties (settable in object initializer or constructor, not afterward), which is most of the way there. But a property of type `List<T>` is still mutable: you can do `record.Items.Add(...)` and the record's identity (its property values) hasn't changed but its state has. For true (deep) immutability, every property type must itself be immutable — use `ImmutableArray<T>`, `IReadOnlyList<T>` backed by an immutable source, or other records all the way down.

**Trade-off:** `ImmutableArray<T>` is allocation-friendlier than `ImmutableList<T>` for read-mostly scenarios; `IReadOnlyList<T>` is the lightest "I won't mutate" hint but only a convention. Pick the strictness level your team needs.

**Real project:** A `record CartDto(List<CartLineDto> Lines)` was being mutated by a logging interceptor that "trimmed sensitive items" — the cache was getting corrupted over the day. Switching to `ImmutableArray<CartLineDto>` made the mutation a compile error.

### 3. What does `with` actually do under the hood?

**Why this matters:** Tests whether candidate knows it's not a pointer-update.

**Answer:** `x with { Y = 5 }` creates a **new instance** by calling the record's auto-generated `<Clone>$()` method (a non-virtual copy constructor) and then applies the property assignments. For positional records, the primary constructor body runs — so any validation throws if the resulting object is invalid. The original object is unmodified. This is the functional-update pattern: state evolves by producing new values, not by mutating old ones.

**Trade-off:** Each `with` allocates. In hot loops on large records, this matters; consider `record struct` or a mutable builder pattern.

**Real project:** A real-time pricing loop called `result with { Total = ... }` per tick. Profiling showed Gen 0 allocations were dominating — switching to `record struct` for the inner-loop type eliminated them without changing the API.

### 4. Why shouldn't EF Core entities be records?

**Why this matters:** Hands-on EF Core knowledge.

**Answer:** EF Core's change tracker uses **reference equality** to know which entity instance corresponds to which row. Record value equality breaks this: two records with the same values are "equal" but are still different references, and the change tracker can get confused. Records also encourage `init`-only properties — but EF Core needs to mutate properties as it materializes objects and tracks changes (`order.Status = OrderStatus.Confirmed`). And inheritance + value equality across derived records doesn't fit the table-per-hierarchy / table-per-type EF mapping. Use classes for entities; reserve records for the read-model DTOs you project entities into.

**Trade-off:** None really — this is settled best practice. The boilerplate of writing equality on entity classes is minimal (almost always identity-based via `Entity<TId>` base).

**Real project:** A team made everything records "for consistency." Their unit-of-work double-saved entities in some cases and dropped updates in others. Migrating entities to classes resolved it; DTOs stayed as records.

### 5. How do `required` members interact with records?

**Why this matters:** C# 11+ feature that fills a real gap.

**Answer:** `required` on a property says "the caller must set this in the object initializer, or the compiler refuses to build." It works on both classes and records (the nominal/`{ }` form). With records, this gives you compile-time enforcement that consumers haven't dropped a property — `new StripeOptions { ApiKey = "..." }` compiles, but `new StripeOptions { }` doesn't. For deserialization scenarios, combine `required` with `[SetsRequiredMembers]` on a constructor to tell the compiler "this ctor satisfies the requirement."

**Trade-off:** Older callers and reflection-based serializers may not know about `required` — `System.Text.Json` does (uses `JsonConstructorAttribute` and constructor binding), but custom code paths might miss it.

**Real project:** A config record was missing `ApiKey` set in `appsettings.Development.json`. The app started up, then crashed on first call to Stripe. Adding `required` made it a startup-time compile-style error caught by `ValidateOnStart`.

### 6. Why would you use `record struct`?

**Why this matters:** Tests awareness of value-type performance.

**Answer:** `record struct` (C# 10+) combines records' generated members with `struct`'s allocation model — instances live on the stack or inline in their container, no heap allocation. Useful for very small data types in hot paths: a `Point`, a small enum-like tuple, a per-iteration intermediate in a pricing pipeline. The trade-off: `struct` semantics (copy on assignment, no inheritance, must always have valid state) apply. Use `readonly record struct` for full immutability and avoid defensive copies in members.

**Trade-off:** Above ~16 bytes, `struct` copy cost outweighs heap allocation cost; profile, don't guess.

**Real project:** A SIMD-friendly geometry library used `readonly record struct Coordinate(double X, double Y)`. Replacing the previous class cut Gen 0 allocations to near-zero in the hottest inner loops.

### 7. What problem does value-based equality solve in tests?

**Why this matters:** Concrete test-writing benefit.

**Answer:** Without value equality, asserting `Assert.Equal(expectedEvent, actualEvent)` on a class compares references and fails (they're different instances). You end up writing per-property assertions for every test or pulling in heavy comparison libraries. With records, `Assert.Equal` "just works" because `Equals` compares property values. Test data builders and integration tests over message payloads become much shorter and less brittle. This is the single biggest day-to-day quality-of-life improvement.

**Trade-off:** If a record contains a mutable collection, equality may still drift; favor `ImmutableArray<T>` for stable equality.

**Real project:** Our integration tests against Service Bus used 50-line property-by-property assertions per event type. Switching events to records collapsed each assertion to one line and caught two pre-existing bugs where fields had silently diverged.

### 8. How should records handle secrets like API keys?

**Why this matters:** Security concern around the auto-generated `ToString`.

**Answer:** The record `ToString` includes every property by default — perfect for diagnostics, dangerous for secrets. If you have a record holding API keys, connection strings, or PII, override `ToString` to redact those fields, mark sensitive members with `[JsonIgnore]` (so they don't leak via serialization), or split the record so secrets live in a separate type that is never logged. For configuration records bound from Key Vault, prefer storing secrets as `SecureString` or never logging the options object at all.

**Trade-off:** Custom `ToString` loses some of the boilerplate benefit, but selective overriding (`ToString` only when secrets exist) is a small cost for safety.

**Real project:** A `record StripeOptions(string ApiKey, string Endpoint)` got logged in a startup `LogInformation("Config: {Options}", options)` — the API key showed up in App Insights. We added `[JsonIgnore]` on the field and overrode `ToString` to redact, and audited similar configs across services.

## Summary Checklist

- [ ] I use records for DTOs, value objects, integration events, and configuration — types defined by their data.
- [ ] I use classes for entities (identity + lifecycle), services, and anything with significant behavior.
- [ ] I `sealed record` by default to avoid inheritance equality surprises.
- [ ] I expose collection properties as `IReadOnlyList<T>` or `ImmutableArray<T>` — never mutable `List<T>`.
- [ ] I put invariant validation in the primary constructor body so `with` expressions validate too.
- [ ] I use `required` members (C# 11) for properties that must be set.
- [ ] I know `with` creates a new instance (allocation) and considers `record struct` for hot paths.
- [ ] I never use records as EF Core entities.
- [ ] I override `ToString` (or use `[JsonIgnore]`) on records holding secrets.
- [ ] I rely on value equality in tests to keep assertions short and readable.
