# Dependency Injection

## What It Is

Dependency Injection (DI) is a design technique where a class **receives** the objects it needs (its *dependencies*) from the outside instead of **creating** them itself. The class declares what it needs through its constructor (or sometimes a method or property), and a separate piece of code — called the **composition root** — decides which concrete implementation to provide at runtime.

In ASP.NET Core, the composition root is normally `Program.cs`. The framework ships with a built-in IoC (Inversion of Control) container — `Microsoft.Extensions.DependencyInjection` — that resolves the dependency graph for you.

Three things are happening when you use DI:

1. **Inversion of Control** — the class no longer controls *how* its collaborators are built; the container does.
2. **Programming to an interface** — the class depends on an abstraction (`IPaymentGateway`) rather than a concrete type (`StripePaymentGateway`).
3. **Lifetime management** — the container decides when objects are created and disposed (`Transient`, `Scoped`, `Singleton`).

```csharp
// Without DI — the class hard-wires its collaborator
public class OrderService
{
    private readonly StripeClient _stripe = new StripeClient("sk_live_xxx"); // tight coupling
}

// With DI — the class declares what it needs
public class OrderService
{
    private readonly IPaymentGateway _payments;
    public OrderService(IPaymentGateway payments) => _payments = payments;
}
```

## Why It Exists

Before DI became mainstream in .NET, backend services typically used the **Service Locator** pattern, static factories, or `new` everywhere. That caused predictable pain:

- You could not swap `SmtpEmailSender` for a fake when writing a unit test.
- Changing `StripePaymentGateway` to `AdyenPaymentGateway` meant editing dozens of files.
- A single class might create its own `HttpClient`, leading to socket exhaustion in production.
- Configuration (connection strings, API keys) was scattered through the codebase.

DI was introduced to solve a deeper problem described by the **Dependency Inversion Principle** (the "D" in SOLID): *high-level modules should not depend on low-level modules; both should depend on abstractions.* Your `CheckoutService` (business logic) should not depend directly on `StripeClient` (infrastructure). It should depend on an `IPaymentGateway` interface, and the infrastructure layer should plug itself in.

DI exists to make this possible without hand-rolling factories and registries everywhere.

## When To Use It

Use DI for almost every non-trivial collaborator in a backend application:

- **External services**: payment gateways, email senders, SMS providers, blob storage, message brokers (Azure Service Bus, RabbitMQ).
- **Persistence**: `DbContext`, repositories, caches (Redis, in-memory).
- **Cross-cutting concerns**: loggers (`ILogger<T>`), telemetry clients, feature flag providers.
- **Configuration objects**: `IOptions<StripeOptions>` injected instead of reading `IConfiguration` everywhere.
- **Time and randomness**: inject `TimeProvider` or `ISystemClock` so tests can control "now".

**Do not use DI for:**

- Pure value objects (`Money`, `Address`, `DateRange`) — these have no behavior worth abstracting.
- Simple DTOs and request/response models.
- Pure utility methods with no state (`StringBuilder`, `Math`).
- Domain entities — they should be created by aggregates and factories, not resolved from the container.

## Why It Is Important

DI is not a fashion — it directly drives four properties that determine whether a service is healthy in production:

1. **Testability**: Without DI, the only way to test `CheckoutService` would be to actually call Stripe. With DI, you inject a `FakePaymentGateway` and drive every branch (declined card, network timeout, fraud flag) in milliseconds.
2. **Replaceability**: When the business says "we're switching from Stripe to Adyen next quarter," you write `AdyenPaymentGateway : IPaymentGateway`, change one line in `Program.cs`, and ship.
3. **Lifetime correctness**: The container guarantees that `DbContext` is disposed at the end of an HTTP request, that `HttpClient` is pooled, and that singletons are thread-safe.
4. **Composition visibility**: All wiring lives in one place. A new engineer can read `Program.cs` and understand what services exist and how they connect.

In a microservices / cloud context (Azure App Service, Azure Functions, AKS), DI is what lets you inject `IConfiguration` backed by Key Vault, `BlobServiceClient` configured for Managed Identity, and `ServiceBusClient` connected to your namespace — all without the business code knowing anything about Azure.

## How It's Used in C# / .NET

