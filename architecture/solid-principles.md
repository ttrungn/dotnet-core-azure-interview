# SOLID Principles

## What It Is

SOLID is an acronym for five object-oriented design principles introduced by Robert C. Martin in the early 2000s. Each letter targets a specific kind of pain that emerges when classes grow, teams grow, or requirements change:

1. **S — Single Responsibility Principle (SRP)** — a class should have one reason to change.
2. **O — Open/Closed Principle (OCP)** — a class should be open for extension but closed for modification.
3. **L — Liskov Substitution Principle (LSP)** — a subtype must be usable wherever its base type is expected, without surprises.
4. **I — Interface Segregation Principle (ISP)** — clients should not be forced to depend on methods they do not use.
5. **D — Dependency Inversion Principle (DIP)** — high-level modules should depend on abstractions, not on low-level modules.

SOLID is not a process or a framework. It is a set of design heuristics you apply to individual classes, modules, and service boundaries. In .NET they translate directly into the way you write controllers, MediatR handlers, EF Core repositories, and infrastructure adapters.

## Why It Exists

Before SOLID was popularized, large object-oriented codebases suffered from predictable rot:

- A single `OrderService` class kept growing — pricing, validation, persistence, email, audit, fraud checks — until any change risked breaking unrelated behavior.
- A new payment method (Apple Pay) required editing a giant `switch (paymentType)` block in five different services.
- A `SqlOrderRepository` subclass changed the meaning of `Add` (silently ignored duplicates) and broke callers that assumed standard behavior.
- A "god interface" like `IOrderService` with 40 methods forced every fake in every unit test to implement methods the test never used.
- Business logic in `CheckoutService` `new`-ed up `StripeClient` directly, making unit tests impossible without an Internet connection.

SOLID exists to give engineers a shared vocabulary and a set of concrete checks for these smells. It is the design layer that sits *under* patterns like Clean Architecture, Domain-Driven Design, and CQRS — those larger patterns are essentially SOLID applied at the module level.

## When To Use It

Use SOLID for:

- Any class with non-trivial behavior that more than one engineer will touch.
- Domain services, application/MediatR handlers, infrastructure adapters (payment, email, storage), background workers.
- Pluggable subsystems — places where a second implementation is realistic (a second payment provider, a second email channel, a second tenant strategy).
- Long-lived services where the cost of a wrong abstraction will be paid many times over the next two years.

**Do not over-apply SOLID for:**

- Pure DTOs (`CreateOrderRequest`, `OrderResponse`) — these are data containers; an `IOrderResponse` interface adds no value.
- Value objects (`Money`, `Address`, `EmailAddress`) — they have value semantics, not behavior worth abstracting.
- Throwaway scripts, migrations, or one-off seed data.
- Internal helpers used in exactly one place. Extracting an interface "just in case" creates dead abstractions.

The trap is the same with every principle: SRP can split a class into 17 micro-classes; OCP can turn a 3-line `switch` into a strategy registry; DIP can wrap `DateTime.UtcNow` in `IClock` even when there is no test for it. Apply each principle when the pain it removes is real, not theoretical.

## Why It Is Important

In a backend that uses ASP.NET Core, EF Core, MediatR, and Azure, SOLID is what determines whether the code is:

- **Testable** — DIP and ISP let you swap `StripePaymentGateway` for a fake in a unit test. Without them, you ship by hoping the integration test environment is healthy.
- **Safe to change** — SRP keeps changes local. If pricing rules change, you edit the pricing service; you do not also risk breaking email sending.
- **Extensible without forking** — OCP lets you add `AdyenPaymentGateway` without modifying `CheckoutHandler`. New code goes into new files; CI only re-tests what changed.
- **Substitutable** — LSP lets you wrap a real repository in `CachedOrderRepository` or `RetryingOrderRepository` without callers caring.
- **Readable for new engineers** — small, focused interfaces tell a new joiner what the module does in seconds.

