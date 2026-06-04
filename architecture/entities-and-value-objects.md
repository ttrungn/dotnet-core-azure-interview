# Entities and Value Objects

## What It Is

Entities and Value Objects are the two foundational building blocks of any object-oriented domain model — and the most basic tactical patterns in Domain-Driven Design.

- **Entity** — a type whose identity matters and persists across time. Two entities can have identical attribute values yet still be different objects because they have different IDs. `Order`, `Customer`, `Shipment`, `User`, `Invoice` are entities. An entity has a **lifecycle** (created, modified, archived) and its state changes through **behavior** (`order.Confirm()`, `customer.ChangeEmail(...)`).

- **Value Object** — a type defined entirely by its values. Two value objects with the same attributes are *interchangeable* — they are equal. `Money(100, USD)` equals another `Money(100, USD)`. Value objects are **immutable**, have **no identity**, and represent **concepts**: an amount of money, an email address, a date range, a geographic coordinate, a postal address.

The distinction sounds academic but drives concrete design decisions: what gets a primary key, what becomes a `record`, what is immutable, what is compared by value, and what protects its own invariants.

## Why It Exists

Without the Entity/Value Object distinction, domain models default to two pathological extremes:

1. **Everything is a primitive** — `decimal Total`, `string Email`, `string Currency`. The compiler can't tell USD from EUR, valid emails from junk, or a price from a quantity. Bugs surface in production: "we sold a $100 item for €100".
2. **Everything is an entity** — `Address` gets a primary key, an ID, and a row in a table. `Money` is tracked, queried, joined. The data model becomes bloated; equality checks become identity checks; conceptually identical values are treated as different objects.

The Entity/Value Object pattern, codified by Eric Evans (2003), exists to:

- Make **identity** an explicit modeling decision, not a database accident.
- Replace **primitive obsession** with self-validating, type-safe wrappers.
- Move **invariants** onto the type that owns them (a `Money` constructor that rejects negative amounts; an `EmailAddress` constructor that rejects bad strings).
- Let the **compiler** prevent whole categories of bugs (you literally cannot pass `Money(USD)` where `Money(EUR)` is expected).

## When To Use It

Use **entities** for:

- Things the business tracks individually over time: `Order`, `Customer`, `Invoice`, `Account`, `Shipment`, `User`, `Subscription`.
- Anything that has lifecycle state changes (`Draft → Submitted → Approved`).
- Anything you query by ID (`GET /api/orders/{id}`).

Use **value objects** for:

- Concepts with no independent identity: `Money`, `Address`, `EmailAddress`, `PhoneNumber`, `DateRange`, `Coordinates`, `Percentage`, `Sku`, `Color`.
- Anything you'd happily replace with an equal-valued instance (`order.ShippingAddress = newAddress`).
- Strongly typed primitives ("typed IDs" like `OrderId`, `CustomerId`).

**Do not use either as a substitute for:**

- DTOs flowing in and out of your API — those are anemic by design (HTTP serialization shape).
- EF Core "owned types" or migration helpers — those are persistence shapes, not domain concepts.
- Configuration POCOs (`StripeOptions`) — those are bound from `IConfiguration`, not domain values.

## Why It Is Important

In a .NET backend, the distinction directly drives:

1. **Type safety.** A method `Charge(Money amount, EmailAddress receipt)` cannot be called with swapped arguments. `Charge(decimal, string)` can.
2. **Invariant locality.** `Money`'s constructor rejects negative amounts and mismatched currencies once; every caller benefits. With `decimal`, every method must re-validate.
3. **Equality semantics that match the domain.** `customerA.Equals(customerB)` checks IDs (right). `address1.Equals(address2)` checks every field (also right). With anemic types, both wrongly default to reference equality.
4. **Persistence clarity.** EF Core configures entities with a primary key and tracking; value objects map as **owned types** or **complex types** sharing the parent row — no extra table, no identity churn.
5. **Testability.** Value objects are pure data — perfect for property-based and parameterized tests. Entities focus on state transitions, which makes their tests behavior-first.

This pattern is the foundation under `[aggregates.md](aggregates.md)`, `[repositories.md](repositories.md)`, and the entire `Domain` layer of `[clean-architecture.md](clean-architecture.md)`.

## How It's Used in C# / .NET

### Modern C# idioms

