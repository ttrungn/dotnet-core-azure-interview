# Aggregates

## What It Is

An **aggregate** is a cluster of domain objects — one or more entities and value objects — that are treated as a single unit for the purpose of data changes. Every aggregate has exactly one **aggregate root**: an entity that is the only object outside code is allowed to hold a reference to. All access to the inner objects goes *through* the root.

Two rules give the concept its power:

1. **Invariants are enforced by the root.** The aggregate is responsible for keeping its internal state consistent. An invariant like *"the sum of order line totals equals the order total"* lives inside the `Order`, not in a controller or a database trigger.
2. **The aggregate is the transactional consistency boundary.** Within one aggregate, changes commit atomically. Between aggregates, consistency is **eventual** — coordinated through domain events, sagas, and the outbox pattern.

```csharp
public sealed class Order              // aggregate root
{
    private readonly List<OrderLine> _lines = new();  // child entity, no external access

    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    public byte[] RowVersion { get; private set; } = default!; // concurrency token

    public static Order Create(CustomerId customerId, IReadOnlyList<OrderLineDto> lines)
    {
        if (lines.Count == 0) throw new DomainException("Order must have at least one line");
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            Total = Money.Zero("USD")
        };
        foreach (var l in lines) order.AddLine(l.Sku, l.Quantity, l.UnitPrice);
        return order;
    }

    public void AddLine(Sku sku, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot modify a non-pending order");
        if (quantity <= 0) throw new DomainException("Quantity must be positive");

        var line = new OrderLine(Id, sku, quantity, unitPrice);
        _lines.Add(line);
        Total = _lines.Aggregate(Money.Zero(unitPrice.Currency), (sum, l) => sum + l.LineTotal);
    }

    public Result Confirm(PaymentIntent intent)
    {
        if (Status != OrderStatus.Pending) return Result.Conflict("Order already confirmed");
        if (_lines.Count == 0) return Result.Invalid("Empty order");
        Status = OrderStatus.Confirmed;
        // Domain event for downstream side effects
        _events.Add(new OrderConfirmedEvent(Id, CustomerId, Total, intent.Id));
        return Result.Ok();
    }
}
```

`OrderLine` is an **entity inside the aggregate** — it has identity (its `Id`) but no external repository. To change a line's quantity, callers call `order.ChangeLineQuantity(lineId, newQty)`. They never touch `OrderLine` directly.

## Why It Exists

Before DDD formalized aggregates (Eric Evans, 2003), the most common domain modeling approach in .NET Framework codebases was a flat object graph: `Customer` had a list of `Orders`, each `Order` had a list of `OrderLines`, each `OrderLine` had a `Product` reference, and any layer could mutate any of them. That produced predictable disasters:

- Two requests on the same order: one added a line, the other confirmed the order. They saved at the same time. The confirmed total did not match the actual lines. *"Customer charged $0 for a $200 order."*
- A controller updated `OrderLine.Quantity` directly without checking the parent order's status, so confirmed orders silently changed.
- A "save customer" call cascaded through every related order, every order line, every product reference — generating thousands of unnecessary `UPDATE` statements.
- Cross-table invariants (*"a confirmed order's lines cannot change"*) were enforced by database triggers, scattered through stored procedures, or simply violated.

Aggregates solve this by **defining a tight consistency boundary** and a single mutator. Outside callers cannot bypass invariants because they cannot reach inside.

## When To Use It

**Use aggregates when:**

- You have a real **domain model** with non-trivial business rules — orders, payments, subscriptions, claims, contracts.
- The system has **concurrent users** who can mutate related data, and you need a clear boundary for optimistic concurrency control.
- You apply **DDD / Clean Architecture** and want the domain to be a first-class citizen, not a passive data bag.
- You are using **event sourcing** — the aggregate is the natural unit of event streams.

**Do not force aggregates when:**

- The system is essentially **CRUD over reference data** (product catalog admin tool, lookup tables). A direct `DbContext`/`Dapper` approach is cleaner.
- The "aggregate" would have a single property and no behavior. That is a record, not an aggregate.
- You are building **read models** for reporting. Use flat DTOs and projections; aggregates are a write-side concept.
- The team is unfamiliar with DDD and the system is small. Premature aggregation creates ceremony without benefit.

## Why It Is Important

In production .NET / Azure systems, well-designed aggregates drive five outcomes:

