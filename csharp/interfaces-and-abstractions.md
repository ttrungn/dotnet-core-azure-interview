# Interfaces and Abstractions

## What It Is

An **interface** in C# is a pure contract — a set of method, property, event, and indexer signatures that an implementing type promises to provide. An **abstraction** is broader: any mechanism (interface, abstract class, delegate, generic constraint) that lets calling code depend on *what* something does rather than *how* it does it. In a modern .NET backend the two concepts intertwine — DI containers wire interfaces to concrete implementations, tests substitute fakes, and modules expose interface contracts as their public API.

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(Money amount, CustomerToken token, CancellationToken ct);
    Task<PaymentResult> RefundAsync(string transactionId, Money amount, CancellationToken ct);
}

public sealed class StripeGateway(IStripeClient stripe, ILogger<StripeGateway> log) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(Money amount, CustomerToken t, CancellationToken ct) { /* … */ }
    public async Task<PaymentResult> RefundAsync(string txId, Money amount, CancellationToken ct)     { /* … */ }
}
```

The interface is the **shape of the relationship**. The class is the **implementation detail** behind that shape.

## Why It Exists

Without abstractions, every caller would hard-code the SDK it talks to. `CheckoutService` would `new StripeClient(...)` inline; tests would need a real Stripe account; migrating to Adyen would mean editing dozens of files. With an interface owned by *your* code, every layer above the gateway is shielded from those choices.

Interfaces also exist because **C# is single-inheritance**. A class can extend one base class but implement many interfaces — that's how a `CustomerRepository` can be `IRepository<Customer>`, `IDisposable`, and `IAsyncDisposable` simultaneously.

Finally, abstractions are the foundation of **dependency inversion** (the "D" in SOLID — see [architecture/solid-principles.md](../architecture/solid-principles.md)): high-level policy (checkout) depends on abstractions (`IPaymentGateway`), and low-level details (Stripe SDK) also depend on those abstractions. Both layers point at the contract; neither depends on the other.

## When To Use It

**Extract an interface when:**

- A **second implementation exists** — real + fake-for-tests counts as two.
- The dependency is **slow, external, or non-deterministic** (HTTP calls, databases, time, randomness, message brokers).
- The implementation might be **swapped per environment** (in-memory cache vs Redis, console logger vs Application Insights).
- Multiple modules need to depend on the **shape** without being coupled to the concrete.
- You want **plug-in extensibility** — new validators, new notification channels, new payment providers.

**Do not extract an interface when:**

- There's exactly one implementation and no second on the horizon.
- The type is a pure data carrier (use `record`, not `interface`).
- The "abstraction" would just mirror every method of one class — `IFooService` ⇄ `FooService` 1:1 is noise.
- The concrete type is a stable, owned-by-you utility (`Money`, `Result<T>`) — you don't extract `IMoney`.

The default question: **"who else needs to depend on the contract, and how do they differ from the current implementation?"** If you can't answer concretely, don't extract.

## Why It Is Important

Interfaces are the seams that let large systems evolve. When a payment provider rate-limits us at 3 AM, we can deploy a `BackupPaymentGateway` decorator the same day because every caller already speaks `IPaymentGateway`. When we move from on-prem SQL Server to Azure SQL, the `IOrderRepository` contract stays — only the EF Core configuration changes.

They are also what makes a codebase **testable**. Every dependency you can replace at construction time is a dependency you can fake in a unit test. Untestable code is unmaintainable code over a five-year horizon.

In Azure-hosted backends, interfaces also let you defer infrastructure decisions: you can ship the v1 cache with `MemoryCache` behind `IResponseCache` and swap in Azure Cache for Redis when traffic justifies it — without rewriting controllers.

## How It's Used in C# / .NET

### 1. Interface vs abstract class

| Use an `interface`                                      | Use an `abstract class`                                  |
| ------------------------------------------------------- | -------------------------------------------------------- |
| You only need a contract.                               | You also need shared state or implementation.            |
| Multiple unrelated types must implement it.             | Implementers share a strong `is-a` relationship.         |
| Cross-cutting capability (`IDisposable`, `IComparable`).| Template-method pattern (`PaymentProcessor` base).       |
| You expect many implementers across modules.            | You expect a small, controlled hierarchy.                |

C# 8 added **default interface members**, but they are best used for backward-compatible API evolution on contracts you own — not as a substitute for abstract classes.

```csharp
public interface IRepository<T>
{
    Task<T?> GetAsync(Guid id, CancellationToken ct);
    Task SaveAsync(T entity, CancellationToken ct);