.NET ships with a first-class DI container in the `Microsoft.Extensions.DependencyInjection` NuGet package. ASP.NET Core, Worker Services, Azure Functions (isolated worker model), gRPC, and MAUI all build on it. You almost never need a third-party container unless you want advanced features (Autofac for modules, Lamar for diagnostics, Scrutor for decorators/assembly scanning).

### 1. Register services on `IServiceCollection`

Three core methods, one per lifetime:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Transient — new instance every time it is resolved
builder.Services.AddTransient<IPasswordHasher, BCryptPasswordHasher>();

// Scoped — one instance per HTTP request (per scope)
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();

// Singleton — one instance for the entire application lifetime
builder.Services.AddSingleton<IClock, SystemClock>();
```

There are overloads for every shape you actually need in production:

```csharp
// Register by interface + implementation type
services.AddScoped<IPaymentGateway, StripePaymentGateway>();

// Register with a factory delegate (needed when construction requires runtime config)
services.AddSingleton<ServiceBusClient>(sp =>
    new ServiceBusClient(
        sp.GetRequiredService<IConfiguration>()["ServiceBus:Namespace"]!,
        new DefaultAzureCredential()));

// Register an existing instance as a singleton
services.AddSingleton<IClock>(SystemClock.Instance);

// Register the same implementation as multiple interfaces
services.AddSingleton<NotificationHub>();
services.AddSingleton<INotificationPublisher>(sp => sp.GetRequiredService<NotificationHub>());
services.AddSingleton<INotificationSubscriber>(sp => sp.GetRequiredService<NotificationHub>());

// TryAdd — only register if no other registration exists (great for library defaults)
services.TryAddScoped<IEmailSender, AzureCommunicationEmailSender>();

// Register multiple implementations of the same interface
services.AddScoped<IOrderValidator, RequiredFieldsValidator>();
services.AddScoped<IOrderValidator, InventoryValidator>();
services.AddScoped<IOrderValidator, FraudValidator>();
// Then inject IEnumerable<IOrderValidator> and run them in a pipeline
```

### 2. Consume services via constructor injection

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orders;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService orders, ILogger<OrdersController> logger)
    {
        _orders = orders;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest req, CancellationToken ct)
    {
        var id = await _orders.PlaceAsync(req, ct);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
}
```

ASP.NET Core wires the constructor automatically — you never new up a controller yourself.

### 3. Framework-provided registration extension methods

Almost every .NET feature exposes an `AddXxx` extension that registers its services with the correct lifetimes:

```csharp
builder.Services.AddControllers();                                 // MVC
builder.Services.AddEndpointsApiExplorer();                        // Minimal APIs metadata
builder.Services.AddSwaggerGen();                                  // OpenAPI
builder.Services.AddAuthentication().AddJwtBearer();               // Auth
builder.Services.AddAuthorization();
builder.Services.AddDbContext<AppDbContext>(o =>                   // EF Core (Scoped)
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sql")));
builder.Services.AddDbContextFactory<AppDbContext>();              // For background services
builder.Services.AddStackExchangeRedisCache(o => /* ... */);       // Redis
builder.Services.AddHttpClient<IStripeClient, StripeClient>();     // Typed HttpClient
builder.Services.AddMediatR(cfg =>                                 // MediatR
    cfg.RegisterServicesFromAssemblyContaining<Program>());
builder.Services.AddHostedService<OutboxPublisher>();              // BackgroundService
builder.Services.AddApplicationInsightsTelemetry();                // Azure Monitor
```

### 4. The Options Pattern — typed configuration via DI

Instead of injecting `IConfiguration` and reading magic strings, bind config sections to strongly-typed classes:

```csharp
// appsettings.json
// "Stripe": { "ApiKey": "sk_live_xxx", "WebhookSecret": "whsec_xxx" }

public class StripeOptions
{
    public string ApiKey { get; init; } = "";
    public string WebhookSecret { get; init; } = "";
}

// Registration
builder.Services.Configure<StripeOptions>(builder.Configuration.GetSection("Stripe"));

// Consumption
public class StripePaymentGateway(IOptions<StripeOptions> options) : IPaymentGateway
{
    private readonly StripeOptions _opts = options.Value;
}
```

Use `IOptionsSnapshot<T>` for per-request reloads, `IOptionsMonitor<T>` for change notifications in singletons.