1. **Invariants cannot be bypassed.** Any path that loads and saves an `Order` goes through the same domain methods. There is no second way to corrupt the data.
2. **Concurrency is manageable.** A single `RowVersion` per aggregate gives you optimistic concurrency that maps cleanly to a single SQL row's `[Timestamp]`/`xmin`. Conflicts are detected and retryable.
3. **Transactional scope is predictable.** A use case modifies *one* aggregate; the Unit of Work commits *one* row family. Locks are small and fast.
4. **Bounded blast radius for change.** Renaming `OrderLine.UnitPrice` to `OrderLine.PriceCents` is a refactor inside `Order`. The application layer, repositories, and tests all keep working because the aggregate's public methods did not change.
5. **Microservice boundaries align naturally.** An aggregate that consistently needs its own scaling, ownership, and data store is a strong candidate for becoming its own bounded context — and later its own microservice.

The pattern's value comes from a single discipline: **all writes go through the root.** Once that rule is enforced, an entire class of concurrency, consistency, and corruption bugs simply cannot occur.

## How It's Used in C# / .NET

### 1. Modeling the aggregate root and child entities

```csharp
// Domain layer — no EF Core
public sealed class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderLine> _lines = new();
    private readonly List<IDomainEvent> _events = new();

    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset? ConfirmedAt { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();

    private Order() { } // EF Core requires a parameterless ctor

    public void ChangeLineQuantity(OrderLineId lineId, int newQuantity)
    {
        if (Status != OrderStatus.Pending) throw new DomainException("Cannot modify confirmed order");
        var line = _lines.FirstOrDefault(l => l.Id == lineId)
                   ?? throw new DomainException("Line not found");
        line.SetQuantity(newQuantity); // delegated; OrderLine validates positivity
        Recalculate();
    }

    public void RemoveLine(OrderLineId lineId) { /* analogous */ }

    private void Recalculate() =>
        Total = _lines.Aggregate(Money.Zero(Total.Currency), (s, l) => s + l.LineTotal);

    public void ClearDomainEvents() => _events.Clear();
}

public sealed class OrderLine : Entity<OrderLineId>
{
    public OrderId OrderId { get; private set; }
    public Sku Sku { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money LineTotal => UnitPrice * Quantity;

    internal OrderLine(OrderId orderId, Sku sku, int quantity, Money unitPrice)
    {
        if (quantity <= 0) throw new DomainException("Quantity must be positive");
        Id = OrderLineId.New();
        OrderId = orderId;
        Sku = sku;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }

    internal void SetQuantity(int quantity)
    {
        if (quantity <= 0) throw new DomainException("Quantity must be positive");
        Quantity = quantity;
    }
}
```

`OrderLine` constructors and mutators are `internal` so only `Order` (in the same assembly) can call them.

### 2. Value objects vs entities

```csharp
public readonly record struct Sku(string Value)
{
    public static Sku Parse(string s) =>
        Regex.IsMatch(s, "^[A-Z0-9-]{3,20}$") ? new Sku(s) : throw new DomainException("Invalid SKU");
}

public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);
    public static Money operator +(Money a, Money b) =>
        a.Currency == b.Currency ? new Money(a.Amount + b.Amount, a.Currency)
                                 : throw new DomainException("Currency mismatch");
    public static Money operator *(Money m, int qty) => new(m.Amount * qty, m.Currency);
}
```

Value objects (`Money`, `Sku`, `Address`, `DateRange`) are immutable, compared by value, and free of identity. They form the vocabulary of the aggregate.