    // Default member added later without breaking implementers
    Task<bool> ExistsAsync(Guid id, CancellationToken ct) => GetAsync(id, ct).ContinueWith(t => t.Result is not null, ct);
}
```

### 2. Role interfaces (small) vs header interfaces (large)

Prefer **role interfaces** — small, focused on one capability:

```csharp
public interface IEmailSender   { Task SendAsync(EmailMessage m, CancellationToken ct); }
public interface ISmsSender     { Task SendAsync(SmsMessage m,   CancellationToken ct); }
public interface IPushNotifier  { Task SendAsync(PushPayload p,  CancellationToken ct); }
```

Avoid the **header interface** anti-pattern that mirrors every method of a service:

```csharp
// BUG — interface is just a duplicate of the class
public interface ICustomerService
{
    Task<Customer> GetAsync(Guid id, CancellationToken ct);
    Task<Customer> CreateAsync(CreateCustomerRequest req, CancellationToken ct);
    Task<Customer> UpdateAsync(Guid id, UpdateCustomerRequest req, CancellationToken ct);
    Task DeleteAsync(Guid id, CancellationToken ct);
    // … 14 more, exactly matching CustomerService
}
```

This is **Interface Segregation Principle** (the "I" in SOLID): consumers should not depend on methods they don't use.

### 3. Explicit interface implementation

When two interfaces define a method with the same signature but different semantics, implement them explicitly:

```csharp
public sealed class PaymentRepository : IRepository<Payment>, IDisposable, IAsyncDisposable
{
    Task<Payment?> IRepository<Payment>.GetAsync(Guid id, CancellationToken ct) { /* … */ }
    Task IRepository<Payment>.SaveAsync(Payment p, CancellationToken ct)        { /* … */ }
    void IDisposable.Dispose()                                                  { /* sync cleanup */ }
    ValueTask IAsyncDisposable.DisposeAsync()                                   { /* async cleanup */ }
}
```

Methods are visible only through the interface type — they don't pollute the class's public surface.

### 4. Generic interfaces and constraints

```csharp
public interface IRepository<T> where T : AggregateRoot
{
    Task<T?> GetAsync(Guid id, CancellationToken ct);
    Task SaveAsync(T entity, CancellationToken ct);
}

public interface IRequestHandler<TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken ct);
}
```

This is how MediatR, EF Core, and most modern .NET libraries express their plug-in models. See [csharp/generics.md](generics.md) for the type-system details.

### 5. Variance — `in` and `out`

```csharp
public interface IReadOnlyRepository<out T>                                          // covariant
{
    Task<T?> GetAsync(Guid id, CancellationToken ct);
}

public interface IValidator<in T>                                                    // contravariant
{
    ValidationResult Validate(T value);
}

IReadOnlyRepository<Customer> repo = GetCustomerRepo();
IReadOnlyRepository<Person> personRepo = repo;   // OK — Customer : Person, output position
```

`out` allows substituting a more derived type as output; `in` allows accepting a more derived type as input. Both are limited to interfaces and delegates — class type parameters cannot be variant.

### 6. DI registration patterns

```csharp
// Single implementation
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

// Multiple implementations resolved as IEnumerable<T>
builder.Services.AddSingleton<INotificationChannel, EmailChannel>();
builder.Services.AddSingleton<INotificationChannel, SmsChannel>();
builder.Services.AddSingleton<INotificationChannel, PushChannel>();

// Keyed DI (.NET 8+) — selects the right one by key
builder.Services.AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, AdyenGateway>("adyen");

// Decorator pattern via Scrutor
builder.Services.Decorate<IPaymentGateway, ResilientPaymentGateway>();
builder.Services.Decorate<IPaymentGateway, LoggingPaymentGateway>();
```

See [csharp/dependency-injection.md](dependency-injection.md) for lifetimes and resolution rules.

### 7. Marker interfaces and capability detection

```csharp
public interface IDomainEvent { DateTimeOffset OccurredAt { get; } }
public interface ITransactional { }