| Concept           | Idiom                                            | Why                                              |
|-------------------|--------------------------------------------------|--------------------------------------------------|
| Value Object      | `sealed record` (positional or with properties)  | Free value equality, immutability, `with` expressions |
| Entity            | `sealed class` with `private set`                | Identity-based equality; controlled mutation     |
| Aggregate Root    | `sealed class` inheriting `AggregateRoot`        | Protects invariants of child entities/VOs        |
| Typed ID          | `readonly record struct OrderId(Guid Value)`     | Zero-allocation, type-safe identifiers           |
| Collections in entity | `private readonly List<T>` + `IReadOnlyList<T>` projection | Prevents external mutation                       |

### Value Object — `Money`

```csharp
public sealed record Money
{
    public decimal Amount { get; }
    public Currency Currency { get; }

    public Money(decimal amount, Currency currency)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount), "Money cannot be negative.");
        Amount = decimal.Round(amount, 2, MidpointRounding.ToEven);
        Currency = currency;
    }

    public static Money Zero(Currency c) => new(0m, c);

    public Money Add(Money other)
    {
        if (other.Currency != Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}.");
        return this with { Amount = Amount + other.Amount };
    }

    public static Money operator *(Money money, int multiplier)
        => money with { Amount = money.Amount * multiplier };
}

public enum Currency { Usd, Eur, Vnd }
```

Notice: validation in constructor, immutability via `record`, mutation via `with`-expressions, operator overloading for natural use.

### Value Object — `EmailAddress`

```csharp
public sealed record EmailAddress
{
    private static readonly Regex Pattern = new(@"^[^@\s]+@[^@\s]+\.[^@\s]+$", RegexOptions.Compiled);

    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !Pattern.IsMatch(value))
            throw new ArgumentException("Invalid email address.", nameof(value));
        Value = value.Trim().ToLowerInvariant();
    }

    public override string ToString() => Value;
    public static implicit operator string(EmailAddress e) => e.Value;
}
```

Now `void Send(EmailAddress to, ...)` cannot be called with an unvalidated string.

### Typed ID

```csharp
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}
```

A method `GetOrder(OrderId id)` cannot accept a `CustomerId` by mistake.

### Entity — `Customer`

```csharp
public sealed class Customer
{
    public CustomerId Id { get; private set; }
    public EmailAddress Email { get; private set; }
    public Address ShippingAddress { get; private set; }
    public DateTimeOffset RegisteredAt { get; private set; }

    private Customer() { } // EF Core

    public static Customer Register(EmailAddress email, Address shipping, TimeProvider clock) => new()
    {
        Id = OrderId.New() == default ? new CustomerId(Guid.NewGuid()) : new CustomerId(Guid.NewGuid()),
        Email = email,
        ShippingAddress = shipping,
        RegisteredAt = clock.GetUtcNow()
    };

    public void ChangeEmail(EmailAddress newEmail) => Email = newEmail;
    public void MoveTo(Address newAddress) => ShippingAddress = newAddress;

    public override bool Equals(object? obj) => obj is Customer c && c.Id == Id;
    public override int GetHashCode() => Id.GetHashCode();
}
```

Notice: identity-based equality, `private set` for controlled mutation, value objects as properties.

### EF Core mapping

```csharp
public sealed class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> b)
    {
        b.HasKey(c => c.Id);
        b.Property(c => c.Id).HasConversion(id => id.Value, v => new CustomerId(v));

        // Value object as owned type — stored on the same row
        b.OwnsOne(c => c.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("ShippingStreet");
            addr.Property(a => a.City).HasColumnName("ShippingCity");
            addr.Property(a => a.PostalCode).HasColumnName("ShippingPostalCode");
        });

        // EmailAddress as a converted scalar
        b.Property(c => c.Email)
         .HasConversion(e => e.Value, v => new EmailAddress(v))
         .HasColumnName("Email");
    }
}
```

In EF Core 8+, `ComplexProperty` is preferred for value objects that don't need their own collection semantics.

## Advantages

- **Type safety at compile time** — wrong currencies, swapped IDs, bad emails can't slip through.
- **Invariants in one place** — validation in the constructor, not duplicated across handlers.
- **Code reads like the domain** — `order.AddLine(productId, qty, price)` instead of nested primitives.
- **Free value equality** with `record` — no boilerplate.
- **Better testability** — value objects are deterministic, easy to parameterize.
- **Clearer EF Core mapping** — owned types/complex types keep the data model tidy.
- **Better refactoring tooling** — renaming a property of `Money` updates every consumer; renaming `decimal` is a global search.

