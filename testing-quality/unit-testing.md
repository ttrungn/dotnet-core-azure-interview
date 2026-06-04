# Unit Testing

## What It Is

A **unit test** is a short, automated test that exercises a single piece of behavior — usually one method on one class — in complete isolation from the outside world. No database, no HTTP, no file system, no real clock, no Azure SDK calls. Only in-process code, fake collaborators, and assertions on the result.

In .NET, a unit test is a method in a class library (`*.Tests.csproj`) referencing a test framework (xUnit, NUnit, or MSTest). The test framework discovers methods marked with `[Fact]` / `[Theory]` (xUnit) and runs them in parallel. The test creates the **system under test (SUT)** with fake dependencies, calls one method on it, and asserts the outcome.

```csharp
[Fact]
public void Place_WhenInventoryAvailable_ReturnsConfirmed()
{
    // Arrange
    var inventory = new FakeInventoryChecker(available: true);
    var sut = new PlaceOrderHandler(inventory);

    // Act
    var result = sut.Place(new OrderRequest(productId: "P-1", quantity: 2));

    // Assert
    result.Status.Should().Be(OrderStatus.Confirmed);
}
```

Three properties define a unit test:

1. **Fast** — milliseconds per test. A 2,000-test suite should finish in under 10 seconds.
2. **Deterministic** — the same input always produces the same result. No flakes from threading, random data, or time.
3. **Isolated** — independent of other tests, the file system, the network, the clock, and shared global state.

If a test waits on `Thread.Sleep`, hits SQL Server, or fails on Tuesdays when the build agent is in a different timezone, it is *not* a unit test. It belongs in [integration-testing.md](integration-testing.md).

## Why It Exists

Before automated tests, teams relied on **manual QA** plus production telemetry. A regression in a discount rule might be discovered three weeks after release when a customer complained. The cost of that defect — engineering time to find it, the refund issued, the brand damage — was orders of magnitude greater than the cost of a 50-line test that would have caught it at the developer's keyboard.

Unit tests exist to make **the feedback loop short enough that bugs are caught while the code is still on screen**. A developer changes the discount calculation, presses `Ctrl+R, T`, and 800 tests pass in 3 seconds. If something breaks, they see the failed assertion *now*, not in three weeks.

They also exist to enable **safe refactoring**. Without tests, every change is risky — touching a 10-year-old `PricingService` could break invoicing in ways no one notices until the next month-end close. With tests around the externally observable behavior, you can rewrite the internals confidently, because any regression turns a green bar red.

## When To Use It

**Use unit tests for**:

- **Business rules and calculations** — discount logic, tax computation, eligibility checks, fraud scoring.
- **Domain entities and value objects** — `Money.Add`, `DateRange.Overlaps`, `OrderStatus` transitions.
- **Application services / command handlers** — `PlaceOrderHandler`, `CancelSubscriptionHandler`.
- **Validators** — FluentValidation rules, custom input checks.
- **Pure functions** — anything that takes data in and returns data out, with no side effects.
- **Mappers** — DTO-to-domain projection logic where mistakes silently corrupt data.

**Do not use unit tests for**:

- **EF Core query behavior** — `Where` / `Include` shapes only run correctly against a real provider. Use integration tests with Testcontainers (see [integration-testing.md](integration-testing.md)).
- **ASP.NET Core routing, model binding, filter pipeline** — these need `WebApplicationFactory`.
- **Database migrations** — must run against a real engine.
- **HTTP client behavior** — JSON serialization edge cases, status code handling, retries — use a real `HttpMessageHandler` test double or `WireMock.Net`.
- **Trivial getters/setters and auto-properties** — they have no behavior worth testing.
- **Code that is mostly framework glue** — testing that `[Authorize]` is applied is more an integration concern.

## Why It Is Important

Unit tests pay back four kinds of value, every day:

1. **Confidence to change code** — without tests, engineers fear touching a working class. With tests, they refactor freely because regressions show up immediately.
2. **Living documentation** — a well-named test (`Place_WhenInventoryUnavailable_ReturnsBackordered`) tells the next reader exactly what the rule is. This survives long after the original engineer leaves.
3. **Design feedback** — if a class is hard to test, it is usually badly designed (too many dependencies, hidden state, mixed concerns). The pain you feel writing a test is a signal — fix the design, not the test.
4. **Cheaper bugs** — a bug caught at the developer's machine costs minutes. The same bug caught in QA costs hours. In production it costs days of incident response plus customer trust.

