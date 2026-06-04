# Domain Events

## What It Is

A **domain event** is a fact that happened inside the business domain, expressed as a small, immutable record. The name is past tense — `OrderPlaced`, `PaymentCaptured`, `InventoryReserved`, `ShipmentDispatched`. The aggregate that owns the consistency boundary raises the event when its state changes; one or more in-process handlers react to it later — typically right after `SaveChanges` commits.

Domain events are **in-process**. They are raised by aggregates, dispatched inside the same service, and consumed by handlers in the same process. That makes them different from **integration events**, which travel across services via a message broker (Azure Service Bus, RabbitMQ).

```csharp
// Domain event — a past-tense record of something that happened in this service
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId, Money Total, DateTimeOffset OccurredAt);

// Aggregate raises it when state changes
public sealed class Order : AggregateRoot
{
    public static Order Place(Guid customerId, IReadOnlyList<OrderLine> lines)
    {
        var order = new Order(Guid.NewGuid(), customerId, lines);
        order.Raise(new OrderPlaced(order.Id, customerId, order.Total, DateTimeOffset.UtcNow));
        return order;
    }
}
```

Three things distinguish a domain event from "any class with a past-tense name":

1. It is raised by an **aggregate**, not by a service or controller.
2. It expresses something **the business cares about**, not a CRUD operation. `RowUpdated` is not a domain event; `OrderShipped` is.
3. It is **dispatched in-process**, usually in the same transaction or immediately after, by a `MediatR.INotification` dispatcher or an EF Core `SaveChanges` interceptor.

## Why It Exists

Without domain events, every place that needs to react to a business change has to be called explicitly from the place that caused the change. The `PlaceOrderHandler` ends up doing this:

```csharp
public async Task Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Place(cmd.CustomerId, cmd.Lines);
    _orders.Add(order);
    await _uow.SaveChangesAsync(ct);

    // Now call every interested collaborator manually
    await _email.SendOrderConfirmationAsync(order, ct);
    await _loyalty.AwardPointsAsync(cmd.CustomerId, order.Total, ct);
    await _analytics.RecordOrderAsync(order, ct);
    await _serviceBus.PublishAsync(new OrderPlacedIntegrationEvent(order.Id), ct);
}
```

Problems:
- The handler knows about loyalty, analytics, email, and Service Bus. Every new reaction requires editing the handler.
- One slow collaborator (e.g., email) makes the whole request slow.
- A failure in step 3 (analytics) might roll back the order even though it shouldn't.
- Testing `PlaceOrderHandler` requires mocking 4+ unrelated services.

Domain events were introduced (by Eric Evans and Vaughn Vernon in the DDD community) so the aggregate could declare *what happened*, and each interested party could subscribe *separately* without the aggregate or the command handler needing to know about them.

## When To Use It

**Use domain events for:**

- **Side effects inside one service** that should react to a business change — sending an email after an order ships, awarding loyalty points after a purchase, invalidating a cache after a price change.
- **Cross-aggregate consistency within the same service** — when `Order` is placed, `Customer.LoyaltyPoints` should update, but they're separate aggregates with separate transactions ideally (or one transaction if simple).
- **Triggering integration events** to other services — a domain event handler maps the in-process event to an integration event row in the [outbox](outbox-pattern.md).
- **Audit and analytics** — record what happened without coupling the aggregate to the audit infrastructure.
- **Updating read models** in [CQRS](cqrs.md) — a denormalized view subscribes to events.

**Do not use domain events for:**

- **Cross-service communication** — that's what integration events and message brokers are for. Domain events are in-process only.
- **Simple CRUD where nothing else cares about the change** — don't raise `CustomerNameUpdated` if nothing reacts to it. You'll create noise and indirection for no value.
- **Anything that must be transactional with the change but you don't trust the dispatcher** — if the side effect must commit with the aggregate change, do it inline.
- **As a replacement for method calls inside one aggregate** — events are between aggregates and components, not internal control flow.

## Why It Is Important

Domain events buy four properties:

1. **Decoupling.** The aggregate states a fact. Subscribers decide what to do. New subscribers (a new email template, a new analytics counter) are added without changing the aggregate or the command handler.
2. **Test focus.** `Order.Place(...)` is tested by asserting it raised `OrderPlaced`. The email handler is tested separately by asserting it called `IEmailSender`. No mocks of 4+ services in one test.
3. **Transactional integrity.** If domain events are dispatched in the same transaction (via EF Core's `SaveChanges` interceptor) and one handler fails, the entire transaction rolls back — the order is *not* placed, and the email is *not* sent. The system stays consistent.
4. **Outbox bridge.** Domain events are the natural input to the [outbox pattern](outbox-pattern.md). A handler maps each domain event to an outbox row, and a background publisher sends integration events to [Service Bus](../azure/azure-service-bus.md) reliably.

In a CQRS + DDD codebase, domain events are the glue between the write model (aggregates) and the rest of the world (read projections, integrations, side effects).

## How It's Used in C# / .NET

The two standard approaches: **MediatR `INotification`** and an **EF Core `SaveChanges` interceptor**. Most production codebases combine them.

### 1. Define the event and the dispatcher abstraction

```csharp
public interface IDomainEvent : INotification { }

public sealed record OrderPlaced(
    Guid OrderId,
    Guid CustomerId,
    Money Total,
    DateTimeOffset OccurredAt) : IDomainEvent;
```

### 2. Base class for aggregates that can raise events

```csharp
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events.AsReadOnly();

    protected void Raise(IDomainEvent @event) => _events.Add(@event);
    public void ClearDomainEvents() => _events.Clear();
}
```

### 3. Aggregate raises events as part of state changes

```csharp
public sealed class Order : AggregateRoot
{
    public Guid Id { get; }
    public OrderStatus Status { get; private set; }

    private Order(Guid id, Guid customerId, IReadOnlyList<OrderLine> lines)
    {
        Id = id;
        Status = OrderStatus.Placed;
        // ... validate, set fields
    }

    public static Order Place(Guid customerId, IReadOnlyList<OrderLine> lines)
    {
        var order = new Order(Guid.NewGuid(), customerId, lines);
        order.Raise(new OrderPlaced(order.Id, customerId, order.Total, DateTimeOffset.UtcNow));
        return order;
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped) throw new InvalidOperationException("Cannot cancel shipped order");
        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelled(Id, reason, DateTimeOffset.UtcNow));
    }
}
```

### 4. Dispatch with an EF Core `SaveChanges` interceptor

The interceptor inspects tracked entities for unprocessed events and dispatches them through MediatR before/after commit. This ties dispatch to the transaction boundary.

```csharp
public sealed class DispatchDomainEventsInterceptor : SaveChangesInterceptor
{
    private readonly IMediator _mediator;
    public DispatchDomainEventsInterceptor(IMediator mediator) => _mediator = mediator;

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData, int result, CancellationToken ct = default)
    {
        if (eventData.Context is null) return result;

        var aggregates = eventData.Context.ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Count > 0)
            .Select(e => e.Entity)
            .ToArray();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToArray();
        foreach (var a in aggregates) a.ClearDomainEvents();

        foreach (var @event in events)
            await _mediator.Publish(@event, ct);

        return result;
    }
}
```

Register it:

```csharp
builder.Services.AddScoped<DispatchDomainEventsInterceptor>();

builder.Services.AddDbContext<AppDbContext>((sp, options) =>
    options
        .UseSqlServer(builder.Configuration.GetConnectionString("Sql"))
        .AddInterceptors(sp.GetRequiredService<DispatchDomainEventsInterceptor>()));
```

### 5. Handlers subscribe via `INotificationHandler<T>`

```csharp
public sealed class SendOrderConfirmationEmail : INotificationHandler<OrderPlaced>
{
    private readonly IEmailSender _email;
    private readonly IOrderRepository _orders;

    public SendOrderConfirmationEmail(IEmailSender email, IOrderRepository orders)
    {
        _email = email;
        _orders = orders;
    }

    public async Task Handle(OrderPlaced @event, CancellationToken ct)
    {
        var order = await _orders.GetAsync(@event.OrderId, ct);
        await _email.SendAsync(order!.CustomerEmail,
            $"Order {order.Number} placed",
            $"Total: {order.Total}", ct);
    }
}

public sealed class AwardLoyaltyPoints : INotificationHandler<OrderPlaced>
{
    private readonly ILoyaltyService _loyalty;
    public AwardLoyaltyPoints(ILoyaltyService loyalty) => _loyalty = loyalty;

    public Task Handle(OrderPlaced @event, CancellationToken ct) =>
        _loyalty.AwardAsync(@event.CustomerId, @event.Total, ct);
}
```

MediatR's `Publish(...)` invokes all registered handlers. Multiple handlers per event are normal.

### 6. Bridging to integration events via the outbox

```csharp
public sealed class WriteOrderPlacedToOutbox : INotificationHandler<OrderPlaced>
{
    private readonly IOutbox _outbox;
    public WriteOrderPlacedToOutbox(IOutbox outbox) => _outbox = outbox;

    public Task Handle(OrderPlaced @event, CancellationToken ct) =>
        _outbox.EnqueueAsync(new OrderPlacedIntegrationEvent(
            @event.OrderId, @event.CustomerId, @event.Total.Amount, @event.Total.Currency), ct);
}
```

`_outbox.EnqueueAsync` writes a row to an `Outbox` table — same `DbContext`, same transaction. A `BackgroundService` later drains it to [Azure Service Bus](../azure/azure-service-bus.md). See the [outbox pattern](outbox-pattern.md).

### 7. Pre-commit vs post-commit dispatch

| Strategy        | Pros                                          | Cons                                                            |
|-----------------|-----------------------------------------------|-----------------------------------------------------------------|
| Pre-commit (`SavingChangesAsync`)  | Handler failures roll back the aggregate change | Handlers can't observe persisted state (no IDs from identity columns yet) |
| Post-commit (`SavedChangesAsync`)  | Handlers see committed state              | A handler failure can't roll back the aggregate; need outbox/retry for resilience |

For side effects that must be transactional (write to another table), prefer pre-commit. For side effects that talk to external systems (email, queues), use post-commit + outbox.

### 8. Library quick reference

| Need                         | API / Library                                          |
|------------------------------|--------------------------------------------------------|
| In-process event contract    | `MediatR.INotification`                                |
| Multi-handler dispatch       | `IMediator.Publish(notification)`                      |
| Tie dispatch to transaction  | EF Core `SaveChangesInterceptor`                       |
| Cross-service publication    | [Outbox pattern](outbox-pattern.md) + [Service Bus](../azure/azure-service-bus.md) |
| Background publisher         | `IHostedService` / `BackgroundService`                 |
| Idempotent consumers         | Message ID + dedup table; MassTransit also handles this |

## Advantages

- **Decoupling** — aggregates state what happened, subscribers decide what to do.
- **Single Responsibility per handler** — one handler = one reaction, easy to test.
- **Transactional safety** — pre-commit dispatch makes side effects atomic with the aggregate change.
- **Composable** — adding a new handler is a single class + DI registration, no edits to existing code.
- **Bridges naturally to integration events** via the outbox.
- **Audit trail** — events are a natural log of what happened in the system.
- **Test focus** — `Order.Place(...)` asserts it raised `OrderPlaced`; handlers are tested separately.

## Disadvantages

- **Indirection** — "Find references" on `OrderPlaced` doesn't follow the dispatch path. New developers must learn that MediatR routes by type.
- **Hidden control flow** — a single SaveChanges may run 8 handlers; debugging requires knowing the dispatch order and timing (pre/post commit).
- **Risk of cascading events** — handler raises another event raises another, eventually exhausting the stack or creating cycles. Hard to debug.
- **Wrong scope used as integration events** — teams send in-process events directly to Service Bus from inside a command handler and lose them on transaction rollback.
- **Easy to over-event** — every CRUD change becomes an event, polluting the system with `FieldChanged`-style noise.
- **Handler ordering is ambiguous** — MediatR doesn't guarantee order; if order matters, you need explicit orchestration.

## Common Mistakes

### 1. Publishing directly to Service Bus from inside the command handler

```csharp
// BUG: DB transaction commits, then Service Bus send fails — event is lost.
//      Or Service Bus send succeeds, then DB throws — phantom event sent.
public async Task Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Place(cmd.CustomerId, cmd.Lines);
    _orders.Add(order);
    await _uow.SaveChangesAsync(ct);

    await _serviceBus.SendAsync(new OrderPlacedIntegrationEvent(order.Id), ct);
}
```

**Fix**: Raise `OrderPlaced` from the aggregate, dispatch via interceptor, have a handler write to the [outbox](outbox-pattern.md), and let a background publisher push to Service Bus.

### 2. Raising events from a service instead of an aggregate

```csharp
// BUG: the service raises the event, but Order's state and the event are not aligned.
public class OrderService
{
    public async Task PlaceAsync(...)
    {
        var order = new Order(...);
        _orders.Add(order);
        await _mediator.Publish(new OrderPlaced(order.Id, ...)); // event lives outside aggregate
        await _uow.SaveChangesAsync(ct);
    }
}
```

**Fix**: The aggregate raises the event because the aggregate owns the invariant. The dispatcher pulls events off the aggregate after `SaveChanges`.

### 3. Doing work in event handlers that should be transactional with the aggregate

A post-commit handler that updates another aggregate (e.g., `Customer.LoyaltyPoints`) can fail after the order was already placed — leaving the system inconsistent. Either move the update into the same transaction (pre-commit), or accept eventual consistency and add idempotency / retries / outbox.

### 4. Handler calls another aggregate that raises another event that calls another aggregate...

```csharp
// BUG: cascading events make control flow unreadable
OrderPlaced -> AwardLoyaltyPoints handler -> Customer.AddPoints raises LoyaltyTierChanged
            -> SendTierUpgradeEmail handler -> ...
```

**Fix**: Keep one event = one round of handlers. If you need a chain, model it as a saga or orchestration so the flow is explicit. See [saga pattern](saga-pattern.md).

### 5. Treating domain events as integration events

```csharp
// BUG: serializing an internal domain event onto Service Bus
await _serviceBus.SendAsync(new OrderPlaced(...)); // OrderPlaced is an internal type
```

Internal types should not leak across services. Map domain events to dedicated **integration event** types with explicit, versioned schemas before publishing. Internal refactors then don't break consumers.

### 6. Forgetting to clear events after dispatch

If `ClearDomainEvents()` isn't called after dispatch, the same event may be re-published on the next `SaveChanges`. Always clear after dispatch (the interceptor above does this).

### 7. Big payloads on events

```csharp
// BUG: event contains the full order with 50 line items + customer profile
public record OrderPlaced(Order Order); // huge object graph
```

Keep events small — IDs and the minimum context needed. Handlers can re-load the aggregate if they need more.

## Best Practices

- **Past tense, no verbs.** `OrderPlaced`, not `PlaceOrder`. Commands are present-tense verbs; events are past-tense facts.
- **One aggregate raises, multiple handlers consume.** Aggregates don't know about handlers.
- **Dispatch through an EF Core SaveChanges interceptor** so dispatch is tied to the transaction boundary.
- **Use pre-commit dispatch** when handlers must be transactional with the aggregate change.
- **Use post-commit dispatch + outbox** when handlers talk to external systems.
- **Keep event payloads small** — IDs, timestamps, the minimum business context.
- **Make handlers idempotent** — they can re-run if dispatch is retried.
- **Map domain events to integration events** at the boundary; don't let internal types leak to Service Bus.
- **Don't raise events for trivial CRUD.** Reserve them for business-meaningful state changes.
- **Test the aggregate by asserting raised events** — `order.DomainEvents.Should().ContainSingle<OrderPlaced>()`.

## Related Concepts

- **[CQRS](cqrs.md)** — domain events are the natural bridge between write-side commands and read-model updates.
- **[Outbox pattern](outbox-pattern.md)** — the way to reliably turn domain events into integration events.
- **[Aggregates](aggregates.md)** — the only thing allowed to raise domain events.
- **[Event-driven architecture](event-driven-architecture.md)** — covers integration events across services.
- **[Saga pattern](saga-pattern.md)** — orchestrate multi-step workflows when one event leads to another.
- **[Domain-driven design](domain-driven-design.md)** — the umbrella that introduced domain events.
- **[Application services](application-services.md)** — orchestrate the unit of work; the interceptor is wired here.
- **MediatR `INotification`** — the standard in-process publish/subscribe dispatcher in .NET.
- **EF Core `SaveChanges` interceptor** — the hook that ties dispatch to the transaction boundary.

## Real-World Usage

### Order fulfillment service

`OrderPlaced` triggers three in-process handlers:
1. `WriteToOutbox` — enqueues `OrderPlacedIntegrationEvent` for fulfillment, billing, and email services.
2. `UpdateCustomerLifetimeValue` — updates a denormalized read model.
3. `EmitTelemetry` — records a custom metric to Application Insights.

All three run inside the same DB transaction (pre-commit). If any fail, the order isn't placed.

### Payment processing

`PaymentCaptured` triggers `RecordRevenueRecognition` (writes to a journal table), `UpdateOrderStatus` (sets the order to `Paid`), and `WriteToOutbox` (enqueues `PaymentCapturedIntegrationEvent` so downstream services like loyalty and notifications can react asynchronously).

### Subscription billing

`SubscriptionRenewed` triggers `ExtendAccessPeriod`, `SendRenewalConfirmation` (post-commit, via outbox), and `UpdateMrrSnapshot`. Each handler is owned by a different team within the same service.

### Inventory

`StockReserved` triggers `EmitStockLowAlertIfThresholdCrossed` (which only sometimes raises another event, `StockLowAlertRaised`, processed in the next round). Carefully designed to avoid cascading: alerts are aggregated, not raised per reservation.

## Code Example — Before and After

### Before: command handler doing everything

```csharp
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly IEmailSender _email;
    private readonly ILoyaltyService _loyalty;
    private readonly IAnalytics _analytics;
    private readonly ServiceBusSender _serviceBus;

    public PlaceOrderHandler(
        IOrderRepository orders, IUnitOfWork uow,
        IEmailSender email, ILoyaltyService loyalty,
        IAnalytics analytics, ServiceBusSender serviceBus)
    {
        _orders = orders; _uow = uow;
        _email = email; _loyalty = loyalty;
        _analytics = analytics; _serviceBus = serviceBus;
    }

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Place(cmd.CustomerId, cmd.Lines);
        _orders.Add(order);
        await _uow.SaveChangesAsync(ct);

        await _email.SendAsync(order.CustomerEmail, "Order placed", $"Order {order.Id}", ct);
        await _loyalty.AwardAsync(cmd.CustomerId, order.Total, ct);
        await _analytics.RecordAsync(order, ct);
        await _serviceBus.SendMessageAsync(new ServiceBusMessage(
            BinaryData.FromObjectAsJson(new OrderPlacedIntegrationEvent(order.Id))), ct);

        return order.Id;
    }
}
```

Problems:
- 6 dependencies; handler tests must mock all 6.
- A failure after `SaveChanges` (e.g., Service Bus throws) leaves the system inconsistent — order placed, no event published.
- Slow side effects (email) block the HTTP response.

### After: aggregate raises event, handlers subscribe, outbox bridges to Service Bus

```csharp
public sealed class Order : AggregateRoot
{
    public static Order Place(Guid customerId, IReadOnlyList<OrderLine> lines)
    {
        var order = new Order(Guid.NewGuid(), customerId, lines);
        order.Raise(new OrderPlaced(order.Id, customerId, order.Total, DateTimeOffset.UtcNow));
        return order;
    }
}

public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;

    public PlaceOrderHandler(IOrderRepository orders, IUnitOfWork uow)
    {
        _orders = orders;
        _uow = uow;
    }

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Place(cmd.CustomerId, cmd.Lines);
        _orders.Add(order);
        await _uow.SaveChangesAsync(ct); // interceptor dispatches OrderPlaced
        return order.Id;
    }
}

// Each side effect is its own handler — single-responsibility, individually testable
public sealed class SendOrderConfirmationEmail : INotificationHandler<OrderPlaced> { /* ... */ }
public sealed class AwardLoyaltyPoints : INotificationHandler<OrderPlaced> { /* ... */ }
public sealed class RecordAnalytics : INotificationHandler<OrderPlaced> { /* ... */ }

// Bridges in-process event to integration event via outbox
public sealed class WriteOrderPlacedToOutbox : INotificationHandler<OrderPlaced>
{
    private readonly IOutbox _outbox;
    public WriteOrderPlacedToOutbox(IOutbox outbox) => _outbox = outbox;

    public Task Handle(OrderPlaced @event, CancellationToken ct) =>
        _outbox.EnqueueAsync(new OrderPlacedIntegrationEvent(
            @event.OrderId, @event.CustomerId, @event.Total.Amount, @event.Total.Currency), ct);
}
```

The handler drops to 2 dependencies. Side effects are independent classes. Integration events flow through the outbox, so DB commit and message publication are atomic.

## Interview Questions and Answers

### 1. Explain the difference between a domain event and an integration event.

**Why this matters**: Mixing these up causes lost messages or service coupling. Senior engineers need to articulate the boundary.

**Answer**: A domain event is a fact that happened *inside one service*, raised by an aggregate and consumed by in-process handlers via MediatR. An integration event is the *contract that travels across services*, published to a message broker (Service Bus, Event Grid). Domain events are internal types — they can change freely. Integration events are external contracts — they need schema versioning. The bridge is usually a handler that maps a domain event to an outbox row, which a background publisher converts into an integration event.

**Trade-off**: Skipping the integration event type and serializing the domain event directly is shorter today, painful tomorrow — every internal rename breaks consumers.

**Real project**: `OrderPlaced` (domain event, raised by `Order` aggregate) → `WriteOrderPlacedToOutbox` handler → outbox row → background publisher → `OrderPlacedIntegrationEvent` (versioned schema in shared contracts NuGet package) → Service Bus topic → fulfillment, billing, notifications.

### 2. Where do you dispatch domain events — before or after the DB commit?

**Why this matters**: Tests whether the candidate understands transaction semantics.

**Answer**: It depends on what the handler does. Handlers that touch the same DB or other aggregates should run **pre-commit** (inside `SavingChangesAsync`), so a failure rolls everything back. Handlers that talk to external systems (email, queues) should run **post-commit** (inside `SavedChangesAsync`) — but then a failure can't undo the DB change, so use idempotency + outbox + retries. Most real projects combine both: in-process consistency handlers pre-commit, external side effects post-commit via outbox.

**Trade-off**: Pre-commit handlers don't see committed state (e.g., DB-generated IDs are still placeholders for new entities). Post-commit handlers do, but at the cost of eventual consistency.

**Real project**: A payment service dispatches `PaymentCaptured` pre-commit so the order status update and the journal entry commit atomically; the integration event (notifying loyalty and email services) goes through the outbox post-commit.

### 3. How would you implement domain events with EF Core?

**Why this matters**: The interceptor pattern is the standard, mature approach. Candidates who answer "use MediatR.Publish in the handler" are missing the transaction boundary.

**Answer**: Make aggregates inherit `AggregateRoot` with a `_events` list and a `Raise(...)` method. Write a `SaveChangesInterceptor` that pulls events off all tracked aggregates after (or before) commit and publishes them through `IMediator.Publish`. Register the interceptor with `AddInterceptors(...)` on `AddDbContext`. Then handlers register naturally as `INotificationHandler<TEvent>`. This ties dispatch to the transaction so consistency is automatic.

**Trade-off**: The interceptor approach hides dispatch — debugging requires knowing where it lives. Document it in the architecture notes.

**Real project**: A fulfillment service has a single `DispatchDomainEventsInterceptor` registered on the `DbContext`. Adding a new event type is one record class + one handler class — no edits to existing wiring.

### 4. Your team is publishing domain events directly to Azure Service Bus from inside command handlers. What's wrong?

**Why this matters**: This is one of the most common production bugs in event-driven systems.

**Answer**: There's no transactional guarantee between the DB write and the Service Bus send. If the DB commits and the send fails, the event is lost. If the send succeeds and the DB throws, you have a phantom event downstream services already acted on. Fix it with the [outbox pattern](outbox-pattern.md): in the same DB transaction as the aggregate write, persist an outbox row. A background publisher reads outbox rows and sends to Service Bus with retries. Mark rows dispatched after Service Bus acknowledges. Consumers must be idempotent because the publisher may retry sends.

**Trade-off**: The outbox adds latency (publisher polls every few seconds) and operational complexity (one more table, one more background worker). The price of correctness.

**Real project**: A team had a 0.5% rate of "missing orders downstream" — orders placed but never received by fulfillment. Root cause: ServiceBus.SendAsync failed silently after SaveChanges. Switching to outbox dropped the rate to zero and made the bug visible (failed outbox rows show in a dashboard).

### 5. A handler for `OrderPlaced` is slow because it sends an email. How do you fix it?

**Why this matters**: Domain event handlers run synchronously by default — slow ones block the original request.

**Answer**: Don't send the email inside the in-process handler. Instead, the in-process handler writes to the outbox; a background publisher pushes a `SendEmailRequested` integration event; an email worker (Azure Function or a hosted service) sends the email asynchronously. The HTTP response for "place order" returns as soon as the DB commits. Email failures get retried by the worker and end up in a DLQ on permanent failure.

**Trade-off**: The user no longer gets immediate feedback that "email sent." Most users don't care about that signal — they care that the order succeeded. If they do, show "we'll email you shortly" in the UI.

**Real project**: A checkout service had 300ms+ p99 latency dominated by SMTP. Moving email to async via outbox + Azure Function worker dropped checkout p99 to 60ms; email arrives within 1-2 seconds anyway.

### 6. How do you test code that uses domain events?

**Why this matters**: Tests for understanding of the decoupled testing model.

**Answer**: Three layers. (1) Test the aggregate by asserting it raised the expected event: `order.DomainEvents.Should().ContainSingle<OrderPlaced>(e => e.OrderId == order.Id)`. (2) Test each handler in isolation by constructing it with fake dependencies and invoking `Handle(event, ct)` directly — no MediatR needed. (3) Write one or two integration tests that go through MediatR + DB to confirm wiring (interceptor fires, handlers are registered). This avoids the trap of one slow end-to-end test per handler.

**Trade-off**: You give up "did the side effect actually happen for this scenario?" coverage at the unit level. Use a small set of integration tests to catch wiring bugs.

**Real project**: A service with 30 domain events has ~120 fast unit tests (aggregate raises + handler executes) plus 5 integration tests verifying the full pipeline. Suite runs in 15 seconds.

### 7. When would you *not* use domain events?

**Why this matters**: A senior engineer knows when patterns add cost without value.

**Answer**: Skip domain events when:
- Nothing reacts to the change. Don't raise `CustomerEmailUpdated` if no handler exists.
- The reaction is a single, always-required step that belongs in the aggregate itself (e.g., "when stock drops below zero, throw") — that's a validation, not an event.
- The service is a thin CRUD layer with no business logic. Adding aggregates + events to a 10-table admin tool is overengineering.
- The team doesn't have the discipline to keep events small and handlers focused — without that, events become a tangled mess of cascading reactions.

**Trade-off**: Sometimes a developer skips events for an early CRUD service, then has to retrofit them later when the domain grows. That's usually fine — refactoring CRUD to events is mechanical once the rules become real.

**Real project**: A read-only reporting service has no domain events because nothing changes; events would be pure noise.

### 8. How do you prevent runaway cascades of events?

**Why this matters**: Cascading events are real production bugs — stack overflows, infinite loops, untestable flows.

**Answer**: Three guardrails. (1) Architectural rule: handlers may **read** other aggregates but should **not** raise more events. If a chain is required, model it explicitly as a [saga](saga-pattern.md) or workflow. (2) Limit handler depth — most codebases keep it to one round of handlers per `SaveChanges`. (3) Add an integration test that asserts the handler graph for each event terminates. Code review should flag any handler that calls `Raise(...)` or `_mediator.Publish(...)`.

**Trade-off**: Some domains genuinely have chains (`OrderPlaced` → `StockReserved` → `LowStockAlertRaised`). The right model is an orchestrator (saga), not implicit cascades.

**Real project**: An inventory service had a cascade bug where reserving stock raised a low-stock event, which raised a re-order event, which raised a stock-incoming event, which... bug fixed by introducing an `InventoryWorkflow` orchestrator that sequenced the steps explicitly and rejected nested in-process events.

## Summary Checklist

- [ ] I can define a domain event as a past-tense fact raised by an aggregate, dispatched in-process.
- [ ] I can distinguish a domain event from an integration event and explain the bridge between them.
- [ ] I can implement domain event dispatch with an EF Core `SaveChangesInterceptor` and MediatR `INotification`.
- [ ] I can explain pre-commit vs post-commit dispatch and pick the right one per handler.
- [ ] I can recognize the "publish to Service Bus from inside the handler" bug and fix it with the [outbox pattern](outbox-pattern.md).
- [ ] I can keep event payloads small and handlers idempotent.
- [ ] I can prevent cascading events with explicit orchestration via a [saga](saga-pattern.md).
- [ ] I can test aggregates by asserting raised events and handlers in isolation.
- [ ] I can decide when domain events are overkill (no subscribers, simple CRUD).
- [ ] I can clear events after dispatch and avoid leaking internal types across services.
