# Integration Testing

## What It Is

An **integration test** verifies that two or more real components work correctly together: your ASP.NET Core HTTP pipeline talking to your handlers, your handlers talking to a real EF Core `DbContext` talking to a real SQL Server, your message consumer talking to a real Service Bus or RabbitMQ broker. Unlike unit tests, integration tests *intentionally* cross at least one process or technology boundary so that mistakes in mapping, configuration, SQL translation, and serialization actually surface.

In .NET, the standard integration testing stack is:

- **`Microsoft.AspNetCore.Mvc.Testing`** with `WebApplicationFactory<TEntryPoint>` to spin up your real `Program.cs` in-memory.
- **Testcontainers for .NET** to launch ephemeral SQL Server, PostgreSQL, Redis, RabbitMQ, MongoDB, or Azure Service Bus emulator instances inside Docker.
- **xUnit** as the runner with `IClassFixture<T>` / `ICollectionFixture<T>` to share expensive setup.
- **FluentAssertions** for readable assertions on `HttpResponseMessage` and JSON.

```csharp
public class OrdersApiTests : IClassFixture<OrdersApiFactory>
{
    private readonly HttpClient _client;

    public OrdersApiTests(OrdersApiFactory factory) => _client = factory.CreateClient();

    [Fact]
    public async Task Post_ValidOrder_ReturnsCreatedAndPersists()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { customerId = Guid.NewGuid(), productId = "P-1", quantity = 2 });

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

An integration test sits one level above a unit test on the **test pyramid** — slower, broader, fewer in number, and exercising the parts that unit tests cannot reach. See [unit-testing.md](unit-testing.md) for the layer below.

## Why It Exists

Unit tests can prove that `DiscountCalculator.Calculate(Gold, 100m) == 10m`. They cannot prove that:

- The `[HttpPost("api/orders")]` route is actually wired up.
- The JSON body deserializes into `CreateOrderRequest` correctly.
- The `[Authorize]` policy rejects requests without a valid JWT.
- The EF Core LINQ query translates into SQL that SQL Server can execute.
- The migration applied successfully and the `Orders` table has the right columns.
- A transient SQL deadlock is correctly retried by the resilience policy.
- The outbound message lands in the Service Bus queue with the right session id.

These bugs only show up when real components are wired together. Historically, teams discovered them in **staging** — manually clicking around — or worse, in **production**, after a deploy. Integration tests pull that discovery into the developer's machine and the CI pipeline, where fixes cost minutes instead of incidents.

They also exist because the gap between "all unit tests green" and "the service actually works" is the most common source of expensive bugs in modern microservices.

## When To Use It

**Use integration tests for**:

- **HTTP endpoints end-to-end** — routing, model binding, validation, authentication, authorization, content negotiation, error handling.
- **EF Core queries and writes** — `Where` / `Include` / `Select` translation, projection shapes, concurrency tokens, migrations.
- **Database constraints** — unique indexes, foreign keys, check constraints, computed columns.
- **Cache integration** — Redis pipelines, TTL behavior, eviction.
- **Messaging** — publishing to Service Bus / RabbitMQ / Kafka and receiving on a handler.
- **Background services** — `IHostedService` and `BackgroundService` lifecycle.
- **Configuration binding** — that `appsettings.json` plus environment variables produce the expected `IOptions<T>`.
- **Contract tests** — verifying the JSON your API produces matches what consumers expect.

**Do not use integration tests for**:

- **Business rule combinatorics** — a 200-case discount matrix belongs in a `[Theory]` unit test, not 200 HTTP round-trips.
- **Third-party SaaS** in CI — don't hit Stripe's real API on every PR. Use sandbox endpoints sparingly or `WireMock.Net`.
- **Pure functions and value-object math** — `Money.Add` does not need a database to test.
- **Trivial controllers** that just delegate to a handler — the handler's unit test plus *one* smoke integration test is enough.

## Why It Is Important

In a microservices / cloud-native architecture, the parts of the system most likely to break are precisely the **seams** — the places where your code meets HTTP, SQL, Redis, Service Bus, and Azure SDKs. Unit tests cover the inside of each box; integration tests cover the connections between them. Both layers are needed.

Without integration tests:

- A `[Required]` attribute on a DTO property gets removed in a refactor and validation silently breaks. The unit test still passes because it tested the handler, not the binding.
- An EF Core `Include` chain that runs perfectly against the `InMemory` provider throws `InvalidOperationException` against SQL Server because of an unsupported projection.
- A new column in the migration is forgotten in the entity mapping and the production deploy crashes on first write.
- An `[Authorize(Roles = "Admin")]` is missing on a sensitive endpoint and the team only discovers it during a pen test.

Integration tests are the **second line of defense after unit tests** and the **first line that proves the system, not just the code, works**. They are typically 5–20% of the test suite by count but catch 50%+ of the regressions that would otherwise reach production.

## How It's Used in C# / .NET

### `WebApplicationFactory<TEntryPoint>` — the in-memory test server

Add a `Program` class entry point to your API (in .NET 6+ minimal APIs, expose the implicit `Program` via `public partial class Program {}`), then:

```csharp
public class OrdersApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");

        builder.ConfigureTestServices(services =>
        {
            // Replace the real payment gateway with a fake
            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton<IPaymentGateway, FakePaymentGateway>();

            // Replace the real time provider with a controllable one
            services.RemoveAll<TimeProvider>();
            services.AddSingleton<TimeProvider>(new FakeTimeProvider(
                DateTimeOffset.Parse("2026-06-04T10:00:00Z")));
        });
    }
}

