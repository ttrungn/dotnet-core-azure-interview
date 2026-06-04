# Clean Architecture

## What It Is

Clean Architecture is a layered design style introduced by Robert C. Martin that organizes a codebase into concentric rings around the business rules. The defining feature is the **Dependency Rule**: source code dependencies point only inward, toward higher-level policies. Frameworks, databases, message brokers, HTTP, and UI live on the outer ring and are plug-in details.

In a .NET solution, Clean Architecture typically materializes as four projects:

1. **Domain** — enterprise business rules: entities, value objects, domain events, domain services. No framework references at all.
2. **Application** — use cases that orchestrate the domain: MediatR handlers, DTOs, validators, abstractions for everything outside (`IOrderRepository`, `IPaymentGateway`, `IEmailSender`). Depends only on Domain.
3. **Infrastructure** — concrete implementations of those abstractions: EF Core, Stripe SDK, Azure Service Bus, SendGrid. Depends on Application (to implement the contracts) and Domain.
4. **Api / Presentation** — ASP.NET Core controllers or minimal APIs, gRPC services, Worker host. Depends on Application; references Infrastructure only in `Program.cs` for DI wiring.

The Dependency Rule means `Application` never references EF Core or `BlobServiceClient`. If pricing logic needs to save data, it depends on `IOrderRepository`; the EF implementation lives in Infrastructure and is injected at startup.

## Why It Exists

Traditional N-tier architectures put business logic into "service" classes that called the database directly via DAL helpers. Over years of growth this produced predictable rot:

- Business logic became inseparable from the framework. Replacing SQL Server with PostgreSQL, or moving an action to a Worker Service, required rewriting business code.
- Tests required a live database, a live message broker, and a real SMTP server. Test suites took 20 minutes and broke nightly.
- Vendor decisions (Azure SDK choices, ORM choices) bled into use cases. Migrating to managed identity meant editing every service.
- Onboarding required learning the framework before understanding the business.

Clean Architecture (and its predecessors Hexagonal / Ports and Adapters by Alistair Cockburn, Onion Architecture by Jeffrey Palermo) exists to invert that dependency direction. Business code becomes the stable center; the volatile parts — frameworks, vendors, transports — become replaceable adapters.

## When To Use It

Use Clean Architecture for:

- Line-of-business APIs and microservices that will live more than a year (order, billing, identity, inventory, shipping, notifications).
- Codebases shared by multiple teams that need clear ownership boundaries.
- Systems with non-trivial business rules — pricing, eligibility, regulatory compliance, multi-region tax.
- Projects where infrastructure is likely to change: a database migration, a move from Service Bus to Event Hubs, a new front-end channel (REST + gRPC + queue triggers).
- Anywhere unit tests need to run in milliseconds without external services.

**Do not use Clean Architecture for:**

- True CRUD admin tools or internal back-office screens — a single ASP.NET Core project with EF Core is faster, simpler, and good enough.
- Prototypes and proof-of-concept code measured in days.
- Pure infrastructure services (a thin proxy in front of Blob Storage) with no business rules to protect.
- Tiny serverless functions where the entire program is 50 lines.

The cost of Clean Architecture is real: 4 projects, more files, more DI wiring, more cognitive overhead. Apply it where the long-term change cost dominates the up-front structural cost.

## Why It Is Important

In a typical .NET shop targeting Azure, Clean Architecture delivers four concrete benefits:

1. **Fast, deterministic tests.** Application handlers are unit-tested against fake repositories, fake payment gateways, and fake clocks. A 200-test suite runs in under 5 seconds and catches behavior regressions before merge.
2. **Replaceable infrastructure.** When the team moves from Azure SQL to Cosmos DB, only the Infrastructure project changes; Application and Domain are unaffected.
3. **Enforced separation of concerns.** Project references make the Dependency Rule a build error, not a code-review heuristic. EF Core literally cannot leak into Domain.
4. **Onboarding and reasoning.** A new engineer can read the Application project as a table of contents of the business: every use case is a handler.