In an interview context, unit testing is also a proxy for several other skills: you cannot write good unit tests without understanding dependency injection (see [../csharp/dependency-injection.md](../csharp/dependency-injection.md)), SOLID principles (see [../architecture/solid-principles.md](../architecture/solid-principles.md)), and clean code (see [clean-code.md](clean-code.md)).

## How It's Used in C# / .NET

### Test frameworks

Three mainstream choices:

| Framework | Strengths                                     | Project template       |
|-----------|-----------------------------------------------|------------------------|
| **xUnit** | Modern, parallel by default, no `[SetUp]` — uses constructor / `IDisposable` | `dotnet new xunit`     |
| **NUnit** | Rich attribute set (`[TestCase]`, `[Setup]`)  | `dotnet new nunit`     |
| **MSTest**| Built into Visual Studio, used at Microsoft   | `dotnet new mstest`    |

xUnit is the de facto default for new ASP.NET Core projects. The examples on this page use xUnit.

```bash
dotnet new xunit -n Billing.Tests
dotnet add Billing.Tests reference src/Billing/Billing.csproj
dotnet add Billing.Tests package FluentAssertions
dotnet add Billing.Tests package NSubstitute
dotnet test
```

### Key xUnit attributes

```csharp
public class DiscountCalculatorTests
{
    // A single test with no parameters
    [Fact]
    public void Calculate_BronzeTier_Returns0Percent()
    {
        var sut = new DiscountCalculator();
        sut.Calculate(CustomerTier.Bronze, orderTotal: 100m).Should().Be(0m);
    }

    // A parameterized test — runs once per InlineData
    [Theory]
    [InlineData(CustomerTier.Bronze, 100, 0)]
    [InlineData(CustomerTier.Silver, 100, 5)]
    [InlineData(CustomerTier.Gold,   100, 10)]
    [InlineData(CustomerTier.Gold,   500, 50)]
    public void Calculate_ReturnsExpectedDiscount(CustomerTier tier, decimal total, decimal expected)
    {
        new DiscountCalculator().Calculate(tier, total).Should().Be(expected);
    }

    // External data source (CSV / database / generator)
    [Theory]
    [MemberData(nameof(EdgeCases))]
    public void Calculate_HandlesEdgeCases(CustomerTier tier, decimal total, decimal expected) { /* ... */ }

    public static IEnumerable<object[]> EdgeCases()
    {
        yield return new object[] { CustomerTier.Gold, 0m, 0m };
        yield return new object[] { CustomerTier.Gold, decimal.MaxValue, decimal.MaxValue * 0.1m };
    }
}
```

### AAA pattern — Arrange, Act, Assert

Every test reads in three blocks. The blank line between them is the only "comment" you need:

```csharp
[Fact]
public async Task Place_WhenCardDeclined_RaisesPaymentFailedEvent()
{
    // Arrange
    var payments = Substitute.For<IPaymentGateway>();
    payments.ChargeAsync(Arg.Any<ChargeRequest>(), Arg.Any<CancellationToken>())
            .Returns(ChargeResult.Declined("insufficient_funds"));
    var events = new FakeEventPublisher();
    var sut = new PlaceOrderHandler(payments, events);

    // Act
    var result = await sut.HandleAsync(new PlaceOrderCommand(customerId: 42, amount: 199m), CancellationToken.None);

    // Assert
    result.Status.Should().Be(OrderStatus.PaymentFailed);
    events.Published.Should().ContainSingle(e => e is PaymentFailed);
}
```

### Naming convention — `Method_State_ExpectedOutcome`

Roy Osherove's convention is the most readable for backend services:

```
Place_WhenInventoryUnavailable_ReturnsBackordered
ChargeAsync_NetworkTimeout_ThrowsPaymentTimeoutException
ApplyDiscount_TotalBelowMinimum_DoesNotApplyDiscount
```

A failed test name should tell you what broke without opening the test file.