public class OrdersApiTests : IClassFixture<OrdersApiFactory>
{
    private readonly OrdersApiFactory _factory;
    private readonly HttpClient _client;

    public OrdersApiTests(OrdersApiFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }
}
```

`WebApplicationFactory` boots your real `Program.cs` — middleware, DI, configuration, authentication — and gives you an `HttpClient` that talks to it in-process (no sockets, no network). This is the canonical way to integration-test ASP.NET Core APIs.

### Overriding services for tests with `ConfigureTestServices`

`ConfigureTestServices` runs *after* `Program.cs` has registered everything, so you can `RemoveAll<T>()` and re-register fakes for genuine externals (payment, email, SMS) while keeping the real database, real validation, real authentication.

```csharp
builder.ConfigureTestServices(services =>
{
    services.RemoveAll<IEmailSender>();
    services.AddSingleton<IEmailSender, CapturingEmailSender>();
});
```

### Testcontainers — real databases on demand

`Microsoft.EntityFrameworkCore.InMemory` is a trap for integration tests — it does not enforce constraints, does not translate LINQ to SQL, and does not behave like SQL Server. Use **Testcontainers** to spin up a real database in Docker for the duration of the test run:

```csharp
public class SqlServerFixture : IAsyncLifetime
{
    public MsSqlContainer Container { get; } = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public string ConnectionString => Container.GetConnectionString();

    public async Task InitializeAsync() => await Container.StartAsync();
    public async Task DisposeAsync() => await Container.DisposeAsync();
}

[CollectionDefinition("Sql")]
public class SqlCollection : ICollectionFixture<SqlServerFixture> { }

[Collection("Sql")]
public class OrderRepositoryTests
{
    private readonly OrderDbContext _db;

    public OrderRepositoryTests(SqlServerFixture sql)
    {
        var options = new DbContextOptionsBuilder<OrderDbContext>()
            .UseSqlServer(sql.ConnectionString)
            .Options;
        _db = new OrderDbContext(options);
        _db.Database.EnsureCreated();
    }