In an Azure-hosted context, Clean Architecture also keeps **Managed Identity, Key Vault, Application Insights**, and **Service Bus** configuration out of business code. You can run the same handlers in App Service, in an Azure Function, in a Worker Service on AKS, or in an integration test — only the host changes.

## How It's Used in C# / .NET

### Project Structure

```
src/
├── MyShop.Domain/                       // No framework refs
│   ├── Orders/
│   │   ├── Order.cs                     // Aggregate root (entity)
│   │   ├── OrderLine.cs                 // Entity
│   │   ├── OrderStatus.cs
│   │   ├── IOrderRepository.cs          // Domain abstraction (optional placement)
│   │   └── Events/OrderPlaced.cs        // Domain event
│   └── SharedKernel/Money.cs            // Value object
│
├── MyShop.Application/                  // Depends on Domain only
│   ├── Orders/
│   │   ├── Commands/PlaceOrder/
│   │   │   ├── PlaceOrderCommand.cs
│   │   │   ├── PlaceOrderHandler.cs     // MediatR IRequestHandler
│   │   │   └── PlaceOrderValidator.cs   // FluentValidation
│   │   └── Queries/GetOrderById/...
│   ├── Common/Interfaces/IPaymentGateway.cs
│   └── DependencyInjection.cs
│
├── MyShop.Infrastructure/               // Depends on Application + Domain
│   ├── Persistence/
│   │   ├── AppDbContext.cs              // EF Core
│   │   ├── Configurations/OrderConfiguration.cs
│   │   └── Repositories/EfOrderRepository.cs
│   ├── Payments/StripePaymentGateway.cs
│   ├── Messaging/ServiceBusOutboxPublisher.cs
│   └── DependencyInjection.cs
│
└── MyShop.Api/                          // Depends on Application; refs Infrastructure in Program.cs
    ├── Controllers/OrdersController.cs
    ├── Program.cs                       // Composition root
    └── appsettings.json
```

### Dependency Rule Enforced by Project References

`MyShop.Domain.csproj` has zero `<ProjectReference>` entries. `MyShop.Application.csproj` references only `MyShop.Domain`. `MyShop.Infrastructure.csproj` references both. `MyShop.Api.csproj` references Application (and Infrastructure for wiring). This makes the Dependency Rule a compiler error.

### Composition Root (`Program.cs`)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplication()         // MediatR, FluentValidation, AutoMapper
    .AddInfrastructure(builder.Configuration); // EF Core, Stripe, Service Bus

builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();
```

### Libraries that fit cleanly per layer

| Concern                  | Library                                  | Layer          |
|--------------------------|------------------------------------------|----------------|
| Use case mediation       | MediatR / Wolverine                      | Application    |
| Validation               | FluentValidation                         | Application    |
| Object mapping           | AutoMapper / Mapster                     | Application    |
| ORM                      | EF Core / Dapper                         | Infrastructure |
| Messaging                | MassTransit / Azure.Messaging.ServiceBus | Infrastructure |
| Caching                  | StackExchange.Redis                      | Infrastructure |
| HTTP clients (Stripe)    | Refit / IHttpClientFactory               | Infrastructure |
| Telemetry                | Application Insights / OpenTelemetry     | Api / Infra    |
| Architecture tests       | NetArchTest                              | Tests          |

### Cross-cutting concerns via MediatR pipeline behaviors

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = validators.Select(v => v.Validate(req)).SelectMany(r => r.Errors).Where(e => e is not null).ToList();
        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}
```

Validation, transactions, logging, and authorization all live in Application as behaviors — they decorate every handler without polluting business code.

## Advantages

- **Testable in isolation** — handlers run against fakes; no DB or network needed.
- **Replaceable infrastructure** — change EF Core for Dapper or Cosmos by editing one project.
- **Build-time enforcement** of the Dependency Rule via project references.
- **Multiple hosts, same business code** — same handlers in API, Functions, Workers.
- **Cleaner onboarding** — the Application project reads as the catalog of use cases.
- **Better fit with DDD, CQRS, and event-driven patterns** — the layers map naturally.