### FluentAssertions

`Assert.Equal(expected, actual)` is fine; FluentAssertions reads better and produces sharper failure messages:

```csharp
result.Should().NotBeNull();
result.Status.Should().Be(OrderStatus.Confirmed);
result.LineItems.Should().HaveCount(3).And.OnlyContain(li => li.Quantity > 0);
result.Total.Should().BeApproximately(199.99m, 0.01m);

// Collections
orders.Should().BeInAscendingOrder(o => o.CreatedAt);
orders.Should().AllSatisfy(o => o.CustomerId.Should().NotBeEmpty());

// Exceptions
var act = () => sut.Charge(-10m);
act.Should().Throw<ArgumentException>().WithMessage("*amount*");

// Async exceptions
var act = async () => await sut.ChargeAsync(req, ct);
await act.Should().ThrowAsync<PaymentDeclinedException>();
```

### Testing async code

`async Task` test methods are awaited by xUnit. Never `.Result` or `.Wait()` inside a test — you can deadlock in some sync contexts and you lose proper exception unwrapping:

```csharp
[Fact]
public async Task SendAsync_RetriesOnTransientFailure()
{
    var sender = new RetryingEmailSender(transient: 2);
    await sender.SendAsync("a@b.com", "subject", "body", CancellationToken.None);
    sender.Attempts.Should().Be(3);
}
```

For testing cancellation, pass a `CancellationTokenSource` with a short timeout or an already-cancelled token, then assert `OperationCanceledException`:

```csharp
[Fact]
public async Task ProcessAsync_RespectsCancellation()
{
    using var cts = new CancellationTokenSource();
    cts.Cancel();
    var act = async () => await sut.ProcessAsync(cts.Token);
    await act.Should().ThrowAsync<OperationCanceledException>();
}
```

### Test isolation

xUnit creates a **new instance of the test class for every test**, so instance fields are per-test:

```csharp
public class OrderRepositoryTests
{
    private readonly InMemoryOrderRepository _repo = new(); // fresh per test, no SetUp needed

    [Fact] public void Add_NewOrder_StoresIt() { /* ... */ }
    [Fact] public void Get_MissingId_ReturnsNull() { /* ... */ }
}
```

Use `IClassFixture<T>` for expensive setup that *can* be shared safely. Avoid `static` fields like the plague — they are the #1 source of order-dependent test flakes.

### Code coverage

`dotnet test --collect:"XPlat Code Coverage"` writes a Cobertura file. Convert it to a human-readable report with ReportGenerator:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:coveragereport -reporttypes:Html
```

Coverage is a smoke detector, not a quality metric. 100% line coverage on a getter proves nothing. Aim for **branch coverage on business rules**, not a number on a dashboard.

## Advantages

- **Sub-second feedback** — change a method, run a focused test, see green or red instantly.
- **Refactoring safety net** — rename, split, or rewrite internals without fear.
- **Executable specification** — test names document the rules; no out-of-date wiki page.
- **Design pressure** — hard-to-test code reveals over-coupling early.
- **Cheap to run in CI** — thousands of tests in seconds, no infrastructure.
- **Localized failures** — a single failing test points to one class, not "something in the order pipeline."

## Disadvantages

- **False confidence with bad assertions** — a green test that asserts nothing meaningful is worse than no test.
- **Brittleness from over-mocking** — tests coupled to implementation details break on every refactor.
- **No coverage of integration seams** — unit tests cannot prove that the SQL query returns the right rows.
- **Setup boilerplate for complex constructors** — six dependencies means six fakes per test.
- **Can encourage navel-gazing** — engineers write 200 unit tests for an internal helper while the HTTP endpoint has zero integration coverage.
- **Maintenance cost** — every test is code that must be kept in sync with the production code.

## Common Mistakes

### 1. Testing implementation details instead of behavior

```csharp
// Bad — asserts that an internal method was called in a specific order
[Fact]
public void Place_CallsValidatorThenRepositoryThenPublisher()
{
    var validator = Substitute.For<IOrderValidator>();
    var repo = Substitute.For<IOrderRepository>();
    var publisher = Substitute.For<IEventPublisher>();
    var sut = new PlaceOrderHandler(validator, repo, publisher);

    sut.Handle(SampleOrder);

    Received.InOrder(() =>
    {
        validator.Validate(Arg.Any<Order>());
        repo.Save(Arg.Any<Order>());
        publisher.Publish(Arg.Any<OrderPlaced>());
    });
}
```

This test fails the moment you refactor (e.g., move publish into a decorator), even though the behavior is identical.

**Fix**: Assert the outcome that the caller actually cares about.

```csharp
[Fact]
public void Place_ValidOrder_PersistsOrderAndPublishesEvent()
{
    var repo = new InMemoryOrderRepository();
    var publisher = new FakeEventPublisher();
    var sut = new PlaceOrderHandler(new AlwaysValidValidator(), repo, publisher);

    var id = sut.Handle(SampleOrder);

    repo.Get(id).Should().NotBeNull();
    publisher.Published.OfType<OrderPlaced>().Should().ContainSingle(e => e.OrderId == id);
}
```

### 2. Shared state between tests

```csharp
// Bad — static field shared by every test in the assembly
public class CustomerRepositoryTests
{
    private static readonly List<Customer> _customers = new();