    [Fact]
    public async Task Add_DuplicateOrderNumber_ThrowsDbUpdateException()
    {
        _db.Orders.Add(new Order("ORD-1"));
        await _db.SaveChangesAsync();

        _db.Orders.Add(new Order("ORD-1"));
        var act = async () => await _db.SaveChangesAsync();
        await act.Should().ThrowAsync<DbUpdateException>(); // unique index fired
    }
}
```

Testcontainer modules also exist for **PostgreSQL** (`Testcontainers.PostgreSql`), **RabbitMQ** (`Testcontainers.RabbitMq`), **Redis** (`Testcontainers.Redis`), **MongoDB**, **Kafka**, and the **Azure Service Bus emulator** (preview). One pattern, one Docker dependency, real databases per test class.

### Hooking Testcontainers into `WebApplicationFactory`

Combine the two so your HTTP integration tests run against a real SQL Server:

```csharp
public class OrdersApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();

    public Task InitializeAsync() => _sql.StartAsync();
    public new Task DisposeAsync() => _sql.DisposeAsync().AsTask();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureAppConfiguration((_, config) =>
        {
            config.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ConnectionStrings:Sql"] = _sql.GetConnectionString()
            });
        });
        builder.ConfigureTestServices(services =>
        {
            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton<IPaymentGateway, FakePaymentGateway>();
        });
    }
}
```

The first test in the class takes ~5 seconds to boot SQL Server; subsequent tests reuse the container.

### Seeding the database

Two patterns, both fine in production codebases:

```csharp
// 1. Per-test seed via a helper
private async Task SeedAsync(Action<OrderDbContext> setup)
{
    using var scope = _factory.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
    setup(db);
    await db.SaveChangesAsync();
}

[Fact]
public async Task Get_ExistingOrder_ReturnsIt()
{
    var id = Guid.NewGuid();
    await SeedAsync(db => db.Orders.Add(new Order(id, "ORD-1")));

    var response = await _client.GetAsync($"/api/orders/{id}");
    response.StatusCode.Should().Be(HttpStatusCode.OK);
}

// 2. Reset between tests with Respawn (https://github.com/jbogard/Respawn)
private static readonly Respawner _respawn =
    Respawner.CreateAsync(connectionString, new RespawnerOptions { ... }).Result;

public async Task ResetAsync() => await _respawn.ResetAsync(connectionString);
```

`Respawn` is the cleanest way to truncate non-schema tables between tests without dropping and recreating the database.

### Test-only authentication

Real JWT signing slows tests down. Wire a test authentication handler that always succeeds:

```csharp
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public TestAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger, UrlEncoder encoder, ISystemClock clock)
        : base(options, logger, encoder, clock) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[] { new Claim(ClaimTypes.Name, "test"), new Claim(ClaimTypes.Role, "Admin") };
        var identity = new ClaimsIdentity(claims, "Test");
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), "Test");
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

// In OrdersApiFactory.ConfigureTestServices:
services.AddAuthentication("Test")
    .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
```

### Asserting on `HttpResponseMessage`

```csharp
var response = await _client.PostAsJsonAsync("/api/orders", request);

response.StatusCode.Should().Be(HttpStatusCode.Created);
response.Headers.Location!.ToString().Should().StartWith("/api/orders/");