### 3. One repository per aggregate root

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Note: NO IOrderLineRepository. OrderLine is only accessible via Order.
```

### 4. EF Core mapping that respects the aggregate

```csharp
public sealed class OrderConfig : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.HasKey(o => o.Id);
        b.Property(o => o.Id).HasConversion(id => id.Value, v => new OrderId(v));
        b.Property(o => o.Status).HasConversion<string>();
        b.OwnsOne(o => o.Total, m =>
        {
            m.Property(x => x.Amount).HasColumnName("Total_Amount").HasColumnType("decimal(18,2)");
            m.Property(x => x.Currency).HasColumnName("Total_Currency").HasMaxLength(3);
        });
        b.Property<byte[]>("RowVersion").IsRowVersion(); // optimistic concurrency
        b.OwnsMany(o => o.Lines, l =>
        {
            l.WithOwner().HasForeignKey(x => x.OrderId);
            l.HasKey(x => x.Id);
            l.Property(x => x.Sku).HasConversion(s => s.Value, v => new Sku(v));
            l.OwnsOne(x => x.UnitPrice);
        });
        b.Ignore(o => o.DomainEvents); // not persisted
    }
}
```

`OwnsMany` tells EF Core that `OrderLine` belongs to `Order` — it has no separate `DbSet`, no separate repository, and its rows live and die with the parent order.

### 5. Domain events for cross-aggregate communication

When confirming an order needs to decrement inventory and notify shipping (both separate aggregates / services), the `Order` aggregate **does not call them directly**. It raises a domain event:

```csharp
public Result Confirm(PaymentIntent intent)
{
    // ... invariants ...
    Status = OrderStatus.Confirmed;
    _events.Add(new OrderConfirmedEvent(Id, CustomerId, Total, intent.Id, DateTimeOffset.UtcNow));
    return Result.Ok();
}
```

A `SaveChanges` interceptor or the application handler picks up `order.DomainEvents`, writes them to the **outbox** in the same transaction, and a dispatcher publishes them to Service Bus. Other aggregates (`Inventory`, `Shipment`) react in their own transactions — **eventual** consistency between aggregates, **immediate** consistency within each.

### 6. Loading patterns

```csharp
// Always load the full aggregate when mutating
public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
    _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);
