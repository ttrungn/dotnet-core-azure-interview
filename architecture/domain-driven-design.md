# Domain-Driven Design

## What It Is

Domain-Driven Design (DDD) is an approach to software design — introduced by Eric Evans in 2003 — that puts the **business domain** at the center of the codebase. Instead of organizing code around technical concerns (controllers, services, repositories) or database tables, DDD organizes code around concepts the business actually talks about: orders, invoices, shipments, refunds, line items, prices, tax regimes, fraud signals.

DDD splits into two halves:

- **Strategic DDD** — how to slice a large system. Bounded Contexts, Ubiquitous Language, Context Maps. This is about *where* to draw service boundaries and *who* owns which model.
- **Tactical DDD** — how to model inside a single context. Entities, Value Objects, Aggregates, Domain Services, Domain Events, Repositories, Factories. This is the day-to-day class design vocabulary.

In a .NET solution, DDD typically lives inside the `Domain` and (partly) `Application` projects of a Clean Architecture layout. The domain layer becomes a rich, behavior-driven set of types — not an anemic bag of properties.

## Why It Exists

In the early 2000s, most line-of-business apps used **anemic** domain models: classes with public getters/setters and zero behavior, driven by transaction script "services" full of `if` statements. Three pain points emerged:

1. **The business and developers used different words.** Stakeholders said "fulfilment", developers said "shipmentRow"; conversations leaked nuance at every translation.
2. **Rules spread everywhere.** "An order over $1,000 needs manager approval" lived in the controller, in the SQL trigger, in the email template, and in two background jobs — never in sync.
3. **Big systems became monoliths of confusion.** A `Customer` class in a 10-team codebase tried to satisfy Sales, Support, Billing, and Marketing. Any change broke something.

DDD exists to give teams a **shared language** with the business, a **place for rules to live** (the entity itself), and a **way to slice big systems** into autonomous contexts that can evolve independently.

## When To Use It

Use DDD for:

- Complex domains where business rules are non-trivial and change often: payments, lending, insurance, fulfilment, regulatory compliance, multi-region tax.
- Long-lived systems shared by multiple teams.
- Systems where requirements are discovered by talking with domain experts and where misunderstandings cost real money.
- Microservice estates where you need a principled way to draw service boundaries (each Bounded Context tends to become a service).

**Do not use DDD for:**

- CRUD systems where the domain *is* the database schema (a contacts app, a simple admin tool).
- Reporting / analytics workloads — those are about queries and projections, not aggregates.
- Throwaway prototypes.
- Pure infrastructure services (a proxy in front of Blob Storage, a metrics collector).

The cost of DDD is real: more classes, more discussions with the business, more discipline. Pay it where complexity dominates; skip it where it doesn't.

## Why It Is Important

In a backend that uses ASP.NET Core, EF Core, MediatR, and Azure, DDD gives you four concrete benefits:

1. **Rules in one place.** `order.Confirm()` enforces every invariant — empty orders cannot be confirmed, paid orders cannot be re-confirmed. No matter who calls it (REST, queue handler, CLI), the rule holds.
2. **Conversations that don't drift.** "Fulfilment" means the same thing in the standup, the JIRA ticket, the C# class, and the API contract. Bug reports become precise.
3. **Microservice boundaries that don't bleed.** A Bounded Context maps cleanly onto a service: Ordering, Catalog, Identity, Shipping. Each owns its data and its language.
4. **Testable business logic.** Rich entities are pure C# — no database, no HTTP, no Azure SDK — so they unit-test in milliseconds.

DDD is also what fills the `Domain` layer of `[clean-architecture.md](clean-architecture.md)` with substance. Without DDD, that layer is just data classes; with DDD, it's the heart of the system.

## How It's Used in C# / .NET

### Strategic DDD — slicing the system

For an e-commerce platform, the bounded contexts might look like:

| Bounded Context | Owns                                          | Sample Aggregates                     |
|-----------------|-----------------------------------------------|---------------------------------------|
| Catalog         | Products, categories, pricing rules           | `Product`, `Category`                 |
| Ordering        | Carts, orders, line items, order state        | `Order`, `Cart`                       |
| Identity        | Users, roles, authentication                  | `User`                                |
| Payments        | Charges, refunds, payment methods             | `Payment`, `Refund`                   |
| Shipping        | Shipments, carriers, tracking                 | `Shipment`                            |
| Inventory       | Stock levels, reservations                    | `StockItem`, `Reservation`            |

The word "Product" means different things across contexts: in Catalog it has price, description, images; in Inventory it has stock level and location; in Shipping it has weight and dimensions. Each context owns its own model — translation happens at the boundary (anti-corruption layer / integration events).

### Tactical DDD — the building blocks

- **Entity** — has identity. `Order`, `Customer`. Two orders with the same data are still different orders.
- **Value Object** — defined by its values, immutable, compared by value. `Money`, `Address`, `EmailAddress`.
- **Aggregate** — a cluster of entities and value objects with a single root entity (the **Aggregate Root**) that enforces invariants. Outside code only references the root.
- **Domain Service** — stateless behavior that doesn't naturally fit on an entity. `TaxCalculator`, `FraudScorer`.
- **Domain Event** — a business fact that happened. `OrderPlaced`, `PaymentCaptured`.
- **Repository** — collection-like access to aggregates (one per aggregate root).
- **Factory** — encapsulates complex creation of aggregates.

### .NET expression — folder structure

```
MyShop.Domain/
├── Common/
│   ├── DomainEvent.cs
│   ├── AggregateRoot.cs
│   └── ValueObject.cs
├── SharedKernel/
│   ├── Money.cs                       // sealed record Money(decimal Amount, Currency Currency)
│   ├── Currency.cs
│   └── Address.cs
└── Ordering/
    ├── Order.cs                       // sealed class Order : AggregateRoot
    ├── OrderLine.cs                   // entity, only mutated via Order
    ├── OrderStatus.cs                 // enum
    ├── IOrderRepository.cs
    └── Events/
        ├── OrderPlaced.cs             // sealed record OrderPlaced(Guid OrderId, Money Total) : IDomainEvent
        └── OrderCancelled.cs
```

### Libraries that fit

| Concern                           | Library                                  |
|-----------------------------------|------------------------------------------|
| Use case mediation                | MediatR / Wolverine                      |
| Domain event dispatch             | MediatR `INotification` after commit     |
| Persistence of aggregates         | EF Core (with `OwnsOne`/`ComplexType` for VOs), Marten for event sourcing |
| Integration events between contexts | MassTransit, Azure Service Bus, Azure Event Grid |
| Validation                        | FluentValidation (in Application layer)  |

### Canonical patterns in C#