var body = await response.Content.ReadFromJsonAsync<OrderResponse>();
body!.Status.Should().Be("Confirmed");
body.LineItems.Should().HaveCount(2);
```

### Running

```bash
dotnet test --filter "Category=Integration" --logger "console;verbosity=detailed"
```

Integration tests are typically tagged with `[Trait("Category", "Integration")]` so PR pipelines can run unit tests on every push and integration tests only on the merge queue (or on demand).

## Advantages

- **Catches real bugs** in routing, model binding, EF Core SQL translation, authentication, and migrations.
- **End-to-end confidence** — proves the system *as deployed*, not just isolated classes.
- **Documents the API contract** — the test file shows the exact shape of requests and responses.
- **Catches infrastructure misconfigurations** — bad connection strings, missing migrations, wrong startup order.
- **Reproduces production-like failures locally** — Testcontainers gives you real SQL Server, real Redis, no shared environment.
- **Works with existing code** — no need to refactor for testability the way unit tests sometimes demand.

## Disadvantages

- **Slow** — seconds per test instead of milliseconds. A 500-integration-test suite can take 5+ minutes.
- **Requires Docker** in CI — adds infrastructure complexity to the build agent.
- **Parallelism is harder** — shared database state forces careful test isolation.
- **Higher maintenance** — schema changes ripple into seed data and assertions.
- **Easier to write flaky tests** — race conditions, ordering, port collisions.
- **Cannot cover every combination** — too slow to exhaustively test business rule matrices; use unit tests for those.
- **Heavier setup** — `WebApplicationFactory`, Testcontainers, fixtures, test auth handlers, seed helpers.

## Common Mistakes

### 1. Using `Microsoft.EntityFrameworkCore.InMemory` as a "database"

```csharp
// Bad — InMemory does not enforce relational constraints, doesn't translate to SQL,
// and silently differs from SQL Server in dozens of ways.
services.AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("test"));
```

The test passes locally. Production crashes because `LEFT JOIN ... GROUP BY` does not project the way EF translated it for SQL Server.

**Fix**: Use **Testcontainers** for SQL Server (or SQLite in-memory as a compromise for cheaper tests, knowing SQLite is more permissive than SQL Server). Reserve `InMemory` for the *handler's* unit test if you absolutely must use it — never for the *repository's* integration test.

### 2. Sharing one database across all tests with no reset

```csharp
[Fact]
public async Task Create_NewOrder_AddsRow()
{
    await _client.PostAsJsonAsync("/api/orders", new { /*...*/ });

    var count = await _db.Orders.CountAsync();
    count.Should().Be(1); // works first run, fails on the second
}
```

**Fix**: Reset between tests with `Respawn`, run each test in a transaction that rolls back, or use a fresh container per class. Pick one and stick with it.

```csharp
public async Task DisposeAsync() => await _respawner.ResetAsync(_factory.ConnectionString);
```

### 3. Hitting real third-party APIs in CI

```csharp
[Fact]
public async Task Pay_ChargesStripe()
{
    var stripe = new StripeClient(realApiKey); // CI hits production-ish Stripe
    await stripe.ChargeAsync(...);
}
```

Slow, flaky when Stripe has an outage, costs money, leaks data into someone's dashboard.

**Fix**: Use Stripe's test mode locally for *one* contract test; for everything else, replace `IPaymentGateway` with `FakePaymentGateway` via `ConfigureTestServices`. For HTTP-level contract verification, use `WireMock.Net`:

```csharp
var server = WireMockServer.Start();
server.Given(Request.Create().WithPath("/charges").UsingPost())
      .RespondWith(Response.Create().WithStatusCode(200).WithBody(@"{""status"":""succeeded""}"));