## Disadvantages

- **More projects, more files** — 4 projects for a service that could have been 1.
- **Ceremony for small features** — adding a single endpoint touches DTO, validator, handler, repository interface, EF implementation.
- **Indirection** — "Go to definition" lands on an interface; junior engineers need to learn the DI graph.
- **Risk of anemic domain** — putting all logic in handlers leaves entities as data bags. Apply DDD to fight this.
- **Cost of bad abstractions** — exposing `IQueryable<T>` from a repository leaks ORM concerns straight back into Application.
- **Over-applied to simple CRUD** — produces 100 files for what should be one controller.

## Common Mistakes

### 1. EF Core Bleeding into Application

```csharp
// BUG: Application handler depends on DbContext directly
public sealed class GetOrdersHandler(AppDbContext db) : IRequestHandler<GetOrdersQuery, List<OrderDto>>
{
    public async Task<List<OrderDto>> Handle(GetOrdersQuery q, CancellationToken ct)
        => await db.Orders.Include(o => o.Lines).Where(...).Select(...).ToListAsync(ct);
}
```

`AppDbContext` is in Infrastructure; if Application references it, the Dependency Rule is broken — Application now drags EF Core, SQL provider, and migrations into every consumer.

**Fix**: Depend on `IOrderRepository` (or `IOrderReadModel` for CQRS reads). The EF implementation stays in Infrastructure.

### 2. Returning `IQueryable<T>` From a Repository

```csharp
public interface IOrderRepository
{
    IQueryable<Order> Query(); // leaks EF
}
```

Callers in Application can now write `.Include(...).Where(...).ToListAsync()`. The interface lies — it pretends to abstract persistence but the consumer must know EF semantics.

**Fix**: Return materialized aggregates (`Task<Order?>`) or accept a `Specification<Order>` if you need flexibility.

### 3. Anemic Domain — Logic Drained Into Handlers

```csharp
// BUG: Handler mutates entity directly
public async Task Handle(ConfirmOrderCommand cmd, CancellationToken ct)
{
    var order = await _repo.GetByIdAsync(cmd.OrderId, ct);
    order.Status = OrderStatus.Confirmed;            // bypasses invariants
    order.ConfirmedAt = _clock.GetUtcNow();
    if (order.Lines.Count == 0) throw new Exception();
    await _uow.CommitAsync(ct);
}
```

The handler now owns rules that belong on `Order`. Two handlers will drift apart.

**Fix**: Put the invariant on the entity: `order.Confirm(clock)`. The handler orchestrates; the entity enforces.

### 4. Putting Domain Events in Infrastructure

```csharp
// BUG: domain event class in Infrastructure
namespace MyShop.Infrastructure.Events;
public record OrderPlaced(Guid OrderId);
```

Domain events describe business facts ("an order was placed"); they belong in Domain. Only the *dispatcher* lives in Infrastructure.

### 5. Reusing DTOs as Domain Entities

```csharp
// BUG: One Order class flows from controller to DB
[ApiController]
public class OrdersController { public Task<Order> GetById(Guid id) => _repo.GetByIdAsync(id); }
```

Now the JSON contract is coupled to the database schema and to the domain model. A property rename in the entity changes the API.

**Fix**: Keep a separate `OrderResponse` DTO. Map with AutoMapper/Mapster in Application.

### 6. Circular Dependencies via "Shared" Project

A `MyShop.Shared` project that holds DTOs *and* interfaces *and* entities pulls everything together and defeats the layering.

**Fix**: Resist the urge. Put truly cross-cutting types (`Result<T>`, common exceptions) in `Domain` (or a tiny `SharedKernel`); keep DTOs in Application and entities in Domain.

### 7. `Program.cs` Wiring Infrastructure Concretes Directly

```csharp
builder.Services.AddScoped<StripePaymentGateway>(); // no interface registered
```

Application code is now forced to depend on `StripePaymentGateway`, defeating DIP and Clean Architecture.

**Fix**:

```csharp
builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
```

## Best Practices