SOLID also unlocks the patterns the rest of this repo relies on: dependency injection (`[../csharp/dependency-injection.md](../csharp/dependency-injection.md)`), Clean Architecture, the Repository pattern, and the Decorator pattern. Without SOLID, those patterns are just ceremony.

## How It's Used in C# / .NET

A typical .NET solution following SOLID looks like this:

```
MyShop.sln
├── src/
│   ├── MyShop.Domain/                 // Entities, value objects, domain services, interfaces
│   ├── MyShop.Application/            // Use cases (MediatR handlers), DTOs, validators
│   ├── MyShop.Infrastructure/         // EF Core, Stripe, Azure Service Bus, SendGrid
│   └── MyShop.Api/                    // ASP.NET Core controllers / minimal APIs
└── tests/
    ├── MyShop.UnitTests/
    └── MyShop.IntegrationTests/
```

Each layer applies SOLID concretely:

- **SRP** — one MediatR handler per use case (`PlaceOrderHandler`, `CancelOrderHandler`).
- **OCP** — new payment provider = a new `IPaymentGateway` implementation registered with the DI container; no existing code changes.
- **LSP** — `CachedProductRepository : IProductRepository` returns the same shape and contract as `EfProductRepository`.
- **ISP** — `IOrderReader`, `IOrderWriter` instead of one mega `IOrderRepository` with read, write, archive, and reporting methods.
- **DIP** — `Application` depends on `IPaymentGateway`; only `Infrastructure` knows about Stripe. Wiring happens once in `Program.cs`.

Libraries that lean on SOLID directly:

- **MediatR / Wolverine** — one handler per command/query (SRP).
- **FluentValidation** — one validator per request (SRP, OCP).
- **EF Core** with the Repository pattern — abstracts persistence (DIP).
- **MassTransit / NServiceBus** — one consumer per message type (SRP).
- **Polly** — decorator-style retry/circuit-breaker policies (OCP).
- **AutoMapper / Mapster** — keeps mapping out of business logic (SRP).

## Advantages

- **Predictable change radius** — most changes touch one or two files.
- **Cheap unit tests** — DIP lets you replace databases, message brokers, and HTTP clients with fakes.
- **Parallel team work** — different engineers can add new strategies (new tax calculator, new shipping carrier) without merge conflicts.
- **Better matching to DI containers** — `Microsoft.Extensions.DependencyInjection` is built around the assumption that consumers depend on interfaces.
- **Foundation for Clean Architecture and DDD** — both patterns assume SOLID is already there.
- **Easier to retire third-party code** — when Stripe gets replaced by Adyen, you write one new adapter.

## Disadvantages

- **Over-engineering risk** — naive SRP creates `IOrderIdGenerator`, `IOrderValidator`, `IOrderNumberFormatter`, where one method on `Order` would do.
- **Indirection cost** — "Go to definition" lands on `IPaymentGateway`, not the real code. New engineers need to learn the DI graph.
- **Interface explosion** — single-implementation interfaces clutter the codebase and slow code navigation.
- **Premature abstraction is hard to undo** — once 30 consumers depend on `IFooService`, removing it is an expensive refactor.
- **Subjective application** — "one reason to change" is context-dependent; teams must align on what counts as a responsibility.

## Common Mistakes

### 1. SRP Misread as "One Class, One Method"

```csharp
// BUG: Splitting "trivial" responsibilities into noise
public interface IOrderIdGenerator { Guid Generate(); }
public interface IOrderNumberFormatter { string Format(Guid id); }
public interface IOrderCreatedAtSetter { DateTimeOffset Now(); }
```

**Fix**: SRP is about *reasons to change*, not method count. Put these on the `Order` entity or use built-in primitives (`Guid.NewGuid()`, `TimeProvider`):

```csharp
public sealed class Order
{
    public Guid Id { get; } = Guid.NewGuid();
    public string Number { get; }
    public DateTimeOffset CreatedAt { get; }

    public Order(string number, TimeProvider clock)
    {
        Number = number;
        CreatedAt = clock.GetUtcNow();
    }
}
```