### 5. Lifetime helpers and scoping

```csharp
// Create a manual scope (e.g., inside a BackgroundService or Singleton)
public class OutboxPublisher(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            // ... publish pending outbox messages
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

### 6. Keyed services (.NET 8+)

When you need multiple implementations of the same interface and want to pick by key:

```csharp
builder.Services.AddKeyedScoped<IPaymentGateway, StripePaymentGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, AdyenPaymentGateway>("adyen");

public class CheckoutService(
    [FromKeyedServices("stripe")] IPaymentGateway primary,
    [FromKeyedServices("adyen")] IPaymentGateway fallback) { /* ... */ }
```

### 7. Container validation

Catch lifetime bugs and missing registrations at startup, not at the first user request:

```csharp
builder.Host.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes = true;     // Throw if a Scoped service is captured by a Singleton
    o.ValidateOnBuild = true;    // Throw at startup if any registration cannot be resolved
});
```

### 8. Quick reference

| API                                | Lifetime  | Typical use                                  |
|------------------------------------|-----------|----------------------------------------------|
| `AddTransient<TService, TImpl>()`  | Transient | Stateless mappers, validators                |
| `AddScoped<TService, TImpl>()`     | Scoped    | `DbContext`, repositories, request handlers  |
| `AddSingleton<TService, TImpl>()`  | Singleton | Config readers, `HttpClient` factories, caches |
| `AddDbContext<T>()`                | Scoped    | EF Core registration                         |
| `AddDbContextFactory<T>()`         | Singleton | Background services and Blazor              |
| `AddHttpClient<TClient, TImpl>()`  | Transient | Typed `HttpClient` with pooling             |
| `AddHostedService<T>()`            | Singleton | `BackgroundService` workers                  |
| `AddKeyedScoped<T>("key")`         | Scoped    | Strategy selection by key (.NET 8+)         |
| `Configure<TOptions>(section)`     | Singleton | Bind config to typed options                |
| `TryAddScoped<T>()`                | Scoped    | Library defaults (don't overwrite)           |

## Advantages

- **Decoupled code** — business logic depends on contracts, not vendors.
- **Easy unit testing** — fake any collaborator with a one-line mock.
- **Centralized configuration** — one composition root instead of scattered `new`.
- **Lifetime management** — container disposes `IDisposable` services automatically.
- **Pluggable architecture** — supports feature flags, A/B tests, multi-tenant strategies (different `ITaxCalculator` per region).
- **Better adherence to SOLID** — especially Dependency Inversion and Single Responsibility.
- **Framework integration** — ASP.NET Core, Azure Functions, gRPC, MassTransit, MediatR all work natively with `IServiceCollection`.

## Disadvantages

- **Indirection** — clicking "Go to definition" on `IPaymentGateway` takes you to an interface, not the actual code that runs. New developers must learn to use the container to follow execution.
- **Runtime errors instead of compile errors** — a missing registration only fails when the request hits the controller. Mitigate with `ValidateOnBuild = true` and `ValidateScopes = true`.
- **Overuse leads to interface explosion** — creating `IOrderMapper`, `IStringTrimmer`, `IDateFormatter` adds noise without testing value.
- **Lifetime bugs are subtle** — a captured `DbContext` in a singleton can corrupt data and only surfaces under load.
- **Slight startup cost** — building large dependency graphs takes time (usually negligible, but matters in serverless cold starts).
- **Container-specific behavior** — auto-resolution of `IEnumerable<T>`, open generics, decorator support differ between containers (MS DI vs Autofac vs Lamar).

## Common Mistakes

### 1. Captive Dependency (Lifetime Mismatch)

A `Singleton` service that injects a `Scoped` service captures the first instance forever. The scoped service is never garbage-collected and stops reflecting per-request state.

```csharp
// BUG: AppCache is Singleton, AppDbContext is Scoped
services.AddSingleton<AppCache>();
services.AddScoped<AppDbContext>();

public class AppCache
{
    public AppCache(AppDbContext db) { /* db is captured forever */ }
}
```

**Fix**: Inject `IServiceScopeFactory` or `IDbContextFactory<AppDbContext>` and create a scope when needed.

### 2. Service Locator Anti-Pattern

```csharp
public class OrderHandler
{
    private readonly IServiceProvider _sp;
    public OrderHandler(IServiceProvider sp) => _sp = sp;