    [Fact]
    public void Add_NewCustomer_IncreasesCount()
    {
        _customers.Add(new Customer("Alice"));
        _customers.Count.Should().Be(1); // fails when other tests already added entries
    }
}
```

**Fix**: Make state instance-level. xUnit gives you a fresh instance per test for free.

```csharp
public class CustomerRepositoryTests
{
    private readonly InMemoryCustomerRepository _repo = new();

    [Fact]
    public void Add_NewCustomer_StoresIt()
    {
        _repo.Add(new Customer("Alice"));
        _repo.All().Should().HaveCount(1);
    }
}
```

### 3. No naming convention — `Test1`, `Test2`, `OrderTest`

```csharp
[Fact] public void Test1() { /* ... */ }
[Fact] public void OrderTest() { /* ... */ }
[Fact] public void Test_works() { /* ... */ }
```

When this test fails in CI, you have no idea what broke.

**Fix**: `Method_State_ExpectedOutcome`. Three underscores, no apologies.

```csharp
[Fact] public void Place_WhenInventoryUnavailable_ReturnsBackordered() { }
[Fact] public void Place_WhenPaymentDeclined_RaisesPaymentFailedEvent() { }
```

### 4. Tests that call the real database, network, or clock

```csharp
// Bad — these are integration tests masquerading as unit tests
[Fact]
public async Task PlaceOrder_StoresInDb()
{
    var db = new SqlConnection("Server=...;");      // real database
    var http = new HttpClient();                    // real network
    var sut = new OrderService(db, http);
    await sut.Place(SampleOrder);
}
```

**Fix**: Inject abstractions. Use `InMemoryDbRepository`, `FakePaymentGateway`, `FakeTimeProvider`. Move the real-database test into the integration suite under [integration-testing.md](integration-testing.md).

### 5. Asserting on the wrong layer with brittle mocks

```csharp
// Bad — verifies a logger call. Tomorrow's "format the message differently" change breaks the test
mockLogger.Verify(l => l.LogInformation(
    "Order {OrderId} placed by {CustomerId}",
    It.IsAny<Guid>(),
    It.IsAny<Guid>()), Times.Once);
```

**Fix**: Don't assert on logging or telemetry calls. Assert on observable state — the database now has the order, the event was published, the response status is `Created`.

### 6. Multiple `Act` blocks per test

```csharp
[Fact]
public void OrderWorkflow()
{
    sut.Place(order);   // Act 1
    sut.Cancel(order);  // Act 2
    sut.Refund(order);  // Act 3
    // many assertions
}
```

When this fails, which step broke? **One test → one Act → one logical assertion topic.**

### 7. Using `[Fact]` when `[Theory]` is correct

```csharp
[Fact] public void Tax_5Percent_OnHundred_Returns5() => Calc(5, 100).Should().Be(5);
[Fact] public void Tax_10Percent_OnHundred_Returns10() => Calc(10, 100).Should().Be(10);
[Fact] public void Tax_0Percent_OnHundred_Returns0() => Calc(0, 100).Should().Be(0);
```

**Fix**: One `[Theory]` with `[InlineData]`:

```csharp
[Theory]
[InlineData(5, 100, 5)]
[InlineData(10, 100, 10)]
[InlineData(0, 100, 0)]
public void Tax_Calculates(decimal rate, decimal amount, decimal expected) =>
    Calc(rate, amount).Should().Be(expected);