```

### 4. Long-running test class with hundreds of tests sharing state

```csharp
[Collection("Shared")]
public class HugeIntegrationTests
{
    // 300 methods, all sharing one DB, all order-dependent, all flaky
}
```

**Fix**: Split into focused classes (`OrdersApiTests`, `RefundsApiTests`, `CustomersApiTests`). Each class gets its own fixture and runs in parallel with other classes.

### 5. Asserting on response body string instead of deserializing

```csharp
var body = await response.Content.ReadAsStringAsync();
body.Should().Contain("\"status\":\"Confirmed\""); // breaks on any formatting change
```

**Fix**: Deserialize to a typed DTO and assert on properties:

```csharp
var body = await response.Content.ReadFromJsonAsync<OrderResponse>();
body!.Status.Should().Be("Confirmed");
```

### 6. Forgetting to dispose Testcontainers and leaking Docker resources

```csharp
public class BadFixture
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();
    public BadFixture() => _sql.StartAsync().Wait();
    // No Dispose — every test run leaves a stopped container behind
}
```

**Fix**: Implement `IAsyncLifetime` (xUnit) so the container is started in `InitializeAsync` and disposed in `DisposeAsync`.

### 7. Running integration tests on every save

A 6-minute suite that runs on every keystroke kills productivity. Split CI:

- **PR pipeline**: unit tests + smoke integration tests (~30s).
- **Merge queue**: full integration suite (~5min).
- **Nightly**: long-running flows + chaos tests.

### 8. Asserting on log output

```csharp
mockLogger.Verify(l => l.LogInformation("Order created..."), Times.Once);
```

Brittle, irrelevant. Assert on the **state** after the call (database row exists, response status correct) — that's what matters to the caller.

## Best Practices

- **Use `WebApplicationFactory<Program>`** for HTTP integration tests; never spin up Kestrel on a real port unless you genuinely need it.
- **Use Testcontainers for real databases.** Never `UseInMemoryDatabase` for repository-level tests.
- **Reset state between tests** with Respawn or per-class containers.
- **Tag with `[Trait("Category", "Integration")]`** so unit and integration suites can run separately.
- **Replace expensive externals via `ConfigureTestServices`** — payment, email, SMS, third-party APIs.
- **Keep the real database, real validation, real auth** — that's the whole point.
- **Use a test authentication handler** to avoid real JWT signing in every test.
- **Seed via helpers, not via SQL scripts** — keep test setup in code, version-controlled.
- **Assert on response status, headers (`Location`), and deserialized DTOs** — not on raw JSON strings.
- **Parallelise per class** with xUnit collections; serialise within a class that shares state.
- **Run smoke integration tests on every PR**; run the full suite on merge or nightly.
- **Pin Docker image versions** — `mcr.microsoft.com/mssql/server:2022-latest` can change without notice; pin to a digest if you need true determinism.

## Related Concepts

- **[unit-testing.md](unit-testing.md)** — the layer below; fast, in-process, no externals.
- **[mocking.md](mocking.md)** — for unit tests; integration tests prefer real implementations.
- **[testable-code-design.md](testable-code-design.md)** — DI seams that allow `ConfigureTestServices` to work.
- **[../aspnet-core/controllers-and-minimal-apis.md](../aspnet-core/controllers-and-minimal-apis.md)** — what you're integration-testing on the HTTP side.
- **[../data-access/entity-framework-core.md](../data-access/entity-framework-core.md)** — what you're integration-testing on the persistence side.
- **[../devops/ci-cd-pipelines.md](../devops/ci-cd-pipelines.md)** — where integration tests run as gates.
- **Contract testing** — Pact, OpenAPI snapshot tests; complementary to integration tests for shared APIs.
- **Component testing** — testing one service with all its inner dependencies real but its outer dependencies stubbed (a middle ground between unit and end-to-end).
- **End-to-end tests** — full deployed environment, real Stripe sandbox, real Service Bus. Slow and flaky; keep to a tiny smoke suite.

## Real-World Usage

A typical .NET microservice (e.g., `Orders.Api` running on Azure App Service backed by Azure SQL) has the following test project structure:

```
tests/
    Orders.UnitTests/              ~500 tests, 5 seconds total
        Application/               handler tests with fakes
        Domain/                    entity and value-object tests
    Orders.IntegrationTests/        ~60 tests, 3 minutes total
        Api/                       WebApplicationFactory + Testcontainers SQL
            OrdersEndpointTests.cs  POST /api/orders happy + error paths
            RefundsEndpointTests.cs POST /api/refunds with auth checks
        Persistence/               Repository tests against Testcontainers SQL
            OrderRepositoryTests.cs unique constraints, concurrency tokens
        Messaging/                 Service Bus emulator tests
            OutboxPublisherTests.cs message published with correct session id
```

CI pipeline:

1. **Build + unit tests** — runs on every push, gates the PR (target < 2 min).
2. **Integration tests** — runs on merge queue, gates the merge to main (target < 8 min).
3. **Smoke tests against deployed staging** — after the deploy, hits real endpoints with a small canary suite.

Coverage is collected from both layers and combined in SonarCloud. The team explicitly does *not* set a single global coverage target — they require ≥90% on `Application/` and `Domain/` (unit tests) and ≥70% on `Api/` controllers (integration tests).

## Code Example — Before and After

### Before — slow, fragile, hits production-like external systems

```csharp
public class OrdersTests
{
    private static readonly HttpClient _http = new() { BaseAddress = new("https://staging.api.contoso.com") };