```

You load the whole aggregate even if you only change one field. The cost of one extra include is far less than the cost of a corruption bug from partial loads.

## Advantages

- **Invariants in one place** — the root enforces them; callers cannot bypass.
- **Optimistic concurrency maps cleanly** to one `RowVersion` per aggregate.
- **Predictable transactional scope** — one aggregate = one transaction.
- **Smaller blast radius for refactoring** — internal restructuring stays internal.
- **Strong alignment with microservices** — bounded contexts grow naturally from clusters of aggregates.
- **Easy unit testing** — aggregates are pure C# with no infrastructure dependencies.

## Disadvantages

- **Learning curve** — teams new to DDD over-design aggregates or model them around tables rather than behavior.
- **Eventual consistency between aggregates** is harder for stakeholders to reason about than "everything in one transaction."
- **Loading cost** — eagerly loading children for every command can be expensive for very large aggregates.
- **Over-modeling risk** — turning every entity into an aggregate produces a sea of one-property classes.
- **EF Core constraints** — `OwnsMany`, navigation properties, and concurrency tokens require careful mapping.

## Common Mistakes

### 1. The God Aggregate

**Problem:** modeling `Customer` as an aggregate that contains every `Order`, every `Address`, every `PaymentMethod`. Loading one customer pulls 50 MB of data and locks half the database.

```csharp
// ❌
public class Customer
{
    public List<Order> Orders { get; }      // 500 orders
    public List<Address> Addresses { get; } // hundreds
    public List<PaymentMethod> Methods { get; }
    public List<SupportTicket> Tickets { get; }
}
```

**Fix:** make `Customer`, `Order`, `Address`, `SupportTicket` separate aggregates. Reference each other by ID, not by navigation property. Coordinate cross-aggregate workflows via domain events.

### 2. Touching child entities from outside the root

**Problem:** the application service mutates `OrderLine` directly, bypassing the aggregate's invariants.

```csharp
// ❌
var line = await _db.OrderLines.FindAsync(lineId);
line.Quantity = 99;
await _db.SaveChangesAsync();
```

The order total is now stale; the order may be confirmed yet a line was changed.

**Fix:** load the aggregate; call the aggregate method; save the aggregate.

```csharp
// ✅
var order = await _orders.GetByIdAsync(orderId, ct);
order.ChangeLineQuantity(lineId, 99);  // enforces "only pending orders" rule + recalculates total
await _uow.SaveChangesAsync(ct);
```

### 3. Cross-aggregate foreign keys with cascading updates

**Problem:** an aggregate references another aggregate by navigation property. A change in one cascades unintentionally to the other; transactions span both.

```csharp
// ❌
public class Order
{
    public Customer Customer { get; set; }  // navigation to another aggregate
}
```

**Fix:** reference by ID, not by object.

```csharp
// ✅
public class Order
{
    public CustomerId CustomerId { get; private set; }
}
```

Loading the customer's name for display is a **read-model** concern, not an aggregate concern.

### 4. Public setters on the root

**Problem:** `order.Status = OrderStatus.Confirmed` from anywhere skips every business rule the aggregate was meant to enforce.

**Fix:** properties are `{ get; private set; }`. State changes happen only through named domain methods: `order.Confirm()`, `order.Cancel(reason)`.

### 5. Trying to keep two aggregates immediately consistent

**Problem:** the handler updates `Order.Status = Shipped` and `Inventory.Quantity -= 1` in the same transaction *because the business expects them to be in lockstep*. Now any inventory edit blocks order processing.

**Fix:** accept that consistency between aggregates is **eventual**. `Order.Confirm()` raises `OrderConfirmedEvent`; an `InventoryReservation` handler reacts and decrements stock in a separate transaction. The outbox guarantees the event is published.

**When immediate consistency is genuinely required:** that is a strong signal the "two aggregates" are really one aggregate, and you should redesign accordingly.

### 6. Putting query / reporting methods on the aggregate

**Problem:** `Order.GetAllShippedSinceLastMonth()` — a query masquerading as a domain method. Aggregates load themselves now, just to answer a question about other aggregates.

**Fix:** queries are not aggregate behavior. Use a **query service** with a flat projection: `IOrderQueries.ListShippedSince(date)`.

## Best Practices

- **One repository per aggregate root, never per child entity.**
- **Public setters are forbidden on aggregates.** Use named methods for every state change.
- **Reference other aggregates by ID, not by navigation property.**
- **One aggregate per transaction.** Cross-aggregate consistency is eventual.
- **Use value objects** for concepts without identity (`Money`, `Address`, `Sku`).
- **Add a `RowVersion` / concurrency token** to every aggregate root.
- **Make child constructors `internal`** so only the root can construct them.
- **Raise domain events** for things other aggregates need to know about; publish them via the outbox.
- **Keep aggregates small** — usually 3–10 child entities, not 300.
- **Test aggregates as pure C#** — no EF Core, no mocks. They are the easiest part of the system to test.
- **Name methods after business intent** — `order.Confirm(intent)`, not `order.SetStatus("Confirmed")`.

## Related Concepts

- [architecture/domain-driven-design.md](domain-driven-design.md) — aggregates are a DDD building block.
- [architecture/entities-and-value-objects.md](entities-and-value-objects.md) — the components inside an aggregate.
- [architecture/repositories.md](repositories.md) — one repository per aggregate root.
- [architecture/unit-of-work.md](unit-of-work.md) — commits one aggregate per use case.
- [architecture/domain-events.md](domain-events.md) — how aggregates communicate.
- [architecture/outbox-pattern.md](outbox-pattern.md) — reliable publishing of aggregate events.
- [architecture/saga-pattern.md](saga-pattern.md) — coordinating multi-aggregate workflows.
- [data-access/optimistic-concurrency.md](../data-access/optimistic-concurrency.md) — `RowVersion` per aggregate.

## Real-World Usage

### ASP.NET Core ordering API

`Order` is the aggregate root. `OrderLine` is owned. `Customer`, `Inventory`, and `Shipment` are separate aggregates in their own bounded contexts. The Orders API serves the write side; a denormalized read model in Cosmos DB serves customer-facing order history queries without ever loading the write-side aggregates.

### Payment processing service

`Payment` is the aggregate root. `PaymentAttempt` (each retry against Stripe) is a child entity. `Refund` is part of the same aggregate. A single `RowVersion` protects against concurrent confirmations and refunds. When a payment succeeds, the aggregate raises `PaymentSucceededEvent`; the orders service consumes it via Service Bus and confirms the corresponding order.

### Inventory aggregate per warehouse

Each warehouse has its own `Inventory` aggregate keyed by `(WarehouseId, Sku)`. That keeps concurrency local — two warehouses can reserve stock simultaneously without contention. Reservations across warehouses are coordinated by a saga.

### Event-sourced subscription service

`Subscription` is an event-sourced aggregate. Every state change (created, plan changed, paused, cancelled) is appended as an event to its stream. The current state is rebuilt by replaying events. The aggregate boundary == the event stream boundary, which makes the model clean.

### Multi-tenant SaaS

Every aggregate carries a `TenantId`. EF Core query filters enforce it on every read. The aggregate is also the natural granularity for tenant-aware sharding — moving "tenant 47's orders" to a different database means moving complete aggregates, never partial ones.

## Code Example — Before and After

### Before — anemic model, flat object graph, no boundary

```csharp
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; }
    public decimal Total { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
    // public setters everywhere
}