## Disadvantages

- **More types upfront** — `Money`, `Currency`, `EmailAddress`, `PostalCode`, `OrderId` are extra files.
- **Serialization friction** — JSON/Swagger needs converters for typed IDs and complex VOs (often a one-liner per type).
- **Allocation cost** — `record` classes allocate; for hot paths consider `readonly record struct`.
- **EF Core mapping ceremony** — converters and `OwnsOne` per VO.
- **Over-modeling risk** — wrapping every string ("UserName", "OrderNumber") in a VO can produce noise.
- **Library boundaries** — third-party APIs (Stripe SDK) take primitives, forcing conversion at the edge.

## Common Mistakes

### 1. Primitive Obsession

```csharp
// BUG
public Task ChargeAsync(string customerId, decimal amount, string currency, string description) { /* ... */ }

await svc.ChargeAsync(orderId.ToString(), 100m, "USD", "Order 42"); // swapped customerId & orderId
```

Compiler is helpless. Replace with typed parameters:

```csharp
public Task ChargeAsync(CustomerId customer, Money amount, string description, CancellationToken ct);
```

### 2. Mutable Value Objects

```csharp
// BUG
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; } = "";
}

var price = new Money { Amount = 100, Currency = "USD" };
order.Total = price;
price.Amount = 9999; // order.Total silently changed!
```

Value objects must be immutable. Use `record` with init-only properties or read-only fields.

### 3. Entity Without Encapsulation

```csharp
// BUG: All public setters
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
}

order.Status = OrderStatus.Confirmed;     // bypasses rule "must have lines to confirm"
order.Lines.Add(...);                     // bypasses rule "no lines after confirmation"
```

`private set`, private collections, expose `IReadOnlyList<T>`.

### 4. Value Object With Identity

```csharp
// BUG
public class Address
{
    public Guid Id { get; set; }      // why does an address need an ID?
    public string Street { get; set; }
    public string City { get; set; }
}
```

`Address` has no identity in business terms — two addresses with the same fields are the same address. Drop the ID; map as an owned type.

### 5. Equals Done Wrong on Entity

```csharp
// BUG: Comparing entities by all fields
public class Customer
{
    public override bool Equals(object? obj) => obj is Customer c
        && c.Name == Name && c.Email == Email && c.RegisteredAt == RegisteredAt;
}
```

Now two customers with identical data are "equal" — they're not; they have different IDs. Compare by ID.

### 6. Forgetting EF Conversion for Typed IDs

```csharp
// BUG: EF tries to map OrderId as an unknown type
public OrderId Id { get; private set; }
```

Without `HasConversion`, EF either ignores it or throws. Configure a value converter once per typed ID.

### 7. Anemic Entity + Public Mutator Service

```csharp
public class OrderUpdater
{
    public void SetStatus(Order order, string status) => order.Status = status;
}
```

This re-introduces the anemic anti-pattern even though you have a value object. The behavior belongs on `Order`.

## Best Practices

- **Use `record` for value objects** — value equality is free and matches semantics.
- **Validate in the constructor.** A `Money` instance is always valid by virtue of existing.
- **Make value objects immutable.** No setters; use `with` for "modified copies".
- **Use typed IDs (`readonly record struct`)** to prevent ID-swap bugs across aggregates.
- **Encapsulate entity state.** `private set`, private collections, behavior methods.
- **Override `Equals`/`GetHashCode` for entities by ID** — or inherit from a base `Entity<TId>`.
- **Map value objects as owned types or complex properties** in EF Core, not separate tables.
- **Configure typed-ID converters once** via `ConfigureConventions` or a reflective scan.
- **Add System.Text.Json converters** for typed IDs so they serialize as strings, not nested objects.
- **Use parameterized tests** (`[Theory] [InlineData]`) on value objects to cover the validation logic.
- **Don't wrap trivial single-use strings** — a `Username` VO that never gets validated and is used once is noise.

## Related Concepts