- **Enforce the Dependency Rule at build time** with project references; add a NetArchTest test to catch accidental usings.
- **Use MediatR handlers for use cases** — one handler per command/query.
- **Keep abstractions in Application** (`IOrderRepository`, `IPaymentGateway`); implementations in Infrastructure.
- **Use pipeline behaviors** for cross-cutting concerns (validation, transactions, logging, authorization).
- **Map at the boundary.** Controllers map between HTTP DTOs and commands/queries. Infrastructure maps between persistence models and domain entities if they diverge.
- **Put business rules on the entity, not in the handler.** Handlers coordinate; entities enforce.
- **Avoid `IServiceProvider` injection.** It hides dependencies and reintroduces Service Locator.
- **Use `IOptions<T>` per module** (`StripeOptions`, `ServiceBusOptions`) — typed configuration is itself a DIP application.
- **Add architecture tests** that fail the build if Application references EF Core or if Domain references anything.
- **Group registration in extension methods**: `AddApplication()`, `AddInfrastructure(config)` — keep `Program.cs` thin.

## Related Concepts

- **[solid-principles.md](solid-principles.md)** — Clean Architecture is SOLID applied at module scale.
- **[../csharp/dependency-injection.md](../csharp/dependency-injection.md)** — how the Dependency Rule is wired in practice.
- **[domain-driven-design.md](domain-driven-design.md)** — DDD fills the Domain layer with rich aggregates and value objects.
- **[application-services.md](application-services.md)** — what lives in the Application layer.
- **[repositories.md](repositories.md)** — the canonical Application/Infrastructure boundary.
- **[unit-of-work.md](unit-of-work.md)** — commits per use case across one handler.
- **[cqrs.md](cqrs.md)** — splitting read and write paths inside the Application layer.
- **Hexagonal Architecture (Ports & Adapters)** — semantically equivalent; "ports" are interfaces in Application, "adapters" are classes in Infrastructure.
- **Onion Architecture** — Jeffrey Palermo's variant; same Dependency Rule, slightly different layer naming.
- **Vertical Slice Architecture** — an alternative for smaller systems: organize by feature, not by layer.

### Onion vs Hexagonal vs Clean — quick comparison

| Aspect              | Hexagonal (Cockburn)             | Onion (Palermo)                       | Clean (Martin)                                 |
|---------------------|----------------------------------|---------------------------------------|------------------------------------------------|
| Year                | 2005                             | 2008                                  | 2012                                           |
| Core idea           | Application + Ports + Adapters   | Concentric layers; deps point inward  | Concentric rings; explicit Use Cases ring      |
| Domain placement    | Center                           | Center                                | Center                                         |
| Layer names         | Application core, ports          | Domain, Domain Services, Application  | Entities, Use Cases, Adapters, Frameworks      |
| .NET convention     | Often 3 projects                 | 3–4 projects                          | 4 projects (Domain/App/Infra/Api)              |

All three share the **Dependency Rule**. Most .NET teams call any of them "Clean Architecture".

## Real-World Usage

### Enterprise Order Management Platform

A multi-region retailer's order service runs the same Application handlers in:

- An ASP.NET Core API on Azure App Service (customer-facing REST).
- A gRPC service for internal microservices.
- An Azure Function triggered by Service Bus for asynchronous order events.

Only `Program.cs` differs per host. The Application and Domain layers are identical NuGet packages reused across all three. When the team migrated from SQL Server to Azure SQL Hyperscale, only `Infrastructure/Persistence/AppDbContext.cs` changed.

### Microservices

In a microservices estate, each service has its own miniature Clean Architecture solution:

- `Ordering.Api` — REST + Service Bus consumer.
- `Ordering.Application` — `PlaceOrderHandler`, `CancelOrderHandler`, `RefundOrderHandler`.
- `Ordering.Domain` — `Order`, `OrderLine`, `Money`, `OrderStatus`.
- `Ordering.Infrastructure` — EF Core, Stripe, Service Bus.

This isolates blast radius: the Catalog team can rewrite its persistence layer without touching Ordering's contracts.

