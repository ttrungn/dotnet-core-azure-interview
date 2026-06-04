# Testable Code Design

## What It Is

**Testable code design** is the practice of writing production code so that automated tests can exercise it cheaply, deterministically, and at the right granularity — without having to spin up databases, network sockets, or real clocks for tests that should be in-process. It is a design discipline, not a testing discipline. By the time you sit down to *write* a test, the testability of the SUT has already been decided by how you structured it.

The five core moves:

1. **Declare dependencies explicitly** through the constructor (see [../csharp/dependency-injection.md](../csharp/dependency-injection.md)).
2. **Keep classes small** with one responsibility (see [../architecture/solid-principles.md](../architecture/solid-principles.md), the **S** in SOLID).
3. **Isolate side effects** behind narrow abstractions — HTTP, SQL, queues, file system, the clock.
4. **Make logic pure where possible** — methods that take inputs and return outputs with no hidden side effects are trivially testable.
5. **Return values, not voids** — give the caller (and the test) something to assert on.

```csharp
// Hard to test — hidden dependencies, side effects, void
public class OrderService
{
    public void PlaceOrder(int customerId)
    {
        var db = new SqlConnection(Environment.GetEnvironmentVariable("CS"));
        db.Open();
        if (DateTime.UtcNow.DayOfWeek == DayOfWeek.Sunday) return;
        // ... 80 lines
    }
}

// Easy to test — explicit dependencies, returns a result, time injected
public sealed class PlaceOrderHandler
{
    private readonly IOrderRepository _repo;
    private readonly TimeProvider _clock;
    private readonly IBusinessHoursPolicy _hours;

    public PlaceOrderHandler(IOrderRepository repo, TimeProvider clock, IBusinessHoursPolicy hours)
    {
        _repo = repo;
        _clock = clock;
        _hours = hours;
    }

    public async Task<OrderResult> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        if (!_hours.IsOpen(_clock.GetUtcNow()))
            return OrderResult.Rejected("store closed");
        var order = new Order(cmd.CustomerId, cmd.Total);
        await _repo.AddAsync(order, ct);
        return OrderResult.Confirmed(order.Id);
    }
}
```

The second version can be unit-tested in 10 milliseconds with no infrastructure. The first cannot be unit-tested at all.

## Why It Exists

Engineers learn early that bad design hurts the product. Testable design hurts a second audience: the test code itself. When the SUT mixes ten concerns and reaches out to four external systems, tests become a mess of mocks, brittle assertions, and shared state. Teams stop writing tests, the safety net frays, and a year later refactoring is terrifying because no one trusts the existing tests.

Testable code design exists to make the **cost of a test proportional to the value it adds**. Well-designed code lets a junior engineer write a test in 30 seconds and trust it. Badly designed code forces senior engineers to write 80 lines of setup per test, and even then the test catches less than it should.

It also exists because the *pain you feel while writing a test* is the cheapest, fastest design feedback you'll ever get. If you cannot write a test for `OrderService.Process()` without instantiating a `DbContext`, a `BlobServiceClient`, and patching `DateTime.UtcNow`, the class is doing too much. Listen to that signal and refactor.

## When To Use It

Apply testability principles to **every class that contains business logic or orchestrates collaborators** — handlers, services, domain entities, policy classes, validators. These are the parts where bugs cost money.

Be more relaxed for:

- **Configuration / startup code** (`Program.cs`) — testability is delivered by integration tests over the boot path, not by abstracting every `AddSingleton`.
- **Pure DTOs / records** — `OrderResponse` has no behavior to test; just keep it immutable.
- **Framework glue** — controllers that delegate to handlers should be thin enough that they don't need their own design effort beyond standard DI.
- **One-off scripts and data migrations** — testability cost may exceed value; tag with comments and move on.

Apply **with judgement** for:

- **Infrastructure adapters** (`StripePaymentGateway`, `EfOrderRepository`) — these should be small enough to integration-test rather than unit-test. Don't over-abstract the inside.

## Why It Is Important

Testable design is the multiplier under every other testing practice. Without it:

- Unit tests become 60 lines of mock setup per assertion.
- Integration tests become the only viable layer, and the suite slows to minutes.
- Refactoring stalls because the tests are too brittle to change.
- New engineers cannot add features safely because they cannot exercise the rules in isolation.
- Coverage metrics lie — you have 90% coverage of glue and 30% of the business rules.

With it:

- Tests are short, focused, and easy to read. They serve as living documentation.
- The dependency graph is visible from the constructor signature alone.
- Refactoring is safe because tests survive internal changes.
- The same SUT can run in production with a real SQL Server and in tests with an in-memory fake — same code, different wiring.
- Time, randomness, and external services stop being sources of flaky tests.

This is one of those topics where an interviewer can tell within two minutes whether the candidate has actually maintained a large test suite over years.

## How It's Used in C# / .NET

### 1. Constructor injection of all collaborators

Make every dependency explicit and immutable:

```csharp
public sealed class RefundService
{
    private readonly IRefundRepository _repo;
    private readonly IPaymentGateway _gateway;
    private readonly IEventPublisher _events;
    private readonly TimeProvider _clock;
    private readonly ILogger<RefundService> _logger;

    public RefundService(
        IRefundRepository repo,
        IPaymentGateway gateway,
        IEventPublisher events,
        TimeProvider clock,
        ILogger<RefundService> logger)
    {
        _repo = repo;
        _gateway = gateway;
        _events = events;
        _clock = clock;
        _logger = logger;
    }
}
```

No `new`, no static lookups, no `IServiceLocator.GetService<>()` inside the body. Each test instantiates `RefundService` directly with fakes.

### 2. Inject `TimeProvider` instead of using `DateTime.UtcNow`

The .NET 8+ way:

```csharp
// Production
services.AddSingleton(TimeProvider.System);

// Test
var clock = new FakeTimeProvider(DateTimeOffset.Parse("2026-01-15T09:00:00Z"));
clock.Advance(TimeSpan.FromMinutes(31));
```

Any call to `_clock.GetUtcNow()` becomes deterministic.

### 3. Wrap external SDKs behind your own narrow interfaces

Don't let `BlobServiceClient`, `ServiceBusClient`, or `StripeClient` leak into your business code:

```csharp
public interface IInvoiceStorage
{
    Task<Uri> UploadAsync(Stream pdf, string fileName, CancellationToken ct);
    Task<Stream> DownloadAsync(string fileName, CancellationToken ct);
}

public sealed class BlobInvoiceStorage : IInvoiceStorage
{
    private readonly BlobContainerClient _container;
    public BlobInvoiceStorage(BlobServiceClient blobs)
        => _container = blobs.GetBlobContainerClient("invoices");
    // ... implementation
}
```

Now business code depends on `IInvoiceStorage` (4 methods you control), not `BlobContainerClient` (60+ methods you cannot mock cleanly).

### 4. Prefer pure functions for business decisions

A pure function depends only on its inputs, has no side effects, and is trivially testable:

```csharp
public static class DiscountPolicy
{
    public static decimal Calculate(CustomerTier tier, decimal subtotal, DateOnly today)
        => tier switch
        {
            CustomerTier.Gold when today.DayOfWeek == DayOfWeek.Friday => subtotal * 0.15m,
            CustomerTier.Gold => subtotal * 0.10m,
            CustomerTier.Silver => subtotal * 0.05m,
            _ => 0m
        };
}
```

`DiscountPolicy.Calculate(...)` needs no setup, no fakes, no `[BeforeEach]`. 30 `[InlineData]` rows cover the full grid in milliseconds.

### 5. Return results, don't reach into the world

```csharp
// Bad — void method, side effects only, nothing to assert on
public void ProcessOrder(Order order) {
    if (!IsValid(order)) throw new InvalidOperationException();
    SaveToDb(order);
    SendEmail(order);
}

// Good — returns OrderResult; the side effects are observable through the repo / sender
public async Task<OrderResult> PlaceAsync(PlaceOrderCommand cmd, CancellationToken ct)
{
    var validation = _validator.Validate(cmd);
    if (!validation.IsValid) return OrderResult.Invalid(validation.Errors);
    var order = new Order(cmd.CustomerId, cmd.Total);
    await _repo.AddAsync(order, ct);
    await _emails.SendAsync(cmd.Email, "Order received", $"#{order.Id}", ct);
    return OrderResult.Confirmed(order.Id);
}
```