if (handler is ITransactional)
{
    using var tx = await db.Database.BeginTransactionAsync(ct);
    await handler.HandleAsync(message, ct);
    await tx.CommitAsync(ct);
}
```

Marker interfaces (no members) are a lightweight form of metadata. Use sparingly — attributes are usually clearer.

### 8. Sealing implementations

```csharp
public sealed class StripeGateway : IPaymentGateway { /* … */ }
```

Sealing implementations lets the JIT de-virtualize calls through the interface when the runtime type is known, and signals that this class is not an extension point — extend by composing or by adding a sibling implementation, not by inheriting.

## Advantages

- **Decoupling** — callers depend on the contract; implementations can change freely.
- **Testability** — every interface dependency can be replaced with a fake or mock.
- **Multiple inheritance of behaviour** — one class can implement many interfaces.
- **Plug-in models** — `IEnumerable<TValidator>`, `IEnumerable<INotificationChannel>` enable open-set extension.
- **Documentation** — a small interface communicates intent better than a 500-line concrete class.
- **Parallel work** — once an interface is agreed, two teams can implement and consume in parallel.

## Disadvantages

- **Indirection** — debugging needs one more "go to implementation" hop.
- **Over-abstraction** — interfaces with one implementation add weight without value.
- **Versioning friction** — adding a method to a shipped public interface is a breaking change (default members help but only on .NET Core 3.0+ targets).
- **Slight performance cost** — virtual dispatch through an interface is marginally slower than direct calls; rarely matters outside hot loops.
- **Discoverability** — hopping through several interfaces to find the real code can frustrate newcomers.

## Common Mistakes

### 1. `IFooService` ⇄ `FooService` 1:1 with one implementation

```csharp
// BUG — interface exists "for testability" but the project only ever has one implementation,
//       and tests use real OrderService anyway
public interface IOrderService { Task<Order> CreateAsync(...); }
public sealed class OrderService : IOrderService { /* … */ }
```

**Fix**: depend on `OrderService` directly. Extract `IOrderService` later, when a fake or alternative actually appears.

### 2. Fat interfaces with mixed concerns

```csharp
// BUG — clients are forced to implement methods they don't need
public interface IOrderManager
{
    Task<Order> GetAsync(Guid id, CancellationToken ct);
    Task SaveAsync(Order o, CancellationToken ct);
    Task SendInvoiceEmailAsync(Order o, CancellationToken ct);
    Task ChargeAsync(Order o, CustomerToken t, CancellationToken ct);
    Task RefundAsync(Order o, Money amt, CancellationToken ct);
}
```

**Fix**: split into role interfaces — `IOrderRepository`, `IInvoiceEmailer`, `IPaymentGateway`. Each has one reason to change.

### 3. Returning concrete collection types

```csharp
public interface ICustomerRepository
{
    Task<List<Customer>> GetActiveAsync(CancellationToken ct);   // BUG — leaks `List<T>` to callers
}
```

**Fix**: return `IReadOnlyList<T>` or `IAsyncEnumerable<T>`. Callers cannot mutate the result; implementations have freedom.

### 4. Interfaces with mutable properties

```csharp
public interface IUser { string Email { get; set; } }   // BUG — implementations cannot enforce invariants
```

**Fix**: read-only properties on the interface; mutation through methods (`ChangeEmail`) that can validate.

### 5. Hiding cancellation tokens

```csharp
public interface IRepository<T>
{
    Task<T?> GetAsync(Guid id);                                  // BUG — no CancellationToken
}
```

**Fix**: every async I/O method takes a `CancellationToken`, no exceptions. Forgetting one infects every caller downstream.

### 6. Leaking implementation types

```csharp
public interface IPaymentGateway
{
    Task<Stripe.Charge> ChargeAsync(...);                        // BUG — Stripe type in the contract
}
```

**Fix**: define your own `PaymentResult` record. The contract should not couple callers to the SDK you happen to use today.

### 7. Default interface methods used as abstract base classes

```csharp
public interface IPaymentProcessor
{
    void LogStart() => Console.WriteLine("starting");            // BUG — implementation in a contract
    void Charge();
}
```

**Fix**: if you need shared implementation, use an abstract class. Reserve default interface members for **non-breaking API evolution** of contracts other teams have already implemented.

### 8. Implementing `IDisposable` without owning anything disposable

```csharp
public sealed class CustomerService(IRepo r) : IDisposable        // BUG — service owns nothing
{
    public void Dispose() { }
}
```

**Fix**: implement `IDisposable`/`IAsyncDisposable` only when you own unmanaged resources or other disposables. DI will manage lifetimes automatically; spurious `Dispose` methods confuse the container.

## Best Practices

- **Define interfaces from the caller's perspective** — name them by what callers need, not by what the class happens to do.
- **Small role interfaces** — every method should be one the consumer actually uses.
- **Read-only by default** — collections, properties, projections.
- **Always include `CancellationToken`** on async I/O operations.
- **Own your contracts** — never put third-party types (Stripe, MongoDB, Kafka) in interface signatures.
- **Seal implementations** unless they are designed for inheritance.
- **Document semantic guarantees** — XML doc on the interface explains idempotency, ordering, exceptions.
- **Keep interface namespace beside callers**, not beside implementations (Clean Architecture: interfaces live in the domain/application layer; implementations in infrastructure — see [architecture/clean-architecture.md](../architecture/clean-architecture.md)).
- **Evolve carefully** — once published, adding required members is a breaking change.

## Related Concepts

- **[csharp/object-oriented-programming.md](object-oriented-programming.md)** — interfaces are the polymorphism backbone.
- **[csharp/dependency-injection.md](dependency-injection.md)** — wires interfaces to implementations.
- **[csharp/generics.md](generics.md)** — generic interfaces and constraints.
- **[csharp/records-and-immutability.md](records-and-immutability.md)** — when a `record` replaces an interface.
- **[architecture/solid-principles.md](../architecture/solid-principles.md)** — Interface Segregation and Dependency Inversion.
- **[architecture/clean-architecture.md](../architecture/clean-architecture.md)** — interface ownership and the dependency rule.
- **[architecture/repositories.md](../architecture/repositories.md)** — `IRepository<T>` as the canonical example.
- **[testing-quality/mocking.md](../testing-quality/mocking.md)** — fakes and mocks rely on interfaces.

## Real-World Usage

### Multi-provider payments in an ASP.NET Core checkout API

```csharp
public interface IPaymentGateway
{
    string Name { get; }
    Task<PaymentResult> ChargeAsync(Money amount, CustomerToken token, CancellationToken ct);
    Task<PaymentResult> RefundAsync(string transactionId, Money amount, CancellationToken ct);
}