public class OrderLine
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }
    public string Sku { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

// In a controller, somewhere
[HttpPost("api/orders/{id}/lines")]
public async Task<IActionResult> AddLine(Guid id, AddLineRequest req)
{
    var line = new OrderLine
    {
        Id = Guid.NewGuid(),
        OrderId = id,
        Sku = req.Sku,
        Quantity = req.Quantity,
        UnitPrice = req.UnitPrice
    };
    _db.OrderLines.Add(line);
    await _db.SaveChangesAsync();

    var order = await _db.Orders.FindAsync(id);
    order.Total = await _db.OrderLines.Where(l => l.OrderId == id).SumAsync(l => l.Quantity * l.UnitPrice);
    await _db.SaveChangesAsync();
    return Ok();
}
```

Problems: status is never checked, quantity is never validated, total is recalculated in two queries (race condition), the order is updated in a second transaction (partial state possible), and the same logic is duplicated wherever lines are added or removed.

### After — aggregate root enforces the rules

```csharp
public sealed class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderLine> _lines = new();
    private readonly List<IDomainEvent> _events = new();

    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();

    private Order() { }

    public static Order Create(CustomerId customerId, Currency currency)
    {
        var o = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            Total = Money.Zero(currency)
        };
        o._events.Add(new OrderCreatedEvent(o.Id, customerId));
        return o;
    }

    public void AddLine(Sku sku, int quantity, Money unitPrice)
    {
        EnsurePending();
        if (quantity <= 0) throw new DomainException("Quantity must be positive");
        if (unitPrice.Currency != Total.Currency) throw new DomainException("Currency mismatch");

        var existing = _lines.FirstOrDefault(l => l.Sku == sku);
        if (existing is not null) existing.IncreaseQuantity(quantity);
        else _lines.Add(new OrderLine(Id, sku, quantity, unitPrice));

        Recalculate();
    }

    public Result Confirm(PaymentIntent intent)
    {
        if (Status != OrderStatus.Pending) return Result.Conflict("Already confirmed");
        if (_lines.Count == 0) return Result.Invalid("Order is empty");
        Status = OrderStatus.Confirmed;
        _events.Add(new OrderConfirmedEvent(Id, CustomerId, Total, intent.Id));
        return Result.Ok();
    }

    private void EnsurePending()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Cannot modify a non-pending order");
    }

    private void Recalculate() =>
        Total = _lines.Aggregate(Money.Zero(Total.Currency), (s, l) => s + l.LineTotal);
}