### 2. OCP Faked With a Giant Switch

```csharp
// BUG: Every new payment method requires editing this method
public PaymentResult Charge(string method, decimal amount) => method switch
{
    "stripe" => _stripe.Charge(amount),
    "adyen"  => _adyen.Charge(amount),
    "paypal" => _paypal.Charge(amount),
    _ => throw new NotSupportedException()
};
```

**Fix**: Strategy pattern + DI keyed services:

```csharp
public interface IPaymentGateway { Task<PaymentResult> ChargeAsync(Money amount, CancellationToken ct); }

services.AddKeyedScoped<IPaymentGateway, StripePaymentGateway>("stripe");
services.AddKeyedScoped<IPaymentGateway, AdyenPaymentGateway>("adyen");

public sealed class CheckoutHandler(IServiceProvider sp)
{
    public Task<PaymentResult> ChargeAsync(string method, Money amount, CancellationToken ct)
        => sp.GetRequiredKeyedService<IPaymentGateway>(method).ChargeAsync(amount, ct);
}
```

### 3. LSP Violation in a Repository Hierarchy

```csharp
// BUG: ReadOnlyOrderRepository throws when callers expect a normal repository
public class ReadOnlyOrderRepository : IOrderRepository
{
    public Order? GetById(Guid id) => _db.Orders.Find(id);
    public void Add(Order o) => throw new NotSupportedException(); // surprise!
}
```

**Fix**: Split the interface so types only implement what they can honor (this is ISP + LSP working together):

```csharp
public interface IOrderReader { Task<Order?> GetByIdAsync(Guid id, CancellationToken ct); }
public interface IOrderWriter { Task AddAsync(Order o, CancellationToken ct); }

public sealed class ReadOnlyOrderRepository : IOrderReader { /* read only */ }
public sealed class EfOrderRepository : IOrderReader, IOrderWriter { /* full */ }
```

### 4. Fat Service Interface Forcing Empty Methods Everywhere

```csharp
// BUG: A 14-method "god" interface
public interface IUserService
{
    Task<User?> GetAsync(Guid id);
    Task UpdateProfileAsync(...);
    Task ResetPasswordAsync(...);
    Task SendVerificationEmailAsync(...);
    Task ExportGdprBundleAsync(...);
    Task DeactivateAsync(...);
    // ... 8 more
}
```

Tests, decorators, and mocks must implement all 14 even when they care about one. Splitting follows ISP:

```csharp
public interface IUserReader { Task<User?> GetAsync(Guid id, CancellationToken ct); }
public interface IUserProfileWriter { Task UpdateProfileAsync(...); }
public interface IUserAuthenticator { Task ResetPasswordAsync(...); }
public interface IGdprExporter { Task<byte[]> ExportAsync(Guid userId, CancellationToken ct); }
```

### 5. DIP Inverted Only Halfway

```csharp
// BUG: Application code "depends on abstraction" but the abstraction is a leaky EF wrapper
public interface IOrderRepository
{
    IQueryable<Order> Query();   // leaks IQueryable -> leaks EF
}
```

Callers can now write `.Include(...).Where(...).ToListAsync()` from the Application layer, which means a Postgres migration or a switch to Dapper breaks every call site.