```

### 8. Testing trivial code

```csharp
[Fact]
public void CustomerName_GetSet_RoundTrips()
{
    var c = new Customer { Name = "Alice" };
    c.Name.Should().Be("Alice"); // testing the C# language, not your code
}
```

These tests have a maintenance cost and no value. Skip them.

## Best Practices

- **One logical assertion per test** — even if it's three `Should()` calls on related properties of one returned object.
- **Use FluentAssertions** for readable failure messages.
- **AAA pattern with blank lines** between sections — no `// Arrange` comments needed.
- **Name tests `Method_State_ExpectedOutcome`** so the failure tells the story.
- **Inject `TimeProvider`** instead of using `DateTime.UtcNow`; use `FakeTimeProvider` in tests.
- **Use `[Theory]` + `[InlineData]`** for the same logic with different inputs.
- **Prefer fakes over mocks** for dependencies you own (see [mocking.md](mocking.md)).
- **Assert on outcomes, not interactions** — except at true boundaries (e.g., "an email *was* sent").
- **Keep tests under 20 lines** — if a test grows, the SUT is doing too much.
- **Run tests on every save** in `dotnet watch test` during development.
- **Treat a flaky test as a P1 bug**, not background noise. Quarantine and fix, do not "rerun until green."
- **Wire up coverage in CI**, but tune thresholds per project — block on regression, not on absolute number.

## Related Concepts

- **[integration-testing.md](integration-testing.md)** — when in-process fakes are not enough; HTTP and database boundaries.
- **[mocking.md](mocking.md)** — fakes, stubs, mocks, spies, and when each is appropriate.
- **[testable-code-design.md](testable-code-design.md)** — design choices that make code easy to unit-test.
- **[../csharp/dependency-injection.md](../csharp/dependency-injection.md)** — the wiring that lets you swap real implementations for test doubles.
- **[../architecture/solid-principles.md](../architecture/solid-principles.md)** — SRP and DIP directly enable testability.
- **TDD (Test-Driven Development)** — write the test first, then the code. Not required for unit testing but pairs well.
- **Property-based testing** — FsCheck for .NET; generates hundreds of inputs to find edge cases automatically.
- **Mutation testing** — Stryker.NET measures whether your tests actually catch real bugs, not just execute lines.

## Real-World Usage

In a typical ASP.NET Core e-commerce service, the unit-test project structure mirrors the source:

```
src/Billing/
    Application/
        PlaceOrderHandler.cs
        DiscountCalculator.cs
    Domain/
        Order.cs
        Money.cs
tests/Billing.Tests/
    Application/
        PlaceOrderHandlerTests.cs       (~15 tests, all branches of the workflow)
        DiscountCalculatorTests.cs      (~30 theory cases for tier × amount matrix)
    Domain/
        OrderTests.cs                   (state transition rules)
        MoneyTests.cs                   (arithmetic, currency mismatch errors)
```

Unit tests cover **application services, domain rules, and value objects**. Integration tests (separate project) cover **EF Core queries, HTTP endpoints, and message handlers**. On every PR, GitHub Actions runs `dotnet test`. Unit tests finish in ~5 seconds and act as a fast pre-flight check before the slower integration suite.

Coverage is published via Coverlet → Cobertura → SonarCloud. The team blocks PRs that drop branch coverage on `src/Billing/Application/` and `src/Billing/Domain/` below 85%. They explicitly *do not* track coverage on `Program.cs` or auto-generated mapping code.

## Code Example — Before and After

### Before — untestable, slow, brittle