    public void Handle()
    {
        var repo = _sp.GetRequiredService<IOrderRepository>(); // hidden dependency
    }
}
```

The class's real dependencies are now invisible. Tests have to set up the whole container. Declare dependencies in the constructor instead.

### 3. Constructor Bloat

A constructor with 12 dependencies is a code smell — the class is doing too much. Split it by use case (Command/Query handlers, smaller services).

### 4. Registering Concrete Types Only

```csharp
services.AddScoped<StripePaymentGateway>(); // can't swap, can't mock
```

Register against the **interface** so consumers depend on the abstraction:

```csharp
services.AddScoped<IPaymentGateway, StripePaymentGateway>();
```

### 5. Creating `HttpClient` Manually

Every `new HttpClient()` leaks sockets. Use `IHttpClientFactory`:

```csharp
services.AddHttpClient<IStripeClient, StripeClient>(c =>
{
    c.BaseAddress = new Uri("https://api.stripe.com");
});
```

### 6. Forgetting `ValidateScopes` in Production

By default, scope validation is on only in `Development`. Enable it everywhere to catch captive dependencies before they hit production:

```csharp
builder.Host.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes = true;
    o.ValidateOnBuild = true;
});
```

### 7. Disposing Services Yourself

If you `using var x = ...` on a container-resolved service, you may dispose something the container expected to manage. Let the container handle disposal of services it created.

## Best Practices

- **Prefer constructor injection.** Method and property injection should be rare exceptions.
- **Depend on interfaces only when there's a real reason** — a second implementation, a testing seam, or a boundary (HTTP, DB, queue). Don't create `IOrder` for `Order`.
- **Group registrations into extension methods** for readability:
  ```csharp
  services.AddPaymentModule(configuration);
  services.AddNotificationModule(configuration);
  ```
- **Use `IOptions<T>` for configuration** instead of injecting `IConfiguration` directly.
- **Use `IHttpClientFactory`** for any outbound HTTP call.
- **Inject `TimeProvider`** instead of using `DateTime.UtcNow` directly.
- **Enable `ValidateOnBuild` and `ValidateScopes`** in all environments.
- **Use `AddDbContextFactory`** when a service needs to span multiple scopes or run background work.
- **Keep `Program.cs` thin** by delegating to `IServiceCollection` extension methods per module.
- **Run a DI integration test** that builds the full container and resolves every controller / hosted service.

## Related Concepts

- **Inversion of Control (IoC)** — DI is one specific form of IoC.
- **Dependency Inversion Principle** — the SOLID principle DI implements.
- **Service Locator** — an older, less safe alternative; usually considered an anti-pattern in modern .NET.
- **Composition Root** — the single place where the object graph is wired (`Program.cs`).
- **Factory Pattern** — sometimes needed alongside DI when objects need runtime parameters (`IPaymentGatewayFactory.Create(currency)`).
- **Options Pattern** — `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>` for typed configuration.
- **Decorator Pattern** — wrap services for cross-cutting concerns (caching, retry, logging) without modifying the original class.
- **Mediator Pattern** — MediatR uses DI to resolve handlers by message type.
- **Scoped Services & Unit of Work** — `DbContext` is scoped so all repositories in one request share a transaction boundary.
- **`IHostedService` / `BackgroundService`** — long-running services resolved at startup; must create their own scopes.

## Real-World Usage

### ASP.NET Core Web API (Azure App Service)

A typical e-commerce checkout API:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Infrastructure
builder.Services.AddDbContext<CheckoutDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Sql")));

builder.Services.AddStackExchangeRedisCache(o =>
    o.Configuration = builder.Configuration["Redis:ConnectionString"]);

builder.Services.AddSingleton(sp =>
    new ServiceBusClient(builder.Configuration["ServiceBus:Namespace"],
                        new DefaultAzureCredential()));

builder.Services.AddHttpClient<IPaymentGateway, StripePaymentGateway>();

// Application
builder.Services.AddScoped<ICheckoutService, CheckoutService>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

// Options
builder.Services.Configure<StripeOptions>(builder.Configuration.GetSection("Stripe"));
```