### 6. Keep classes small and focused

The Single Responsibility Principle is a testability principle in disguise. A handler with one reason to change has one set of tests; a service with eight reasons to change has eight overlapping suites.

```csharp
// Bad — 800 lines, "OrderService" does everything
public class OrderService
{
    public void Place(...) { /* validation, pricing, payment, email, audit, dispatch */ }
}

// Good — split by use case
public sealed class PlaceOrderHandler { /* one concern */ }
public sealed class CancelOrderHandler { /* one concern */ }
public sealed class RefundOrderHandler { /* one concern */ }
public sealed class OrderPricingPolicy { /* pure logic */ }
```

### 7. Avoid `static` mutable state

```csharp
// Untestable — every test in the assembly shares the same counter
public static class OrderCounter
{
    private static int _count;
    public static int Next() => ++_count;
}

// Testable — instance-based, injectable
public interface IOrderNumberGenerator { int Next(); }
public sealed class SequentialOrderNumberGenerator : IOrderNumberGenerator
{
    private int _count;
    public int Next() => Interlocked.Increment(ref _count);
}
```

Static mutable state is the #1 source of order-dependent test flakes. Banish it for anything stateful.

### 8. Hide infrastructure behind ports (Hexagonal / Clean Architecture)

```
Domain          (pure, no deps)             ← perfectly testable
Application     (commands/handlers/ports)   ← needs only fakes
Infrastructure  (adapters: EF Core, HTTP)   ← integration-tested
Presentation    (API controllers)           ← integration-tested
```

The Domain and Application layers should be unit-testable without any framework references. The Infrastructure layer is integration-tested directly.

### 9. Use `internal` + `InternalsVisibleTo` sparingly

If you find yourself wanting `internal` types visible to the test assembly, prefer to **make the type public if it's part of the API**, or **delete the type if it's a helper that should be inlined**, or **promote it to its own class with a real interface**. `InternalsVisibleTo` is fine occasionally but should not be the default escape hatch.

### 10. Make `Guid` and randomness injectable

```csharp
public interface IIdGenerator { Guid NewId(); }
public sealed class GuidIdGenerator : IIdGenerator { public Guid NewId() => Guid.NewGuid(); }

// Tests use a deterministic implementation
public sealed class SequentialIdGenerator : IIdGenerator
{
    private int _n;
    public Guid NewId() => new Guid(++_n, 0, 0, new byte[8]);
}
```

This lets you assert exact ids in tests instead of `Should().NotBeEmpty()`.

## Advantages

- **Tests stay short and readable** — no setup theatre.
- **The dependency graph is visible** — constructor signatures are documentation.
- **Refactoring is safe** — tests rely on contracts, not internals.
- **Time, randomness, and external services are deterministic** in tests.
- **You can swap implementations** — Stripe to Adyen, SQL to Postgres — without touching business code.
- **Onboarding is faster** — a new engineer can understand a class by reading its constructor.
- **Coverage actually means something** — branches in real logic are covered, not just glue.

## Disadvantages