```csharp
public class OrderService
{
    public async Task<OrderResult> PlaceAsync(OrderRequest request)
    {
        // Hard-coded SQL — needs a real database
        using var conn = new SqlConnection(
            Environment.GetEnvironmentVariable("CONNECTION_STRING"));
        await conn.OpenAsync();

        // Direct HTTP — needs the network
        using var http = new HttpClient();
        var fx = await http.GetStringAsync("https://api.fx.com/usd");

        // Direct clock — different result depending on time of day
        if (DateTime.UtcNow.Hour < 9)
            return new OrderResult { Status = "rejected", Reason = "store closed" };

        // ... 80 more lines mixing validation, pricing, persistence, emails
        return new OrderResult { Status = "ok" };
    }
}
```

Unit-testing this requires SQL Server, an internet connection, and a 9am clock. The test runs in 800ms when the network is fast, fails when it's not.

### After — testable, fast, deterministic

```csharp
public sealed class PlaceOrderHandler
{
    private readonly IOrderRepository _repo;
    private readonly IFxRateProvider _fx;
    private readonly IBusinessHoursPolicy _hours;
    private readonly TimeProvider _clock;

    public PlaceOrderHandler(IOrderRepository repo, IFxRateProvider fx,
                             IBusinessHoursPolicy hours, TimeProvider clock)
    {
        _repo = repo;
        _fx = fx;
        _hours = hours;
        _clock = clock;
    }

    public async Task<OrderResult> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        if (!_hours.IsOpen(_clock.GetUtcNow()))
            return OrderResult.Rejected("store closed");

        var rate = await _fx.GetRateAsync("USD", cmd.Currency, ct);
        var total = cmd.Amount * rate;

        var order = new Order(cmd.CustomerId, total);
        await _repo.SaveAsync(order, ct);
        return OrderResult.Confirmed(order.Id);
    }
}

// The test — no SQL, no HTTP, no real clock
public class PlaceOrderHandlerTests
{
    private readonly InMemoryOrderRepository _repo = new();
    private readonly FakeFxRateProvider _fx = new(rate: 1.0m);
    private readonly AlwaysOpenHours _hours = new();
    private readonly FakeTimeProvider _clock = new(DateTimeOffset.Parse("2026-06-04T10:00:00Z"));

    [Fact]
    public async Task Handle_DuringBusinessHours_ConfirmsOrder()
    {
        var sut = new PlaceOrderHandler(_repo, _fx, _hours, _clock);

        var result = await sut.HandleAsync(
            new PlaceOrderCommand(customerId: Guid.NewGuid(), amount: 100m, currency: "USD"),
            CancellationToken.None);

        result.Status.Should().Be(OrderStatus.Confirmed);
        _repo.All().Should().ContainSingle(o => o.Total == 100m);
    }

    [Fact]
    public async Task Handle_OutsideBusinessHours_RejectsWithReason()
    {
        var closedHours = new AlwaysClosedHours();
        var sut = new PlaceOrderHandler(_repo, _fx, closedHours, _clock);

        var result = await sut.HandleAsync(
            new PlaceOrderCommand(customerId: Guid.NewGuid(), amount: 100m, currency: "USD"),
            CancellationToken.None);

        result.Status.Should().Be(OrderStatus.Rejected);
        result.Reason.Should().Be("store closed");
    }
}
```

This runs in under 10ms. It is deterministic. It covers two branches explicitly and can be extended to cover FX failure, repository failure, and invalid currency without any infrastructure.

## Interview Questions and Answers

### 1. What makes something a "unit test" versus an integration test?

**Why this matters**: A surprising number of "unit tests" in real codebases hit a real SQL Server through EF Core's `UseSqlServer`. Knowing the boundary lets you build a healthy test pyramid.

**Answer**: A unit test runs entirely in-process with no external systems — no database, no HTTP, no file system, no real clock, no message broker. It calls one method on one class and asserts the outcome. If the test needs Docker, a connection string, or a network, it is an integration test. The practical signal is speed: a unit test runs in single-digit milliseconds.

**Trade-off**: The line is fuzzy in the middle. Using `Microsoft.EntityFrameworkCore.InMemory` is technically in-process but tests almost nothing about your real persistence layer — call that an integration test of the *handler* but not of the *query*.

**Real project**: In a checkout service, `DiscountCalculator` and `PlaceOrderHandler` get unit tests. `EfOrderRepository` and `POST /api/orders` get integration tests with Testcontainers SQL Server.