`CheckoutService` only knows about `IPaymentGateway`, `IOrderRepository`, and `IServiceBusSender`. It has no idea Stripe, SQL Server, or Azure Service Bus exist.

### Azure Functions

```csharp
[Function("ProcessOrder")]
public async Task Run([ServiceBusTrigger("orders")] OrderMessage msg)
{
    await _orderProcessor.HandleAsync(msg);
}
```

`_orderProcessor` is injected through the same `IServiceCollection` registered in `Program.cs` — identical mental model to ASP.NET Core.

### Multi-Tenant SaaS

In a multi-tenant system, you resolve an `ITenantContext` per request, then use it to choose tenant-specific implementations (different database, different feature flags) via a `TenantAwareFactory`.

### Testing

```csharp
[Fact]
public async Task Checkout_DeclinedCard_ReturnsPaymentFailed()
{
    var fakeGateway = new FakePaymentGateway(ChargeResult.Declined);
    var service = new CheckoutService(fakeGateway, new InMemoryOrderRepo());

    var result = await service.PlaceOrderAsync(SampleOrder);

    result.Status.Should().Be(OrderStatus.PaymentFailed);
}
```

No HTTP, no database, no Azure — the constructor signature **is** the test contract.

## Code Example — Before and After

### Before: Tight Coupling, Untestable

```csharp
public class InvoiceService
{
    public void SendInvoice(Invoice invoice)
    {
        // Direct dependency on SMTP — cannot test without a mail server
        var smtp = new SmtpClient("smtp.contoso.com", 25);
        var msg = new MailMessage("billing@contoso.com", invoice.CustomerEmail)
        {
            Subject = $"Invoice {invoice.Number}",
            Body = $"Amount due: {invoice.Total:C}"
        };
        smtp.Send(msg);

        // Direct dependency on SQL — must hit a real database
        using var conn = new SqlConnection("Server=...;");
        conn.Open();
        new SqlCommand($"UPDATE Invoices SET SentAt = GETUTCDATE() WHERE Id = {invoice.Id}", conn)
            .ExecuteNonQuery();
    }
}
```

Problems:
- Cannot unit-test without a real SMTP server and SQL Server.
- Cannot swap email providers (SendGrid, Azure Communication Services).
- SQL injection risk from string interpolation.
- Connection string hard-coded.

### After: DI, Testable, Swappable

```csharp
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body, CancellationToken ct);
}

public interface IInvoiceRepository
{
    Task MarkSentAsync(Guid invoiceId, DateTimeOffset sentAt, CancellationToken ct);
}

public sealed class InvoiceService
{
    private readonly IEmailSender _email;
    private readonly IInvoiceRepository _invoices;
    private readonly TimeProvider _clock;
    private readonly ILogger<InvoiceService> _logger;

    public InvoiceService(
        IEmailSender email,
        IInvoiceRepository invoices,
        TimeProvider clock,
        ILogger<InvoiceService> logger)
    {
        _email = email;
        _invoices = invoices;
        _clock = clock;
        _logger = logger;
    }

    public async Task SendInvoiceAsync(Invoice invoice, CancellationToken ct)
    {
        await _email.SendAsync(
            invoice.CustomerEmail,
            $"Invoice {invoice.Number}",
            $"Amount due: {invoice.Total:C}",
            ct);

        await _invoices.MarkSentAsync(invoice.Id, _clock.GetUtcNow(), ct);

        _logger.LogInformation("Invoice {InvoiceId} sent to {Email}", invoice.Id, invoice.CustomerEmail);
    }
}

// Composition root
builder.Services.AddScoped<IEmailSender, AzureCommunicationEmailSender>();
builder.Services.AddScoped<IInvoiceRepository, EfInvoiceRepository>();
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<InvoiceService>();
```

Now the service is:
- **Testable**: inject in-memory fakes, no SMTP or SQL needed.
- **Replaceable**: swap `AzureCommunicationEmailSender` for `SendGridEmailSender` in one line.
- **Deterministic**: tests can freeze time via a `FakeTimeProvider`.
- **Safe**: no SQL strings, no hard-coded connection strings.

## Interview Questions and Answers

### 1. Why is constructor injection preferred over property or method injection?

**Why this matters**: Constructor injection makes dependencies **required** and **immutable**. The compiler enforces that you can't create the object without providing them, which means you can't accidentally use a service in a half-built state.

