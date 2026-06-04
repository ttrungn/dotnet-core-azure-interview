# Mocking

## What It Is

**Mocking** is the practice of replacing a real collaborator in a test with a controllable stand-in so the test can exercise the **system under test (SUT)** in isolation. The term is often used loosely; precisely, there are five distinct types of **test doubles** (a vocabulary introduced by Gerard Meszaros in *xUnit Test Patterns*):

| Double  | Purpose                                                                 | Example |
|---------|-------------------------------------------------------------------------|---------|
| **Dummy** | Passed around but never used — fills a parameter slot.                 | `null!` for an unused `ILogger` in a constructor. |
| **Stub**  | Returns canned answers to calls.                                       | `IRateProvider` that always returns 1.0. |
| **Fake**  | A working, simplified in-memory implementation.                        | `InMemoryOrderRepository : IOrderRepository`. |
| **Mock**  | A double that **verifies expected interactions** were performed.       | `Mock<IEmailSender>` with `Verify(...Times.Once)`. |
| **Spy**   | Records all calls made to it, for inspection in the assert phase.      | A `CapturingEmailSender` that exposes a `List<Email> Sent`. |

In .NET, three libraries dominate:

- **Moq** — the long-standing default, `Mock<T>` + `.Setup(x => x.Method(...))`.
- **NSubstitute** — friendlier syntax, `Substitute.For<T>()` + `received.Method(...)`.
- **FakeItEasy** — fluent, similar in spirit to NSubstitute.

This page covers all three, but treats **fakes** and **stubs** as the preferred default for code you own, with **mocks** reserved for true external boundaries.

```csharp
// Moq
var gateway = new Mock<IPaymentGateway>();
gateway.Setup(g => g.ChargeAsync(It.IsAny<ChargeRequest>(), It.IsAny<CancellationToken>()))
       .ReturnsAsync(ChargeResult.Declined("insufficient_funds"));

// NSubstitute
var gateway = Substitute.For<IPaymentGateway>();
gateway.ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>())
       .Returns(ChargeResult.Declined("insufficient_funds"));

// Hand-rolled fake (often the best choice)
public class FakePaymentGateway : IPaymentGateway
{
    private readonly ChargeResult _result;
    public List<ChargeRequest> Charged { get; } = new();
    public FakePaymentGateway(ChargeResult result) => _result = result;
    public Task<ChargeResult> ChargeAsync(ChargeRequest req, CancellationToken ct)
    {
        Charged.Add(req);
        return Task.FromResult(_result);
    }
}
```

## Why It Exists

Without test doubles, a `PlaceOrderHandler` test would have to:

- Boot a real database (SQL Server) to hold orders.
- Open an HTTP connection to Stripe to charge cards.
- Run an SMTP server to send confirmation emails.
- Connect to Azure Service Bus to publish events.

Each of those would make the test slow, flaky, expensive, and dependent on the environment. The test would also be **non-deterministic** — what if Stripe declines the test card because their test mode is rate-limited?

Test doubles exist to **substitute the controlled, fast, in-process equivalents** of those external systems so the test can focus on the SUT's behavior. They are the mechanism that makes the [unit-testing.md](unit-testing.md) layer practical at all.

Beyond test isolation, test doubles also let you simulate **failure conditions** that are hard or impossible to trigger in real systems: a network timeout, a 503 from the payment gateway, a duplicate-key violation from the database. These edge cases are exactly the ones that cause production incidents, and a real Stripe sandbox cannot reliably produce them on demand.

## When To Use It

**Use mocking / fakes for**:

- **External boundaries**: payment gateways, email/SMS senders, blob storage, message brokers, third-party HTTP APIs.
- **Time and randomness**: `TimeProvider`, `IRandomProvider`.
- **Heavy dependencies your test shouldn't pay for**: full `DbContext` chains, gRPC clients, Azure SDK clients.
- **Failure-injection**: simulate `HttpRequestException`, `SqlException`, `OperationCanceledException`.

**Do not use mocking for**:

- **Pure value objects** — there is nothing to mock in `Money`, `OrderStatus`, `DateRange`. Use the real type.
- **Pure functions** — `DiscountCalculator.Calculate(...)` has no dependencies worth replacing.
- **Concrete classes you own with no abstraction** — extract an interface first if mocking is truly needed; usually a fake or the real class is better.
- **Static .NET APIs you trust** — don't wrap `JsonSerializer.Serialize` in `IJsonSerializer` just to mock it. You'd be testing your wrapper, not your code.
- **Internal helpers inside the SUT** — if you're tempted to mock a private method, the SUT is doing too much.

## Why It Is Important

Mocking is the seam that lets unit tests be **fast, deterministic, and isolated** — the three properties that make a test suite a productivity tool instead of a tax. Without it, you have only integration tests, which are slow and expensive to maintain at scale.

But mocking is also the single biggest source of **brittle tests** in production codebases. A poorly mocked test couples to *how* the SUT works rather than *what* it produces. The result: every refactor breaks dozens of tests, the team loses faith in the suite, and either the tests get deleted or the refactoring stops. Understanding the difference between a **fake** (encodes a contract) and a **mock** (encodes an interaction sequence) is the senior-vs-junior distinction.

The interview question "how do you decide between a mock and a real implementation?" is a reliable filter — it reveals whether the candidate has felt the pain of an over-mocked codebase.

## How It's Used in C# / .NET

### Moq — `Mock<T>`, `Setup`, `Verify`

```csharp
using Moq;

[Fact]
public async Task Place_DeclinedPayment_DoesNotSendConfirmation()
{
    var gateway = new Mock<IPaymentGateway>();
    gateway
        .Setup(g => g.ChargeAsync(It.IsAny<ChargeRequest>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(ChargeResult.Declined("card_declined"));

    var email = new Mock<IEmailSender>();
    var repo = new Mock<IOrderRepository>();

    var sut = new PlaceOrderHandler(gateway.Object, email.Object, repo.Object);

    var result = await sut.HandleAsync(new PlaceOrderCommand(/*...*/), CancellationToken.None);

    result.Status.Should().Be(OrderStatus.PaymentFailed);
    email.Verify(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(),
                                  It.IsAny<string>(), It.IsAny<CancellationToken>()),
                 Times.Never);
}
```

Argument matchers: `It.IsAny<T>()`, `It.Is<T>(x => x.Id == 42)`, `It.IsRegex("^ORD-")`, `It.IsInRange(1, 10, Range.Inclusive)`.

Sequenced returns:
```csharp
gateway.SetupSequence(g => g.ChargeAsync(It.IsAny<ChargeRequest>(), default))
       .ReturnsAsync(ChargeResult.TransientFailure())
       .ReturnsAsync(ChargeResult.Confirmed("ch_123"));
```

Strict vs loose:
```csharp
var mock = new Mock<IPaymentGateway>(MockBehavior.Strict); // unexpected calls throw
// vs MockBehavior.Loose (default) — unexpected calls return default(T)
```

### NSubstitute — `Substitute.For<T>`, `Received()`

```csharp
using NSubstitute;

[Fact]
public async Task Charge_RetriesOnTransientFailure()
{
    var gateway = Substitute.For<IPaymentGateway>();
    gateway.ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>())
           .Returns(
               ChargeResult.TransientFailure(),     // 1st call
               ChargeResult.TransientFailure(),     // 2nd call
               ChargeResult.Confirmed("ch_xyz"));   // 3rd call

    var sut = new RetryingPaymentService(gateway, maxAttempts: 3);

    var result = await sut.ChargeAsync(SampleRequest, CancellationToken.None);

    result.Status.Should().Be(ChargeStatus.Confirmed);
    await gateway.Received(3).ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>());
}
```

NSubstitute is generally easier to read; Moq has slightly more powerful argument matching. Either is fine — pick one per codebase and stay consistent.

### FakeItEasy

```csharp
using FakeItEasy;

var gateway = A.Fake<IPaymentGateway>();
A.CallTo(() => gateway.ChargeAsync(A<ChargeRequest>._, A<CancellationToken>._))
 .Returns(ChargeResult.Confirmed("ch_1"));

// later
A.CallTo(() => gateway.ChargeAsync(A<ChargeRequest>.That.Matches(r => r.Amount > 1000m), A<CancellationToken>._))
 .MustHaveHappenedOnceExactly();
```

### Hand-rolled fakes — usually the cleanest choice