### 2. How do you decide what to assert in a unit test?

**Why this matters**: Over-asserting on internals creates brittle tests; under-asserting creates green tests that prove nothing.

**Answer**: Assert on **observable outcomes** that the caller cares about. For a query, that's the returned value. For a command, that's the state change in an injected repository plus any events raised. Do not assert on which private method was called, in what order, or what was logged. The test should still pass if you rewrite the internals to be more efficient, as long as the contract holds.

**Trade-off**: At true boundaries — payment, email, audit — you *do* want to verify the interaction. "Was the charge sent to the payment gateway with the right amount?" is a legitimate assertion because that's the contract with the outside world.

**Real project**: A `PlaceOrderHandler` test asserts the order ended up in the repository with the expected total and that `OrderPlaced` was published. It does *not* assert that `_validator.Validate()` was called first; if a future refactor moves validation into a decorator, the test should not care.

### 3. Walk me through how you'd write the first test for a `RefundCalculator` you've never seen before.

**Answer**: Read the class's public surface, pick the simplest happy path, and write the AAA test for it:

```csharp
[Fact]
public void Calculate_FullRefund_ReturnsOriginalAmount()
{
    var sut = new RefundCalculator();
    var result = sut.Calculate(originalAmount: 100m, refundType: RefundType.Full);
    result.Should().Be(100m);
}
```

Then add tests for edge cases (partial refund, refund with restocking fee, refund past the return window, currency-specific rounding). For each branch in the production code, add at least one test that covers it. Use `[Theory]` + `[InlineData]` whenever you'd otherwise duplicate the same test shape.

**Trade-off**: Don't aim for 100% line coverage on day one. Cover the rules that lose money if they break first; cover the rest as you touch them.

**Real project**: When inheriting a refund service, I write three tests in the first hour — full refund, partial refund, refund past window — that establish the baseline behavior. Now any change has a safety net, and I can refactor with confidence.

### 4. Your test passes locally but fails in CI. What's your debugging checklist?

**Why this matters**: This is the single most common real-world testing problem and it reveals whether the candidate writes deterministic tests.

**Answer**: Walk through the usual culprits:

1. **Time** — does the test use `DateTime.UtcNow` directly? Inject `TimeProvider`.
2. **Culture** — does it parse numbers or dates without `CultureInfo.InvariantCulture`? CI agents often run in `en-US`, your laptop in `en-GB` or similar.
3. **Time zone** — same issue with `DateTime.Now`. Use UTC everywhere.
4. **Random data** — `Guid.NewGuid()` in assertions, `Random.Shared` without a seed.
5. **Order dependency** — does it touch a `static` field or a file on disk? xUnit parallelises, CI parallelises differently.
6. **Async timing** — `Task.Delay` "should be enough" — never is. Replace with deterministic signals.
7. **Environment variables / appsettings** — missing config silently falling back to a default.

**Trade-off**: The temptation is to retry the test until it passes. Don't. Mark it `[Fact(Skip = "flaky, see #1234")]` and fix the root cause before merging more code that depends on it.

**Real project**: A test was failing once a week in CI. Root cause: it used `DateTime.UtcNow.Date` in arrange and `DateTime.UtcNow` in act. When the test crossed midnight UTC between the two lines, the dates differed. Fixed by injecting `FakeTimeProvider`.

### 5. How do you test code that throws exceptions, including async ones?

**Answer**: Capture the action and assert with FluentAssertions:

```csharp
// Sync
var act = () => sut.Charge(amount: -10m);
act.Should().Throw<ArgumentException>()
   .WithMessage("*amount must be positive*")
   .And.ParamName.Should().Be("amount");

// Async — note the async lambda and await on the assertion
var act = async () => await sut.ChargeAsync(declinedRequest, CancellationToken.None);
await act.Should().ThrowAsync<PaymentDeclinedException>()
                  .Where(e => e.DeclineCode == "insufficient_funds");
```

**Trade-off**: Don't catch the exception with `try/catch` inside the test — `act.Should().Throw<>()` is clearer and produces better failure messages when the wrong exception type comes out.

**Real project**: A test for `OrderRepository.AddAsync` asserts that calling it with a null order throws `ArgumentNullException` with `ParamName == "order"`. This catches the bug where a refactor accidentally removed the null check.