- **[aggregates.md](aggregates.md)** — entities + value objects organized behind a consistency boundary.
- **[domain-driven-design.md](domain-driven-design.md)** — the wider framework these patterns sit inside.
- **[repositories.md](repositories.md)** — load and persist aggregate roots (entities).
- **[../csharp/records-and-immutability.md](../csharp/records-and-immutability.md)** — the C# 9+ feature that makes value objects natural.
- **[../data-access/entity-framework-core.md](../data-access/entity-framework-core.md)** — `OwnsOne`, `ComplexProperty`, value converters.
- **[solid-principles.md](solid-principles.md)** — encapsulation, SRP, and DIP all apply.
- **Anti-patterns**: Anemic Domain Model, Primitive Obsession, God Object.

## Real-World Usage

### Financial Platform

A lending platform uses `Money`, `InterestRate`, `LoanTerm` (a value object combining months and start date), and typed IDs (`BorrowerId`, `LoanId`). Currency mismatch bugs disappeared after the migration from `decimal`; a single regulator-requested rounding change (round-half-to-even) was made once in `Money` and applied to every calculation. The `Loan` entity controls state transitions (`Draft → Approved → Disbursed → Repaying → Closed`) and rejects illegal transitions in its methods.

### E-Commerce Checkout

The `Cart` aggregate root holds a list of `CartLine` entities (each with `ProductId`, `Quantity`, `UnitPrice` of type `Money`) and a `ShippingAddress` value object. Adding the same product twice merges quantities rather than duplicating lines — a rule enforced inside `Cart.Add(...)`. Tests use `new Cart().Add(productId, 2, new Money(10m, Currency.Usd))` directly — no setup, no mocks.

### Multi-Tenant SaaS

A `TenantId` typed ID is the safety net. Every query, every aggregate, every API uses `TenantId tenantId` instead of `Guid`. A junior engineer who writes `repo.GetById(orderId.Value)` gets a compile error if `orderId.Value` is a `Guid` and the method expects `OrderId`. This catches data-leak bugs at compile time, not at security review.

## Code Example — Before and After

### Before — Primitives, mutable bag of properties

```csharp
public class Order
{
    public Guid Id { get; set; }
    public string Status { get; set; } = "Draft";
    public decimal Total { get; set; }
    public string Currency { get; set; } = "USD";
    public string CustomerEmail { get; set; } = "";
    public string ShippingStreet { get; set; } = "";
    public string ShippingCity { get; set; } = "";
    public string ShippingZip { get; set; } = "";
    public List<OrderLine> Lines { get; set; } = new();
}

public class OrderLine
{
    public Guid Id { get; set; }
    public Guid ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

// Anywhere in the codebase:
order.Status = "Confirmed";              // typo not caught
order.Total = -50m;                       // negative total!
order.Currency = "USDD";                  // typo not caught
order.CustomerEmail = "definitely-not-an-email"; // not validated
```

### After — Entities, Value Objects, Typed IDs

```csharp
// Value Objects
public sealed record Money
{
    public decimal Amount { get; }
    public Currency Currency { get; }
    public Money(decimal amount, Currency currency)
    {
        if (amount < 0) throw new ArgumentOutOfRangeException(nameof(amount));
        Amount = decimal.Round(amount, 2, MidpointRounding.ToEven);
        Currency = currency;
    }
    public static Money Zero(Currency c) => new(0m, c);
    public Money Add(Money other)
    {
        if (other.Currency != Currency) throw new InvalidOperationException("Currency mismatch.");
        return this with { Amount = Amount + other.Amount };
    }
    public static Money operator *(Money m, int q) => m with { Amount = m.Amount * q };
}

public enum Currency { Usd, Eur, Vnd }

public sealed record EmailAddress
{
    public string Value { get; }
    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email.", nameof(value));
        Value = value.Trim().ToLowerInvariant();
    }
}

public sealed record Address(string Street, string City, string PostalCode, string Country);

// Typed IDs
public readonly record struct OrderId(Guid Value) { public static OrderId New() => new(Guid.NewGuid()); }
public readonly record struct ProductId(Guid Value);

public enum OrderStatus { Draft, Confirmed, Shipped, Cancelled }

// Entity (aggregate root)
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public OrderId Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public Currency Currency { get; private set; }
    public EmailAddress CustomerEmail { get; private set; } = null!;
    public Address ShippingAddress { get; private set; } = null!;
    public Money Total => _lines.Aggregate(Money.Zero(Currency), (acc, l) => acc.Add(l.LineTotal));
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() { }

    public static Order Create(EmailAddress customer, Address shipping, Currency currency)
    {
        return new Order
        {
            Id = OrderId.New(),
            Status = OrderStatus.Draft,
            Currency = currency,
            CustomerEmail = customer,
            ShippingAddress = shipping
        };
    }

    public void AddLine(ProductId product, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order is not editable.");
        if (unitPrice.Currency != Currency) throw new InvalidOperationException("Currency mismatch.");
        _lines.Add(new OrderLine(product, quantity, unitPrice));
    }

    public void Confirm()
    {
        if (_lines.Count == 0) throw new InvalidOperationException("Cannot confirm an empty order.");
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Already finalised.");
        Status = OrderStatus.Confirmed;
    }

    public override bool Equals(object? obj) => obj is Order o && o.Id == Id;
    public override int GetHashCode() => Id.GetHashCode();
}

// Entity child (only mutated through Order)
public sealed class OrderLine
{
    public ProductId ProductId { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money LineTotal => UnitPrice * Quantity;

    private OrderLine() { }
    internal OrderLine(ProductId productId, int quantity, Money unitPrice)
    {
        if (quantity <= 0) throw new ArgumentOutOfRangeException(nameof(quantity));
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }
}
```