### Azure-Hosted Workloads

Clean Architecture pairs naturally with Azure:

- **Key Vault** secrets bound via `IOptions<T>` — Application never sees the connection string.
- **Managed Identity** for `BlobServiceClient`, `ServiceBusClient` — configured once in Infrastructure DI, transparent to handlers.
- **Application Insights** sampled in Api layer via `AddApplicationInsightsTelemetry()` — handlers emit `ILogger<T>` log entries that flow there automatically.
- **Health checks** live in Api: `services.AddHealthChecks().AddDbContextCheck<AppDbContext>().AddAzureServiceBusQueue(...)`.

The handlers themselves contain no Azure references and run identically against in-memory fakes for tests.

## Code Example — Before and After

### Before — Single-project N-tier service with leaks

```csharp
// One project, controllers calling DAL helpers, business rules in services using EF directly

public class OrdersController : ControllerBase
{
    private readonly AppDbContext _db;
    public OrdersController(AppDbContext db) => _db = db;

    [HttpPost]
    public async Task<IActionResult> Place(OrderRequest req)
    {
        // Validation, pricing, persistence, Stripe, email - all here
        var stripe = new StripeClient("sk_live_xxx");
        var total = req.Items.Sum(i => i.Price * i.Quantity);
        var charge = stripe.Charge(req.CardToken, total);

        var order = new Order { Id = Guid.NewGuid(), Total = total, Status = "Paid" };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();

        new SmtpClient("smtp.contoso.com").Send(
            new MailMessage("noreply@contoso.com", req.Email) { Subject = "Order placed" });

        return Ok(order);
    }
}
```

Problems: controller knows about Stripe, SMTP, EF, validation. Untestable without network and DB. Replacing Stripe means editing controllers. Replacing the email provider means editing controllers.

### After — Clean Architecture

**Domain (`MyShop.Domain/Orders/Order.cs`)**:

```csharp
public sealed class Order
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Money Total { get; private set; }
    public OrderStatus Status { get; private set; }

    private Order() { } // EF
    public Order(Money total) { Total = total; Status = OrderStatus.PendingPayment; }

    public void MarkPaid(PaymentReference reference)
    {
        if (Status != OrderStatus.PendingPayment)
            throw new InvalidOperationException("Order is not awaiting payment.");
        Status = OrderStatus.Paid;
    }
}
```

**Application (`MyShop.Application/Orders/Commands/PlaceOrder/PlaceOrderHandler.cs`)**:

```csharp
public sealed record PlaceOrderCommand(string Email, string CardToken, IReadOnlyList<OrderItem> Items)
    : IRequest<Guid>;

public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    IPaymentGateway payments,
    IEmailSender email,
    IUnitOfWork uow,
    IPricingService pricing) : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var total = pricing.Calculate(cmd.Items);
        var order = new Order(total);

        var payment = await payments.ChargeAsync(cmd.CardToken, total, ct);
        if (!payment.Succeeded) throw new PaymentFailedException(payment.DeclineReason);
        order.MarkPaid(new PaymentReference(payment.TransactionId));

        await orders.AddAsync(order, ct);
        await uow.CommitAsync(ct);

        await email.SendAsync(cmd.Email, "Order placed", $"Total: {total}", ct);
        return order.Id;
    }
}
```

**Infrastructure (`MyShop.Infrastructure/Payments/StripePaymentGateway.cs`)**:

```csharp
public sealed class StripePaymentGateway(IOptions<StripeOptions> options, HttpClient http) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(string token, Money amount, CancellationToken ct)
    {
        // Stripe SDK call here
        return PaymentResult.Succeeded("ch_123");
    }
}
```

**Api (`MyShop.Api/Controllers/OrdersController.cs`)**:

```csharp
[ApiController, Route("api/orders")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Place(PlaceOrderRequest req, CancellationToken ct)
    {
        var id = await mediator.Send(req.ToCommand(), ct);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
}
```

**Composition root (`Program.cs`)**:

```csharp
builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);
```