public sealed class StripeGateway(IStripeClient stripe, ILogger<StripeGateway> log) : IPaymentGateway
{
    public string Name => "stripe";
    public async Task<PaymentResult> ChargeAsync(Money amt, CustomerToken t, CancellationToken ct) { /* … */ }
    public async Task<PaymentResult> RefundAsync(string id, Money amt, CancellationToken ct)      { /* … */ }
}

public sealed class AdyenGateway(IAdyenClient adyen, ILogger<AdyenGateway> log) : IPaymentGateway
{
    public string Name => "adyen";
    public async Task<PaymentResult> ChargeAsync(Money amt, CustomerToken t, CancellationToken ct) { /* … */ }
    public async Task<PaymentResult> RefundAsync(string id, Money amt, CancellationToken ct)      { /* … */ }
}
```

Resolver:

```csharp
public sealed class PaymentGatewayResolver(IEnumerable<IPaymentGateway> gateways)
{
    private readonly Dictionary<string, IPaymentGateway> _map =
        gateways.ToDictionary(g => g.Name, StringComparer.OrdinalIgnoreCase);

    public IPaymentGateway For(string provider) =>
        _map.TryGetValue(provider, out var g)
            ? g
            : throw new NotSupportedException($"Unknown gateway '{provider}'");
}
```

`CheckoutService` asks the resolver for the right gateway based on the customer's saved preference or A/B test bucket.

### Azure Function consuming Service Bus with the same abstraction

```csharp
public sealed class RefundFunction(PaymentGatewayResolver resolver, ILogger<RefundFunction> log)
{
    [Function("ProcessRefund")]
    public async Task RunAsync(
        [ServiceBusTrigger("refunds", Connection = "SbConn")] RefundRequested req,
        CancellationToken ct)
    {
        var gateway = resolver.For(req.Provider);
        var result = await gateway.RefundAsync(req.TransactionId, req.Amount, ct);
        log.LogInformation("Refund {Status} for {Tx}", result.Status, req.TransactionId);
    }
}
```

The function does not know about Stripe or Adyen — it depends only on the contract.

### Testing with a fake

```csharp
public sealed class FakePaymentGateway : IPaymentGateway
{
    public string Name => "fake";
    public List<(Money, CustomerToken)> Charges { get; } = new();
    public Task<PaymentResult> ChargeAsync(Money a, CustomerToken t, CancellationToken ct)
    {
        Charges.Add((a, t));
        return Task.FromResult(PaymentResult.Success("tx_test_123"));
    }
    public Task<PaymentResult> RefundAsync(string id, Money a, CancellationToken ct)
        => Task.FromResult(PaymentResult.Success("rf_test_123"));
}