### 6. Should you ever mock a class you own? Why not just use the real class?

**Why this matters**: New engineers mock everything reflexively. Senior engineers know when a fake or the real class is better.

**Answer**: Prefer the real class for **pure logic and value objects** — there is nothing to gain by mocking `Money` or `OrderStatus`. Use a fake or in-memory implementation for **stateful collaborators you own** (e.g., `InMemoryOrderRepository` instead of `Mock<IOrderRepository>`). Reserve `Moq` / `NSubstitute` for **external boundaries** — payment gateways, email senders, message publishers — where you cannot have a real instance in a unit test.

**Trade-off**: Mocking gives you fine control over return values and lets you verify interactions, but every `mock.Setup(...)` couples the test to the SUT's internal calls. Fakes encode the contract once and stay stable across refactors.

**Real project**: In a billing service, `IInvoiceRepository` has both `EfInvoiceRepository` (production) and `InMemoryInvoiceRepository` (tests). Handlers are tested against the in-memory one. The EF one has its own integration tests against Testcontainers SQL Server.

### 7. A teammate writes a test that runs `Thread.Sleep(1000)` to wait for an async event. What do you say in review?

**Answer**: Reject it. `Thread.Sleep` makes the test slow *and* flaky — too short and the test fails intermittently when CI is busy; too long and the suite takes minutes. The fix is to make the wait deterministic: use a `TaskCompletionSource` signalled by the event handler, or use `await Task.WhenAny(myTask, Task.Delay(timeout))` with a sensible timeout that fails the test if it triggers.

```csharp
[Fact]
public async Task Publish_FiresHandler()
{
    var tcs = new TaskCompletionSource<OrderPlaced>();
    bus.Subscribe<OrderPlaced>(e => { tcs.SetResult(e); return Task.CompletedTask; });

    await bus.PublishAsync(new OrderPlaced(orderId: 42));

    var completed = await Task.WhenAny(tcs.Task, Task.Delay(TimeSpan.FromSeconds(2)));
    completed.Should().Be(tcs.Task, "event was not received within 2 seconds");
    (await tcs.Task).OrderId.Should().Be(42);
}
```

**Real project**: A test suite had 30 `Thread.Sleep(500)` calls — that's 15 seconds of pure wait per run. Replacing them with `TaskCompletionSource` cut suite time from 8 minutes to 90 seconds.

### 8. How do you decide when to stop adding unit tests for a feature?

**Why this matters**: Coverage is not the goal; risk reduction is. Senior engineers know when to ship.

**Answer**: Stop when **every branch in the business rules has at least one test**, and **every defect you've ever seen in this area is reproduced by a test**. Don't aim for 100% line coverage — testing exhaustively against `null`-guard branches in DTOs is theatre. Focus on the rules that, if broken, would cost the business money or trust: pricing, discounts, refunds, access control, payments, expiry dates.

**Trade-off**: Diminishing returns set in fast. The first ten tests catch most regressions; tests 30–50 catch corner cases; tests beyond that mostly catch refactoring noise. Use the time you save on better integration tests, which are usually under-invested.

**Real project**: A discount engine has 12 unit tests covering each tier × amount combination, two edge cases (zero amount, max amount), and three regression tests for past bugs. The team explicitly chose not to write tests for the `ToString()` override — it added zero risk reduction.

## Summary Checklist

- [ ] My tests run in-process with no database, network, or real clock.
- [ ] I name tests `Method_State_ExpectedOutcome` so failures self-document.
- [ ] I follow AAA with blank lines between Arrange, Act, and Assert.
- [ ] I use `[Theory]` + `[InlineData]` to remove duplicated test bodies.
- [ ] I assert on observable outcomes, not on internal method calls.
- [ ] I inject `TimeProvider` instead of using `DateTime.UtcNow`.
- [ ] I prefer in-memory fakes for collaborators I own over mock setups.
- [ ] My async tests are awaited and never use `.Result` or `Thread.Sleep`.
- [ ] My suite runs in seconds; flaky tests get fixed, not retried.
- [ ] My coverage focus is branches on business rules, not lines on glue code.