    [Fact]
    public async Task CreateOrder_Works()
    {
        var token = await GetJwtFromAzureAd();   // real network call
        _http.DefaultRequestHeaders.Authorization = new("Bearer", token);

        var response = await _http.PostAsJsonAsync("/api/orders",
            new { customerId = Guid.NewGuid(), productId = "P-1", quantity = 2 });

        response.IsSuccessStatusCode.Should().BeTrue();
        // Now manually clean up via another HTTP call
        var id = (await response.Content.ReadFromJsonAsync<OrderResponse>())!.Id;
        await _http.DeleteAsync($"/api/orders/{id}");
    }
}
```

Slow (5+ seconds per test), depends on staging being up, leaks data into the staging database, breaks when someone redeploys staging mid-test.

### After — in-process, real DB via Testcontainers, deterministic

```csharp
public class OrdersApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public async Task InitializeAsync()
    {
        await _sql.StartAsync();
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync() => await _sql.DisposeAsync();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureAppConfiguration((_, c) =>
            c.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ConnectionStrings:Sql"] = _sql.GetConnectionString()
            }));
        builder.ConfigureTestServices(s =>
        {
            s.RemoveAll<IPaymentGateway>();
            s.AddSingleton<IPaymentGateway, FakePaymentGateway>();
            s.AddAuthentication("Test")
             .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
        });
    }
}

[Trait("Category", "Integration")]
public class OrdersApiTests : IClassFixture<OrdersApiFactory>, IAsyncLifetime
{
    private readonly OrdersApiFactory _factory;
    private readonly HttpClient _client;

    public OrdersApiTests(OrdersApiFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public Task InitializeAsync() => Task.CompletedTask;

    public async Task DisposeAsync()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
        await db.Database.ExecuteSqlRawAsync("DELETE FROM Orders");
    }

    [Fact]
    public async Task Post_ValidOrder_ReturnsCreatedAndPersists()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { customerId = Guid.NewGuid(), productId = "P-1", quantity = 2 });

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var id = response.Headers.Location!.Segments.Last();

        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
        var saved = await db.Orders.FindAsync(Guid.Parse(id));
        saved.Should().NotBeNull();
        saved!.Quantity.Should().Be(2);
    }

    [Fact]
    public async Task Post_InvalidPayload_ReturnsValidationProblem()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { customerId = Guid.Empty, productId = "", quantity = 0 });

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problem!.Errors.Should().ContainKey("ProductId");
    }
}
```

In-process, no staging dependency, real SQL Server semantics, cleans up after itself, runs in ~1.5 seconds after the container warms up.

## Interview Questions and Answers

### 1. What's the difference between a unit test and an integration test in your projects?

**Why this matters**: This single answer reveals whether the candidate understands the testing pyramid.

**Answer**: A unit test runs entirely in-process — no database, no HTTP, no real clock — and exercises one class in isolation. An integration test deliberately crosses at least one real boundary: HTTP via `WebApplicationFactory`, SQL via Testcontainers, or a message broker. Unit tests run in milliseconds and cover business rules and value-object behavior. Integration tests run in seconds and cover routing, model binding, EF Core SQL translation, migrations, authentication, and serialization.

**Trade-off**: Integration tests are the only way to catch a whole class of bugs (bad LINQ, missing migration, wrong route attribute), but they are 100× slower than unit tests. A healthy suite is mostly unit tests at the base, fewer integration tests in the middle, and a tiny end-to-end smoke suite at the top.

**Real project**: In a billing service, 500 unit tests run in 5 seconds and cover the discount engine and command handlers. 60 integration tests run in 3 minutes and cover every HTTP endpoint and EF Core query. Five end-to-end tests run nightly against staging.

### 2. Why is `Microsoft.EntityFrameworkCore.InMemory` a bad choice for integration tests?

**Answer**: It silently differs from a real database in dozens of ways. It does not enforce unique indexes, foreign keys, or check constraints. It does not translate LINQ to SQL — so queries that fail at runtime against SQL Server pass against InMemory. It doesn't handle case sensitivity, collation, or concurrency tokens the same way. A test suite that's green against InMemory tells you nothing about production behavior.

**Trade-off**: It's faster than spinning up a real database, so it's tempting for handler tests where the database is incidental. Even then, prefer SQLite in-memory mode (a closer real-relational engine) or a Testcontainers SQL Server shared across the class.

**Real project**: A team migrated from InMemory to Testcontainers SQL Server and immediately found three bugs that had been hiding for months: a missing `[ForeignKey]` causing duplicate rows, a `Select` projection that didn't translate, and a concurrency token that was never validated.

### 3. How do you keep integration tests isolated when they all hit the same database?

**Why this matters**: Test isolation is the #1 cause of flaky integration suites.

**Answer**: Pick one of three strategies and stick with it:

1. **Reset between tests** with [Respawn](https://github.com/jbogard/Respawn) — fast, only truncates tables, preserves schema.
2. **Run each test in a transaction** that rolls back in teardown — works for code paths that don't open their own transactions.
3. **Use a fresh container per test class** — strong isolation but slower.

```csharp
public async Task DisposeAsync() => await _respawner.ResetAsync(_connectionString);
```

**Trade-off**: Per-test database reset is slow; per-class reset requires careful in-class ordering. The transaction approach silently breaks when production code opens its own transaction. Respawn is the most common production choice.

**Real project**: A 200-test suite went from 12 minutes (drop & recreate DB per test) to 90 seconds (Respawn between tests) with zero change in coverage.

### 4. How would you integration-test a `POST /api/orders` endpoint that publishes to Azure Service Bus on success?

**Answer**: Use `WebApplicationFactory` with `ConfigureTestServices` to replace `ServiceBusSender` with a capturing fake. Hit the endpoint via `_client.PostAsJsonAsync`, assert the HTTP response, then assert that the fake captured a message with the right body and metadata.

```csharp
public class CapturingBusSender : IBusSender
{
    public List<object> Messages { get; } = new();
    public Task SendAsync(object msg, CancellationToken ct) { Messages.Add(msg); return Task.CompletedTask; }
}