// Application
public sealed class AddOrderLineHandler : IRequestHandler<AddOrderLineCommand, Result>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;

    public AddOrderLineHandler(IOrderRepository orders, IUnitOfWork uow)
    {
        _orders = orders; _uow = uow;
    }

    public async Task<Result> Handle(AddOrderLineCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct);
        if (order is null) return Result.NotFound();
        order.AddLine(cmd.Sku, cmd.Quantity, cmd.UnitPrice);
        await _uow.SaveChangesAsync(ct);
        return Result.Ok();
    }
}
```

Now: status is checked once, quantity is validated, the total is always consistent with the lines, the change commits atomically, and there is no other path that can add a line to an order.

## Interview Questions and Answers

### 1. How do you decide what belongs inside an aggregate vs in a separate one?

**Why this matters:** aggregate boundaries are the single most consequential modeling decision in DDD.

**Answer:** Two questions guide me. First, *"do these objects need to be transactionally consistent?"* If yes, same aggregate. If the rule is "eventually" or "soon," separate aggregates. Second, *"will I ever load object A without object B in this use case?"* If you frequently load one without the other, they probably do not belong together. I deliberately err toward **smaller aggregates** — a 1,000-entity aggregate is always wrong; a 5-entity aggregate is usually fine.

**Real project:** I once inherited a `Customer` aggregate containing every order, address, and ticket. Loading a customer to update their email pulled megabytes of data and locked rows used by other features. Splitting `Order` and `SupportTicket` into their own aggregates dropped p99 latency on the customer endpoint from 1.2s to 40ms.

### 2. Why is one aggregate per transaction a rule and not just a guideline?

**Why this matters:** ignoring this rule is the root cause of most concurrency disasters in DDD codebases.

**Answer:** Two reasons. **Locking and contention** — modifying multiple aggregates in one transaction holds locks across unrelated parts of the system, killing throughput. **Consistency boundary clarity** — when aggregates can mutate each other in one transaction, the boundary becomes meaningless and the invariants stop being enforceable. The rule forces you to use **domain events + outbox + eventual consistency** between aggregates, which is also the only model that scales horizontally and that translates cleanly to microservices later.

**Trade-off:** harder for product owners to reason about "the inventory should decrement immediately." We pay for that simplicity with concurrency bugs that never get found until production.

### 3. Should aggregates reference each other directly?

**Why this matters:** navigation properties between aggregates are the textbook mistake.

**Answer:** Only by **ID**, never by navigation property. `Order.CustomerId` is fine; `Order.Customer` is a code smell. The reasons: it prevents accidental cross-aggregate transactions, it keeps load sizes small, it makes serialization predictable, and it lets each aggregate evolve independently. If a use case needs the customer's name for display, it asks the read model — not the aggregate.

### 4. How do you handle optimistic concurrency on an aggregate?

**Why this matters:** real systems have concurrent writers; pretending otherwise leads to lost updates.

**Answer:** A single `RowVersion`/`[Timestamp]` column on the aggregate root, mapped via EF Core's `IsRowVersion()`. EF Core throws `DbUpdateConcurrencyException` at `SaveChangesAsync` when two writers collide. The handler catches it, reloads the aggregate, re-applies the command, and saves again — with a small retry budget. Because the entire aggregate commits together, one concurrency token protects all child entities and the root itself.

**Real project:** an order without `RowVersion` had a "lost update" bug — two CSRs editing the shipping address overwrote each other's changes. Adding the row version + 3-attempt retry surfaced conflicts cleanly as 409s and ended the silent data loss.

### 5. When are aggregates the wrong tool?

**Why this matters:** maturity to know when not to apply a pattern.

**Answer:** Three cases. **Pure CRUD over reference data** — a lookup-table admin UI does not need aggregates; it needs `DbSet<T>` and `SaveChanges`. **Read models for reporting** — DTOs and projections beat aggregates every time for queries. **Trivial domains** — if there are no real invariants beyond "field X cannot be null," skip the ceremony. I introduce aggregates when there is genuine domain behavior worth protecting; I delete them when the system has drifted into CRUD.

### 6. How do aggregates relate to microservice boundaries?

**Why this matters:** people confuse "one aggregate per service" with the actual relationship.

**Answer:** Aggregates do **not** map 1:1 to microservices. A **bounded context** — a cohesive cluster of aggregates that share a model and a team — maps to a microservice. Within one microservice you typically have 2–10 aggregates that change together. Splitting one aggregate across services destroys the consistency boundary; splitting one service per aggregate creates an explosion of services with chatty integration. The aggregate gives you the consistency boundary; the bounded context gives you the deployment boundary.

### 7. How do you publish a domain event reliably without distributed transactions?

**Why this matters:** ghost-data bugs (saved but never published) are the most common production failure in DDD systems.

**Answer:** The **outbox pattern**. Inside the same `SaveChangesAsync` that mutates the aggregate, an interceptor reads `aggregate.DomainEvents` and writes them as rows to an `OutboxMessages` table. Because the business change and the event rows are in the same transaction, no event is ever lost. A background dispatcher reads the outbox and publishes to Service Bus, marking each row sent. Service Bus delivers at least once; consumers are idempotent. No 2PC required.

### 8. How do you test an aggregate?

**Why this matters:** if you cannot test aggregates trivially, your design is broken.

**Answer:** Pure xUnit tests with **zero mocks**. Aggregates are plain C# — no `DbContext`, no `HttpClient`, no DI. I write tests like `[Fact] public void Confirming_empty_order_fails()` that arrange state, call the domain method, and assert on the resulting state and raised events. Hundreds of these tests run in a second. If a test needs a mock to exercise an aggregate, the aggregate has crept outside its boundary.

**Real project:** the `Order` aggregate had 80 behavioral tests covering every transition, every invariant, every domain event. A refactor that changed EF Core mappings broke zero of them, because the tests touched only domain code. That confidence is the whole point.

## Summary Checklist

- [ ] I model aggregates around behavior and invariants, not around tables.
- [ ] Every aggregate has exactly one root and that root is the only public entry point.
- [ ] Child entities have `internal` constructors; outside code cannot create them directly.
- [ ] Properties have `private set;`; state changes go through named domain methods.
- [ ] Aggregates reference each other by ID, never by navigation property.
- [ ] One aggregate is modified per transaction; cross-aggregate consistency is eventual via domain events + outbox.
- [ ] Each aggregate root has a `RowVersion` concurrency token and a retry strategy for `DbUpdateConcurrencyException`.
- [ ] One repository per aggregate root — no `IOrderLineRepository`.
- [ ] Aggregates can be unit-tested as pure C# with no mocks and no infrastructure.
- [ ] I know when to skip aggregates entirely (CRUD admin tools, read models, trivial domains).
