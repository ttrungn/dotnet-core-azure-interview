# Saga Pattern

## What It Is

A **saga** is a sequence of local transactions across multiple services that together complete one business process. Each service commits its own change in its own database, and the saga either proceeds to the next step or runs **compensating actions** to semantically undo previous steps when something later fails. It is the standard way to coordinate long-running, multi-service business workflows (checkout, payment, fulfillment, refund) without using a distributed transaction.

There are two flavors: **orchestration** (a central coordinator commands each step) and **choreography** (services react to each other's events with no central coordinator).

```csharp
// Without a saga — one method tries to atomically touch 4 services. Doesn't work in microservices.
public async Task<Guid> CheckoutAsync(CheckoutRequest req)
{
    using var tx = new TransactionScope();                     // distributed tx; not viable across HTTP/queues
    var orderId = await _orderSvc.CreateAsync(req);
    await _inventorySvc.ReserveAsync(orderId, req.Items);
    await _paymentSvc.CaptureAsync(orderId, req.AmountCents);
    await _shippingSvc.CreateShipmentAsync(orderId);
    tx.Complete();
    return orderId;
}

// With a saga — each step is a local transaction; failures trigger compensation.
public class CheckoutSaga : MassTransitStateMachine<CheckoutState>
{
    public CheckoutSaga()
    {
        InstanceState(s => s.CurrentState);

        During(Initial,
            When(CheckoutStarted).Then(c => c.Saga.OrderId = c.Message.OrderId)
                                 .PublishAsync(c => c.Init<ReserveInventory>(new { c.Message.OrderId, c.Message.Items }))
                                 .TransitionTo(ReservingInventory));

        During(ReservingInventory,
            When(InventoryReserved).PublishAsync(c => c.Init<CapturePayment>(new { c.Saga.OrderId, c.Saga.AmountCents }))
                                    .TransitionTo(CapturingPayment),
            When(InventoryReservationFailed).TransitionTo(Failed));

        During(CapturingPayment,
            When(PaymentCaptured).PublishAsync(c => c.Init<CreateShipment>(new { c.Saga.OrderId }))
                                  .TransitionTo(CreatingShipment),
            When(PaymentFailed).PublishAsync(c => c.Init<ReleaseInventory>(new { c.Saga.OrderId }))    // compensate
                                .TransitionTo(Compensating));
        // ... etc
    }
}
```

## Why It Exists

In a microservices architecture, every service owns its database. A checkout flow touches the Order, Inventory, Payment, Shipping, and Notification services — each with its own SQL or Cosmos instance, each deployed independently. You cannot wrap all five in a single ACID transaction because:

- **2PC (Two-Phase Commit)** would require a coordinator that participants block on. Azure SQL, Cosmos, Service Bus, and most managed services do not support XA. Even when supported, 2PC blocks participants during commit and is operationally fragile.
- **Cross-service network calls inside a transaction** mean any participant's slowdown freezes the whole thing.
- **Long-running workflows** (a saga that waits for a customer to confirm shipping address, or for an external provider's webhook) can take hours or days. A transaction held that long is impossible.

The saga pattern emerged from the 1987 paper by Garcia-Molina and Salem to solve exactly this: model the workflow as **a sequence of local transactions, each with a defined compensating transaction**, accept eventual consistency, and recover failure through semantic reversal rather than rollback.

Concretely, the production incidents that drove saga adoption:

- A checkout reserved inventory and captured payment but failed to create a shipment. Inventory was held forever. Customer was charged. Manual cleanup at 2 AM.
- A bank transfer debited the source account but the credit failed. Funds disappeared from the system.
- A booking system held seats during a 30-minute payment window with a distributed lock; one node crash and the entire booking workflow stalled.

## When To Use It

**Use a saga when:**

- A business workflow spans **multiple services with separate databases**.
- The workflow has **multiple steps that each commit to their own store** (order, payment, inventory, shipping).
- You need **compensation logic** when a later step fails (refund, release stock, cancel shipment).
- The workflow may run for **seconds to days** and cannot hold a transaction open.
- You're already using a **message broker** (Service Bus, RabbitMQ, Kafka) for inter-service communication.

**Do not use a saga when:**

- All steps live in **one database** — use a single transaction or `IUnitOfWork`. See [data-access/transactions.md](../data-access/transactions.md).
- The workflow is **read-only** — there's nothing to compensate.
- A step is **physically irreversible** and the business cannot accept partial completion (e.g., sending a physical item before payment; usually the workflow must be redesigned).
- You are building a **modular monolith** where a local transaction is sufficient — don't introduce sagas because microservices sound exciting.
- The team lacks the operational maturity to run brokers, monitor saga state, and handle stuck sagas — start simpler.

## Why It Is Important

Sagas are how mature event-driven systems stay correct when things break. Five production properties depend on them:

1. **Bounded inconsistency.** The system is *temporarily* inconsistent during a saga but converges to a defined final state (completed, failed, compensated).
2. **No distributed locks.** Each step is short and local; no participant holds a row lock waiting for another service.
3. **Restart safety.** Saga state is persisted; a crashed orchestrator resumes from the last persisted state on restart.
4. **Auditability.** The saga's state transitions and emitted events form a complete audit trail of what happened to a customer's order.
5. **Operational repair.** Support engineers can see which step failed, replay events, or manually advance a stuck saga — instead of writing SQL fixes against five databases.

In cloud terms: sagas + outbox + idempotent consumers + Service Bus are the canonical recipe for trustworthy distributed business workflows on Azure.

## Orchestration vs Choreography

| Aspect | Orchestration | Choreography |
|---|---|---|
| Coordinator | Central orchestrator (state machine) | None; services react to events |
| Visibility | One place to see the workflow | Scattered across services |
| Coupling | Orchestrator knows all participants | Services only know event contracts |
| Failure handling | Centralized compensation logic | Each service decides how to react |
| Complexity to add steps | Edit one state machine | Coordinate event contract changes across services |
| Best for | Complex workflows with branches | Simple linear flows |
| Debugging | Easy (one log stream) | Hard (correlate across services) |
| Risk | Orchestrator becomes a god-service | Implicit workflow no one understands |

Most teams pick **orchestration for non-trivial workflows** (checkout, refund, KYC) and **choreography for simple linear flows** (`OrderPlaced → BillingNotified → InventoryDecremented`).

## How It's Used in C# / .NET

The mainstream .NET stack for sagas is **MassTransit** with its built-in state machine (originally Automatonymous, now integrated). NServiceBus is the enterprise alternative. For lightweight cases, you can hand-roll with EF Core + Service Bus.

### 1. Define the saga state

```csharp
public class CheckoutState : SagaStateMachineInstance
{
    public Guid    CorrelationId   { get; set; }    // saga instance id
    public string  CurrentState    { get; set; } = null!;
    public Guid    OrderId         { get; set; }
    public Guid    CustomerId      { get; set; }
    public long    AmountCents     { get; set; }
    public IReadOnlyList<OrderItem> Items { get; set; } = [];
    public string? PaymentTransactionId { get; set; }
    public string? ShipmentId    { get; set; }
    public string? FailureReason { get; set; }
    public DateTime StartedAtUtc  { get; set; }
}
```

### 2. Define the state machine

```csharp
public class CheckoutStateMachine : MassTransitStateMachine<CheckoutState>
{
    public State ReservingInventory { get; private set; } = null!;
    public State CapturingPayment   { get; private set; } = null!;
    public State CreatingShipment   { get; private set; } = null!;
    public State Completed          { get; private set; } = null!;
    public State Compensating       { get; private set; } = null!;
    public State Failed             { get; private set; } = null!;

    public Event<CheckoutRequested>           CheckoutRequested           { get; private set; } = null!;
    public Event<InventoryReserved>           InventoryReserved           { get; private set; } = null!;
    public Event<InventoryReservationFailed>  InventoryReservationFailed  { get; private set; } = null!;
    public Event<PaymentCaptured>             PaymentCaptured             { get; private set; } = null!;
    public Event<PaymentFailed>               PaymentFailed               { get; private set; } = null!;
    public Event<ShipmentCreated>             ShipmentCreated             { get; private set; } = null!;
    public Event<ShipmentCreationFailed>      ShipmentCreationFailed      { get; private set; } = null!;
    public Event<InventoryReleased>           InventoryReleased           { get; private set; } = null!;
    public Event<PaymentRefunded>             PaymentRefunded             { get; private set; } = null!;

    public CheckoutStateMachine()
    {
        InstanceState(s => s.CurrentState);

        // Correlate every event by OrderId.
        Event(() => CheckoutRequested,         x => x.CorrelateById(m => m.Message.OrderId).SelectId(m => m.Message.OrderId));
        Event(() => InventoryReserved,         x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => InventoryReservationFailed,x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentCaptured,           x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailed,             x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => ShipmentCreated,           x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => ShipmentCreationFailed,    x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => InventoryReleased,         x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentRefunded,           x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(CheckoutRequested)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId      = ctx.Message.OrderId;
                    ctx.Saga.CustomerId   = ctx.Message.CustomerId;
                    ctx.Saga.AmountCents  = ctx.Message.AmountCents;
                    ctx.Saga.Items        = ctx.Message.Items;
                    ctx.Saga.StartedAtUtc = DateTime.UtcNow;
                })
                .PublishAsync(ctx => ctx.Init<ReserveInventory>(new
                {
                    ctx.Saga.OrderId, ctx.Saga.Items, IdempotencyKey = ctx.Saga.CorrelationId
                }))
                .TransitionTo(ReservingInventory));

        During(ReservingInventory,
            When(InventoryReserved)
                .PublishAsync(ctx => ctx.Init<CapturePayment>(new
                {
                    ctx.Saga.OrderId, ctx.Saga.AmountCents,
                    IdempotencyKey = ctx.Saga.CorrelationId
                }))
                .TransitionTo(CapturingPayment),

            When(InventoryReservationFailed)
                .Then(ctx => ctx.Saga.FailureReason = "Inventory unavailable")
                .TransitionTo(Failed));

        During(CapturingPayment,
            When(PaymentCaptured)
                .Then(ctx => ctx.Saga.PaymentTransactionId = ctx.Message.TransactionId)
                .PublishAsync(ctx => ctx.Init<CreateShipment>(new { ctx.Saga.OrderId, ctx.Saga.CustomerId }))
                .TransitionTo(CreatingShipment),

            When(PaymentFailed)
                .Then(ctx => ctx.Saga.FailureReason = ctx.Message.Reason)
                .PublishAsync(ctx => ctx.Init<ReleaseInventory>(new { ctx.Saga.OrderId }))   // compensate
                .TransitionTo(Compensating));

        During(CreatingShipment,
            When(ShipmentCreated)
                .Then(ctx => ctx.Saga.ShipmentId = ctx.Message.ShipmentId)
                .PublishAsync(ctx => ctx.Init<NotifyCustomer>(new { ctx.Saga.OrderId, ctx.Saga.CustomerId }))
                .TransitionTo(Completed),

            When(ShipmentCreationFailed)
                .Then(ctx => ctx.Saga.FailureReason = "Shipment creation failed")
                .PublishAsync(ctx => ctx.Init<RefundPayment>(new                              // compensate payment
                {
                    ctx.Saga.OrderId, ctx.Saga.PaymentTransactionId,
                    IdempotencyKey = ctx.Saga.CorrelationId
                }))
                .TransitionTo(Compensating));

        During(Compensating,
            When(InventoryReleased).TransitionTo(Failed),
            When(PaymentRefunded).PublishAsync(ctx => ctx.Init<ReleaseInventory>(new { ctx.Saga.OrderId }))
                                  .TransitionTo(Failed));

        SetCompletedWhenFinalized();   // remove from saga repository when terminal
    }
}
```

### 3. Wire MassTransit and persist saga state to SQL

```csharp
// NuGet: MassTransit, MassTransit.Azure.ServiceBus.Core,
//        MassTransit.EntityFrameworkCore, MassTransit.SagaStateMachine
builder.Services.AddDbContext<SagaDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Saga")));

builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<CheckoutStateMachine, CheckoutState>()
     .EntityFrameworkRepository(r =>
     {
         r.ExistingDbContext<SagaDbContext>();
         r.UsePostgres();   // or .UseSqlServer();
         r.ConcurrencyMode = ConcurrencyMode.Pessimistic;   // lock row per saga
     });

    x.AddEntityFrameworkOutbox<SagaDbContext>(o =>
    {
        o.UseSqlServer();
        o.UseBusOutbox();   // outgoing messages atomic with saga state
    });

    x.UsingAzureServiceBus((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["ServiceBus:ConnectionString"]);
        cfg.ConfigureEndpoints(ctx);
    });
});
```

### 4. Participant service — idempotent payment handler

```csharp
public class CapturePaymentConsumer(IPaymentGateway gateway, PaymentDb db)
    : IConsumer<CapturePayment>
{
    public async Task Consume(ConsumeContext<CapturePayment> ctx)
    {
        // Idempotency: same IdempotencyKey returns same result, never double-charge.
        var existing = await db.Captures.FirstOrDefaultAsync(c => c.IdempotencyKey == ctx.Message.IdempotencyKey);
        if (existing is not null)
        {
            await ctx.Publish(new PaymentCaptured(ctx.Message.OrderId, existing.TransactionId));
            return;
        }

        try
        {
            var result = await gateway.ChargeAsync(ctx.Message.OrderId, ctx.Message.AmountCents,
                                                   ctx.Message.IdempotencyKey.ToString(), ctx.CancellationToken);
            db.Captures.Add(new Capture(ctx.Message.IdempotencyKey, result.TransactionId));
            await db.SaveChangesAsync(ctx.CancellationToken);
            await ctx.Publish(new PaymentCaptured(ctx.Message.OrderId, result.TransactionId));
        }
        catch (PaymentDeclinedException ex)
        {
            await ctx.Publish(new PaymentFailed(ctx.Message.OrderId, ex.Reason));
        }
    }
}
```

### 5. Hand-rolled saga without MassTransit

For very simple cases or non-MassTransit shops, persist saga state in EF Core, use a Service Bus session per saga (to guarantee ordering), and dispatch by message type in a `BackgroundService` consumer. Works but requires building everything MassTransit gives you for free.

## Advantages

- **Works across services with independent databases** — no distributed transaction needed.
- **Supports long-running workflows** — minutes, hours, days.
- **Restart-safe** — saga state persisted between steps.
- **Auditable** — state transitions form a complete history.
- **Operationally repairable** — stuck sagas can be inspected, retried, or compensated by support.
- **Decoupled** — services only know about events and commands, not each other's internals.
- **Compensation gives explicit failure handling** — instead of silent partial completion.

## Disadvantages

- **Eventual consistency is customer-visible** — "your order is being processed" UX.
- **Compensation logic doubles the design surface** — every forward step needs a reverse.
- **Not every action is compensatable** — sent emails, shipped items, executed trades.
- **Higher operational complexity** — saga state store, broker, dead-letter queues, monitoring dashboards.
- **Debugging is harder** — one workflow touches multiple services with multiple log streams.
- **Duplicate messages must be handled** — every command and event needs idempotency.
- **Latency** — multi-step async workflow takes longer than a single synchronous transaction.

## Common Mistakes

### 1. Treating compensation as rollback

```csharp
// BAD — compensation is not "DELETE the row." It's a business action that
// semantically reverses the previous step. A captured payment is not undone by deleting it;
// it's undone by issuing a refund.
public Task CompensatePayment(Guid paymentId) => _db.Payments.Where(p => p.Id == paymentId).ExecuteDeleteAsync();
```

```csharp
// FIX — issue a real refund through the provider, idempotent.
public async Task CompensatePayment(Guid paymentId, string idempotencyKey)
{
    var payment = await _db.Payments.FindAsync(paymentId);
    await _gateway.RefundAsync(payment!.TransactionId, idempotencyKey);
    payment.Status = PaymentStatus.Refunded;
    await _db.SaveChangesAsync();
}
```

### 2. Non-idempotent step handlers

```csharp
// BAD — duplicate ReserveInventory message reserves the same items twice.
public async Task Consume(ConsumeContext<ReserveInventory> ctx)
{
    foreach (var item in ctx.Message.Items)
        await _inventory.DecrementAsync(item.Sku, item.Quantity);
    await ctx.Publish(new InventoryReserved(ctx.Message.OrderId));
}
```

```csharp
// FIX — keyed on OrderId; second attempt for the same OrderId is a no-op.
if (await _db.Reservations.AnyAsync(r => r.OrderId == ctx.Message.OrderId)) {
    await ctx.Publish(new InventoryReserved(ctx.Message.OrderId));   // re-publish for the saga
    return;
}
```

### 3. No correlation id

Without a correlation id flowing through every command, event, and log line, debugging a saga across 5 services is impossible. Every message must carry it; every log statement must include it.

### 4. Compensation that can also fail (and no plan for it)

```csharp
// BAD — refund call throws; saga is now "compensating" forever.
await ctx.Publish(new RefundPayment(orderId, txId));
```

**Fix** — compensations themselves must be retried with backoff, and after N attempts move to a `ManualInterventionRequired` state with a clear support ticket and alert.

### 5. Putting all coordination in HTTP calls

```csharp
// BAD — orchestrator makes synchronous HTTP calls to participants.
// One slow service blocks the entire workflow; one crashed orchestrator loses the workflow.
```

**Fix** — communicate via durable messages (Service Bus, Kafka). Saga state is persisted; messages survive restarts.

### 6. Saga that holds business locks

```csharp
// BAD — InventoryService reserves stock with a row-level lock and waits for the saga to finish
// (which may take 30 seconds across payment + shipping). Inventory rows are locked the whole time.
```

**Fix** — reservation is a **state transition**, not a lock. Insert a `Reservation` row with status `Held`, valid for N minutes; release it on failure or expiry, mark `Consumed` on completion.

### 7. Hiding the saga state from support

If support engineers cannot see "where is order 12345 right now?" they cannot help customers when the saga is stuck. Surface saga state in an admin tool with the current state, last transition, and recent events.

## Best Practices

- **Use orchestration for non-trivial workflows** (3+ steps, branches, failure paths).
- **Use choreography for simple linear flows** with stable contracts.
- **Persist saga state durably** — EF Core repository on SQL, Cosmos, or Redis.
- **Every command and event carries an idempotency key / correlation id.**
- **Compensations are real business actions** (refund, release, cancel), not row deletes.
- **Use the outbox pattern** so saga state and outgoing messages commit atomically. See [outbox-pattern.md](outbox-pattern.md).
- **Use an inbox / message dedup on consumers** to handle duplicate deliveries.
- **Add a timeout per step** — if `PaymentCaptured` doesn't arrive in 30s, transition to a timeout state and decide compensate vs. retry.
- **Define explicit `Failed`, `Compensating`, `ManualInterventionRequired` states** so support can find stuck sagas.
- **Build admin tooling** to inspect saga state and replay events.
- **Test failure paths** — chaos test by killing participants mid-saga.
- **Monitor**: active sagas by state, average completion time, stuck saga count, compensation count.

## Related Concepts

- [outbox-pattern.md](outbox-pattern.md) — reliable event publishing; saga state + outgoing message commit together.
- [reliability-design.md](reliability-design.md) — sagas are a reliability pattern for distributed workflows.
- [event-driven-architecture.md](event-driven-architecture.md) — sagas live on top of an event-driven backbone.
- [microservices-architecture.md](microservices-architecture.md) — the architecture that necessitates sagas.
- [cqrs.md](cqrs.md) — commands trigger saga steps; events advance them.
- [domain-events.md](domain-events.md) — saga events are usually integration events derived from domain events.
- [data-access/transactions.md](../data-access/transactions.md) — each saga step is a local transaction.
- [azure/azure-service-bus.md](../azure/azure-service-bus.md) — most common broker for saga messages.

## Real-World Usage

### E-commerce checkout

A `CheckoutSaga` orchestrates: validate cart → reserve inventory → capture payment → create shipment → notify customer. Each step is a separate microservice. Payment uses Stripe with an idempotency key equal to the saga correlation id. On payment failure, inventory is released. On shipment failure, payment is refunded *and* inventory is released. Stuck sagas (no transition in 24 hours) alert the on-call engineer.

### Subscription billing

A `RenewSubscriptionSaga` runs monthly: charge card → extend subscription → send invoice email. If the charge fails, the saga enters a `RetryDunning` state that retries with backoff over 7 days before transitioning to `Suspended` and emitting `SubscriptionSuspended`. The saga is restart-safe across multiple background worker pods on AKS.

### Bank transfer

A `TransferSaga` debits the source account → credits the destination account → records ledger entries. Compensation on credit failure: reverse the debit. Compensation cannot fail (it's local SQL); if it does, the saga moves to `ManualReconciliation` with a high-priority alert. Every step is double-booked in the ledger for auditability.

### Order refund

A `RefundSaga` orchestrates: validate refund eligibility → reverse charge on Stripe → restock inventory → notify customer → update accounting ledger. Each step is idempotent and the saga can be safely retried at any step.

### KYC onboarding

A long-running saga (minutes to hours): collect documents → run identity verification → run sanctions check → manual review (waits for human action) → activate account. The "wait for human" step is a state the saga can sit in for days; it's resumed by a `KycReviewed` event.

## Code Example — Before and After

### Before — synchronous HTTP chain, no compensation, partial failure leaves money lost

```csharp
public class CheckoutService
{
    private readonly HttpClient _inventory;
    private readonly HttpClient _payment;
    private readonly HttpClient _shipping;
    private readonly HttpClient _notification;

    public async Task<Guid> CheckoutAsync(CheckoutRequest req)
    {
        var orderId = Guid.NewGuid();

        // Synchronous chain across 4 services. No retries, no compensation, no correlation.
        await _inventory.PostAsJsonAsync($"/reserve/{orderId}", req.Items);
        await _payment.PostAsJsonAsync($"/charge/{orderId}", new { req.AmountCents });
        await _shipping.PostAsJsonAsync($"/ship/{orderId}", new { req.Address });
        await _notification.PostAsJsonAsync($"/notify/{orderId}", new { req.CustomerEmail });

        return orderId;
    }
}
```

Production failure modes:
- Payment succeeds, shipping returns 500. Customer is charged, no shipment, inventory held.
- Inventory reserved, payment times out at 30s. Customer sees error, retries → inventory reserved *again*.
- Notification service is down; checkout fails entirely even though order was already paid.
- No way to know "where did this order get stuck?" from logs.

### After — orchestrated saga, idempotent steps, compensation, observable state

```csharp
// 1. Saga state machine (excerpt from above) drives the workflow.
// 2. Each participant is an idempotent consumer.
// 3. Saga state is persisted; the system is restart-safe.

public record CheckoutRequested(Guid OrderId, Guid CustomerId, long AmountCents,
                                IReadOnlyList<OrderItem> Items, string CustomerEmail);

// API endpoint kicks off the saga and returns immediately.
app.MapPost("/api/checkout", async (CheckoutRequest req, IPublishEndpoint bus, CancellationToken ct) =>
{
    var orderId = Guid.NewGuid();
    await bus.Publish(new CheckoutRequested(orderId, req.CustomerId, req.AmountCents,
                                            req.Items, req.CustomerEmail), ct);
    return Results.Accepted($"/api/orders/{orderId}", new { orderId, status = "processing" });
});

// Customer polls or subscribes to SignalR for updates.
app.MapGet("/api/orders/{id}", async (Guid id, SagaDbContext db) =>
{
    var saga = await db.Set<CheckoutState>().FirstOrDefaultAsync(s => s.OrderId == id);
    return saga is null ? Results.NotFound() : Results.Ok(new
    {
        saga.OrderId,
        State = saga.CurrentState,
        saga.FailureReason,
        IsCompensating = saga.CurrentState is "Compensating",
        IsCompleted    = saga.CurrentState is "Completed"
    });
});
```

What's better:
- A crashed orchestrator pod is replaced; the saga resumes from the last persisted state.
- Payment failure triggers `ReleaseInventory` automatically — inventory is never stuck.
- Every message carries the saga correlation id; logs and traces are stitched end-to-end.
- Idempotent consumers + Service Bus dedup mean duplicate deliveries are safe.
- Support can query `CheckoutState` table to see exactly where any order is stuck.

## Interview Questions and Answers

### 1. Explain the saga pattern in your own words and why it exists.

**Why this matters** — opens the topic; checks whether you can avoid jargon.

**Answer** — A saga is a sequence of local transactions across services, where each step commits to its own database and failures are handled by **compensating actions** rather than a global rollback. It exists because microservices own separate databases, so a single ACID transaction across them is impractical (2PC is operationally expensive and rarely supported in cloud-managed services). Instead, we accept eventual consistency, commit each step locally, and design explicit compensation for each step we might need to reverse.

**Trade-off** — You give up the simplicity of "all or nothing" in exchange for cross-service workflows that actually work in production.

**Real project** — An e-commerce checkout used to chain 5 synchronous HTTP calls across services. A 0.5% failure rate at any step caused customer-visible inconsistencies (charged but no order, reserved but no payment). Replacing the chain with a MassTransit saga eliminated the partial-failure incidents.

### 2. Orchestration vs choreography — which would you choose for a checkout flow and why?

**Why this matters** — tests judgment, not memorization.

**Answer** — For checkout, **orchestration**. Checkout has branches (payment declined → compensate inventory; shipping fails → refund payment + release inventory), multiple failure paths, and needs central support visibility ("where is this order right now?"). A state machine in one place is dramatically easier to reason about, test, and operate than a web of services reacting to each other's events. Choreography is better for simple linear flows like `OrderPlaced → BillingNotified → InventoryDecremented` where each consumer is independent and there's no compensation logic.

**Trade-off** — The orchestrator can become a "god service" that needs careful boundaries; choreography is decoupled but the workflow is implicit and hard to debug.

**Real project** — A team built choreographed sagas for a 7-step onboarding flow. After 6 months, no one could explain the full workflow from any single service's code. Rewriting it as an orchestrated MassTransit state machine cut new-engineer ramp time from 2 weeks to 2 days.

### 3. What's a compensating transaction, and how is it different from a rollback?

**Why this matters** — fundamental concept; if you confuse them, the interview ends.

**Answer** — A rollback (ACID) undoes the *technical* effect of a committed change as if it never happened. A compensation is a **new business action** that semantically reverses a previous *committed* step. A captured Stripe payment cannot be rolled back — it's already moved money. The compensation is to issue a refund. A reserved inventory item is "compensated" by releasing the reservation. Both produce audit trail records of "we charged, then we refunded," which is the correct accounting view; a rollback would erase history.

**Trade-off** — Not every action is compensatable. A sent email cannot be unsent. A shipped item cannot be un-shipped. Design ordering matters: do the irreversible thing last, or accept that compensation is partial.

**Real project** — A booking system "compensated" failed bookings by deleting the seat reservation row. When the same flow needed audit logs, we had to redesign every compensation as a state transition (`Reserved → Released`) with timestamps and reasons.

### 4. How would you make a saga step idempotent?

**Why this matters** — at-least-once delivery means duplicate handlers are inevitable.

**Answer** — Three layers. (1) Pass an **idempotency key** in every command — typically the saga correlation id or a composite of (saga id, step name). (2) The handler checks an **inbox table** (or business-level unique constraint) keyed by that idempotency key; if seen, it re-publishes the success event and returns. (3) The downstream provider (Stripe, Service Bus) also receives the idempotency key for its own dedup. Combined, even if the same `CapturePayment` command is delivered 5 times, the customer is charged exactly once.

**Trade-off** — Adds an extra write per handler invocation; trivial cost compared to double-charging.

**Real project** — Service Bus redelivered messages during a node restart, double-charging customers. Adding an `IdempotencyKey` unique index on `payment_captures` made the system safe within a day; the prior production fix had taken a week of manual refunds.

### 5. Where do you store saga state, and what happens if the orchestrator crashes?

**Why this matters** — restart safety is what distinguishes a real saga from a workflow framework toy.

**Answer** — Saga state is persisted in a durable store — typically SQL via MassTransit's EF Core repository, or Cosmos DB for high write throughput. Each saga instance has a row keyed by correlation id. When a message arrives, MassTransit loads the saga, applies the transition, and persists the new state — all in one transaction (combined with the outbox pattern so outgoing messages commit atomically). If the orchestrator pod crashes mid-handling, the message is not acknowledged on Service Bus, so it redelivers to another pod. The other pod loads the saga from the database and processes the message from the persisted state. Nothing is lost.

**Trade-off** — Persisted saga state adds load on the database; for high-volume workflows (thousands of sagas/sec), Cosmos or partitioned SQL may be needed.

**Real project** — During a Kubernetes rolling update, 30 orchestrator pods restarted simultaneously. Active sagas (~12k) resumed cleanly on the new pods because state was in SQL. Before the saga rewrite, the previous design used in-memory state and lost every in-flight workflow on deploy.

### 6. How do you handle a step that times out — say, payment provider does not respond?

**Why this matters** — real-world failure isn't always "error returned"; it's silence.

**Answer** — Schedule a **timeout message** when entering the step: `Schedule.RequestTimeout(orderId, TimeSpan.FromSeconds(30))`. If `PaymentCaptured` or `PaymentFailed` arrives first, cancel the scheduled timeout. If the timeout fires first, transition to a `PaymentUnknown` state and start a reconciliation flow: query the provider's API directly to learn whether the payment actually went through. Based on the answer, either advance the saga (treat as success) or compensate. This handles the worst-case "provider captured but our message was lost" scenario.

**Trade-off** — Reconciliation adds complexity and a polling/query path, but the alternative — assuming failure and double-charging the customer when retrying — is unacceptable.

**Real project** — A payment provider had a 4-minute outage during which captures succeeded but webhook responses were lost. Without reconciliation, the saga assumed failure and refunded. After: saga waits in `PaymentUnknown`, queries the provider, finds the capture, advances to shipping. Zero double-refunds during the next incident.

### 7. What metrics and alerts would you put on a production saga system?

**Why this matters** — sagas fail silently if no one is watching.

**Answer** — (1) **Active saga count by state** — sudden spike in `Compensating` indicates upstream failure. (2) **Stuck sagas** — sagas in a non-terminal state longer than expected (e.g., > 5 minutes for checkout, > 24 hours for KYC); alert on count and provide a dashboard with details. (3) **Compensation rate per workflow** — rising compensations point to a failing participant. (4) **Average completion time** — regression here signals downstream slowdown. (5) **Saga state store latency and lock contention.** (6) **DLQ depth** on every saga-related Service Bus subscription. (7) **`ManualInterventionRequired` count** — pages on-call immediately.

**Trade-off** — Some metrics (per-saga-instance) are expensive at high volume; aggregate by state and use sampling for detail.

**Real project** — A retail platform had no saga observability; support discovered stuck sagas through customer complaints. Adding "sagas in state X older than Y" dashboards cut MTTR from days to minutes.

### 8. When would you NOT use a saga?

**Why this matters** — knowing the limits separates seniors from cargo-culters.

**Answer** — (1) When all participants live in one database — use a single transaction. (2) When the workflow is read-only — there's nothing to compensate. (3) When the business cannot accept eventual consistency or visible "your order is processing" UX — strong consistency is required, redesign the workflow. (4) When the system is small enough that a modular monolith with one DB transaction solves the problem more simply. (5) When the team lacks operational maturity to run brokers, monitor saga state, and handle stuck sagas — start simpler, introduce sagas when actual pain demands them.

**Trade-off** — Avoiding a saga keeps the system simple but caps growth options. Adopting one prematurely creates operational burden with no proportional benefit.

**Real project** — A team built a saga for a 2-step admin workflow inside one bounded context with one database. The saga's broker, state store, and monitoring cost more engineering than the feature it implemented. Replacing it with a single SQL transaction and a domain event cut the code by 80% with identical guarantees.

## Summary Checklist

- [ ] The workflow spans multiple services with separate databases (otherwise use one transaction).
- [ ] Saga state is persisted durably (EF Core repository on SQL/Cosmos), survives restarts.
- [ ] Each step has a defined compensation that is itself a real business action, not a row delete.
- [ ] Every command and event carries a correlation id / idempotency key; consumers are idempotent.
- [ ] Outgoing messages commit atomically with saga state via the outbox pattern.
- [ ] Per-step timeouts are scheduled; timeout transitions to a reconciliation or compensation state.
- [ ] Explicit terminal states exist: `Completed`, `Failed`, `Compensated`, `ManualInterventionRequired`.
- [ ] Compensation paths are themselves idempotent and retried with backoff; failed compensations alert.
- [ ] Saga state is queryable by support (admin tool with current state, history, last transition).
- [ ] Monitoring covers active counts by state, stuck sagas, compensation rate, completion time, DLQ depth.
- [ ] Orchestration is the default for complex workflows; choreography only for simple linear flows.
- [ ] Failure paths are tested (chaos: kill orchestrator mid-saga, kill participants, redeliver messages).