builder.ConfigureTestServices(s =>
{
    s.RemoveAll<IBusSender>();
    s.AddSingleton<IBusSender, CapturingBusSender>();
});

// In the test
var response = await _client.PostAsJsonAsync("/api/orders", request);
var sender = (CapturingBusSender)_factory.Services.GetRequiredService<IBusSender>();
sender.Messages.OfType<OrderPlaced>().Should().ContainSingle(m => m.CustomerId == request.CustomerId);
```

**Trade-off**: This doesn't prove the message survives a real Service Bus round-trip. For that, use Testcontainers with the Azure Service Bus emulator in a separate, smaller "infrastructure contract" test set.

**Real project**: A team caught a serialization bug where `decimal` was being silently truncated to `int` in messages — the fake sender captured the object before serialization, hiding the bug. They added one Testcontainers Service Bus emulator test per message type to verify the full round-trip.

### 5. Your integration test passes locally and fails in CI. What do you check first?

**Answer**: The same checklist as for flaky unit tests, plus integration-specific concerns:

1. **Docker not available** or wrong version on the agent.
2. **Port conflicts** — Testcontainers picks random ports, but local tests can leak fixed-port containers.
3. **Timezone/culture** — the CI agent is `UTC en-US`; your machine is not.
4. **Slow container startup** — increase the health-check timeout; SQL Server takes ~5 seconds.
5. **Shared state across test classes** — does your fixture use `IClassFixture` or `ICollectionFixture`? Wrong scope can leak state.
6. **Migrations not applied** — `EnsureCreated` vs `MigrateAsync` behave differently.
7. **Configuration overrides not applied** — `ConfigureTestServices` runs after `Program.cs`; double-check service order.

**Real project**: A test passed locally and failed in CI because Docker Desktop on the dev laptop pulled the SQL Server image quickly, but the CI agent had a small image cache and the test timed out waiting. Increased `WithStartupCallback` timeout from 60s to 120s — green.

### 6. How do you test authorization and authentication in integration tests without dealing with real JWTs?

**Answer**: Register a test-only authentication scheme via `ConfigureTestServices` that always succeeds with a `ClaimsPrincipal` you control:

```csharp
services.AddAuthentication(defaultScheme: "Test")
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
```

For per-test claims, expose a helper:

```csharp
public HttpClient CreateClientAs(params Claim[] claims)
{
    return _factory.WithWebHostBuilder(b => b.ConfigureTestServices(s =>
    {
        s.RemoveAll<IAuthenticationSchemeProvider>();
        s.AddAuthentication("Test")
         .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", o => { });
        s.AddSingleton(new TestAuthClaims(claims));
    })).CreateClient();
}