[Fact]
public async Task Checkout_ChargesCustomer()
{
    var fake = new FakePaymentGateway();
    var sut  = new CheckoutService(fake, new InMemoryOrderRepository(), NullLogger<CheckoutService>.Instance);
    var order = new Order(Guid.CreateVersion7()); order.AddLine(new("SKU-1", new Money(50m, "USD"), 1));

    await sut.CheckoutAsync(order, new CustomerToken("tok_test"), default);

    fake.Charges.Should().HaveCount(1);
}
```

No Stripe credentials, no HTTP, no network — runs in milliseconds in CI.

## Code Example — Before and After

### Before: concretes everywhere, no testing seam

```csharp
public class OrderService
{
    private readonly SqlConnection _db = new("Server=...;Database=Orders;...");
    private readonly StripeClient  _stripe = new("sk_live_...");
    private readonly SmtpClient    _smtp = new("smtp.contoso.com");

    public async Task<bool> CheckoutAsync(Order o, CustomerToken t)
    {
        // direct SQL on _db
        // direct Stripe call on _stripe
        // direct SMTP on _smtp
        return true;
    }
}
```

Problems:

- Cannot test without a live database, a Stripe key, and an SMTP server.
- Cannot swap providers — every change is a code edit.
- All concerns (persistence, payment, email) bound to one class — fragile, untestable, undeployable in different environments.

### After: interfaces and DI

```csharp
public interface IOrderRepository { Task SaveAsync(Order o, CancellationToken ct); }
public interface IPaymentGateway  { Task<PaymentResult> ChargeAsync(Money a, CustomerToken t, CancellationToken ct); }
public interface IEmailSender     { Task SendAsync(EmailMessage m, CancellationToken ct); }