Now negative totals, currency mismatches, bad emails, and illegal status transitions are *impossible* — not "validated", impossible.

## Interview Questions and Answers

### 1. What is the practical difference between an entity and a value object?

**Why this matters**: Tests foundational DDD vocabulary and design instinct.

**Answer**: An entity has identity that persists across time — two `Order` instances with the same data are still different orders. A value object is defined by its values and is interchangeable — two `Money(100, USD)` instances are equal. Entities have lifecycle (created, modified, archived) and behavior that mutates state; value objects are immutable, equality-by-value, and replaceable. The practical impact: entities have a primary key and an ID-based `Equals`; value objects are `record` types with no key, mapped as owned types in EF Core.

**Trade-off**: A few more types upfront. Saves debugging time when a primitive bug would have cost a postmortem.

**Real project**: Replacing `decimal Amount` and `string Currency` with `Money` eliminated three currency-mixing bugs caught in code review the same sprint.

### 2. Why use a `record` for value objects?

**Why this matters**: Tests modern C# fluency.

**Answer**: `record` (since C# 9) provides value-based equality, `GetHashCode`, `ToString`, and `with`-expressions for free. That matches the semantics of a value object: equal-by-value, immutable, and copy-on-modify. Before records, you had to write 30 lines of `Equals`/`GetHashCode`/operator overrides per type and they were easy to get wrong. For hot paths or zero-allocation needs, `readonly record struct` is even better.

**Trade-off**: `record class` allocates per instance; for very hot paths choose `readonly record struct`. Serialization needs converters for typed IDs.

**Real project**: Switching `Money` from a hand-written class to a `record` removed 40 lines of equality plumbing and made unit tests pass instantly.

### 3. How do you map a value object in EF Core without creating a new table?

**Why this matters**: Practical persistence question; many candidates know DDD theory but stumble on EF.

**Answer**: Two options:

- **`OwnsOne`** (EF Core 2+): `builder.OwnsOne(o => o.ShippingAddress, addr => { addr.Property(a => a.Street).HasColumnName("ShippingStreet"); ... });` — columns live on the parent table.
- **`ComplexProperty`** (EF Core 8+): preferred for true value semantics; no shadow key, no identity.

For single-property value objects like `EmailAddress(string Value)` or typed IDs `OrderId(Guid Value)`, use `HasConversion` to map to/from the underlying primitive.

**Trade-off**: `OwnsOne` was implemented as an owned entity (hidden FK, can't easily be shared). `ComplexProperty` fixes that — prefer it in EF Core 8+.

**Real project**: We migrated 17 owned types to `ComplexProperty` in EF Core 8 — startup time dropped because no shadow entities were tracked.

### 4. What is "primitive obsession" and why is it bad?

**Why this matters**: Identifies whether the candidate sees beyond compile success.

**Answer**: Primitive obsession is the habit of representing domain concepts with primitives — `decimal` for money, `string` for email, `Guid` for any ID. The compiler can't tell USD from EUR, valid email from junk, `OrderId` from `CustomerId`. Bugs surface in production. The fix is to wrap each concept in a small value object with validation in the constructor; the type system then prevents whole categories of bugs. The cost is small (a few extra files); the safety is large.

**Trade-off**: Over-wrapping creates noise. Wrap things the business names and validates ("Money", "EmailAddress", "Sku"). Don't wrap "Description" or "InternalNote".

**Real project**: Wrapping all our IDs as `readonly record struct` typed IDs caught a `tenantId/userId` swap at compile time during a refactor — would have been a data leak otherwise.

### 5. How do you override equality on an entity, and why?

**Why this matters**: Common interview question, often answered wrongly.

**Answer**: An entity is equal to another entity of the same type if their IDs are equal. Override `Equals(object?)` to check the runtime type and compare `Id`. Override `GetHashCode()` to return `Id.GetHashCode()`. Optionally implement `IEquatable<T>` for performance. Do **not** compare by all fields — two `Customer` instances with the same data are still different customers if they have different IDs. Conversely, the **same** customer loaded into two different `DbContext` instances must still be equal.

**Trade-off**: Hand-written equality is boilerplate; use a base `Entity<TId>` class that handles it once.

**Real project**: Using a base `Entity<TId>` removed ~200 lines of duplicated `Equals` plumbing across 25 aggregates.

### 6. When would a value object *not* fit a domain concept that looks like a "value"?

**Why this matters**: Tests judgment, not pattern memorization.

**Answer**: When the concept actually needs identity. `Order` looks like a value at first ("a snapshot of items and total") but the business cares about *this specific order #1234* across its lifecycle — refunds, returns, customer support — so it's an entity. Similarly, a `Reservation` for a flight seat has identity even though the data looks values-like, because it can be cancelled, modified, or transferred. Heuristic: if you ever say "the same X", it's an entity; if you say "a copy of X", it's a value object.

**Trade-off**: Sometimes the right answer is "both" — a value-object snapshot stored on an entity (e.g., `Order` keeps a `PriceSnapshot` value object capturing pricing at confirmation time).

**Real project**: We modeled `LoanTerms` (rate + months + start date) as a value object because the business never edits them — they only ever create a new `Loan` with new terms.

### 7. What happens when a value object has too many fields?

**Why this matters**: Probes pragmatism about over-modeling.

**Answer**: It usually means the value object is hiding two value objects. Break it apart — e.g., a `CustomerProfile` with 12 fields might split into `ContactInfo` (email, phone), `BillingAddress`, and `Preferences`. Sometimes the fields actually belong on the entity, not in a VO. The smell to watch for is "I only ever change one or two fields at a time" — that's a sign the VO is too coarse or that those fields are entity properties.

**Trade-off**: Too-fine VOs add ceremony. Aim for cohesive groups of values that the business names and always changes together.

**Real project**: A 9-field `ShippingPreferences` VO was split into `DeliveryMethod` + `DeliveryWindow` + `SpecialInstructions` — each was used (and tested) independently afterwards.

### 8. How do you handle JSON serialization of typed IDs like `OrderId(Guid Value)`?

**Why this matters**: A practical wiring question that bites every team.

**Answer**: Without a converter, `System.Text.Json` serializes `OrderId` as `{"value":"..."}` — ugly and breaks every consumer. Register a custom `JsonConverter<OrderId>` that writes/reads the underlying `Guid`. The same converter can be auto-registered via a `JsonConverterFactory` for every `readonly record struct` ending in `Id`. Also configure Swagger via `SchemaFilter` so OpenAPI shows the typed ID as a `string`/`uuid`, not a nested object.

**Trade-off**: Small upfront wiring per project. Pays back across every controller and consumer.

**Real project**: A converter factory automatically handled 22 typed IDs — adding a new one required zero serialization code.

## Summary Checklist

- [ ] I can define entity vs value object and give a backend example of each.
- [ ] I can spot primitive obsession and wrap concepts as value objects.
- [ ] I can implement `Money`, `EmailAddress`, and typed IDs using `record`/`readonly record struct`.
- [ ] I can override `Equals` and `GetHashCode` on an entity by ID and on a value object by value.
- [ ] I can map value objects in EF Core via `OwnsOne` / `ComplexProperty` and typed IDs via `HasConversion`.
- [ ] I can encapsulate entity state with `private set`, private collections, and behavior methods.
- [ ] I can recognise when a candidate "value" actually needs identity and is an entity.
- [ ] I can register custom JSON converters for typed IDs.
- [ ] I can refactor an anemic, primitive-heavy model into rich, type-safe entities and value objects.
- [ ] I know when *not* to wrap a primitive and avoid noise.