[Fact]
public async Task AdminEndpoint_RequiresAdminRole()
{
    var client = CreateClientAs(new Claim(ClaimTypes.Role, "User"));
    var response = await client.GetAsync("/api/admin/users");
    response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}
```

**Trade-off**: You're not testing the *real* token issuance. Keep one integration test that hits a real Identity Provider in test mode to verify the token validation parameters are correct.

**Real project**: This pattern caught a real bug where an `[Authorize(Policy = "AdminOnly")]` was using the wrong claim type (`role` vs `roles`) — easily caught with two integration tests, never visible to unit tests.

### 7. When is an end-to-end test worth writing, and when is it overkill?

**Why this matters**: E2E tests are seductive but often net-negative on velocity.

**Answer**: An E2E test (real deployment, real third-party services, browser or HTTP client) is worth writing when:

- The flow crosses **multiple services** that integration tests cannot cover in-process.
- A regression in this flow would be **business-critical** (checkout, login, refund).
- No cheaper test (unit / integration / contract) covers it.

It is overkill when:

- The same coverage is achievable with a Testcontainers integration test.
- The test is just walking through screens for screenshots — automate that with simpler tools.
- The flow changes often and the test will be in maintenance hell.

**Trade-off**: A handful of E2E tests (5–20) gated to nightly runs is sustainable. A hundred E2E tests on every PR is not — they will be flaky, slow, and disabled within weeks.

**Real project**: A team had 30 E2E tests on every PR, ~40% flaky. They moved 25 to nightly and rewrote 5 as Testcontainers integration tests. PR feedback dropped from 25 minutes to 4 minutes, with the same defect-catch rate.

### 8. How do you parallelise integration tests safely?

**Answer**: xUnit parallelises **across test classes** by default (not within a class). For integration tests:

- Use `IClassFixture<TFactory>` so each class gets its own `WebApplicationFactory` (and therefore its own database container).
- Use `[Collection("name")]` to **serialise** classes that share expensive infrastructure (e.g., one shared SQL Server container across five classes).
- Disable parallelism within a class by default — sharing state across tests in the same class needs careful ordering.
- Set `[assembly: CollectionBehavior(MaxParallelThreads = 4)]` to cap concurrent container starts on agents with limited CPU/RAM.

**Trade-off**: Full parallelism is fastest but burns CI memory (each SQL Server container takes ~500 MB). Tune `MaxParallelThreads` based on agent specs.

**Real project**: A 60-test integration suite ran in 8 minutes serially. With class-level parallelism capped at 4 and one Testcontainer per class, it dropped to 2 minutes — the bottleneck became container startup, not test execution.

## Summary Checklist

- [ ] I use `WebApplicationFactory<Program>` to integration-test ASP.NET Core endpoints in-process.
- [ ] I use Testcontainers for real SQL Server / Postgres / Redis / RabbitMQ — never `UseInMemoryDatabase`.
- [ ] I replace true externals (payment, email, third-party APIs) via `ConfigureTestServices`.
- [ ] I keep real authentication semantics with a test auth handler, not by disabling `[Authorize]`.
- [ ] I reset database state between tests with Respawn, transactions, or fresh fixtures.
- [ ] I tag integration tests with `[Trait("Category", "Integration")]` to separate them in CI.
- [ ] I assert on deserialized DTOs, status codes, and headers — not on raw JSON strings.
- [ ] I parallelise across classes, serialise within a class, and cap container concurrency.
- [ ] My integration suite runs in minutes, not tens of minutes — split nightly if it grows.
- [ ] I have one end-to-end smoke test per critical flow, no more.