public sealed class CheckoutService(
    IOrderRepository orders,
    IPaymentGateway  gateway,
    IEmailSender     email,
    ILogger<CheckoutService> log)
{
    public async Task<Result> CheckoutAsync(Order order, CustomerToken token, CancellationToken ct)
    {
        var payment = await gateway.ChargeAsync(order.Total, token, ct);
        if (!payment.IsSuccess) return Result.Fail(payment.ErrorCode);

        order.MarkPaid(payment.TransactionId);
        await orders.SaveAsync(order, ct);
        await email.SendAsync(EmailMessage.InvoiceFor(order), ct);
        log.LogInformation("Checkout complete for {OrderId}", order.Id);
        return Result.Ok();
    }
}
```

Composition root:

```csharp
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripeGateway>();
builder.Services.Decorate <IPaymentGateway, ResilientPaymentGateway>();
builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();
builder.Services.AddScoped<CheckoutService>();
```

What improved:

- **Testable** — every dependency has a fake counterpart.
- **Configurable** — swap Stripe for Adyen via registration; no production code change.
- **Resilient** — the `Decorate` line wraps the gateway with retry/timeout policies (decorator pattern).
- **Each class has one reason to change** — Single Responsibility achieved.

## Interview Questions and Answers

### 1. When do you choose an interface over an abstract class?

**Why this matters**: Reveals whether you understand both tools, not just one.

**Answer**: Interface for pure contracts shared across unrelated types or to enable plug-in extensibility. Abstract class when implementers share genuine state or behavior (template-method, common logging). With C# 8+ default interface members, the line has narrowed — but I still reach for abstract classes when there's shared state, and interfaces when there's only a contract.

**Trade-off**: An abstract class locks implementers into single inheritance — pick it only when that's acceptable.

**Real project**: `IPaymentGateway` is an interface (Stripe/Adyen share nothing); `PaymentProcessor` abstract base is appropriate when every implementer needs the same logging/telemetry wrapper.

### 2. What is the Interface Segregation Principle?

**Answer**: Clients should not be forced to depend on methods they don't use. Practically: split big interfaces into role interfaces (`IEmailSender`, `ISmsSender`, `IPushNotifier`) rather than one `INotificationService` with every method.

**Real project**: A reporting module only needed `IRepository.GetAsync` but was forced to mock `SaveAsync`, `DeleteAsync`, `BulkInsertAsync` because they were on the same interface. Splitting into `IReadRepository<T>` and `IWriteRepository<T>` halved test setup boilerplate.

### 3. Should every service have an interface "for testability"?

**Answer**: No. Extract an interface when:
- A second implementation exists (real + fake counts).
- The dependency is slow/external/non-deterministic.
- You need a swap point per environment.

For pure utility classes with no I/O (`Money`, `Result<T>`), no interface needed — they're already testable.

**Trade-off**: Premature interfaces add indirection without value and pollute the codebase with one-implementation contracts.

### 4. How do you evolve a published interface without breaking consumers?

**Answer**: 
- Add new methods as **default interface methods** (C# 8+) — existing implementers compile unchanged.
- For non-default-method scenarios, define a new interface (`IFooV2`) that extends the old one — consumers opt in.
- Mark old methods `[Obsolete]` and remove in a major version.
- Never change method signatures or return types — that's always a break.

**Real project**: Adding `BulkSaveAsync` to `IRepository<T>` was rolled out as a default method calling `SaveAsync` in a loop; implementations that cared overrode it with EF Core's `BulkExtensions`.

### 5. What is variance (`in`/`out`) and when does it matter?

**Answer**: 
- `out T` (covariant) — the parameter only appears in output positions; allows `IReadOnlyRepository<Customer>` to be assigned to `IReadOnlyRepository<Person>` if `Customer : Person`.
- `in T` (contravariant) — appears only in input positions; allows `IValidator<Person>` to validate `Customer`.
- Invariant by default; variance only applies to interfaces and delegates.

In practice it matters for collection projections (`IEnumerable<out T>`) and for validator/handler hierarchies — without variance, you'd need explicit casts everywhere.

### 6. Explain default interface members and when you'd use them.

**Answer**: C# 8+ allows interface methods to have a default implementation. Use them for:
- **Backward-compatible API evolution** on contracts you've shipped.
- **Convenience overloads** (`SaveAsync(entity)` calling `SaveAsync(entity, CancellationToken.None)` — though I'd still prefer an extension method for that).

Don't use them as a substitute for abstract base classes; they can't access state, can't be `protected`, and they confuse the "interface = contract" mental model when overused.

### 7. How do you handle multiple implementations of the same interface in DI?

**Answer**: Three patterns:
1. **Inject `IEnumerable<TInterface>`** — iterate and dispatch by some discriminator (`Name`, `Kind`).
2. **Keyed DI** (.NET 8+) — `AddKeyedScoped<IFoo, FooA>("a")` and `[FromKeyedServices("a")]` on the consumer.
3. **A resolver service** — small class that owns the dictionary and exposes `For(name)`.

I prefer the resolver — it isolates the lookup logic and gives a clear failure mode when the name is unknown.

### 8. When have you decided NOT to extract an interface, and what was the consequence?

**Why this matters**: Mature interviewers want to hear about restraint, not "I extract interfaces everywhere".

**Answer**: For a `MoneyFormatter` utility class — pure function, no I/O, no second implementation — I left it as a concrete static-like service. Three years on, it's still one class; no test or production scenario needed substitution. Had I extracted `IMoneyFormatter`, every consumer would be one indirection deeper for zero benefit.

**Trade-off**: If a second formatting strategy ever appears (e.g., a localized variant), the cost to extract then is one refactor — much smaller than the cumulative cost of carrying a useless interface for three years.

## Summary Checklist

- [ ] Every interface in my codebase has at least two implementations (real + fake, or two production variants).
- [ ] Interfaces are small role interfaces — every method is used by every consumer.
- [ ] Async methods always take a `CancellationToken`.
- [ ] Interface signatures use my types, never third-party SDK types.
- [ ] Collection-returning methods return `IReadOnlyList<T>` or `IAsyncEnumerable<T>`, not `List<T>`.
- [ ] I extract interfaces when a second implementation appears, not on speculation.
- [ ] I prefer abstract classes only when implementers share genuine state.
- [ ] Default interface members are reserved for backward-compatible API evolution.
- [ ] Implementations are `sealed` unless designed for inheritance.
- [ ] Interface ownership follows Clean Architecture — contracts live with the consumers, not the implementations.