```csharp
public sealed class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _store = new();

    public Task<Order?> GetAsync(Guid id, CancellationToken ct) =>
        Task.FromResult(_store.GetValueOrDefault(id));

    public Task AddAsync(Order order, CancellationToken ct)
    {
        if (_store.ContainsKey(order.Id))
            throw new DuplicateOrderException(order.Id);
        _store[order.Id] = order;
        return Task.CompletedTask;
    }

    public IReadOnlyCollection<Order> All() => _store.Values.ToList();
}
```

A fake encodes the **contract** of the interface once, and every test that uses it gets stable, refactor-resilient behavior. Sharing one well-built fake across 50 tests is far cheaper than writing 50 `mock.Setup(...)` blocks.

### `HttpClient` mocking via `HttpMessageHandler`

Don't mock `HttpClient` itself (it's a sealed-ish class with awkward virtual surface). Mock the `HttpMessageHandler`:

```csharp
public class FakeHttpMessageHandler : HttpMessageHandler
{
    public Queue<HttpResponseMessage> Responses { get; } = new();
    public List<HttpRequestMessage> Requests { get; } = new();

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        Requests.Add(request);
        return Task.FromResult(Responses.Dequeue());
    }
}

var handler = new FakeHttpMessageHandler();
handler.Responses.Enqueue(new HttpResponseMessage(HttpStatusCode.OK)
{
    Content = new StringContent("""{"status":"succeeded"}""")
});
var http = new HttpClient(handler) { BaseAddress = new Uri("https://api.stripe.test") };
var sut = new StripeClient(http);
```