- **Upfront design cost** — extracting interfaces and injecting dependencies takes longer than `new()`.
- **Interface explosion if overdone** — `IStringTrimmer`, `IMathHelper`, `IPropertyMapper` add noise without value.
- **Indirection** — "go to definition" lands on an interface, not the implementation; new devs must learn the DI registration to follow execution.
- **Slight runtime cost** — DI resolution and virtual dispatch are usually negligible, but matter in tight loops.
- **Frameworks fight you** — some legacy ASP.NET / WinForms / ORM code resists injection without rewriting.
- **Boilerplate fatigue** — six dependencies + six fields + six constructor parameters per handler. Use primary constructors (C# 12) to shrink this.

## Common Mistakes

### 1. `DateTime.UtcNow` everywhere

```csharp
// Bad — test must run at the right minute of the right day
if (DateTime.UtcNow > order.ExpiresAt) order.MarkExpired();
```

**Fix**: Inject `TimeProvider`. Use `FakeTimeProvider` in tests.

```csharp
if (_clock.GetUtcNow() > order.ExpiresAt) order.MarkExpired();
```

### 2. `new` of an HTTP / SQL / SDK client inside the body

```csharp
// Bad — cannot test without the network / DB
public class WeatherService
{
    public async Task<Forecast> GetAsync(string city)
    {
        using var http = new HttpClient();
        return await http.GetFromJsonAsync<Forecast>($"https://api.weather.com/{city}");
    }
}
```

**Fix**: Inject `IWeatherApi` (a narrow interface you own) or `HttpClient` via `IHttpClientFactory`.

### 3. Service Locator inside the class

```csharp
// Bad — hidden dependency, untestable without spinning up the full container
public class OrderHandler
{
    private readonly IServiceProvider _sp;
    public OrderHandler(IServiceProvider sp) => _sp = sp;
    public void Handle() { var repo = _sp.GetRequiredService<IOrderRepository>(); /*...*/ }
}
```

**Fix**: Inject `IOrderRepository` directly. The constructor is the contract.

### 4. Constructor with 12 dependencies

If a class needs that many collaborators, it's doing too many things. Split it by use case (Command / Query handlers, separate services). See [../csharp/dependency-injection.md](../csharp/dependency-injection.md) for the refactor pattern.

### 5. Random data and `Guid.NewGuid()` inside the SUT

```csharp
// Bad — different id every run; test cannot assert exact value
public Order Create(...) => new(Guid.NewGuid(), ...);
```

**Fix**: Inject `IIdGenerator`. Use a deterministic generator in tests.

### 6. Mixing pure logic with I/O

```csharp
// Bad — discount calculation interleaved with EF queries
public async Task<decimal> CalculateDiscount(Order o, CancellationToken ct)
{
    var customer = await _db.Customers.FindAsync(o.CustomerId);
    if (customer.IsGold) return o.Total * 0.1m;
    var historical = await _db.Orders.Where(x => x.CustomerId == o.CustomerId).CountAsync();
    if (historical > 10) return o.Total * 0.05m;
    return 0m;
}
```

**Fix**: Pull the *decision* into a pure function; do the I/O at the edges:

```csharp
public async Task<decimal> CalculateDiscount(Order o, CancellationToken ct)
{
    var profile = await _customers.GetProfileAsync(o.CustomerId, ct);
    return DiscountPolicy.For(profile, o.Total);
}

public static class DiscountPolicy
{
    public static decimal For(CustomerProfile p, decimal subtotal) => /* pure */;
}
```

The policy gets 30 unit tests; the loader gets one integration test.

### 7. `sealed` everywhere or `sealed` nowhere

Default to `sealed` for production classes — it documents intent and helps the JIT. But avoid `sealed` on infrastructure adapters you might need to extend in tests (use composition instead of inheritance — `class TestRefundService : IRefundService` instead of subclassing).

### 8. Adding interfaces only to enable mocking

```csharp
// Bad — IOrder for a value object with no second implementation
public interface IOrder { Guid Id { get; } decimal Total { get; } }
```

**Fix**: Use the concrete `Order` type. Add an interface only when there is a real reason: a second implementation, a real boundary, or a clear extensibility point.

## Best Practices

- **Constructor injection only.** No property/setter injection except in edge cases (legacy frameworks).
- **Inject `TimeProvider`, `IIdGenerator`, `IRandomProvider`** for anything time- or randomness-dependent.
- **Wrap third-party SDKs** behind narrow interfaces you own.
- **Keep handlers under 100 lines and 5 dependencies.** If you exceed, split by use case.
- **Make business decisions pure functions.** Inject collaborators only where the function genuinely needs them.
- **Return results from public methods.** Void is acceptable only for true fire-and-forget commands.
- **`sealed` by default** on production classes; favor composition over inheritance.
- **No `static` mutable state.** Use scoped services or pass state through method parameters.
- **Avoid `InternalsVisibleTo` as a habit.** Promote the type or rethink the design.
- **Test the design pain.** If a test is hard to write, the SUT is hard to use — fix the SUT, not the test.
- **Co-locate the abstraction with its consumer**, not its implementation. `IPaymentGateway` lives in `Application/`, `StripePaymentGateway` lives in `Infrastructure/`.

## Related Concepts

- **[unit-testing.md](unit-testing.md)** — the consumer of testable design.
- **[mocking.md](mocking.md)** — the technique that testable design enables.
- **[integration-testing.md](integration-testing.md)** — for the parts you deliberately leave untestable in isolation.
- **[../csharp/dependency-injection.md](../csharp/dependency-injection.md)** — the wiring that makes injection work.
- **[../architecture/solid-principles.md](../architecture/solid-principles.md)** — SRP, OCP, DIP all underpin testable design.
- **[../architecture/clean-architecture.md](../architecture/clean-architecture.md)** — the layering that protects pure logic from infrastructure.
- **[clean-code.md](clean-code.md)** — small functions, descriptive names, low cyclomatic complexity all improve testability.
- **Hexagonal / Ports and Adapters** — the pattern that names "ports" (interfaces) and "adapters" (implementations) as a design vocabulary.
- **Functional core, imperative shell** — Gary Bernhardt's framing: pure logic in the core, side effects at the edge.

## Real-World Usage

In a typical Clean Architecture .NET service:

```
src/Billing.Domain/                   ← pure C# class library, no framework deps
    Order.cs, Money.cs, Refund.cs     unit-tested, 0 ms per test
src/Billing.Application/              ← class library, depends only on Domain
    PlaceOrderHandler.cs              unit-tested with fakes, 5 ms per test
    Ports/IOrderRepository.cs         interface defined here
    Ports/IPaymentGateway.cs
src/Billing.Infrastructure/           ← depends on Application, plus EF Core, Azure SDKs
    Persistence/EfOrderRepository.cs  integration-tested against Testcontainers SQL
    Payments/StripePaymentGateway.cs  contract-tested with WireMock
src/Billing.Api/                      ← ASP.NET Core, wires everything in Program.cs
    Controllers/OrdersController.cs   integration-tested with WebApplicationFactory
```

The discipline: anything in `Domain` or `Application` must be unit-testable with zero external dependencies. The CI build *fails* if a Domain project takes a dependency on EF Core, Azure SDKs, or anything outside the .NET base class library — enforced via project references and an architecture test using NetArchTest.

This separation is what lets the team run 500 unit tests in 5 seconds. The structure is the test strategy.

## Code Example — Before and After

### Before — untestable god class

```csharp
public class OrderProcessor
{
    private static SqlConnection _conn;
    private static int _lastOrderNumber;

    public void Process(int customerId, string productCode, int qty)
    {
        // Hidden static initialization
        _conn ??= new SqlConnection(ConfigurationManager.ConnectionStrings["Db"].ConnectionString);
        _conn.Open();

        // Reads the clock directly
        if (DateTime.UtcNow.DayOfWeek == DayOfWeek.Sunday) return;

        // Static mutable state — every test shares this counter
        _lastOrderNumber++;

        // Inlined HTTP call
        using var http = new HttpClient();
        var rate = http.GetStringAsync("https://fx.api/usd").Result;

        // Inline SQL
        var cmd = _conn.CreateCommand();
        cmd.CommandText = $"INSERT INTO Orders VALUES ({_lastOrderNumber}, {customerId}, '{productCode}', {qty}, GETUTCDATE())";
        cmd.ExecuteNonQuery();

        // Direct SMTP
        var smtp = new SmtpClient("smtp.contoso.com");
        smtp.Send(new MailMessage("orders@contoso.com", "ops@contoso.com")
        {
            Subject = $"Order {_lastOrderNumber} placed",
            Body = $"Customer {customerId} ordered {qty} of {productCode}"
        });
    }
}
```

Unit test cost: enormous. Requires SQL Server, HTTPS connectivity to a real FX provider, an SMTP server, and the test must avoid running on Sunday.

### After — small, testable, composable

```csharp
public interface IFxRateProvider { Task<decimal> GetRateAsync(string from, string to, CancellationToken ct); }
public interface IOrderRepository { Task AddAsync(Order order, CancellationToken ct); }
public interface IEmailSender   { Task SendAsync(string to, string subject, string body, CancellationToken ct); }
public interface IBusinessHoursPolicy { bool IsOpen(DateTimeOffset now); }

public sealed class PlaceOrderHandler
{
    private readonly IOrderRepository _repo;
    private readonly IFxRateProvider _fx;
    private readonly IEmailSender _email;
    private readonly IBusinessHoursPolicy _hours;
    private readonly TimeProvider _clock;
    private readonly IIdGenerator _ids;

    public PlaceOrderHandler(
        IOrderRepository repo, IFxRateProvider fx, IEmailSender email,
        IBusinessHoursPolicy hours, TimeProvider clock, IIdGenerator ids)
    {
        _repo = repo; _fx = fx; _email = email;
        _hours = hours; _clock = clock; _ids = ids;
    }

    public async Task<OrderResult> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        if (!_hours.IsOpen(_clock.GetUtcNow()))
            return OrderResult.Rejected("store closed");

        var rate = await _fx.GetRateAsync("USD", cmd.Currency, ct);
        var order = new Order(_ids.NewId(), cmd.CustomerId, cmd.Amount * rate);
        await _repo.AddAsync(order, ct);

        await _email.SendAsync("ops@contoso.com",
            $"Order {order.Id} placed",
            $"Customer {cmd.CustomerId} ordered {cmd.Quantity} of {cmd.ProductCode}",
            ct);

        return OrderResult.Confirmed(order.Id);
    }
}

[Fact]
public async Task Handle_DuringOpenHours_PersistsOrder_AndSendsConfirmation()
{
    var repo = new InMemoryOrderRepository();
    var email = new CapturingEmailSender();
    var sut = new PlaceOrderHandler(
        repo,
        new FixedFxProvider(rate: 1.0m),
        email,
        new AlwaysOpenHours(),
        new FakeTimeProvider(DateTimeOffset.UtcNow),
        new SequentialIdGenerator());

    var result = await sut.HandleAsync(
        new PlaceOrderCommand(customerId: 42, productCode: "P-1", quantity: 1, amount: 100m, currency: "USD"),
        CancellationToken.None);

    result.Status.Should().Be(OrderStatus.Confirmed);
    repo.All().Should().ContainSingle(o => o.Total == 100m);
    email.Sent.Should().ContainSingle(s => s.To == "ops@contoso.com");
}
```

Cost: 5 ms. Deterministic. Covers business behaviour exhaustively without any infrastructure.

## Interview Questions and Answers

### 1. What does "testable code design" mean to you in concrete terms?

**Why this matters**: A vague answer ("you should be able to test it") signals surface-level understanding. Concrete answers reveal experience.

**Answer**: Five things, in order of impact: (1) constructor injection of every collaborator so the class's dependencies are obvious from its signature; (2) inject `TimeProvider`, `IIdGenerator`, and any randomness so behavior is deterministic; (3) wrap external SDKs behind narrow interfaces I own; (4) keep handlers small and single-purpose so each test covers one concern; (5) prefer pure functions for business decisions so the most important logic is trivially testable. The signal that I've done this right is that I can write a meaningful test in under a minute, with under 10 lines of setup.

**Trade-off**: Done in excess, it produces interface explosion (`IStringTrimmer`, `IMathHelper`). The discipline is to add abstractions only at *real boundaries* — external systems, the clock, business policies — not to wrap every helper.

**Real project**: In a billing service, refactoring an `OrderProcessor` god-class into seven handlers + `TimeProvider` + `IIdGenerator` cut test setup from 60 lines per test to 4 lines, and the suite ran 3× faster.

### 2. Why is `DateTime.UtcNow` inside the SUT a problem?

**Answer**: It makes the code non-deterministic — the test result depends on what time it runs. A "session expired after 15 minutes" check passes locally if you run it at 10:00:01 and fails at 10:14:59 because of slight execution drift. It also makes time-traveling tests impossible — you can't easily test "what happens at end-of-day if no payment arrived" without sleeping or freezing the clock with reflection.

The fix is `TimeProvider` (.NET 8+). Inject it, then use `FakeTimeProvider` in tests with `Advance(...)` to simulate time passing. Before .NET 8, the equivalent was `IClock` / `ISystemClock` from `Microsoft.AspNetCore.Authentication`.

**Trade-off**: For one-shot console scripts or constant references where time isn't behavior, `DateTime.UtcNow` is fine. The discipline applies to *anywhere the time affects the outcome*.

**Real project**: A token-refresh service used `DateTime.UtcNow.AddMinutes(15)` directly. Migration to `TimeProvider` + `FakeTimeProvider` exposed three off-by-one bugs in token expiry checks — all caught by tests that previously couldn't be written.

### 3. A teammate says interfaces slow down code. Should you stop using them?

**Why this matters**: Reveals whether the candidate can defend design decisions with evidence.

**Answer**: Interface dispatch adds an indirect call, which in microbenchmarks is ~1–2 ns slower than a direct call. For 99% of business code, this is irrelevant — the cost is dwarfed by anything I/O. For tight loops where it matters (millions of iterations on hot data), JIT devirtualisation handles the common case, and you can fall back to `sealed` classes with concrete dependencies for that specific spot.

The right answer is: keep interfaces at architectural boundaries (where they enable testability and replacement), and remove them only when a profiler identifies them as the bottleneck. Don't optimise speculatively.

**Trade-off**: Interface explosion has a *real* readability cost. Don't add `IFoo` just because; add it when there's a second implementation or a test seam.

**Real project**: A team removed interfaces from a hot computation path after profiling showed 5% CPU spent on virtual dispatch in a 100,000-iteration loop. The rest of the codebase kept its interfaces — the optimization was local to one method.

### 4. Walk through how you'd refactor a class that has 12 dependencies and is hard to test.

**Answer**: Twelve dependencies is a Single Responsibility Principle violation. I'd:

1. **List the use cases** the class handles. If `OrderService` does Place, Cancel, Refund, Ship, and Reprice, that's five handlers.
2. **Split into per-use-case classes** — `PlaceOrderHandler`, `CancelOrderHandler`, etc. Each injects only what it needs (usually 3–5 deps).
3. **Extract shared policies** (`PricingPolicy`, `DiscountPolicy`) as pure static methods or small services that handlers call.
4. **Move cross-cutting concerns** (logging, telemetry, validation) into pipeline behaviors (MediatR) or filters (ASP.NET) so they don't show up as constructor parameters.
5. **Migrate tests one handler at a time** — the existing tests give you a safety net.

**Trade-off**: This is invasive. Do it incrementally, behind a feature flag if you're under delivery pressure, and split the PRs by handler.

**Real project**: A `CheckoutService` with 12 dependencies became four MediatR handlers with 3–4 dependencies each. Test count went up (better coverage of edge cases) but total LOC went down.

### 5. When does adding an interface for testability go too far?

**Answer**: When the interface has exactly one implementation, will never have a second, and the only consumer is the test. Examples I'd push back on:

- `IStringTrimmer` to mock `string.Trim()`.
- `IGuidProvider` if `Guid.NewGuid()` is already wrapped by `IIdGenerator`.
- `IFileSystem` if you're testing file *content*, not *paths* — a real temp directory in `Path.GetTempPath()` is simpler.

The rule: add an abstraction when it represents a **boundary** (external system, replaceable strategy, test seam at a meaningful contract). Don't add one because "tests need it" if a real class or a temp resource works.

**Trade-off**: For some teams the right move is the opposite — they're under-mocking and shipping bugs. Calibrate to your codebase's actual pain.

### 6. How do you make a method that calls `Guid.NewGuid()` testable?

**Answer**: Inject an `IIdGenerator`:

```csharp
public interface IIdGenerator { Guid NewId(); }
public sealed class GuidIdGenerator : IIdGenerator { public Guid NewId() => Guid.NewGuid(); }

public sealed class SequentialIdGenerator : IIdGenerator
{
    private int _n;
    public Guid NewId()
    {
        var bytes = new byte[16];
        BinaryPrimitives.WriteInt32BigEndian(bytes, ++_n);
        return new Guid(bytes);
    }
}
```

In production, register `GuidIdGenerator`. In tests, instantiate `SequentialIdGenerator` so ids are `00000001-...`, `00000002-...`. Now you can assert exact ids.

**Trade-off**: If you don't need to assert the exact id, you can leave `Guid.NewGuid()` in the SUT and assert `Should().NotBeEmpty()`. Inject only when the id appears in correlation logs, response payloads, or anywhere a stable value helps.

### 7. How do you balance testability with the YAGNI principle?

**Why this matters**: Tests "you ain't gonna need it" maturity.

**Answer**: Apply testability principles *only where they pay back*. Specifically:

- **Always** for business logic and handlers — these change often and bugs cost money.
- **Always** for time and randomness — these break tests in ways that are confusing to debug.
- **Always** for true external boundaries — payment, email, queues, third-party APIs.
- **Not** for one-line wrappers (`new Order(...)`), pure value objects (`Money`), or framework glue (`AddSingleton<>`).
- **Not** for code with no behavior to test (DTOs, auto-properties, `ToString()` overrides).

The signal is whether you can describe a real test that would catch a real bug if the abstraction were missing. If not, skip it.

**Trade-off**: Over-engineering for hypothetical future tests has a real cost — interface noise, indirection, harder onboarding. Listen to actual test pain, not imagined.

### 8. How do you architect a service so 80% of tests can be pure unit tests?

**Answer**: Clean / Hexagonal Architecture, with strict project references:

- **Domain** project — entities, value objects, domain services. No external dependencies. 100% unit-testable.
- **Application** project — command/query handlers, ports (interfaces). Depends only on Domain. Fakes injected for ports → 100% unit-testable.
- **Infrastructure** project — EF adapters, Azure SDK adapters, HTTP clients. Depends on Application's port interfaces. Integration-tested.
- **Api / Presentation** project — controllers, minimal API endpoints, middleware. Integration-tested via `WebApplicationFactory`.

Enforce the boundary with project references *and* an architecture test (NetArchTest):

```csharp
[Fact]
public void Domain_ShouldNot_Reference_Infrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot().HaveDependencyOn("Billing.Infrastructure").GetResult();
    result.IsSuccessful.Should().BeTrue();
}
```

The structure *forces* testability — there's no way to accidentally couple a domain rule to EF Core because the project won't compile.

**Trade-off**: This is a big architectural commitment. For a tiny CRUD service it's overkill. For anything where business rules matter and the team has more than three engineers, it pays back in months.

**Real project**: A billing service with this structure runs 600 unit tests in 4 seconds and 80 integration tests in 90 seconds. Coverage on Domain + Application is 92%; on Infrastructure is 65%; total project coverage is 80%.

## Summary Checklist

- [ ] Every collaborator is injected through the constructor.
- [ ] I inject `TimeProvider`, `IIdGenerator`, and any randomness — never use `DateTime.UtcNow` or `Guid.NewGuid()` directly in business code.
- [ ] External SDKs are wrapped behind narrow interfaces I own.
- [ ] Handlers stay under 100 lines and 5 dependencies; bigger means split.
- [ ] Business decisions are pure functions wherever possible.
- [ ] Public methods return results; void is reserved for true fire-and-forget.
- [ ] No `static` mutable state anywhere in business code.
- [ ] Domain and Application projects compile without referencing EF Core, Azure SDKs, or ASP.NET Core.
- [ ] When a test is hard to write, I refactor the SUT, not the test.
- [ ] Architecture tests (NetArchTest or similar) enforce the layering automatically.