The handler is unit-testable with `FakePaymentGateway` and `InMemoryOrderRepository`. Swapping Stripe for Adyen changes one line in `AddInfrastructure`. Moving the same `PlaceOrderHandler` into an Azure Function takes 30 minutes.

## Interview Questions and Answers

### 1. What is the Dependency Rule, and how do you enforce it in a .NET solution?

**Why this matters**: Tests whether the candidate understands the *defining property* of Clean Architecture, not just the project names.

**Answer**: Source code dependencies point only inward — outer rings know about inner rings, never the reverse. In .NET, enforce it with project references: `Domain` references nothing; `Application` references `Domain`; `Infrastructure` references both; `Api` references `Application` and (only in `Program.cs`) `Infrastructure`. Add a NetArchTest that fails the build if `Application` references `Microsoft.EntityFrameworkCore`. Build-time enforcement beats code-review enforcement.

**Trade-off**: 4 projects add cognitive load. For a tiny service, a single project is fine. The structure pays off in services that will live for years.

**Real project**: An architecture test caught a PR adding `using Microsoft.EntityFrameworkCore;` in Application after a refactor — would have bled EF into 6 handlers.

### 2. Why not just put the business logic in the controller and use EF directly?

**Why this matters**: Many real codebases look like that. The answer must explain *when* the trade-off flips.

**Answer**: It's fine for true CRUD or short-lived services. It breaks down once business rules grow: the controller becomes untestable without HTTP + DB + downstream services; replacing the database is a rewrite; the same logic ends up duplicated when you add a Worker Service or a Function. Clean Architecture is an investment that pays back when behavior accumulates and infrastructure churns.

**Trade-off**: For a 10-endpoint internal admin tool, Clean Architecture is overkill. For a payment platform, the cost is recovered in the first replacement of a vendor.

**Real project**: We left a back-office tool as a single ASP.NET Core project for two years; we used Clean Architecture in the customer-facing payment API and replaced Stripe with Adyen in three weeks without touching handlers.

### 3. Where do domain events live, and who dispatches them?

**Why this matters**: Common confusion — events feel like infrastructure but they are business facts.

**Answer**: The event types (`OrderPlaced`, `PaymentCaptured`) live in `Domain`. Entities raise them via a `_domainEvents` collection. Application handlers (or a `SaveChanges` interceptor) collect and dispatch them — usually via MediatR `INotification` after `UnitOfWork.Commit()`. Infrastructure provides the dispatcher (e.g., a Service Bus publisher). This keeps the domain pure but lets you route events to integration channels.

**Trade-off**: Dispatching events inside the same transaction risks deadlocks; dispatching after commit risks lost events if the publisher fails. The Outbox pattern resolves this.

**Real project**: We moved domain event dispatch from in-transaction to a post-commit outbox, eliminating a class of "publisher timeout fails the request" incidents.

### 4. A teammate is returning `IQueryable<Order>` from `IOrderRepository`. What is wrong, and what would you do?

**Why this matters**: A classic Clean Architecture leak that looks "performant" but breaks the abstraction.

**Answer**: It leaks EF Core semantics into Application: callers must know about `Include`, `AsNoTracking`, expression tree limitations, and the cost of materialization. Replacing EF with Dapper or Cosmos becomes a massive refactor. I would replace the method with intent-revealing methods (`GetByIdAsync`, `GetPendingForCustomerAsync`) that return materialized aggregates or DTOs, or accept a `Specification<Order>` if filtering really must be flexible.

**Trade-off**: For read-heavy reporting screens, the Specification pattern can be more work than its benefits warrant. Use a separate `IOrderReadModel` returning DTOs (CQRS read side) instead.

**Real project**: A reporting screen evolved into 18 conditional filters in the repository call site; we extracted a Specification + projection and finally cleaned the test suite.

### 5. How does Clean Architecture differ from Hexagonal and Onion Architecture?

**Why this matters**: All three are routinely conflated.