**Fix**: Expose business-meaningful methods, return materialized aggregates:

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetPendingForCustomerAsync(Guid customerId, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

### 6. Treating SOLID as a Code-Review Checklist

Reviewers reject PRs because "this isn't SRP enough" without pointing to a real reason to change. SOLID is a *guide*; if a class has 3 small responsibilities that always change together, leaving them together is fine.

**Fix**: Anchor every SOLID critique to a concrete change story: "When pricing changes, we'd also have to redeploy email — that's two reasons to change, please split."

### 7. Interface for Every Class, Even Singletons

```csharp
// BUG
public interface IOrderMapper { OrderDto Map(Order o); }
public sealed class OrderMapper : IOrderMapper { /* ... */ }
```

If there is one implementation and no test ever fakes it, the interface is dead weight. Make it `internal sealed` until a second implementation actually shows up.

## Best Practices

- **Drive each principle from real pain.** Apply SRP when changes hurt; apply OCP when adding strategies repeatedly modifies the same file; apply DIP when something blocks unit tests.
- **Use folder/project boundaries to enforce DIP.** `MyShop.Application` should not reference `MyShop.Infrastructure`. Add a build check (`InternalsVisibleTo`, project references) that fails the build if the rule is broken.
- **Prefer composition over inheritance.** Most "subtype" needs are better served by injecting a collaborator.
- **One MediatR handler per use case.** Natural SRP boundaries: `PlaceOrderHandler`, `RefundOrderHandler`, `ArchiveOrderHandler`.
- **Use Decorators for cross-cutting concerns** (caching, logging, retries) — that is OCP in action. Libraries like Scrutor make `services.Decorate<T>` trivial.
- **Keep interfaces small.** If you cannot fake an interface in one line for a test, it is probably too big.
- **Avoid `IOptions<T>` everywhere.** That is itself a DIP application: depend on typed configuration, not raw `IConfiguration`.
- **Pair SOLID with architecture tests.** Use NetArchTest or similar to enforce that the Domain project depends on nothing, that handlers depend on abstractions only, and that no `Microsoft.EntityFrameworkCore` reference leaks into Application.

## Related Concepts

- **[../csharp/dependency-injection.md](../csharp/dependency-injection.md)** — DIP's concrete implementation in .NET.
- **[clean-architecture.md](clean-architecture.md)** — applies SOLID at the architectural level.
- **[domain-driven-design.md](domain-driven-design.md)** — uses SOLID inside aggregates and domain services.
- **[repositories.md](repositories.md)** — the canonical DIP/ISP example in line-of-business code.
- **[application-services.md](application-services.md)** — the natural home of SRP-aligned use cases.
- **Decorator pattern** — OCP for cross-cutting concerns.
- **Strategy pattern** — OCP for swappable algorithms (payment providers, pricing rules).
- **Hexagonal / Ports and Adapters** — a generalization of DIP.
- **CQRS** — separates read and write concerns (SRP + ISP at the use-case level).

## Real-World Usage

### Enterprise Order Management Platform

A retailer's order platform processes millions of orders across countries. SOLID is enforced at multiple levels:

- Each country's tax rules live behind `ITaxCalculator`, registered as keyed services per region (OCP).
- `IOrderRepository` returns aggregates; query screens use a separate `IOrderReadModel` (CQRS-flavored ISP).
- Payment providers (Stripe in US, Adyen in EU, MercadoPago in LATAM) sit behind `IPaymentGateway` (DIP, OCP).
- Promotion rules use a chain of `IPromotionRule` objects evaluated in order (OCP + LSP).

### Microservices Architecture

In a microservice for shipping rates:

- `IShippingRateProvider` has implementations for UPS, FedEx, DHL — adding a new carrier is a new class + DI registration (OCP).
- `Application/Handlers/GetRatesHandler` depends only on `IShippingRateProvider` and `IRateCache` (DIP).
- Each carrier client lives in `Infrastructure/Carriers/Ups` and only implements what UPS exposes (ISP).
- Integration tests use `InMemoryShippingRateProvider` — the production code is unchanged.

### Azure-Hosted Workloads

Running on Azure App Service / Functions / AKS:

- `IBlobStorage` abstracts `BlobServiceClient` so unit tests can swap to an in-memory implementation (DIP).
- `IFeatureFlagProvider` is implemented by `AzureAppConfigurationFeatureFlagProvider`; a `FixedFeatureFlagProvider` is used in tests (DIP).
- `IClock` (or built-in `TimeProvider`) replaces `DateTime.UtcNow` so retry/expiry behavior is deterministic in tests.
- Background workers (`IHostedService`) depend on `IOutboxPublisher` rather than Azure Service Bus directly — when you move to Event Hubs, only the adapter changes.

## Code Example — Before and After

### Before — God service violating SRP, OCP, DIP

```csharp
public class OrderService
{
    public Guid PlaceOrder(OrderRequest req)
    {
        // 1. Validate (responsibility #1)
        if (req.Items.Count == 0) throw new Exception("empty");
        if (req.Email is null || !req.Email.Contains('@')) throw new Exception("bad email");

        // 2. Calculate pricing (responsibility #2)
        decimal total = req.Items.Sum(i => i.Price * i.Quantity);
        if (req.PromoCode == "SUMMER10") total *= 0.9m;

        // 3. Charge the card (responsibility #3) — hard dependency on Stripe
        var stripe = new StripeClient("sk_live_xxx");
        var charge = stripe.Charge(req.CardToken, total);
        if (!charge.Succeeded) throw new Exception("payment failed");

        // 4. Persist (responsibility #4) — hard dependency on SQL
        using var conn = new SqlConnection("Server=...;");
        conn.Open();
        var id = Guid.NewGuid();
        new SqlCommand($"INSERT INTO Orders ... VALUES ('{id}', {total})", conn).ExecuteNonQuery();

        // 5. Send email (responsibility #5)
        var smtp = new SmtpClient("smtp.contoso.com");
        smtp.Send(new MailMessage("noreply@contoso.com", req.Email) { Subject = "Order placed" });

        return id;
    }
}
```

Problems: 5 reasons to change, untestable, SQL injection, no way to add a second payment provider, no way to swap email.

### After — Each principle applied

```csharp
// Domain — SRP
public sealed class Order
{
    public Guid Id { get; } = Guid.NewGuid();
    public Money Total { get; private set; }
    public OrderStatus Status { get; private set; } = OrderStatus.PendingPayment;

    public Order(Money total) => Total = total;

    public void MarkPaid(PaymentReference reference) => Status = OrderStatus.Paid;
}

// Application — DIP + SRP (one handler per use case)
public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    IPaymentGateway payments,
    IEmailSender email,
    IUnitOfWork uow,
    IPricingService pricing,
    ILogger<PlaceOrderHandler> logger)
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var total = pricing.Calculate(cmd.Items, cmd.PromoCode);
        var order = new Order(total);

        var result = await payments.ChargeAsync(cmd.CardToken, total, ct);
        if (!result.Succeeded)
            throw new PaymentFailedException(result.DeclineReason);

        order.MarkPaid(new PaymentReference(result.TransactionId));
        await orders.AddAsync(order, ct);
        await uow.CommitAsync(ct);

        await email.SendAsync(cmd.Email, "Order placed", $"Total: {total}", ct);
        logger.LogInformation("Order {OrderId} placed for {Total}", order.Id, total);
        return order.Id;
    }
}

// Infrastructure — OCP (new providers added without touching application code)
public sealed class StripePaymentGateway : IPaymentGateway { /* ... */ }
public sealed class AdyenPaymentGateway  : IPaymentGateway { /* ... */ }

// ISP — small focused interfaces
public interface IOrderReader { Task<Order?> GetByIdAsync(Guid id, CancellationToken ct); }
public interface IOrderWriter { Task AddAsync(Order order, CancellationToken ct); }
public interface IOrderRepository : IOrderReader, IOrderWriter { }

// LSP — decorator that adds caching without breaking the contract
public sealed class CachedOrderReader(IOrderReader inner, IDistributedCache cache) : IOrderReader
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        // Same contract: returns the same shape, never throws unexpectedly
        return await inner.GetByIdAsync(id, ct);
    }
}
```

Outcome: each class has one reason to change, payment providers and email senders are pluggable, the handler is unit-testable in milliseconds with fake collaborators, and Stripe can be replaced by Adyen by changing one DI line.

## Interview Questions and Answers

### 1. Walk me through SOLID with one short backend example per letter.

**Why this matters**: Tests whether you treat SOLID as a coherent design discipline or as memorized acronyms.

**Answer**:
- **SRP**: Splitting a 5-responsibility `OrderService` into `PlaceOrderHandler`, `PricingService`, `EmailSender`, `OrderRepository`.
- **OCP**: Adding `AdyenPaymentGateway : IPaymentGateway` and a DI registration instead of editing `CheckoutHandler`.
- **LSP**: `CachedOrderReader` wraps `EfOrderReader` — both honor `IOrderReader.GetByIdAsync` returning the same shape.
- **ISP**: Splitting `IUserService` (14 methods) into `IUserReader`, `IUserAuthenticator`, `IGdprExporter`.
- **DIP**: `PlaceOrderHandler` depends on `IPaymentGateway`, not on `StripeClient`.

**Trade-off**: Each principle has a cost (more files, more interfaces). Apply when the change radius or testability pain is real.

**Real project**: Refactoring a 1,200-line `OrderService` god class let us add Apple Pay in 2 hours with no risk to existing flows.

### 2. A junior asks: "What is the difference between SRP and ISP?"

**Why this matters**: They often get confused. Senior engineers can articulate the distinction.

**Answer**: SRP is about a *class* — it should have one reason to change. ISP is about an *interface* — its consumers should not depend on methods they do not use. A class can satisfy SRP while exposing an interface that violates ISP if the interface is bloated for the consumer's needs. Example: `OrderRepository` can be SRP-clean (only persists orders) but expose 20 methods on `IOrderRepository`; a read-only controller that only needs `GetByIdAsync` is forced to depend on the other 19.

**Trade-off**: Splitting interfaces ad infinitum can produce too many files. Group methods that are always used together.

**Real project**: We split `IBookingService` into `IBookingReader`, `IBookingWriter`, `IBookingCancellation` — the reporting microservice only consumed `IBookingReader` and got a much smaller test surface.

### 3. How does the Dependency Inversion Principle relate to Dependency Injection?

**Why this matters**: A common confusion — DIP is a *principle*, DI is a *technique*.

**Answer**: DIP says "depend on abstractions, not concretions". DI is one way to honor it — instead of `new`-ing a dependency, the consumer accepts it through a constructor and a container wires the implementation. You can violate DIP while using DI (depending on a concrete class registered in the container) and you can honor DIP without a DI container (factories, manual composition). They reinforce each other but they are not the same thing.

**Trade-off**: A DI container only helps if you actually depend on interfaces. Registering only concrete types defeats the purpose.

**Real project**: Our team enabled `ValidateOnBuild` and added an architecture test that fails if Application code references a concrete type from Infrastructure. That made DIP enforceable, not aspirational.

### 4. Show me a Liskov Substitution violation you have seen in real code.

**Why this matters**: LSP is the most abstract principle — concrete examples prove understanding.

**Answer**: A common one is a "read-only" subclass of a writeable repository that throws on `Add`. Callers expecting `IOrderRepository.Add` now break at runtime. Another is overriding `Equals` in a derived entity to compare by business key while the base class compares by identity — collections silently behave wrong. The fix is usually to split the interface (ISP) or favor composition over inheritance.

**Trade-off**: Pure LSP can push you toward more interfaces and more types. Sometimes a runtime feature flag is simpler than a separate type.

**Real project**: We had `ArchivedOrderRepository : OrderRepository` that overrode `SaveChangesAsync` to no-op. It silently dropped updates. We split into `IArchivedOrderReader` (no writes) and removed the inheritance.

### 5. The team has 60 single-implementation interfaces. Is that good SOLID?

**Why this matters**: Tests whether the candidate distinguishes "principled" from "ceremonial".

**Answer**: No. Interfaces are valuable when (a) there is or will be a second implementation, (b) they form a boundary like HTTP/DB/queue, or (c) they enable testing seams. A `IOrderMapper` with one implementation that is never faked in tests is dead weight. The principle is "depend on abstractions where it matters", not "abstract everything". I would propose deleting interfaces with no second implementation and no test fake.

**Trade-off**: Removing interfaces is a refactor with downstream impact. Doing it after a code review of usage is worth the effort.

**Real project**: We removed 22 single-implementation interfaces from an internal API; "Go to definition" started landing on real code and onboarding got noticeably faster.

### 6. How would you use SOLID to add caching to an existing `IProductRepository`?

**Why this matters**: A practical OCP question — extending without modifying.

**Answer**: Use the Decorator pattern. Write `CachedProductRepository : IProductRepository` that wraps the existing `EfProductRepository`, checks `IDistributedCache` first, and falls back to the inner repository. Register it via DI (Scrutor's `services.Decorate<IProductRepository, CachedProductRepository>()` or manual factory). No existing code changes — OCP honored, LSP honored (same contract), DIP unchanged.

**Trade-off**: Decorators add a layer of indirection; debugging requires stepping through wrappers. Acceptable for cross-cutting concerns; overkill for one-off changes.

**Real project**: We added Redis caching + Polly retries to a flaky downstream API by decorating its client — no business code was touched, and the change rolled out behind a feature flag.

### 7. When is SRP wrong?

**Why this matters**: Probes nuance. Naive engineers treat SOLID as absolute.

**Answer**: When you split for splits' sake. If three pieces of behavior always change together (same business rule, same deploy unit), separating them just produces three classes that always change together, plus glue. A `MoneyFormatter`, `MoneyParser`, `MoneyComparer` trilogy is usually worse than `Money` with three methods. SRP should reduce *coupled* change, not produce more files.

**Trade-off**: This judgment is hard; it improves with experience reading and modifying production code. When unsure, keep it together until pain appears.

**Real project**: We merged three "responsibility" classes that changed in every PR back into a single `InvoiceCalculator` — the test suite shrank and reviews got faster.

### 8. How would you migrate a legacy 5,000-line `CustomerManager` toward SOLID without freezing delivery?

**Why this matters**: Tests realistic refactoring strategy, not green-field design.

**Answer**: 
1. Pin behavior with characterization tests (call the public methods, snapshot the output and the DB writes).
2. Identify the natural seams — registration, profile updates, GDPR export, password reset.
3. For each seam, extract a focused handler (`RegisterCustomerHandler`, etc.) called *from inside* `CustomerManager`. Existing callers do not change.
4. Move the relevant data access behind small interfaces (`ICustomerReader`, `ICustomerWriter`).
5. Once handlers are stable, change the public callers (controllers, jobs) to call the handlers directly and delete the manager method.
6. Ship every step behind a feature flag; observe production metrics between releases.

**Trade-off**: Slower than a "big-bang rewrite" on paper, but rewrites of large modules almost always slip and break edge cases the team forgot existed. Incremental wins compound.

**Real project**: Took us 6 weeks to dissolve a 4,000-line `BillingService` into 12 handlers — zero production incidents, and the team finally started accepting backlog tickets again.

## Summary Checklist

- [ ] I can name all five principles and give a backend example for each.
- [ ] I can distinguish SRP (class) from ISP (interface) with a concrete case.
- [ ] I can explain why DIP and DI are related but not identical.
- [ ] I can spot a Liskov violation in inheritance or in a "read-only" subtype that throws.
- [ ] I can show how OCP plus the Strategy or Decorator pattern lets me add features without modifying existing code.
- [ ] I know when *not* to apply each principle and can name the cost of over-abstraction.
- [ ] I can refactor a god class incrementally without freezing delivery.
- [ ] I can connect SOLID to Clean Architecture, DDD, MediatR, and EF Core in a .NET solution.
- [ ] I can write an architecture test (NetArchTest) that enforces a SOLID rule at build time.
- [ ] I can explain SOLID to a non-engineer as "fewer surprises when we change code".
