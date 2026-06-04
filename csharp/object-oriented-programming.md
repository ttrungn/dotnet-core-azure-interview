# Object-Oriented Programming in C#

## What It Is

Object-Oriented Programming (OOP) in C# is a paradigm where business logic is organized into **types** (classes, records, structs, interfaces) that bundle **data** with the **behavior** that operates on it, and that relate to each other through four mechanisms: **encapsulation**, **inheritance**, **polymorphism**, and **abstraction**. C# is multi-paradigm — modern code blends OOP with functional features (`record`, `with`, pattern matching, immutability) and procedural workhorses (DI, middleware, minimal APIs) — but at its core, a well-designed .NET backend is still a graph of objects with clear responsibilities.

```csharp
// Procedural — anyone can do anything to the data, invariants live nowhere
public class OrderData { public List<OrderLineData> Lines; public decimal Total; }
public static class OrderHelper
{
    public static void AddLine(OrderData o, OrderLineData line) { o.Lines.Add(line); o.Total += line.Price; }
}

// OOP — the object owns its data and enforces the rules
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;
    public Money Total { get; private set; } = Money.Zero("USD");

    public void AddLine(OrderLine line)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot edit a placed order");
        _lines.Add(line);
        Total = Total.Add(line.LineTotal);
    }
}
```

## Why It Exists