**Answer**: Property injection leaves dependencies optional (`null` until set), so the class needs null checks everywhere and tests can run with missing collaborators. Method injection is fine for true per-call dependencies (like `CancellationToken`) but pollutes the public API for every other dependency. Constructor injection puts the contract in one place — the constructor signature is also the test setup signature.

**Trade-off**: Constructor injection forces you to think about responsibilities early. If you find yourself writing a constructor with 10 parameters, the class is doing too much — that's the signal, not a reason to switch injection styles.

**Real project**: In a typical ASP.NET Core controller, every dependency comes through the constructor: `ILogger<T>`, `IMediator`, `ICurrentUser`. You'll rarely see anything else in production code.

### 2. A singleton service needs to read from the database. How do you handle this?

**Why this matters**: Tests for *captive dependency* bugs. A junior may inject `DbContext` directly, which silently breaks under load.

**Answer**: Never inject a `Scoped` `DbContext` into a `Singleton`. Instead, inject `IServiceScopeFactory` (or `IDbContextFactory<T>`) and create a fresh scope per operation:

```csharp
public class CachedConfigService // Singleton
{
    private readonly IServiceScopeFactory _scopeFactory;
    public CachedConfigService(IServiceScopeFactory f) => _scopeFactory = f;

    public async Task RefreshAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db, then scope disposes everything
    }
}
```

**Common mistake**: Using `_serviceProvider.GetService<AppDbContext>()` from the root provider — that gives you a context tied to the root scope, which will leak memory and connections.

**Real project**: A `BackgroundService` that polls a feature flag table every 60 seconds. It must create a scope per poll, never reuse the same `DbContext`.

### 3. Explain the difference between `Transient`, `Scoped`, and `Singleton` with a concrete example.

**Answer**:

| Lifetime    | Created                     | Use for                                              |
|-------------|-----------------------------|------------------------------------------------------|
| `Transient` | Every time it is resolved   | Cheap, stateless services (a small mapper, a validator) |
| `Scoped`    | Once per HTTP request       | `DbContext`, repositories, request-scoped caches     |
| `Singleton` | Once for the app's lifetime | `HttpClient` (via factory), config readers, thread-safe caches |

**Why it matters**: Picking the wrong lifetime is one of the most common production bugs. A `Scoped` service injected into a `Singleton` is captured for the app's lifetime and stops reflecting per-request data. A `Singleton` with non-thread-safe state (like a mutable `Dictionary`) will corrupt data under concurrent requests.

**Real project**: `DbContext` must be scoped because EF Core's change tracker holds state per unit-of-work. An `IDateTimeProvider` is naturally singleton because it has no state. A `MapperConfiguration` (AutoMapper) is singleton; the `IMapper` is scoped.

### 4. The team wants to add Redis caching to `IProductRepository` without changing existing code. How would you do it with DI?

**Why this matters**: Tests the candidate's knowledge of the **Decorator pattern** and how DI containers support it.

**Answer**: Use a decorator. Create `CachedProductRepository` that wraps `IProductRepository`, then register both:

```csharp
public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly IDistributedCache _cache;

    public CachedProductRepository(IProductRepository inner, IDistributedCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Product?> GetAsync(Guid id, CancellationToken ct)
    {
        var key = $"product:{id}";
        var cached = await _cache.GetStringAsync(key, ct);
        if (cached is not null) return JsonSerializer.Deserialize<Product>(cached);

        var product = await _inner.GetAsync(id, ct);
        if (product is not null)
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(product), ct);
        return product;
    }
}

// Registration
services.AddScoped<EfProductRepository>();
services.AddScoped<IProductRepository>(sp =>
    new CachedProductRepository(
        sp.GetRequiredService<EfProductRepository>(),
        sp.GetRequiredService<IDistributedCache>()));
```

**Trade-off**: Manual decorator wiring gets verbose. Libraries like **Scrutor** add `services.Decorate<IProductRepository, CachedProductRepository>()` for cleaner syntax.

**Real project**: Adding caching, retry policies (Polly), or telemetry to an existing repository without touching its consumers.

### 5. Your team's `Program.cs` has 400 lines of `AddScoped` calls. How do you clean it up?

**Answer**: Group registrations into extension methods, one per module or concern. The composition root should read like a table of contents:

```csharp
// Program.cs
builder.Services
    .AddPersistence(builder.Configuration)
    .AddPaymentModule(builder.Configuration)
    .AddNotificationModule(builder.Configuration)
    .AddBackgroundJobs();

// Infrastructure/PersistenceServiceCollectionExtensions.cs
public static IServiceCollection AddPersistence(this IServiceCollection s, IConfiguration c)
{
    s.AddDbContext<AppDbContext>(o => o.UseSqlServer(c.GetConnectionString("Sql")));
    s.AddScoped<IOrderRepository, EfOrderRepository>();
    s.AddScoped<IProductRepository, EfProductRepository>();
    return s;
}
```

**Why**: It makes the dependency graph navigable, supports per-module unit tests, and lets you remove a feature module by deleting one extension call instead of hunting through `Program.cs`.

### 6. How do you test that all services are wired correctly without spinning up the full web host?

**Answer**: Write a DI smoke test:

```csharp
[Fact]
public void Container_Resolves_AllControllers()
{
    using var app = new WebApplicationFactory<Program>();
    using var scope = app.Services.CreateScope();
    var controllers = typeof(Program).Assembly.GetTypes()
        .Where(t => typeof(ControllerBase).IsAssignableFrom(t) && !t.IsAbstract);

    foreach (var t in controllers)
        scope.ServiceProvider.GetRequiredService(t); // throws if any dep is missing
}
```

Combine with `options.ValidateOnBuild = true` to catch missing registrations at startup. This single test has caught real production outages caused by forgetting an `AddScoped` after a refactor.

### 7. When would you *not* use DI?

**Why this matters**: A mature engineer knows when a pattern adds more cost than value.

**Answer**: Skip DI for:
- **Domain entities and value objects** — they should be created by aggregates and factories. Resolving an `Order` from the container leaks persistence concerns into the domain.
- **Simple console utilities** with one or two collaborators where direct instantiation is clearer.
- **Static helpers** that have no state and no external dependencies (`Guid.NewGuid()`, `JsonSerializer.Serialize`).
- **Truly cross-cutting infrastructure** that .NET already provides factories for (e.g., `Activity.Current` for tracing — don't wrap it in an interface).

**Trade-off**: Over-abstracting to "make it DI-friendly" produces dozens of single-implementation interfaces that obscure the real design. Add an interface when you have a real reason: a second implementation, a test seam, or a layer boundary.

### 8. A constructor has 12 dependencies. What does this tell you and how do you fix it?

**Answer**: It signals **Single Responsibility Principle** violation — the class is orchestrating too many concerns. Common fixes:

1. **Split by use case**: Replace `OrderService` with `PlaceOrderHandler`, `CancelOrderHandler`, `RefundOrderHandler` — each handler injects only what it uses.
2. **Aggregate related dependencies** into a domain service that encapsulates a workflow.
3. **Use a facade**: If three services are always used together (e.g., `IBlobClient`, `IThumbnailGenerator`, `IVirusScanner`), wrap them in an `IFileProcessingPipeline`.
4. **Move infrastructure concerns** (logging, telemetry, validation) into decorators or pipeline behaviors (MediatR pipeline, ASP.NET filters).

**Real project**: Refactoring a 12-dependency `CheckoutService` into MediatR command handlers reduced each handler to 3-4 dependencies and made the test suite three times faster.

## Summary Checklist

- [ ] I can explain DI as a way to invert control of object construction.
- [ ] I can describe constructor, property, and method injection and explain why constructor injection is preferred.
- [ ] I can compare `Transient`, `Scoped`, and `Singleton` and give a concrete example of each.
- [ ] I can recognize and fix a captive dependency (Scoped injected into Singleton).
- [ ] I can register and resolve services through `IServiceCollection` and extension methods.
- [ ] I can explain why Service Locator and `new HttpClient()` are anti-patterns.
- [ ] I can apply DI to make a legacy class testable using interfaces and fakes.
- [ ] I can use the Decorator pattern (manually or via Scrutor) to add caching/logging/retry transparently.
- [ ] I can enable `ValidateOnBuild` and `ValidateScopes` and write a smoke test that resolves all controllers.
- [ ] I can recognize when *not* to use DI (domain entities, value objects, static helpers).