**Value Object as a `record`** (modern C#):

```csharp
public sealed record Money(decimal Amount, Currency Currency)
{
    public static Money Zero(Currency c) => new(0m, c);
    public Money Add(Money other)
    {
        if (other.Currency != Currency)
            throw new InvalidOperationException("Currency mismatch.");
        return this with { Amount = Amount + other.Amount };
    }
}
```

`record` gives value equality, immutability, and `with`-expressions for free.

**Entity with `private set` for encapsulation**:

```csharp
public sealed class Order : AggregateRoot
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total => _lines.Aggregate(Money.Zero(Currency.Usd), (acc, l) => acc.Add(l.LineTotal));
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() { } // EF Core

    public static Order Create(Guid customerId)
    {
        var order = new Order { Id = Guid.NewGuid(), Status = OrderStatus.Draft };
        order.Raise(new OrderCreated(order.Id, customerId));
        return order;
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify a non-draft order.");
        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Confirm(TimeProvider clock)
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Cannot confirm an empty order.");
        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmed(Id, Total, clock.GetUtcNow()));
    }
}
```

External callers cannot mutate `_lines` directly — invariants are protected.

## Advantages

- **Shared language with the business** — fewer translation bugs.
- **Rules live with the data** — `order.Confirm()` cannot be bypassed by adding a new entry point.
- **Better microservice boundaries** — Bounded Contexts map naturally onto autonomous services.
- **Testable business logic** — pure C#, runs in milliseconds.
- **Pairs naturally with Clean Architecture, CQRS, and Event Sourcing**.
- **Improves long-term maintainability** — newcomers can read the Domain layer to understand the business.

## Disadvantages

- **Steep learning curve** — Entities vs. Value Objects vs. Aggregates is non-obvious at first.
- **Verbose for simple domains** — a contacts app does not need aggregates.
- **Requires sustained access to domain experts** — without them, you invent a fictional language.
- **EF Core friction** — mapping rich aggregates with private setters, value objects, and collections takes more configuration.
- **Over-modeling risk** — trying to make every concept an aggregate produces a brittle, fragmented domain.
- **Strategic DDD requires org buy-in** — Bounded Contexts that cut across team boundaries fail without leadership support.

## Common Mistakes

### 1. Anemic Domain Model

```csharp
// BUG: Entity is a data bag; rules live in services
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
}

public class OrderService
{
    public void Confirm(Order order)
    {
        if (order.Lines.Count == 0) throw new Exception();
        order.Status = OrderStatus.Confirmed;
    }
}
```

The rule "no empty confirmations" lives in `OrderService`, not on `Order`. Any new caller can mutate `Status` directly and bypass it.

**Fix**: Put behavior on the entity, hide setters:

```csharp
public sealed class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderLine> _lines = new();

    public void Confirm()
    {
        if (_lines.Count == 0) throw new InvalidOperationException("Empty order");
        Status = OrderStatus.Confirmed;
    }
}
```

### 2. One Big Aggregate That Loads Half the Database

```csharp
// BUG: Loading a Customer pulls in every order, invoice, ticket, refund
public class Customer : AggregateRoot
{
    public List<Order> Orders { get; set; } = new();
    public List<Invoice> Invoices { get; set; } = new();
    public List<SupportTicket> Tickets { get; set; } = new();
}
```

Now any change to a single ticket loads thousands of rows and contends for the same row in the database.

**Fix**: Each of these is its own aggregate. Reference by ID, load via its own repository.

### 3. Sharing One `Customer` Class Across Contexts

```csharp
// BUG: One Customer used by Sales, Support, Billing
public class Customer
{
    public string Name { get; set; }       // used by Sales
    public List<Ticket> Tickets { get; set; } = new(); // used by Support
    public string TaxId { get; set; }      // used by Billing
    public CreditLimit CreditLimit { get; set; } // used by Billing
}
```

Sales doesn't care about credit limit. Support doesn't care about tax ID. Every change risks breaking unrelated teams.

**Fix**: Each context has its own `Customer` with the attributes it needs. Translate at boundaries via integration events (`CustomerRegistered`, `CustomerCreditLimitChanged`).

### 4. Using Primitives Instead of Value Objects

```csharp
// BUG: Email and money as strings/decimals
public void RegisterCustomer(string email, decimal balance, string currency) { /* ... */ }
```

Caller can swap parameters, pass an unvalidated email, or mix currencies silently.

**Fix**:

```csharp
public void RegisterCustomer(EmailAddress email, Money openingBalance) { /* ... */ }
```

Validation happens in the value object's constructor; the compiler enforces correctness.

### 5. Domain Logic in Repositories

```csharp
// BUG: Repository computes business state
public class OrderRepository
{
    public async Task ConfirmAsync(Guid id, CancellationToken ct)
    {
        var o = await _db.Orders.FindAsync(id);
        if (o.Lines.Count > 0) o.Status = OrderStatus.Confirmed;
        await _db.SaveChangesAsync(ct);
    }
}
```

Repositories should fetch and persist aggregates — that is all. Business decisions belong on the aggregate.

**Fix**: `repository.GetByIdAsync(id)` → `order.Confirm()` → `repository.UpdateAsync(order)` (or rely on EF tracking + `UnitOfWork.Commit`).

### 6. Domain Events That Reach Across Contexts Synchronously

```csharp
// BUG: domain event handler calls another aggregate in the same transaction
public class OrderConfirmedHandler(IInventoryRepository inv) : INotificationHandler<OrderConfirmed>
{
    public async Task Handle(OrderConfirmed e, CancellationToken ct)
    {
        var stock = await inv.GetAsync(e.ProductId, ct);
        stock.Reserve(e.Quantity); // crosses aggregate + context boundary inside the txn
    }
}
```

This creates large transactions across aggregates and contexts.

**Fix**: Use domain events inside the context (handled in the same transaction by the same aggregate's neighbors), and **integration events** (published via the Outbox pattern) across contexts for eventual consistency.

### 7. Skipping Ubiquitous Language

The team uses "row" while the business says "line item". After six months, the codebase has `OrderRow`, `OrderLineItem`, `OrderEntry`, and `LineRecord` for the same concept.

**Fix**: Pick the business term, write it in the code, in the API, and in the docs. If the term changes, do a global rename.

## Best Practices

- **Run an Event Storming workshop** with domain experts before drawing boundaries. Big sticky notes beat UML.
- **Pick a Ubiquitous Language per context** and use it in code, schemas, API contracts, and conversations.
- **Keep aggregates small.** If two parts of an aggregate never change together, they are probably two aggregates.
- **Reference other aggregates by ID, not by navigation property.** Load them via their own repository.
- **Use `record` for value objects** in modern C#; value equality and immutability are free.
- **Use `private set` and private collections with read-only exposures** to protect invariants.
- **Use `OwnsOne` / `ComplexType` in EF Core** to map value objects without separate tables.
- **One repository per aggregate root.** Repositories return whole aggregates.
- **Raise domain events from the entity.** Dispatch them after the transaction commits (via MediatR + interceptor or the Outbox pattern).
- **Cross context with integration events**, not direct calls. Use Azure Service Bus / Event Grid / MassTransit.
- **Write architecture tests** that fail the build if Domain references EF Core, ASP.NET, or Azure SDKs.
- **Don't fight the framework into the domain.** If EF mapping requires a parameterless constructor, mark it `private` — that's a minor concession, not a violation.

## Related Concepts

- **[entities-and-value-objects.md](entities-and-value-objects.md)** — the tactical core of DDD.
- **[aggregates.md](aggregates.md)** — consistency boundaries within a context.
- **[repositories.md](repositories.md)** — collection-like persistence per aggregate.
- **[unit-of-work.md](unit-of-work.md)** — committing all changes in an aggregate atomically.
- **[application-services.md](application-services.md)** — the orchestration layer between API and Domain.
- **[clean-architecture.md](clean-architecture.md)** — the layering that DDD fills out.
- **[cqrs.md](cqrs.md)** — separating commands (use rich aggregates) from queries (use flat read models).
- **[domain-events.md](domain-events.md)** — within-context business facts.
- **[event-driven-architecture.md](event-driven-architecture.md)** — across-context integration events.
- **[outbox-pattern.md](outbox-pattern.md)** — reliable dispatch of integration events.
- **[saga-pattern.md](saga-pattern.md)** — long-running workflows across contexts.

## Real-World Usage

### Enterprise E-Commerce Platform

A retailer's platform is split into bounded contexts: Catalog, Ordering, Identity, Payments, Shipping, Inventory, Promotions. Each owns its database (often its own Azure SQL or Cosmos DB), publishes integration events to Service Bus, and exposes a REST API. Within Ordering, the `Order` aggregate enforces all checkout invariants: minimum quantity, max total, allowed payment method per region, promo eligibility. The Catalog context publishes `ProductPriceChanged` events; Ordering subscribes and updates its own pricing read model — it never queries Catalog's database.

### Microservice Decomposition

When breaking a monolith, DDD's Context Map is the planning tool. A team maps which contexts share which language, which integrate via API, which via events, and which are wholly independent. Each Bounded Context becomes a candidate microservice. Anti-corruption layers translate between contexts that use different vocabularies (e.g., a legacy "Customer" with 80 fields becomes a clean `Buyer` with 8).

### Azure-Hosted Workloads

Inside a single context running on Azure:

- The aggregate lives in C#, hosted in ASP.NET Core on App Service or an Azure Function.
- EF Core persists aggregates to Azure SQL. `OwnsOne` maps `Money` and `Address` to columns on the same table.
- After commit, an EF Core interceptor copies domain events into an `OutboxMessages` table.
- A background `OutboxPublisher` sends integration events to Azure Service Bus.
- Subscribers in other contexts (Inventory, Shipping) consume via `ServiceBusTrigger` in Azure Functions and update their own aggregates.

The domain code itself is pure C# — completely portable, independent of Azure.

## Code Example — Before and After

### Before — Anemic, single `Order` shared everywhere, primitives in signatures

```csharp
public class Order
{
    public Guid Id { get; set; }
    public string Status { get; set; } = "Draft";
    public decimal Total { get; set; }
    public string Currency { get; set; } = "USD";
    public List<OrderLine> Lines { get; set; } = new();
}

public class OrderLine
{
    public Guid ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

public class OrderService(AppDbContext db)
{
    public async Task<Guid> Place(Guid customerId, List<(Guid pid, int qty, decimal price)> items, string currency)
    {
        var order = new Order { Id = Guid.NewGuid(), Currency = currency };
        foreach (var (pid, qty, price) in items)
            order.Lines.Add(new OrderLine { ProductId = pid, Quantity = qty, Price = price });
        order.Total = order.Lines.Sum(l => l.Price * l.Quantity);
        order.Status = "Confirmed";
        db.Orders.Add(order);
        await db.SaveChangesAsync();
        return order.Id;
    }
}
```

Problems: any caller can set `Status` to anything, mix currencies, confirm an empty order, and the rule about "no empty confirmations" lives nowhere.

### After — Rich aggregate, value objects, domain events

```csharp
// Domain
public sealed record Money(decimal Amount, Currency Currency)
{
    public static Money Zero(Currency c) => new(0m, c);
    public Money Add(Money other)
    {
        if (other.Currency != Currency) throw new InvalidOperationException("Currency mismatch.");
        return this with { Amount = Amount + other.Amount };
    }
    public static Money operator *(Money m, int qty) => m with { Amount = m.Amount * qty };
}

public enum OrderStatus { Draft, Confirmed, Cancelled }

public sealed class OrderLine
{
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money LineTotal => UnitPrice * Quantity;

    private OrderLine() { }
    internal OrderLine(Guid productId, int quantity, Money unitPrice)
    {
        if (quantity <= 0) throw new ArgumentOutOfRangeException(nameof(quantity));
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }
}

public sealed class Order : AggregateRoot
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Currency Currency { get; private set; }
    public Money Total => _lines.Aggregate(Money.Zero(Currency), (acc, l) => acc.Add(l.LineTotal));
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() { }

    public static Order Create(Guid customerId, Currency currency)
    {
        var order = new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft, Currency = currency };
        order.Raise(new OrderCreated(order.Id, customerId));
        return order;
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order is not editable.");
        if (unitPrice.Currency != Currency) throw new InvalidOperationException("Line currency mismatch.");
        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Confirm(TimeProvider clock)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order already finalised.");
        if (_lines.Count == 0) throw new InvalidOperationException("Cannot confirm an empty order.");
        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmed(Id, Total, clock.GetUtcNow()));
    }
}

// Domain event (record)
public sealed record OrderConfirmed(Guid OrderId, Money Total, DateTimeOffset ConfirmedAt) : IDomainEvent;
```

Application handler stays thin:

```csharp
public sealed class PlaceOrderHandler(IOrderRepository repo, IUnitOfWork uow, TimeProvider clock)
    : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Currency);
        foreach (var item in cmd.Items)
            order.AddLine(item.ProductId, item.Quantity, new Money(item.UnitPrice, cmd.Currency));
        order.Confirm(clock);

        await repo.AddAsync(order, ct);
        await uow.CommitAsync(ct);  // triggers outbox dispatch of OrderConfirmed
        return order.Id;
    }
}
```

The aggregate enforces every rule; the handler orchestrates; the domain event flows out to other contexts via the Outbox.

## Interview Questions and Answers

### 1. What is the difference between an entity and a value object, and why does it matter?

**Why this matters**: Tests the foundational vocabulary of DDD.

**Answer**: An entity has an identity that persists over time — two `Order` instances with the same data are still different orders. A value object is defined entirely by its values and is interchangeable — two `Money(100, USD)` values are the same. Entities have lifecycle (created, mutated, archived); value objects are immutable. Mixing them up causes real bugs: using a plain `decimal` for money skips currency safety; treating `Address` as an entity creates needless rows and identity churn.

**Trade-off**: Value objects are slightly more code (a `record` and validation). The payoff is type safety and immutability.

**Real project**: Replacing `decimal` parameters with `Money` caught three currency-mixing bugs in code review the same week.

### 2. What is a Bounded Context, and how is it different from a microservice?

**Why this matters**: Confusing the two leads to badly drawn service boundaries.

**Answer**: A Bounded Context is a *linguistic* boundary — inside it, every term (Customer, Order, Product) has one precise meaning. A microservice is a *deployment* boundary. They often align (each context becomes a service), but not always: a small organization may keep two contexts in one deployable to start. The Bounded Context decision comes first; the service decision follows when the team is ready to own a separate deployable, database, and on-call rotation.

**Trade-off**: Drawing too many contexts early creates noise; drawing too few creates the "god model" that DDD was invented to avoid.

**Real project**: We kept Ordering and Fulfilment as two contexts in one ASP.NET Core service for the first 6 months, then split them when fulfilment grew its own team.

### 3. Why should an aggregate not load other aggregates?

**Why this matters**: A common over-modeling mistake.

**Answer**: Aggregates are consistency boundaries. Loading another aggregate via navigation pulls it into the same transaction, blurs ownership, and creates large reads. Reference by ID instead; if the other aggregate's state matters, either query its read model or react to its domain/integration events. This keeps transactions small, scales horizontally, and lets each aggregate evolve independently.

**Trade-off**: You lose the convenience of `order.Customer.Email`. You gain transactional clarity and the ability to shard.

**Real project**: Splitting the `Customer -> Orders -> Invoices` chain into three aggregates removed a chronic lock contention hotspot and made invoice imports 10x faster.

### 4. How do you persist a rich aggregate with private setters and value objects using EF Core?

**Why this matters**: A practical DDD-meets-EF question.

**Answer**: Use `EntityTypeConfiguration<T>` to map private fields and collections with `Metadata.FindNavigation(...).SetPropertyAccessMode(PropertyAccessMode.Field)`. Use `OwnsOne` for value objects (`builder.OwnsOne(o => o.ShippingAddress)`) or `ComplexType` (EF Core 8+) for stack-allocated VOs. Use a `private` parameterless constructor for EF. Keep mapping in Infrastructure so the Domain stays clean of EF references.

**Trade-off**: A small amount of EF-specific configuration per aggregate. Worth it to keep the domain rich and pure.

**Real project**: Mapping `Money` as a complex type let us keep `Amount` and `Currency` columns on the parent table without exposing setters.

### 5. Where do domain events live, and how do they differ from integration events?

**Why this matters**: Conflating them produces brittle, leaky designs.

**Answer**: Domain events are in-process facts that happened in this context (`OrderConfirmed`). They flow synchronously inside the same transaction or just after commit; consumers are other parts of the same context. Integration events are durable, schema-stable messages published to other contexts (`order.confirmed.v1`). They go through Azure Service Bus / Kafka, use a versioned contract, and are dispatched via the Outbox pattern for reliability. Mixing them produces tightly coupled contexts and "lost event" incidents.

**Trade-off**: Two event types feel redundant at first. Keeping them separate prevents the domain language leaking out and the integration schema leaking in.

**Real project**: Introducing an explicit `OrderConfirmedIntegrationEvent v1` decoupled it from the internal `OrderConfirmed` domain event — we could refactor the domain freely without breaking subscribers.

### 6. The team has one big `Customer` aggregate that owns orders, tickets, invoices, and preferences. What would you do?

**Why this matters**: Tests the "small aggregates" heuristic.

**Answer**: Split into four aggregates: `Customer` (profile and identity), `Order` (own root), `SupportTicket` (own root), `Invoice` (own root). Reference each other by `CustomerId`. Each gets its own repository. Cross-aggregate workflows use domain events ("invoice paid → mark order shipped") or sagas for longer ones. Watch the database: large aggregates often hide locking and read-amplification problems.

**Trade-off**: Splitting requires moving code and updating queries. Worth it for any aggregate that loads or contends on hundreds of rows per request.

**Real project**: We split a `Patient` aggregate into `Patient`, `Visit`, `Prescription`, `LabResult` — visit-only updates stopped blocking lab result inserts during peak hours.

### 7. Domain experts use the word "shipment" but the codebase uses "delivery". How do you fix it?

**Why this matters**: Tests the discipline behind Ubiquitous Language.

**Answer**: Pick one term — usually whichever the business uses — and rename it everywhere: code, schema (via a migration), API contract (via a deprecated alias if external), tickets, dashboards. Document the decision in a glossary inside the repo. Add a lint rule that flags the wrong term in new code. The cost of inconsistent language compounds; the cost of a global rename is one-time.

**Trade-off**: External API renames may need a deprecation window. Internal renames should happen immediately.

**Real project**: Renaming `Delivery` to `Shipment` across 80 files took an afternoon and ended three months of standup confusion.

### 8. When is DDD overkill, and what would you use instead?

**Why this matters**: A mature engineer knows the cost.

**Answer**: For pure CRUD apps (an internal contacts directory, a feature-flag admin tool), reporting systems, simple proxies, and prototypes. The domain *is* the database schema — there are no invariants to protect. Use a minimal ASP.NET Core project with EF Core and DTO-based responses; skip aggregates, repositories, and the four-project layout. The signal to upgrade is "we now have branching business rules that we want to test independently and protect from accidental change".

**Trade-off**: Starting too small and hitting the wall is easier to fix than starting too big and dragging ceremony through 2 years of development.

**Real project**: An internal admin tool stayed as a single project with EF Core for years; the customer-facing payment service used full DDD from day one and benefited.

## Summary Checklist

- [ ] I can distinguish strategic (Bounded Context, Ubiquitous Language) from tactical (Entity, VO, Aggregate) DDD.
- [ ] I can model a rich aggregate in C# with `private set`, private collections, and `record`-based value objects.
- [ ] I can persist an aggregate via EF Core using `OwnsOne` and private constructors without leaking EF into Domain.
- [ ] I can explain why aggregates should not navigate to other aggregates.
- [ ] I can separate domain events (in-context) from integration events (cross-context) and dispatch the latter via the Outbox pattern.
- [ ] I can draw a Context Map for a small e-commerce system and justify each boundary.
- [ ] I can spot and fix an anemic domain model.
- [ ] I can refactor a "god aggregate" into multiple smaller aggregates.
- [ ] I can apply Ubiquitous Language discipline in code, schema, and conversation.
- [ ] I know when DDD is overkill and use a simpler structure instead.