Or use [WireMock.Net](https://github.com/WireMock-Net/WireMock.Net) for a real local HTTP server with matching rules — closer to a contract test than a mock.

### Mocking `TimeProvider` (.NET 8+)

```csharp
var clock = new FakeTimeProvider(DateTimeOffset.Parse("2026-06-04T10:00:00Z"));
var sut = new TokenIssuer(clock);

var token = sut.IssueToken(userId: 42);
token.ExpiresAt.Should().Be(DateTimeOffset.Parse("2026-06-04T10:15:00Z"));

clock.Advance(TimeSpan.FromMinutes(16));
sut.IsValid(token).Should().BeFalse();
```

## Advantages

- **Test isolation** — exercise the SUT without booting external systems.
- **Failure injection** — easy to simulate timeouts, 5xx, deadlocks, expired tokens.
- **Speed** — milliseconds per test, no network or disk.
- **Deterministic** — control return values, exceptions, timing.
- **Verifies interactions** at true boundaries — "an email *was* sent to the right address."
- **Decouples test from third-party** — no Stripe sandbox quota issues, no SMTP downtime.

## Disadvantages

- **Tight coupling to implementation** — over-mocking turns refactors into mass test failures.
- **False positives** — a mocked test can pass while the real integration is broken (the mocked return value doesn't match what the real service produces).
- **Setup verbosity** — `mock.Setup(x => x.Y(It.IsAny<>()...)).ReturnsAsync(...)` × six dependencies adds noise.
- **Hidden contract drift** — when the real interface changes, the mock returns stale shapes.
- **Risk of testing the mock, not the SUT** — assertions on `Verify(...)` instead of state.
- **Library learning curve** — Moq's syntax is non-obvious; NSubstitute's auto-substitution can surprise.

## Common Mistakes

### 1. Mocking everything, including pure logic

```csharp
// Bad — Money is a value object with no dependencies
var money = new Mock<Money>();
money.Setup(m => m.Add(It.IsAny<Money>())).Returns(new Money(100m, "USD"));
```

**Fix**: Use the real `Money` class. Mock only what you cannot construct cheaply.

### 2. Over-verifying internal interactions

```csharp
// Bad — couples test to internal sequencing
mockValidator.Verify(v => v.Validate(It.IsAny<Order>()), Times.Once);
mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>(), default), Times.Once);
mockPublisher.Verify(p => p.PublishAsync(It.IsAny<OrderPlaced>(), default), Times.Once);
mockLogger.Verify(l => l.LogInformation("Order placed: {OrderId}", It.IsAny<Guid>()), Times.Once);
```

**Fix**: Assert on the observable result. Use a `FakeOrderRepository` and check it contains the order; use a `CapturingEventPublisher` and check it has the right event. Forget the validator and logger.

```csharp
repo.All().Should().ContainSingle(o => o.Id == result.OrderId);
publisher.Published.Should().ContainSingle(e => e is OrderPlaced);
```

### 3. Mocking concrete classes (or worse, `sealed` ones)

```csharp
// Bad — mocking a concrete class with virtual methods is a smell
var service = new Mock<OrderService>(); // why? extract IOrderService or use the real class
```

**Fix**: Either depend on an interface (and mock that), or use the real class with a fake repository. Avoid `Mock<ConcreteClass>` — it leads to half-mocked objects that behave inconsistently.

### 4. Setting up calls the test never makes

```csharp
// Bad — 12 lines of Setup, only 2 are actually invoked by the SUT
mock.Setup(x => x.MethodA(...)).Returns(...);
mock.Setup(x => x.MethodB(...)).Returns(...);  // never called
mock.Setup(x => x.MethodC(...)).Returns(...);  // never called
// ...
```

**Fix**: Only set up what the SUT actually calls. Use `MockBehavior.Strict` for one test to surface unused setups, then keep them tight.

### 5. Asserting on `It.IsAny<>` for arguments that matter

```csharp
// Bad — the test passes even if you send a $0 charge
gateway.Verify(g => g.ChargeAsync(It.IsAny<ChargeRequest>(), default), Times.Once);
```

**Fix**: Match on the specific argument shape:

```csharp
gateway.Verify(g => g.ChargeAsync(
    It.Is<ChargeRequest>(r => r.Amount == 199m && r.Currency == "USD"),
    It.IsAny<CancellationToken>()), Times.Once);
```

### 6. Using mocks instead of fakes for stateful collaborators

```csharp
// Bad — every test re-sets up Get and Add behaviour, brittle and verbose
repoMock.Setup(r => r.GetAsync(It.IsAny<Guid>(), default)).ReturnsAsync((Order?)null);
repoMock.Setup(r => r.AddAsync(It.IsAny<Order>(), default)).Returns(Task.CompletedTask);
```

**Fix**: Write `InMemoryOrderRepository` once and share it across the test suite. Tests become shorter and refactor-resilient.

### 7. Mocking the clock with `DateTime.UtcNow` patches

```csharp
// Bad — depends on test running before midnight UTC, untestable boundary
if (DateTime.UtcNow > order.ExpiresAt) order.MarkExpired();
```

**Fix**: Inject `TimeProvider`, use `FakeTimeProvider` in tests.

### 8. Mocking `DbContext`

```csharp
// Bad — DbContext has too many touchpoints to mock reliably
var db = new Mock<AppDbContext>();
db.Setup(x => x.Set<Order>()).Returns(/* a mocked DbSet */);
```

**Fix**: Use the **repository pattern** so the handler depends on `IOrderRepository`, then fake that. If the SUT *is* the repository, write an integration test against Testcontainers SQL Server (see [integration-testing.md](integration-testing.md)).

## Best Practices

- **Prefer fakes for stateful collaborators you own**, mocks for behavior-only externals (email, payments).
- **Mock the interface, not the class** — design for testability with abstractions at real boundaries.
- **One mock library per codebase** — pick Moq or NSubstitute; mixing them creates context-switching cost.
- **Assert on state, not interactions** — except at true boundaries where "was X called" *is* the contract.
- **Use `It.Is<T>(...)` to match meaningful arguments**, not `It.IsAny<T>()` for everything.
- **Reset mocks between tests** if you reuse them in a class fixture (better: don't — make them per-test).
- **Don't mock what you don't own** unless wrapped in an interface — mocking `HttpClient`, `DbContext`, third-party SDKs leads to brittle tests; wrap them.
- **Verify call count carefully** — `Times.AtLeastOnce` is usually right when you care, `Times.Exactly(n)` when count matters.
- **Use spies (capturing fakes) over verify-mocks** for boundary tests — assert on captured data in the same readable shape as other assertions.
- **Combine with [testable-code-design.md](testable-code-design.md)** — small classes with few dependencies are easy to mock and easy to use real.

## Related Concepts

- **[unit-testing.md](unit-testing.md)** — the layer that uses mocks/fakes.
- **[integration-testing.md](integration-testing.md)** — where you stop mocking and use real components.
- **[testable-code-design.md](testable-code-design.md)** — design choices that make mocking effective.
- **[../csharp/dependency-injection.md](../csharp/dependency-injection.md)** — the seam that allows test doubles to be injected.
- **Test Double pattern (Gerard Meszaros)** — the formal taxonomy of dummy/stub/fake/mock/spy.
- **London school vs Detroit school of TDD** — London emphasizes interaction testing with mocks; Detroit (Chicago) emphasizes state testing. Most real teams blend both.
- **Property-based testing** — FsCheck generates inputs, often pairs with fakes for the SUT's dependencies.
- **WireMock.Net** — a local HTTP server for testing HTTP clients without mocking `HttpClient`.

## Real-World Usage

In a typical .NET checkout service:

- `PlaceOrderHandler` is unit-tested with:
  - **Fakes** for `IOrderRepository`, `IEventPublisher`, `IInventoryChecker` (you own them, in-memory is cleanest).
  - **Mocks/spies** for `IPaymentGateway`, `IEmailSender` (true boundaries; verify "charge was attempted with $199" and "confirmation email was queued").
  - **`FakeTimeProvider`** for any time-sensitive logic.
- `EfOrderRepository` is *integration-tested* against Testcontainers SQL Server — no mocking of `DbContext`.
- `StripePaymentGateway` is contract-tested with WireMock simulating Stripe's HTTP responses; an additional manual test hits the real Stripe sandbox once per release.

Mock library: NSubstitute. The team standardised on it after a Moq codebase grew 4,000 `.Setup(...)` calls that broke on every refactor; switching to fakes for stateful collaborators cut the test count by 20% while improving coverage.

## Code Example — Before and After

### Before — over-mocked, brittle, tests implementation

```csharp
[Fact]
public async Task Place_HappyPath_CallsEverythingInOrder()
{
    var validator = new Mock<IOrderValidator>();
    var inventory = new Mock<IInventoryChecker>();
    var pricing = new Mock<IPricingService>();
    var repo = new Mock<IOrderRepository>();
    var publisher = new Mock<IEventPublisher>();
    var payment = new Mock<IPaymentGateway>();
    var email = new Mock<IEmailSender>();

    validator.Setup(v => v.Validate(It.IsAny<PlaceOrderCommand>()))
             .Returns(ValidationResult.Success());
    inventory.Setup(i => i.IsAvailable(It.IsAny<string>(), It.IsAny<int>(), default))
             .ReturnsAsync(true);
    pricing.Setup(p => p.Price(It.IsAny<string>(), It.IsAny<int>(), default))
           .ReturnsAsync(199m);
    repo.Setup(r => r.AddAsync(It.IsAny<Order>(), default)).Returns(Task.CompletedTask);
    payment.Setup(p => p.ChargeAsync(It.IsAny<ChargeRequest>(), default))
           .ReturnsAsync(ChargeResult.Confirmed("ch_1"));
    publisher.Setup(p => p.PublishAsync(It.IsAny<OrderPlaced>(), default))
             .Returns(Task.CompletedTask);
    email.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(),
                                  It.IsAny<string>(), default))
         .Returns(Task.CompletedTask);

    var sut = new PlaceOrderHandler(
        validator.Object, inventory.Object, pricing.Object,
        repo.Object, publisher.Object, payment.Object, email.Object);

    await sut.HandleAsync(new PlaceOrderCommand("cust", "P-1", 1), default);

    validator.Verify(v => v.Validate(It.IsAny<PlaceOrderCommand>()), Times.Once);
    inventory.Verify(i => i.IsAvailable("P-1", 1, default), Times.Once);
    pricing.Verify(p => p.Price("P-1", 1, default), Times.Once);
    repo.Verify(r => r.AddAsync(It.IsAny<Order>(), default), Times.Once);
    payment.Verify(p => p.ChargeAsync(It.IsAny<ChargeRequest>(), default), Times.Once);
    publisher.Verify(p => p.PublishAsync(It.IsAny<OrderPlaced>(), default), Times.Once);
    email.Verify(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>(),
                                   It.IsAny<string>(), default), Times.Once);
}
```

40 lines of plumbing. Every internal refactor breaks the test. No assertion on the actual outcome.

### After — fakes for owned state, mocks only at boundaries, assert on outcome

```csharp
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _store = new();
    public Task AddAsync(Order o, CancellationToken ct) { _store[o.Id] = o; return Task.CompletedTask; }
    public Task<Order?> GetAsync(Guid id, CancellationToken ct) =>
        Task.FromResult(_store.GetValueOrDefault(id));
    public IReadOnlyCollection<Order> All() => _store.Values.ToList();
}

public class CapturingEventPublisher : IEventPublisher
{
    public List<object> Events { get; } = new();
    public Task PublishAsync(object evt, CancellationToken ct) { Events.Add(evt); return Task.CompletedTask; }
}

public class CapturingEmailSender : IEmailSender
{
    public List<(string To, string Subject)> Sent { get; } = new();
    public Task SendAsync(string to, string subject, string body, CancellationToken ct)
    {
        Sent.Add((to, subject));
        return Task.CompletedTask;
    }
}

[Fact]
public async Task Place_HappyPath_PersistsOrder_PublishesEvent_SendsEmail()
{
    var repo = new InMemoryOrderRepository();
    var publisher = new CapturingEventPublisher();
    var email = new CapturingEmailSender();
    var payment = Substitute.For<IPaymentGateway>();
    payment.ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>())
           .Returns(ChargeResult.Confirmed("ch_1"));

    var sut = new PlaceOrderHandler(
        new AlwaysValidValidator(),
        new AlwaysInStockChecker(),
        new FixedPricer(199m),
        repo, publisher, payment, email);

    var result = await sut.HandleAsync(
        new PlaceOrderCommand("alice@contoso.com", "P-1", 1), CancellationToken.None);

    result.Status.Should().Be(OrderStatus.Confirmed);
    repo.All().Should().ContainSingle(o => o.Total == 199m);
    publisher.Events.OfType<OrderPlaced>().Should().ContainSingle();
    email.Sent.Should().ContainSingle(s => s.To == "alice@contoso.com" && s.Subject.Contains("Confirmation"));

    await payment.Received(1).ChargeAsync(
        Arg.Is<ChargeRequest>(r => r.Amount == 199m),
        Arg.Any<CancellationToken>());
}
```

Half the lines. Refactor the internal pipeline freely — the test only cares about the outcome and the boundary contract. The single `payment.Received(1)` mock-style assertion is justified because charging the card *is* the contract that matters.

## Interview Questions and Answers

### 1. What's the difference between a mock, a fake, and a stub? When do you reach for each?

**Why this matters**: A confident answer here signals years of test-suite ownership. Vague answers signal copy-pasting `Mock<T>` without understanding.

**Answer**: A **stub** returns canned values, nothing more. A **fake** is a simplified but working implementation (e.g., `InMemoryOrderRepository`) that respects the interface contract. A **mock** is a stub that *also* records calls so the test can `Verify` interactions. Use stubs for inputs you don't care about (`IRateProvider` returning 1.0), fakes for collaborators you own with state (repositories, caches, event publishers), and mocks for true external boundaries where you need to verify "this was called with these arguments" (payment, email, audit log).

**Trade-off**: Mocks couple tests to implementation; fakes encode the contract once. Prefer fakes for owned dependencies, reserve mocks for the moment of truth at the boundary.

**Real project**: In a billing service, `IOrderRepository` had 200 tests using `Mock<IOrderRepository>.Setup(...)`. Replacing with `InMemoryOrderRepository` cut 1,200 setup lines and made the suite refactor-resilient.

### 2. When should you NOT mock something?

**Answer**: Don't mock:

- **Pure logic and value objects** — there's nothing to substitute.
- **Concrete classes you own** — extract an interface only if the abstraction is genuinely useful, otherwise use the real class with fake collaborators.
- **Static .NET APIs you trust** — `JsonSerializer`, `Guid.NewGuid`, `Convert.ToInt32`. Wrapping them in `IJsonSerializer` just to mock adds noise.
- **`DbContext`** — too many interaction points, mocking it produces tests that pass while the real database fails. Use a repository abstraction instead.
- **Anything where the real implementation is fast and side-effect free** — `Money.Add`, `RegexCompiledOnce.IsMatch`.

**Trade-off**: Avoiding mocking when possible reduces test brittleness but requires good design (small classes, narrow interfaces). The pain of needing to mock something is often a signal to refactor the SUT, not to add a new abstraction.

**Real project**: A team wrapped `DateTime.UtcNow` in `ITimeService` purely to mock it. When .NET 8 shipped `TimeProvider`, they migrated to `FakeTimeProvider` and deleted 30 lines of pointless abstraction.

### 3. Walk through testing a method that retries a failing HTTP call three times. How do you set up the mock?

**Answer**: Use NSubstitute's sequenced returns (or Moq's `SetupSequence`) so the same call returns different results in order. Assert both the final state and that the call was attempted the expected number of times:

```csharp
[Fact]
public async Task ChargeAsync_RetriesUpToThreeTimes_OnTransientFailure()
{
    var gateway = Substitute.For<IPaymentGateway>();
    gateway.ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>())
           .Returns(
               ChargeResult.TransientFailure(),
               ChargeResult.TransientFailure(),
               ChargeResult.Confirmed("ch_1"));

    var sut = new RetryingPaymentService(gateway, maxAttempts: 3);

    var result = await sut.ChargeAsync(SampleRequest, CancellationToken.None);

    result.Status.Should().Be(ChargeStatus.Confirmed);
    await gateway.Received(3)
                 .ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>());
}
```

**Trade-off**: This test couples to the retry count. If you want to test the *policy* (3 attempts), this is fine. If you want to test that "eventually it succeeds", remove the `Received(3)` assertion.

**Real project**: For a Polly-wrapped Stripe client, this exact pattern verified that `WaitAndRetryAsync(3, ...)` configurations were honored — and caught a bug where the policy was misconfigured to only retry on 4xx, never 5xx.

### 4. How do you simulate a network timeout in a unit test?

**Answer**: Configure the mock to throw the exception your real client would throw:

```csharp
var gateway = Substitute.For<IPaymentGateway>();
gateway.ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>())
       .ThrowsAsync(new TaskCanceledException("Timed out", new TimeoutException()));

var sut = new PlaceOrderHandler(/*...*/, gateway);

var result = await sut.HandleAsync(SampleCmd, CancellationToken.None);

result.Status.Should().Be(OrderStatus.PaymentPending);
// And the SUT should NOT have published OrderConfirmed
```

For `HttpClient`-based code, throw inside the `HttpMessageHandler`:

```csharp
public class TimingOutHandler : HttpMessageHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage r, CancellationToken ct)
        => throw new TaskCanceledException("Request timed out");
}
```

**Trade-off**: Simulated exceptions are deterministic but only test code paths *if* the SUT correctly catches that specific exception type. Cross-check with one integration test that uses a real slow endpoint (e.g., WireMock with `WithDelay`).

**Real project**: A test simulating a Stripe timeout caught that the SUT was catching `Exception` and treating timeouts as "payment confirmed" — a money-losing bug. Production logs later confirmed the same exception had silently caused 12 unintended charges.

### 5. Your team has 4,000 `Mock<T>.Setup(...)` calls and tests break on every refactor. How would you fix this?

**Why this matters**: This is the most common reality in older .NET codebases. The candidate's response reveals their engineering maturity.

**Answer**: Three parallel actions:

1. **Replace stateful mocks with hand-rolled fakes** — `InMemoryOrderRepository`, `CapturingEventPublisher`, `FakeFeatureFlagProvider`. One fake replaces dozens of `.Setup` blocks.
2. **Shift assertions from `.Verify(...)` to state checks** — `repo.All().Should().Contain(...)` instead of `mock.Verify(r => r.Add(...), Times.Once)`.
3. **Bring in integration tests for the layers that don't need mocking** — use Testcontainers + `WebApplicationFactory` so repository-level tests don't need mocked `DbContext` at all.

Do this incrementally — pick the most-edited handler, refactor its tests, prove the team it's faster to maintain, then spread. Don't try to convert all 4,000 in one PR.

**Trade-off**: Some teams will resist "rewriting tests" — frame it as "removing test debt" with the velocity metric (PRs blocked on test fixes) as the justification.

**Real project**: A team I worked with had 30% of PRs blocked on test failures unrelated to the code change. After 3 months of incremental fake-isation, that dropped to 5% and the test suite ran 40% faster.

### 6. How do you test that the right event was published without coupling to "what other methods were called"?

**Answer**: Use a capturing fake event publisher and assert on the captured collection:

```csharp
public class CapturingEventPublisher : IEventPublisher
{
    public List<object> Published { get; } = new();
    public Task PublishAsync(object evt, CancellationToken ct)
    {
        Published.Add(evt);
        return Task.CompletedTask;
    }
}

[Fact]
public async Task Place_Successful_PublishesOrderPlaced()
{
    var publisher = new CapturingEventPublisher();
    var sut = BuildHandler(publisher: publisher);

    await sut.HandleAsync(SampleCmd, CancellationToken.None);

    publisher.Published
             .OfType<OrderPlaced>()
             .Should()
             .ContainSingle(e => e.OrderId != Guid.Empty && e.CustomerId == SampleCmd.CustomerId);
}
```

This is robust to refactoring: move the publish into a decorator, a pipeline behavior, or a domain event dispatcher — the test still passes as long as `OrderPlaced` ends up in `Published`.

**Trade-off**: A `Mock<IEventPublisher>.Verify(...)` is one line shorter. The fake pays off as soon as a second test needs to assert on the same publisher.

### 7. A senior engineer says "fakes are duplication; we should mock everything." How do you respond?

**Why this matters**: Tests the candidate's ability to push back constructively.

**Answer**: I'd reframe: fakes *concentrate* knowledge of the contract in one place. The `InMemoryOrderRepository` encodes "Add throws on duplicate id, Get returns null for missing" once; every test gets that behavior for free. Mocks, by contrast, *duplicate* that knowledge across every `.Setup(...)` call — 50 tests means 50 places that re-encode the same contract.

When the interface changes (say, `AddAsync` becomes `AddAsync(Order, IdempotencyKey, CancellationToken)`), the fake breaks once, in one file, with a compiler error pointing exactly where. Mocks break in 50 tests at runtime, often silently passing if the setup doesn't match (because `Loose` mocks return defaults).

I'd offer a concrete experiment: pick one repository, write its fake, migrate ten tests, and compare the line count and the refactor pain over two weeks.

**Trade-off**: Fakes do require upfront design — the initial in-memory implementation takes time. Mocks are quicker to start but pay a higher long-term tax.

**Real project**: After migrating ten `IOrderRepository.Setup` chains to one `InMemoryOrderRepository`, the team adopted it for all repositories. Test code shrank by 25% across the codebase.

### 8. When does a mock-heavy test still add value versus being deleted?

**Answer**: Even an over-mocked test adds value when:

- It's the **only test** for a critical money path (delete it last).
- It documents a non-obvious **boundary contract** ("the charge call must always include the idempotency key").
- It reproduces a **regression** ("after this PR, the SUT stopped publishing the event").

It should be deleted (or rewritten) when:

- The setup is longer than the production method.
- It fails every time anyone refactors the SUT, even though the behavior is identical.
- The only assertion is "method X was called" with no consequence to the user.
- Two or more tests in the same file repeat the same mock setup verbatim — they should share a fake or a helper.

**Trade-off**: Don't crusade for deletion in one PR. Mark suspect tests with `[Trait("Smell", "OverMocked")]`, then revisit them when they next fail.

**Real project**: A team triaged 4,000 mock-heavy tests over a quarter, deleted 600 redundant ones, rewrote 1,200 to use fakes, and kept the rest. Net result: 25% fewer tests, 40% faster suite, and noticeably less PR friction.

## Summary Checklist

- [ ] I know the difference between dummy, stub, fake, mock, and spy.
- [ ] I prefer fakes for stateful collaborators I own.
- [ ] I reserve mocks for true external boundaries (payment, email, third-party APIs).
- [ ] I assert on state, not interactions — except at those boundaries.
- [ ] I never mock pure value objects, static APIs, or `DbContext`.
- [ ] I use `It.Is<T>(...)` / `Arg.Is<T>(...)` to match meaningful arguments, not `It.IsAny<>` for everything.
- [ ] I use `FakeTimeProvider` instead of patching `DateTime.UtcNow`.
- [ ] I write `HttpMessageHandler` fakes instead of mocking `HttpClient`.
- [ ] My tests survive internal refactors of the SUT without changes.
- [ ] When a test needs 40 lines of `.Setup`, I treat that as a design signal, not a test problem.