Real backend systems have **invariants** — rules that must always hold (an order's total equals the sum of its lines; a refund cannot exceed the captured amount; a `User` cannot be `Locked` and `Active` at the same time). Procedural code scatters those rules across helper functions; *anyone* can mutate the data and forget the rule. OOP gives one type **ownership** of its data and centralizes the rules — every change has to go through the methods the owner exposes.

OOP also exists because **real systems change**. A `PaymentProcessor` interface can be implemented by Stripe today, Adyen next quarter, and a mock in tests — the calling code never changes. Without polymorphism, you would have `if (provider == "stripe") ... else if (provider == "adyen") ...` in every method.

Finally, OOP gives the team a **shared vocabulary**. When someone says "the `Order` aggregate enforces the no-overdraft rule", every engineer knows where to look. See [architecture/domain-driven-design.md](../architecture/domain-driven-design.md) for how this scales to large domains.

## When To Use It

**Use for:**

- **Domain entities and aggregates** — `Order`, `Customer`, `Invoice`, `Subscription` — anywhere invariants matter.
- **Behavior families with multiple implementations** — payment gateways, notification channels, storage providers.
- **Stateful services with lifetime** — caches, connection pools, background processors.
- **Long-lived APIs you publish** — public types that other teams depend on benefit from explicit contracts.

**Do not over-use for:**

- **Pure data shapes on the wire** — use `record` DTOs, not class hierarchies. See [csharp/records-and-immutability.md](records-and-immutability.md).
- **One-off scripts and helpers** — `static` methods or local functions are clearer than ceremonial classes.
- **Configuration containers** — use `IOptions<T>` with `record` settings, not classes with behavior.
- **Forced inheritance for code reuse** — prefer composition.

The wrong question is "is this class OOP?" — every C# type technically is. The right question is "does this type own data + rules that belong together, and does it expose a small surface that keeps callers from breaking those rules?"

## Why It Is Important

When OOP is done well, code reviewers can point at a single class and say "this enforces the rule" — incidents shrink. When it's done badly, the same rule is duplicated in five services with three subtly different versions, and the production bug ticket reads "refund went through twice on Tuesday but only once on Friday".

In Azure-hosted systems, you also depend on OOP to make code **testable**. Stripe's SDK is real and expensive — you cannot call it from a unit test. By coding against `IPaymentGateway` (abstraction), you can substitute a `FakePaymentGateway` in tests, run the suite offline in CI, and still ship confidently. The whole DI container (see [csharp/dependency-injection.md](dependency-injection.md)) is built on this assumption.

## How It's Used in C# / .NET

### 1. Encapsulation — make invalid states unrepresentable

```csharp
public sealed class Invoice
{
    public Guid Id { get; }
    public Money Amount { get; }
    public InvoiceStatus Status { get; private set; }
    public DateTimeOffset? PaidAt { get; private set; }

    public Invoice(Guid id, Money amount)
    {
        if (amount.Value <= 0) throw new ArgumentException("Invoice must be positive", nameof(amount));
        Id = id;
        Amount = amount;
        Status = InvoiceStatus.Issued;
    }

    public void MarkPaid(DateTimeOffset paidAt)
    {
        if (Status != InvoiceStatus.Issued)
            throw new InvalidOperationException($"Cannot mark a {Status} invoice as paid");
        Status = InvoiceStatus.Paid;
        PaidAt = paidAt;
    }
}
```

The state transition lives in **one** method. There is no setter on `Status` for callers to misuse. Validation runs in the constructor — you cannot build a zero-amount invoice.

### 2. Inheritance — small, intentional hierarchies

```csharp
public abstract class PaymentProcessor
{
    protected readonly ILogger Logger;
    protected PaymentProcessor(ILogger logger) => Logger = logger;

    public async Task<PaymentResult> ChargeAsync(ChargeRequest req, CancellationToken ct)
    {
        Logger.LogInformation("Charging {Amount} via {Provider}", req.Amount, GetType().Name);
        var result = await DoChargeAsync(req, ct);
        Logger.LogInformation("Charge {Status} for {Id}", result.Status, result.TransactionId);
        return result;
    }

    protected abstract Task<PaymentResult> DoChargeAsync(ChargeRequest req, CancellationToken ct);
}

public sealed class StripeProcessor(IStripeClient stripe, ILogger<StripeProcessor> log) : PaymentProcessor(log)
{
    protected override async Task<PaymentResult> DoChargeAsync(ChargeRequest req, CancellationToken ct)
    {
        var charge = await stripe.Charges.CreateAsync(/* ... */, cancellationToken: ct);
        return PaymentResult.From(charge);
    }
}
```

The base captures **cross-cutting concerns** (logging, telemetry) and forces children to fill in the gateway-specific part — the **template method** pattern.

### 3. Polymorphism — one call site, many implementations

```csharp
public sealed class CheckoutService(IPaymentGateway gateway, IOrderRepository orders)
{
    public async Task<Result> PlaceAsync(Order order, CancellationToken ct)
    {
        var payment = await gateway.ChargeAsync(order.Total, order.CustomerId, ct);
        if (!payment.IsSuccess) return Result.Fail(payment.ErrorCode);
        order.MarkPaid(payment.TransactionId);
        await orders.SaveAsync(order, ct);
        return Result.Ok();
    }
}
```

`CheckoutService` does not know — and does not care — whether `IPaymentGateway` is Stripe, Adyen, or `FakePaymentGateway` in a test. DI wires the right one at runtime (see [csharp/dependency-injection.md](dependency-injection.md)).

### 4. Abstraction — interfaces describe the *what*, not the *how*

```csharp
public interface INotificationChannel
{
    string Name { get; }
    Task SendAsync(Notification message, CancellationToken ct);
}

public sealed class EmailChannel(SendGridClient sg) : INotificationChannel { /* ... */ }
public sealed class SmsChannel(AcsClient acs) : INotificationChannel { /* ... */ }
public sealed class PushChannel(NotificationHubClient nh) : INotificationChannel { /* ... */ }
```

Callers loop over `IEnumerable<INotificationChannel>` and never know which transport handled the message. Adding a new channel is a single new class — no `switch` to update.

### 5. Composition over inheritance

Inheritance is a strong, brittle coupling. Composition is loose.

```csharp
// Inheritance — locked into the order's lifetime, hard to test independently
public class OrderWithRetries : Order
{
    public new void Place() { /* retry logic */ base.Place(); }
}

// Composition — retry policy is a separate object, swappable per environment
public sealed class ResilientOrderService(Order order, IAsyncPolicy retry)
{
    public Task PlaceAsync(CancellationToken ct) => retry.ExecuteAsync(_ => Task.Run(order.Place, _), ct);
}
```

The C# community guidance since at least 2010 has been: **prefer composition, reach for inheritance when types share a true `is-a` relationship**.

### 6. `sealed`, `abstract`, `virtual`, `override`, `new`

```csharp
public abstract class Notification          // must be inherited
{
    public abstract string Subject { get; }                       // must be overridden
    public virtual int Priority => 5;                             // can be overridden
}

public sealed class InvoicePaidNotification : Notification        // cannot be inherited further
{
    public override string Subject => "Invoice paid";
    public override int Priority => 1;
}
```

Make classes `sealed` by default. Unseal only when inheritance is part of the design — otherwise reviewers waste cycles worrying about subclasses that never come.

### 7. Properties, init, required

```csharp
public sealed class Customer
{
    public required Guid Id { get; init; }
    public required string Email { get; init; }
    public CustomerTier Tier { get; private set; } = CustomerTier.Standard;

    public void Promote()
    {
        if (Tier == CustomerTier.Premium) return;
        Tier = CustomerTier.Premium;
    }
}
```

`required` + `init` give construction-time immutability without ceremony; behaviour-controlled mutation lives in named methods (`Promote`), not in setters.

### 8. Static members and extension methods

```csharp
public static class MoneyExtensions
{
    public static Money SumOf(this IEnumerable<OrderLine> lines, string currency) =>
        new(lines.Sum(l => l.Price.Value), currency);
}
```

Extension methods extend types you don't own. Use them sparingly — abuse leads to "where on earth is this method defined?" Often a real method on a real type is better.

## Advantages

- **Encapsulation** keeps invariants in one place; bugs become local.
- **Polymorphism** decouples callers from implementations — lets you swap Stripe for Adyen without touching `CheckoutService`.
- **Testability** — abstractions enable fakes, mocks, and in-memory implementations.
- **Vocabulary** — `Order`, `Invoice`, `Subscription` become shared nouns across product, design, and engineering.
- **Tooling** — IDEs leverage type hierarchies for navigation, refactoring, and code analysis.
- **Maintainability** — small, well-named classes are easier to change than 800-line `*Helper` files.

## Disadvantages

- **Over-design risk** — deep inheritance trees, factories of factories, "AbstractEnterpriseManagerProxy" syndrome.
- **Performance** — virtual calls cost more than direct calls; aggressive abstraction can hurt hot paths.
- **Hidden state** — mutable objects can be modified anywhere they're referenced; reasoning becomes harder.
- **Learning curve** — knowing *when* to extract an interface, when to use composition, when to seal — these take experience.
- **Premature abstraction** — extracting `IFoo` for the one class that exists today produces noise without benefit.

## Common Mistakes

### 1. Anemic domain model

```csharp
// BUG — Order is a bag of public setters; rules live in services
public class Order
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
}

public class OrderService
{
    public void AddLine(Order o, OrderLine line) // anyone can also bypass this and call o.Lines.Add
    {
        if (o.Status != OrderStatus.Draft) throw new();
        o.Lines.Add(line);
        o.Total += line.Price;
    }
}
```

**Fix**: move the rule into `Order` and make the data private. The service becomes a thin coordinator.

### 2. Deep inheritance chains

```csharp
// BUG — every change to BaseEntity breaks 12 subclasses; testing one requires constructing all parents
public abstract class BaseEntity { }
public abstract class AuditedEntity : BaseEntity { }
public abstract class SoftDeletableEntity : AuditedEntity { }
public abstract class TenantedEntity : SoftDeletableEntity { }
public class Order : TenantedEntity { }
```

**Fix**: composition with interfaces (`IAuditable`, `ISoftDeletable`) and EF Core conventions handle the same cross-cutting concerns without inheritance.

### 3. Leaky encapsulation through public mutable collections

See `Common Mistakes #5` in [csharp/fundamentals.md](fundamentals.md). A public `List<T>` lets any caller bypass invariants.

### 4. Inheriting from concrete classes you don't control

```csharp
// BUG — fragile base class; subtle changes in HttpClient internals break this
public class LoggingHttpClient : HttpClient { /* ... */ }
```

**Fix**: use a `DelegatingHandler` or wrap `HttpClient` in a composed service.

### 5. `new` to hide a method

```csharp
public class Parent { public virtual void Save() { } }
public class Child : Parent { public new void Save() { } } // hides — does NOT override
```

Calling `Save` on a `Parent` reference of a `Child` instance silently calls `Parent.Save`. Almost always a bug — use `override`, not `new`.

### 6. `protected` data instead of `protected` methods

Exposing fields as `protected` lets every subclass touch the data — same problem as `public` fields, just narrower. Expose `protected virtual` methods instead.

### 7. Premature interface extraction

A single-implementation `IFooService` paired with `FooService` is noise. Wait until the second implementation appears (real one + fake for tests counts as two — that's the threshold).

### 8. Mixing behavior into DTOs

```csharp
// BUG — request shape carries business logic
public class CreateOrderRequest
{
    public List<OrderLineDto> Lines { get; set; }
    public decimal Total => Lines.Sum(l => l.Price); // computed on the wire? in the controller? at the DB?
}
```

**Fix**: DTOs are dumb data carriers (`record` types); behavior belongs to entities or services.

## Best Practices

- **Make types `sealed` by default**; unseal intentionally.
- **Make data `private`**; expose behavior, not fields.
- **Prefer composition over inheritance**; inherit only for genuine `is-a` relationships.
- **Constructor-validate** so invalid objects cannot exist.
- **One reason to change per class** (Single Responsibility — see [architecture/solid-principles.md](../architecture/solid-principles.md)).
- **Inject dependencies, do not `new` them** — see [csharp/dependency-injection.md](dependency-injection.md).
- **Extract interfaces lazily** — at the second implementation, not the first.
- **Name types after domain concepts** (`Invoice`, `Subscription`) — not technical buckets (`InvoiceManager`, `OrderHelper`).
- **Encapsulate collections** behind read-only interfaces and named mutation methods.
- **Use `record` for value-equality types**; `class` for identity-equality entities.

## Related Concepts

- **[csharp/interfaces-and-abstractions.md](interfaces-and-abstractions.md)** — the contracts polymorphism is built on.
- **[csharp/records-and-immutability.md](records-and-immutability.md)** — when a `record` is the right shape.
- **[csharp/dependency-injection.md](dependency-injection.md)** — wires polymorphic implementations at runtime.
- **[csharp/generics.md](generics.md)** — parameterised OOP.
- **[architecture/solid-principles.md](../architecture/solid-principles.md)** — five rules that make OO codebases survive five years.
- **[architecture/domain-driven-design.md](../architecture/domain-driven-design.md)** — OOP for the domain layer.
- **[architecture/aggregates.md](../architecture/aggregates.md)** — encapsulation at the aggregate boundary.
- **[architecture/clean-architecture.md](../architecture/clean-architecture.md)** — keeps the domain free of infrastructure dependencies.

## Real-World Usage

### Notification dispatcher in an ASP.NET Core Web API

```csharp
public interface INotificationChannel
{
    NotificationKind Kind { get; }
    Task SendAsync(Notification n, CancellationToken ct);
}

public sealed class EmailChannel(ISendGridClient sg, ILogger<EmailChannel> log) : INotificationChannel
{
    public NotificationKind Kind => NotificationKind.Email;
    public async Task SendAsync(Notification n, CancellationToken ct)
    {
        log.LogInformation("Sending email to {To}", n.Recipient);
        await sg.SendEmailAsync(/* … */, ct);
    }
}

public sealed class SmsChannel(IAcsSmsClient acs, ILogger<SmsChannel> log) : INotificationChannel
{
    public NotificationKind Kind => NotificationKind.Sms;
    public async Task SendAsync(Notification n, CancellationToken ct)
    {
        log.LogInformation("Sending SMS to {To}", n.Recipient);
        await acs.SendAsync(/* … */, ct);
    }
}

public sealed class NotificationDispatcher(IEnumerable<INotificationChannel> channels)
{
    private readonly Dictionary<NotificationKind, INotificationChannel> _map =
        channels.ToDictionary(c => c.Kind);

    public Task DispatchAsync(Notification n, CancellationToken ct)
        => _map.TryGetValue(n.Kind, out var ch)
            ? ch.SendAsync(n, ct)
            : throw new NotSupportedException($"Channel {n.Kind} not registered");
}
```

Registration:

```csharp
builder.Services.AddSingleton<INotificationChannel, EmailChannel>();
builder.Services.AddSingleton<INotificationChannel, SmsChannel>();
builder.Services.AddSingleton<NotificationDispatcher>();
```

Adding **WhatsApp** means a new class implementing `INotificationChannel` and one registration line. `NotificationDispatcher` does not change.

### Azure Functions — Service Bus consumer using the same hierarchy

```csharp
public sealed class NotificationFunction(NotificationDispatcher dispatcher)
{
    [Function("HandleNotification")]
    public Task RunAsync(
        [ServiceBusTrigger("notifications", Connection = "SbConn")] Notification n,
        CancellationToken ct)
        => dispatcher.DispatchAsync(n, ct);
}
```

Same `INotificationChannel` family, same dispatcher — works in the Web API *and* the Function host because there is no infrastructure leak.

### Unit testing the dispatcher

```csharp
public sealed class FakeChannel(NotificationKind kind) : INotificationChannel
{
    public NotificationKind Kind => kind;
    public List<Notification> Sent { get; } = new();
    public Task SendAsync(Notification n, CancellationToken ct) { Sent.Add(n); return Task.CompletedTask; }
}

[Fact]
public async Task Dispatch_ForwardsToMatchingChannel()
{
    var email = new FakeChannel(NotificationKind.Email);
    var sms   = new FakeChannel(NotificationKind.Sms);
    var sut   = new NotificationDispatcher(new INotificationChannel[] { email, sms });

    await sut.DispatchAsync(new Notification { Kind = NotificationKind.Sms, Recipient = "+15555550100" }, default);

    sms.Sent.Should().HaveCount(1);
    email.Sent.Should().BeEmpty();
}
```

No mocking framework, no SendGrid, no Azure — fast, deterministic, runs in CI in milliseconds.

## Code Example — Before and After

### Before: god-class, public state, no polymorphism

```csharp
public class OrderManager
{
    public List<Order> Orders = new();

    public bool PlaceOrder(Order o, string paymentProvider, string cardNumber, decimal amount)
    {
        if (o.Lines.Count == 0) return false;
        if (paymentProvider == "stripe")
        {
            // hand-roll Stripe HTTP call
            // log to disk
            // update o.Status = "Paid"
        }
        else if (paymentProvider == "adyen")
        {
            // hand-roll Adyen call
        }
        Orders.Add(o);
        return true;
    }

    public decimal CalculateTax(Order o, string country) { /* big switch */ }
    public void SendEmail(Order o) { /* SMTP code */ }
    public void Refund(Order o) { /* yet more provider switches */ }
}
```

Problems:

- `Orders` is public — any caller can corrupt the list.
- Provider selection by string literal — adding a third provider touches every method.
- One class does payments, tax, email, persistence, and refunds — every change risks every behavior.
- No interfaces — impossible to unit test without real Stripe/SMTP.

### After: small classes, polymorphism, DI, encapsulated state

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(Money amount, CustomerToken token, CancellationToken ct);
    Task<PaymentResult> RefundAsync(string transactionId, Money amount, CancellationToken ct);
}

public sealed class StripeGateway(IStripeClient stripe, ILogger<StripeGateway> log) : IPaymentGateway { /* … */ }
public sealed class AdyenGateway (IAdyenClient adyen, ILogger<AdyenGateway>  log) : IPaymentGateway { /* … */ }

public sealed class Order
{
    private readonly List<OrderLine> _lines = new();
    public Guid Id { get; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;
    public Money Total => Money.SumOf(_lines, "USD");

    public Order(Guid id) { Id = id; Status = OrderStatus.Draft; }

    public void AddLine(OrderLine line)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException();
        _lines.Add(line);
    }

    public void MarkPaid(string transactionId)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException();
        Status = OrderStatus.Paid;
        TransactionId = transactionId;
    }

    public string? TransactionId { get; private set; }
}