**Answer**: They are nearly the same idea expressed differently. Hexagonal (Cockburn, 2005) talks about ports (interfaces) and adapters (implementations). Onion (Palermo, 2008) talks about concentric layers with the domain at the center. Clean (Martin, 2012) explicitly names the "Use Cases" ring and labels the outer ring "Frameworks & Drivers". All three enforce the same Dependency Rule. In .NET, "Clean Architecture" has become the umbrella term, often realized via the Jason Taylor template.

**Trade-off**: Spending time arguing terminology is wasted; what matters is that the Dependency Rule is enforced.

**Real project**: We adopted the Jason Taylor template as a starting point and customized the Infrastructure project for Azure-specific clients.

### 6. Where do validators (FluentValidation) live, and why?

**Why this matters**: A subtle choice that affects testability and reuse.

**Answer**: In `Application` next to the command/query they validate. They depend only on the command shape and on Domain types, so they have no infrastructure dependencies and can be unit-tested in milliseconds. They are wired into the MediatR pipeline via a `ValidationBehavior<TRequest, TResponse>` so every handler is automatically validated. Domain-level invariants (e.g., a confirmed order cannot be modified) belong on the entity, not in the validator.

**Trade-off**: Some teams duplicate validation in DTO attributes and FluentValidation. Pick one as the source of truth (usually FluentValidation in Application).

**Real project**: Moving validation into a MediatR pipeline behavior removed 200 lines of guard-clause duplication from handlers.

### 7. You need to run the same logic from a REST API, an Azure Function triggered by Service Bus, and a CLI tool. How do you organize the code?

**Why this matters**: Tests Clean Architecture's payoff — reuse across hosts.

**Answer**: Put the use case in `Application` as a MediatR handler that knows nothing about the host. Add three host projects: `MyShop.Api` (controllers), `MyShop.Functions` (function handlers that call `IMediator.Send`), `MyShop.Cli` (a `Program.Main` that builds the host and dispatches). Each host project has its own `Program.cs` that calls `AddApplication()` and `AddInfrastructure()`. The handlers run identically in all three.

**Trade-off**: Three deployment artifacts add ops surface area. Justify it with a real reuse need.

**Real project**: A migration tool reused the same `ImportInvoiceHandler` as the REST API by hosting it in a CLI; the only new code was the input parsing.

### 8. When would you avoid Clean Architecture entirely?

**Why this matters**: A senior engineer knows when *not* to apply a pattern.

**Answer**: For genuine CRUD admin tools (4–10 endpoints, one table per page), throwaway prototypes, short-lived seasonal microsites, or pure infrastructure proxies with no business rules. The 4-project structure adds cost without recouping it. A single ASP.NET Core project with EF Core and minimal APIs ships faster and stays understandable. The signal for upgrading is "we have more than a handful of branching rules and at least one infrastructure change is on the horizon".

**Trade-off**: Many teams default to Clean Architecture and over-build small services. Match the structure to the lifetime and complexity.

**Real project**: We started an MVP as a single project; when the business rules took over, we extracted a `Domain` + `Application` from the existing code in two days — much faster than starting big and stripping it down.

## Summary Checklist

- [ ] I can state the Dependency Rule and enforce it via project references and architecture tests.
- [ ] I can lay out a 4-project .NET solution (Domain / Application / Infrastructure / Api) and explain what each contains.
- [ ] I can place MediatR handlers, FluentValidation, EF Core, and Azure SDK clients in the correct layer.
- [ ] I can spot common leaks (EF in Application, `IQueryable<T>` from repos, anemic domain) and fix them.
- [ ] I can implement cross-cutting concerns (validation, transactions, logging) via MediatR pipeline behaviors.
- [ ] I can compare Hexagonal, Onion, and Clean and explain why they amount to the same Dependency Rule.
- [ ] I can reuse Application handlers across REST, gRPC, Functions, and Workers.
- [ ] I can name when Clean Architecture is overkill and a single-project service is better.
- [ ] I can wire Azure-specific clients (Key Vault, Service Bus, Blob) in Infrastructure without leaking them into Application.
- [ ] I can refactor an N-tier legacy service into Clean Architecture incrementally, behind feature flags.