public sealed class CheckoutService(
    IPaymentGateway gateway,
    IOrderRepository orders,
    INotificationChannel email,
    ILogger<CheckoutService> log)
{
    public async Task<Result> CheckoutAsync(Order order, CustomerToken token, CancellationToken ct)
    {
        var result = await gateway.ChargeAsync(order.Total, token, ct);
        if (!result.IsSuccess) return Result.Fail(result.ErrorCode);

        order.MarkPaid(result.TransactionId);
        await orders.SaveAsync(order, ct);

        await email.SendAsync(new Notification(order, NotificationKind.Email), ct);
        log.LogInformation("Checkout complete for {OrderId}", order.Id);
        return Result.Ok();
    }
}
```

What improved:

- **`Order` owns its rules** — no caller can corrupt the line list or status.
- **`IPaymentGateway`** — Stripe vs Adyen is a DI registration, not a `switch`.
- **Single Responsibility** — `CheckoutService` coordinates; it doesn't talk SMTP, HTTP, or SQL directly.
- **Testable** — every collaborator can be substituted with a fake.

## Interview Questions and Answers

### 1. What are the four pillars of OOP, and how do they show up in real .NET backend code?

**Why this matters**: This is the canonical interview opener; how concretely you answer reveals whether you've shipped systems or just read tutorials.

**Answer**:

- **Encapsulation** — `Order` owns its lines and total; mutation only via `AddLine`, which enforces invariants.
- **Inheritance** — `PaymentProcessor` base class adds logging/telemetry; `StripeProcessor`/`AdyenProcessor` implement `DoChargeAsync`.
- **Polymorphism** — `CheckoutService` works with `IPaymentGateway`; DI provides the right implementation per environment.
- **Abstraction** — `INotificationChannel` describes "send a message" without committing to email, SMS, or push.

**Trade-off**: Over-applying any pillar — especially inheritance — produces brittle code. The skill is judging when each is the right tool.

**Real project**: A checkout system used these four pillars to add a second payment provider in two weeks (one new class, one registration, zero changes to checkout flow or tests).

### 2. When do you choose composition over inheritance? Give a real example.

**Answer**: Always prefer composition; reach for inheritance when there's a true `is-a` relationship and a small, stable shared API.

**Example**: A `ResilientPaymentGateway` that wraps `IPaymentGateway` with retry/timeout policies via Polly is composition:

```csharp
public sealed class ResilientPaymentGateway(IPaymentGateway inner, IAsyncPolicy<PaymentResult> policy)
    : IPaymentGateway
{
    public Task<PaymentResult> ChargeAsync(Money amount, CustomerToken t, CancellationToken ct)
        => policy.ExecuteAsync(_ => inner.ChargeAsync(amount, t, _), ct);
    public Task<PaymentResult> RefundAsync(string id, Money a, CancellationToken ct)
        => policy.ExecuteAsync(_ => inner.RefundAsync(id, a, _), ct);
}
```

If we'd inherited from `StripeGateway` to add retries, we'd be locked to Stripe — Adyen wouldn't benefit.

### 3. How do you keep a domain entity from becoming an "anemic" data bag?

**Answer**:

- Make properties **`private set`** (or `init`); expose **methods** for state changes.
- Validate in the **constructor** so invalid instances are unreachable.
- Hide collections behind `IReadOnlyList<T>` with explicit `Add*`/`Remove*` methods.
- Keep behavior **close to the data it operates on**, not in `*Service` classes.

**Real project**: Moving the "no edits after placement" rule from `OrderService` into `Order.AddLine` eliminated three bug variants where service callers forgot to check status.

### 4. What does `sealed` do, and why use it by default?

**Answer**: `sealed` prevents further inheritance. Defaulting to `sealed`:

- Allows the JIT to **de-virtualize** virtual calls — modest performance gain.
- Communicates intent — readers don't have to wonder about subclasses.
- Prevents accidental fragile-base-class problems.

You unseal when you've designed an extension point — and then it's `abstract` or has `virtual` methods, not just `class`.

### 5. What's the difference between `override` and `new`, and when is `new` correct?

**Answer**:

- `override` replaces a `virtual`/`abstract` method — polymorphism resolves to the derived implementation.
- `new` *hides* the inherited member — the call resolves to the static type of the reference, not the runtime type.

`new` is almost always a mistake. Legitimate uses are vanishingly rare (mainly explicit interface re-implementation or hiding a base name in a derived class with different semantics on purpose). If you find yourself reaching for `new`, the design probably needs rethinking.

### 6. How do you make code that depends on a third-party SDK (like Stripe) testable?

**Answer**: Define an interface owned by *your* code (`IPaymentGateway`) and put the SDK call inside a thin adapter (`StripeGateway : IPaymentGateway`). Calling code depends on the interface. In tests, register a `FakePaymentGateway` that records calls and returns canned `PaymentResult` instances. The Stripe SDK is loaded only at runtime in production.

This is the **Dependency Inversion Principle** — high-level policy (checkout) depends on abstractions, not on Stripe (see [architecture/solid-principles.md](../architecture/solid-principles.md)).

### 7. When should you NOT extract an interface?

**Answer**:

- When there is exactly one implementation and no foreseeable second one.
- When the abstraction would have **every method** of the concrete class — it's just noise.
- When the concrete is already a value object/DTO with no behavior worth abstracting (`record`).

A common smell: `IFooService` ⇄ `FooService` 1:1 — that interface adds files, indirection, and friction for no benefit. Extract at the second implementation (real + fake counts).

### 8. How do you keep a class hierarchy from drifting into a "BaseEntityFromHell"?

**Why this matters**: Every long-lived .NET codebase has at least one such class; the question is whether the team has learned to stop adding to it.

**Answer**:

- Cap inheritance depth at **2 levels** as a rule of thumb (abstract base + concrete) and require justification for a third.
- Move cross-cutting concerns (auditing, soft-delete, tenancy) into **interfaces + EF Core interceptors/conventions**, not into base classes.
- Treat any field added to the base as a project-wide change — make it expensive on purpose.
- Periodically split: if a base class has 12 protected members and 8 unrelated subclasses, the abstraction is wrong.

## Summary Checklist

- [ ] My domain entities encapsulate their data — no public setters on invariants.
- [ ] State transitions live on the entity (`MarkPaid`, `Cancel`) — not in helper services.
- [ ] I prefer composition over inheritance, and seal classes by default.
- [ ] My interfaces describe a *role*, not a 1:1 mirror of a class.
- [ ] Behavior families (gateways, channels, validators) all implement a small, stable interface.
- [ ] DI wires polymorphic implementations — `switch` on provider names is gone.
- [ ] DTOs are dumb (`record`) — no business logic in request/response shapes.
- [ ] Collections on aggregates are exposed as `IReadOnlyList<T>` with explicit mutation methods.
- [ ] Inheritance is two levels deep or less; deeper trees require explicit justification.
- [ ] Every public type has a single, well-named responsibility I can describe in one sentence.
