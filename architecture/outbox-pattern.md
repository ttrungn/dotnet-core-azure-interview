# Outbox Pattern

## What It Is

The Outbox Pattern is a way to **reliably publish messages or integration events** when those messages must reflect a database change. Instead of writing to the database and *then* publishing to a broker (two separate operations that can fail independently), the application writes the business row **and** an `outbox_messages` row in the **same database transaction**. A separate publisher process later reads unpublished outbox rows and sends them to the broker (Azure Service Bus, RabbitMQ, Kafka, Event Grid).

This trades immediate publishing for **atomicity between state and event**, and it shifts the system from "lost messages are possible" to "messages are delivered at least once and consumers must be idempotent."

```csharp
// BROKEN — dual write. SQL commits, then process crashes before publish.
// Order exists, but no OrderPlaced event was sent. Billing never bills.
await _db.Orders.AddAsync(order);
await _db.SaveChangesAsync();                                 // (1) commit
await _serviceBus.SendAsync(new OrderPlaced(order.Id));       // (2) crash here = lost event

// OUTBOX — one transaction, then async delivery.
await _db.Orders.AddAsync(order);
await _db.OutboxMessages.AddAsync(OutboxMessage.From(new OrderPlaced(order.Id)));
await _db.SaveChangesAsync();                                 // atomic: order + event row commit together
// A BackgroundService publishes the outbox row to Service Bus later.
```

## Why It Exists

The outbox exists because of the **dual-write problem**: you cannot atomically commit to two different systems (a relational database and a message broker) without a distributed transaction coordinator, and those coordinators (MS DTC, XA) are operationally heavy, slow, and largely unavailable in cloud-managed services like Azure SQL or Service Bus.

The naive "save then publish" approach has three failure modes that all corrupt your system:

1. **Save succeeds, publish fails (or process crashes between).** The order exists in SQL. Billing, shipping, and inventory never hear about it. The customer sees their order in "history" but nothing happens.
2. **Publish succeeds, save rolls back.** Worse: downstream services react to an `OrderPlaced` event for an order that does not exist in the source-of-truth database.
3. **Both succeed but in the wrong order.** Consumers see the event before the source database is queryable (read replica lag, eventual consistency).

The outbox pattern collapses these to a single failure mode you can handle: **the message will be delivered eventually, possibly more than once**. From there, idempotent consumers and an inbox table close the loop.

This problem became unavoidable with microservices and event-driven architectures. A modular monolith can use one transaction across modules; a payment service publishing `PaymentCaptured` to a separate billing service cannot.

## When To Use It

**Use the outbox pattern when:**

- A service writes to its database **and** must publish an integration event (most microservice writes).
- You use Service Bus, Kafka, RabbitMQ, EventGrid, or any broker that is operationally separate from your DB.
- A saga depends on the event being published reliably ([saga-pattern.md](saga-pattern.md)).
- Compliance/audit requires every state change to produce a downstream record.
- You currently see "ghost orders" or "missing events" in incident reports.

**Do not use the outbox when:**

- The event consumer lives in the same database transaction (same monolith, same DB). Just call it directly.
- You truly do not care if the event is lost (e.g., debug telemetry, analytics best-effort).
- You're using a DB+broker combination with native transactional outbox (some Kafka+Debezium setups, EventStore).
- The volume of events is so high that polling/CDC overhead becomes the bottleneck and you have a single-system alternative (e.g., Kafka as both store and broker).

## Why It Is Important

Outbox is a foundation for **trustworthy event-driven systems**. Five production properties depend on it:

1. **No lost events after commit.** Once `SaveChangesAsync` returns, the event is durably queued and will be delivered.
2. **No phantom events.** Events for rolled-back transactions never get published, because the outbox row was rolled back with the business data.
3. **Restart safety.** If the publisher crashes, the next instance picks up unpublished rows. If the consumer crashes, the broker redelivers.
4. **Operational visibility.** The outbox table is itself a real-time audit log: which events were emitted, when, in what order, with what payload.
5. **Saga reliability.** Long-running workflows can only advance if each step's event is reliably published. Without an outbox, sagas silently stall.

In the cloud, the outbox is the bridge between Azure SQL (transactional, source of truth) and Azure Service Bus / Event Grid / Event Hubs (delivery). Without it, you have a beautifully consistent database publishing events into a void.

## When To Use It vs Direct Publishing

| Aspect | Direct Publish After Save | Outbox |
|---|---|---|
| Atomicity | None (dual write) | Atomic with business data |
| Latency | Lower | Slight delay (polling interval) |
| Lost events on crash | Possible | Impossible after commit |
| Consumer dedupe needed | If retries exist | Always (at-least-once) |
| Infra complexity | Low | Outbox table + publisher + monitoring |
| Right choice when | Best-effort signals only | Money, inventory, orders, sagas |

## How It's Used in C# / .NET

### 1. Define the outbox table

```csharp
public class OutboxMessage
{
    public Guid     Id           { get; set; }            // primary key, also message id
    public string   Type         { get; set; } = null!;   // "OrderPlaced", fully-qualified or short
    public string   Payload      { get; set; } = null!;   // JSON of the event
    public DateTime OccurredOnUtc{ get; set; }
    public DateTime? ProcessedOnUtc { get; set; }         // null = not yet published
    public string?  Error        { get; set; }
    public int      Attempts     { get; set; }
    public string?  CorrelationId{ get; set; }

    public static OutboxMessage From<T>(T evt) where T : notnull => new()
    {
        Id            = Guid.NewGuid(),
        Type          = typeof(T).AssemblyQualifiedName!,
        Payload       = JsonSerializer.Serialize(evt),
        OccurredOnUtc = DateTime.UtcNow
    };
}
```

### 2. Configure with EF Core

```csharp
public class OrderDbContext(DbContextOptions<OrderDbContext> opts) : DbContext(opts)
{
    public DbSet<Order>           Orders          => Set<Order>();
    public DbSet<OutboxMessage>   OutboxMessages  => Set<OutboxMessage>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<OutboxMessage>(e =>
        {
            e.ToTable("outbox_messages");
            e.HasKey(x => x.Id);
            e.HasIndex(x => x.ProcessedOnUtc).HasFilter("[ProcessedOnUtc] IS NULL");
            e.Property(x => x.Payload).HasColumnType("nvarchar(max)");
        });
    }
}
```

The filtered index on `ProcessedOnUtc IS NULL` is critical — the publisher scans only unpublished rows.

### 3. Write business data + outbox row atomically

```csharp
public async Task<Guid> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = new Order(cmd.CustomerId, cmd.Items, cmd.AmountCents);
    _db.Orders.Add(order);

    _db.OutboxMessages.Add(OutboxMessage.From(
        new OrderPlaced(order.Id, order.CustomerId, order.AmountCents, DateTime.UtcNow)));

    await _db.SaveChangesAsync(ct);                       // one transaction, atomic
    return order.Id;
}
```

### 4. Background publisher — `BackgroundService` polling Azure Service Bus

```csharp
public class OutboxPublisher(
    IServiceScopeFactory scopes,
    ServiceBusClient sbClient,
    ILogger<OutboxPublisher> log) : BackgroundService
{
    private readonly ServiceBusSender _sender = sbClient.CreateSender("integration-events");

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try { await PublishBatchAsync(stoppingToken); }
            catch (Exception ex) { log.LogError(ex, "Outbox publish loop error"); }
            await Task.Delay(TimeSpan.FromSeconds(2), stoppingToken);
        }
    }

    private async Task PublishBatchAsync(CancellationToken ct)
    {
        using var scope = scopes.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderDbContext>();

        // Claim a batch using row-level locks so two instances don't grab the same rows.
        var batch = await db.OutboxMessages
            .FromSqlRaw(@"
                SELECT TOP (50) * FROM outbox_messages WITH (UPDLOCK, READPAST)
                WHERE ProcessedOnUtc IS NULL
                ORDER BY OccurredOnUtc")
            .ToListAsync(ct);

        if (batch.Count == 0) return;

        foreach (var msg in batch)
        {
            try
            {
                var sbMsg = new ServiceBusMessage(msg.Payload)
                {
                    MessageId   = msg.Id.ToString(),       // enables Service Bus dedup
                    Subject     = msg.Type,
                    ContentType = "application/json",
                    CorrelationId = msg.CorrelationId
                };
                await _sender.SendMessageAsync(sbMsg, ct);
                msg.ProcessedOnUtc = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                msg.Attempts++;
                msg.Error = ex.Message;
                log.LogWarning(ex, "Failed to publish outbox {Id}, attempt {N}", msg.Id, msg.Attempts);
            }
        }

        await db.SaveChangesAsync(ct);
    }
}

builder.Services.AddHostedService<OutboxPublisher>();
```

Key details:
- `WITH (UPDLOCK, READPAST)` lets multiple publisher instances run safely; each skips locked rows.
- `MessageId = msg.Id` enables **Service Bus duplicate detection** (when configured on the queue, 1-7 day window), giving you a second deduplication layer.
- The filtered index makes the scan O(unpublished count), not O(table size).

### 5. MassTransit's transactional outbox

For most production systems, use a library instead of hand-rolling:

```csharp
// NuGet: MassTransit, MassTransit.Azure.ServiceBus.Core, MassTransit.EntityFrameworkCore
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<OrderDbContext>(o =>
    {
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.UseSqlServer();
        o.UseBusOutbox();
    });

    x.AddConsumer<OrderPlacedConsumer>();
    x.UsingAzureServiceBus((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["ServiceBus:ConnectionString"]);
        cfg.ConfigureEndpoints(ctx);
    });
});
```

MassTransit creates `InboxState`, `OutboxMessage`, and `OutboxState` tables for you, handles the publishing loop, and gives consumers an inbox for deduplication automatically.

### 6. CDC alternative (Debezium / SQL Server CDC)

Instead of polling, enable SQL Server Change Data Capture on `outbox_messages` and let Debezium stream inserts to Kafka. Eliminates polling lag at the cost of operational complexity (Debezium connector, Kafka Connect cluster, schema registry).

### 7. Inbox pattern for consumers

```csharp
public async Task HandleAsync(ServiceBusReceivedMessage msg, CancellationToken ct)
{
    var msgId = Guid.Parse(msg.MessageId);
    if (await _db.ProcessedMessages.AnyAsync(p => p.Id == msgId, ct))
        return;                                            // already handled, drop

    using var tx = await _db.Database.BeginTransactionAsync(ct);
    await ApplyBusinessChangeAsync(msg, ct);
    _db.ProcessedMessages.Add(new ProcessedMessage(msgId, DateTime.UtcNow));
    await _db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
```

## Advantages

- **Atomicity** between business state and integration event — no lost or phantom events.
- **Restart-safe** — crashes anywhere between save and publish are recovered automatically.
- **Built-in audit log** — every event ever emitted is in one table, queryable for debugging.
- **Broker-agnostic** — works identically for Service Bus, Kafka, RabbitMQ, Event Grid.
- **Supports complex workflows** — sagas, choreography, and event sourcing all depend on reliable publish.
- **No distributed transactions** — works with any cloud-managed DB and broker.

## Disadvantages

- **At-least-once delivery, never exactly-once.** Consumers must deduplicate. Anyone who tells you otherwise is wrong.
- **Publishing latency** — polling interval (1-5 seconds typical) adds delay between commit and delivery.
- **Extra writes** to the same database — small but non-zero load increase.
- **Cleanup discipline** — `outbox_messages` grows unbounded without archival/deletion.
- **Operational surface** — publisher health, lag metrics, DLQ for poison messages, schema versioning.
- **Coordination required** with multiple publisher instances — needs row-level locks or leader election.

## Common Mistakes

### 1. Publishing inside the same SaveChanges call

```csharp
// BAD — defeats the entire purpose. The publish blocks the transaction, and a network
// blip during publish rolls back the order.
_db.Orders.Add(order);
await _serviceBus.SendAsync(new OrderPlaced(order.Id));   // synchronous publish in tx
await _db.SaveChangesAsync();
```

```csharp
// FIX — write the outbox row, let the background publisher deliver.
_db.Orders.Add(order);
_db.OutboxMessages.Add(OutboxMessage.From(new OrderPlaced(order.Id)));
await _db.SaveChangesAsync();
```

### 2. No deduplication on the consumer side

```csharp
// BAD — every retry creates a new invoice.
public async Task Handle(OrderPlaced evt) =>
    await _invoices.CreateAsync(new Invoice(evt.OrderId, evt.AmountCents));
```

```csharp
// FIX — inbox table or unique constraint on (OrderId).
if (await _db.Invoices.AnyAsync(i => i.OrderId == evt.OrderId)) return;
```

### 3. Two publisher instances racing on the same rows

```csharp
// BAD — both publishers read the same batch, both publish, customer billed twice.
var batch = await _db.OutboxMessages.Where(m => m.ProcessedOnUtc == null).Take(50).ToListAsync();
```

```csharp
// FIX — row-level locks with READPAST so each instance grabs a disjoint batch.
var batch = await _db.OutboxMessages.FromSqlRaw(@"
    SELECT TOP (50) * FROM outbox_messages WITH (UPDLOCK, READPAST)
    WHERE ProcessedOnUtc IS NULL ORDER BY OccurredOnUtc").ToListAsync();
```

### 4. Unbounded outbox growth

```sql
-- After 6 months, outbox_messages has 80M rows, the publisher's filtered scan slows down,
-- backups balloon, and DML on the table starts to lock.

-- FIX — scheduled cleanup of published rows older than retention window.
DELETE FROM outbox_messages
WHERE ProcessedOnUtc IS NOT NULL AND ProcessedOnUtc < DATEADD(day, -7, GETUTCDATE());
```

### 5. Storing full aggregates in the payload

```csharp
// BAD — payload includes the full Order with 200 line items and PII; the event is fragile,
// large, and leaks internal model changes to every consumer.
OutboxMessage.From(order);
```

```csharp
// FIX — explicit integration event with stable, minimal contract.
OutboxMessage.From(new OrderPlaced(order.Id, order.CustomerId, order.TotalCents, order.PlacedAtUtc));
```

### 6. Claiming "exactly-once" delivery

It does not exist over a network. Outbox guarantees **at-least-once publish**; consumers must be idempotent (inbox table, unique constraints, conditional writes).

### 7. No alerting on publish lag

If the publisher dies at 2 AM, the outbox queue grows for 6 hours. A simple alert on `MAX(NOW - OccurredOnUtc) WHERE ProcessedOnUtc IS NULL > 60s` catches it within a minute.

## Best Practices

- **One transaction, always.** Business data + outbox row must commit together.
- **Set Service Bus `MessageId` to the outbox row id** so the broker dedupes if the publisher retries.
- **Filtered index on `ProcessedOnUtc IS NULL`** keeps the publisher scan O(backlog), not O(history).
- **Row-level locks with `READPAST`** allow horizontal publisher scaling without race conditions.
- **Define an explicit integration event contract** — never publish raw domain entities.
- **Version events** with `EventType` + `SchemaVersion`. Add fields additively.
- **Inbox table on consumers** dedupes by `MessageId`; combine with business unique constraints for defense in depth.
- **Cleanup job** archives or deletes processed rows after the retention window.
- **Monitor publish lag and unpublished depth** — these are first-class SLO metrics.
- **Use MassTransit's EF outbox** unless you have a strong reason to hand-roll.
- **Configure Service Bus duplicate detection** (10-minute to 7-day window) as a second safety net.

## Related Concepts

- [saga-pattern.md](saga-pattern.md) — sagas advance only when each step's event is reliably published.
- [reliability-design.md](reliability-design.md) — outbox is a core reliability primitive.
- [domain-events.md](domain-events.md) — domain events become integration events through the outbox.
- [event-driven-architecture.md](event-driven-architecture.md) — outbox is what makes EDA trustworthy.
- [cqrs.md](cqrs.md) — command-side writes often emit events via outbox.
- [unit-of-work.md](unit-of-work.md) — the unit of work commits business data and outbox row together.
- [data-access/transactions.md](../data-access/transactions.md) — the local transaction is the foundation.
- [azure/azure-service-bus.md](../azure/azure-service-bus.md) — most common broker target for the outbox.

## Real-World Usage

### E-commerce checkout

Order service writes `orders` row + `OrderPlaced` outbox row in one transaction. Publisher delivers to Service Bus topic `integration-events`. Billing, Inventory, Shipping, and Notification services each have their own subscription with their own dead-letter queue. Each consumer maintains an inbox table.

### Payment capture

Payment service records `payment_attempts` row + `PaymentCaptured` outbox row atomically. Even if the Service Bus is down, the row is safe. When Service Bus recovers, the backlog drains. Downstream `Billing` consumer is idempotent on `PaymentAttemptId`.

### Saga step advancement

A `MassTransit` state machine for order fulfillment uses the outbox so each state transition's event (`InventoryReserved`, `PaymentCaptured`, `ShipmentRequested`) is durably published. Without the outbox, a crash between state save and event publish would stall the saga indefinitely.

### Audit and compliance

A regulated banking platform stores 2 years of outbox rows (archived monthly to Azure Blob Storage cold tier). Every integration event ever emitted is replayable and auditable, which satisfies financial regulators.

### High-throughput CDC variant

A trading platform handles 50k events/sec. Polling overhead is unacceptable, so they use SQL Server CDC + Debezium to stream `outbox_messages` inserts to Kafka in real time, with end-to-end p99 lag under 100ms.

## Code Example — Before and After

### Before — dual-write, lost events, double-charge on replay

```csharp
public class OrderService
{
    private readonly OrderDbContext _db;
    private readonly ServiceBusSender _sender;

    public async Task<Guid> PlaceOrderAsync(PlaceOrderCommand cmd)
    {
        var order = new Order(cmd.CustomerId, cmd.AmountCents);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();                               // commit 1

        var evt = new OrderPlaced(order.Id, order.AmountCents);
        await _sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.Serialize(evt)));
        // ^ if this throws, the order exists but billing/shipping never hear about it.
        // if the process crashes between, same outcome.

        return order.Id;
    }
}

public class BillingConsumer
{
    public async Task Handle(OrderPlaced evt)
    {
        // No dedup. On retry/redelivery, the customer is billed twice.
        await _billing.ChargeAsync(evt.OrderId, evt.AmountCents);
    }
}
```

Production failure modes seen here:
- 0.3% of orders silently never reach billing during Service Bus throttling incidents.
- Periodic Service Bus connection resets cause `SendMessageAsync` to retry; consumers receive duplicates and double-charge.
- A failed deploy leaves orphan orders that require manual SQL repair.

### After — atomic outbox, idempotent consumer, observable

```csharp
public class OrderService(OrderDbContext db, ILogger<OrderService> log)
{
    public async Task<Guid> PlaceOrderAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order(cmd.CustomerId, cmd.AmountCents);
        db.Orders.Add(order);

        db.OutboxMessages.Add(OutboxMessage.From(
            new OrderPlaced(order.Id, order.CustomerId, order.AmountCents, DateTime.UtcNow)));

        await db.SaveChangesAsync(ct);                              // atomic
        log.LogInformation("Order {OrderId} placed, outbox event queued", order.Id);
        return order.Id;
    }
}

public class OutboxPublisher(IServiceScopeFactory scopes, ServiceBusClient sb,
                             IMeterFactory meters, ILogger<OutboxPublisher> log) : BackgroundService
{
    private readonly ServiceBusSender _sender = sb.CreateSender("integration-events");
    private readonly Histogram<double> _lag =
        meters.Create("checkout").CreateHistogram<double>("outbox.publish_lag_ms");

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<OrderDbContext>();

            var batch = await db.OutboxMessages.FromSqlRaw(@"
                SELECT TOP (100) * FROM outbox_messages WITH (UPDLOCK, READPAST)
                WHERE ProcessedOnUtc IS NULL ORDER BY OccurredOnUtc").ToListAsync(ct);

            foreach (var m in batch)
            {
                try
                {
                    await _sender.SendMessageAsync(new ServiceBusMessage(m.Payload)
                    {
                        MessageId = m.Id.ToString(),
                        Subject   = m.Type
                    }, ct);
                    m.ProcessedOnUtc = DateTime.UtcNow;
                    _lag.Record((m.ProcessedOnUtc.Value - m.OccurredOnUtc).TotalMilliseconds);
                }
                catch (Exception ex) { m.Attempts++; m.Error = ex.Message; log.LogWarning(ex, "publish failed {Id}", m.Id); }
            }
            await db.SaveChangesAsync(ct);
            if (batch.Count == 0) await Task.Delay(2000, ct);
        }
    }
}

public class BillingConsumer(BillingDbContext db, IPaymentGateway gateway)
{
    public async Task HandleAsync(ServiceBusReceivedMessage msg, CancellationToken ct)
    {
        var msgId = Guid.Parse(msg.MessageId);
        if (await db.ProcessedMessages.AnyAsync(p => p.Id == msgId, ct)) return;     // inbox dedup

        var evt = JsonSerializer.Deserialize<OrderPlaced>(msg.Body.ToString())!;
        await gateway.ChargeAsync(evt.OrderId, evt.AmountCents, idempotencyKey: msgId.ToString(), ct);

        db.ProcessedMessages.Add(new ProcessedMessage(msgId, DateTime.UtcNow));
        await db.SaveChangesAsync(ct);
    }
}
```

What's better:
- The order and its event commit atomically. No more ghost orders.
- The publisher is restart-safe; multiple instances coordinate via `UPDLOCK, READPAST`.
- The consumer dedupes on `MessageId`; the payment gateway *also* dedupes on the same id. Double defense.
- `publish_lag_ms` metric surfaces publisher health; alert when p95 > 5s.

## Interview Questions and Answers

### 1. What problem does the outbox pattern solve, in one sentence?

**Why this matters** — tests whether you can articulate the dual-write problem without buzzwords.

**Answer** — The outbox solves the **dual-write problem**: a service cannot atomically commit to its database and publish to a message broker, so the outbox stores the message in the same DB transaction as the business change and a separate publisher delivers it asynchronously.

**Trade-off** — You shift from "messages may be lost" to "messages are at-least-once," which means consumers must be idempotent.

**Real project** — A B2B order platform was losing ~0.5% of `OrderPlaced` events during Service Bus throttling. Introducing an outbox eliminated the loss and made the inconsistency operationally observable through publish-lag metrics.

### 2. Does the outbox give you exactly-once delivery?

**Why this matters** — separates engineers who repeat marketing claims from those who understand distributed systems.

**Answer** — No. End-to-end exactly-once over a network is impossible. The outbox guarantees **at-least-once publish** to the broker. If the publisher sends, then crashes before marking the row processed, the next run sends again. Consumers must deduplicate using `MessageId` (inbox table) or business-level idempotency (unique constraints, conditional writes). Service Bus duplicate detection adds another layer but is bounded by the dedup window.

**Trade-off** — You accept slightly more complexity in consumers in exchange for never losing a message after commit.

**Real project** — Confirmed during a failover drill: the publisher was killed mid-batch. The replacement instance redelivered ~12 messages. Without consumer dedup, that would have meant 12 duplicate invoices.

### 3. How do you prevent two publisher instances from sending the same outbox row twice?

**Why this matters** — operational reality. Horizontal scaling is normal, so the design must be safe for it.

**Answer** — Use database row-level locks when claiming the batch. In SQL Server, `WITH (UPDLOCK, READPAST)` lets each publisher take an exclusive lock on its rows and skip rows locked by other publishers, so the two instances naturally claim disjoint batches. Alternatives: leader election (only one publisher active at a time), or a `claimed_by`/`claimed_until` column with optimistic acquisition.

**Trade-off** — Row-level locking has minor overhead but scales fine to dozens of publishers; leader election is simpler conceptually but reduces throughput to a single instance.

**Real project** — A retail platform ran 4 publisher pods. Without `READPAST`, periodic Service Bus duplicates were caused by both pods grabbing the same batch. Adding `UPDLOCK, READPAST` eliminated the cross-pod duplication entirely.

### 4. Polling vs CDC — when would you choose each?

**Why this matters** — tests judgment on infrastructure vs simplicity trade-offs.

**Answer** — **Polling** is the right default. It's simple (one `BackgroundService`), needs no extra infrastructure, and a 1-2 second poll interval is acceptable for almost every event-driven workflow. **CDC** (Debezium, SQL Server Change Data Capture, native Postgres logical replication) is justified when (a) polling lag is unacceptable (sub-100ms requirements), (b) outbox volume is so high that polling overhead is meaningful (tens of thousands of events/sec), or (c) you already operate Kafka Connect / a streaming platform.

**Trade-off** — CDC adds operational complexity (connector lifecycle, schema registry, monitoring) that is rarely worth it for a 2-second latency improvement.

**Real project** — A 50k-events/sec trading platform used CDC + Debezium + Kafka. A typical SaaS at 100 events/sec polling every 2 seconds met all its SLOs and avoided the CDC operations burden.

### 5. The outbox table has grown to 200M rows and queries are slow. What do you do?

**Why this matters** — checks that you treat operational hygiene as a first-class design concern.

**Answer** — Three things. (1) Verify the filtered index on `ProcessedOnUtc IS NULL` exists; if so, the publisher scan should still be fast on the small unpublished portion. (2) Run a scheduled cleanup job that deletes (or archives to Blob Storage) processed rows older than the retention window — typically 7-30 days. (3) Delete in batches with `WAITFOR DELAY` to avoid blocking DML on the table; consider partition switching for very high volume. Long term, document retention as part of the service runbook.

**Trade-off** — Aggressive deletion loses audit history; long retention costs storage. Pick based on regulatory and debugging needs.

**Real project** — A 6-month-old payment platform had 80M outbox rows, backups took 4 hours, and the publisher's `WHERE ProcessedOnUtc IS NULL` scan started showing in slow-query logs because the index had been dropped during a migration. Recreating the filtered index restored sub-millisecond scans; a 7-day cleanup job brought the table back to 2M rows.

### 6. Walk me through how the outbox interacts with a saga.

**Why this matters** — tests integration of two related patterns.

**Answer** — A saga step is a local transaction. After the step commits, the saga must publish an event/command to trigger the next step. Without the outbox, a crash between commit and publish stalls the saga forever (state moved forward, but nobody knows). With the outbox: the saga writes the step's state change **and** the outgoing message in one transaction. The publisher delivers asynchronously. The next saga participant consumes the message (with inbox dedup) and runs its own local transaction + outbox row. The pattern repeats until completion or compensation.

**Trade-off** — Saga steps are slightly slower (publish lag), but the saga is now restart-safe and recoverable.

**Real project** — An order fulfillment saga (reserve inventory → capture payment → create shipment → notify) was losing ~1 in 500 sagas to "stuck" states. Introducing MassTransit's transactional outbox eliminated the stuck sagas because every state transition was now durably published.

### 7. How would you version an event schema without breaking consumers?

**Why this matters** — events are public contracts; breaking them breaks production.

**Answer** — Treat integration events like public APIs. **Only additive changes** in v1 of a schema — add new optional fields, never remove or rename. When breaking changes are needed, publish a new event type (`OrderPlacedV2`) alongside the old one for a deprecation window. Consumers handle both, the producer dual-publishes, and after all consumers migrate, the old type is retired. Embed `SchemaVersion` in the payload for explicit handling. Maintain a registry (Confluent Schema Registry or a wiki) of all event contracts.

**Trade-off** — Dual-publishing during migration doubles outbox volume and adds consumer complexity, but it's the only safe way to evolve.

**Real project** — A payment platform renamed `Amount` to `AmountCents` in `PaymentCaptured`. A naive deploy broke 3 downstream consumers. The recovery pattern became mandatory: publish `PaymentCapturedV2`, leave `PaymentCaptured` in place, migrate consumers one by one over 6 weeks, then retire V1.

### 8. What metrics would you monitor and alert on for the outbox?

**Why this matters** — silent failures in async pipelines are the worst kind.

**Answer** — Key metrics: (1) **Unpublished depth** — `COUNT(*) WHERE ProcessedOnUtc IS NULL`, alert if > threshold. (2) **Publish lag** — `MAX(NOW - OccurredOnUtc) WHERE ProcessedOnUtc IS NULL`, alert if > 60s. (3) **Publish error rate** per publisher pod. (4) **Attempts distribution** — high `Attempts` indicates poison messages. (5) **DLQ depth** on the broker side, since downstream failures back-pressure the system. (6) **Cleanup job last-run timestamp** — silent cleanup failure leads to table bloat.

**Trade-off** — Too many alerts cause fatigue. Page only on lag/depth above SLO; ticket the rest.

**Real project** — A health check that returned 200 even when the outbox publisher was deadlocked led to a 3-hour silent outage. Adding `publish_lag_ms` as a first-class metric with a 1-minute alert reduced detection from hours to seconds.

## Summary Checklist

- [ ] Business data and the outbox row commit in **one** database transaction.
- [ ] The outbox table has a filtered index on `ProcessedOnUtc IS NULL`.
- [ ] The background publisher uses row-level locks (`UPDLOCK, READPAST`) so multiple instances run safely.
- [ ] `MessageId` on the broker message equals the outbox row id (enables broker-level dedup).
- [ ] Consumers maintain an inbox table (or use a unique business constraint) to dedupe by `MessageId`.
- [ ] Integration events have explicit, minimal, versioned contracts — never raw domain entities.
- [ ] A cleanup job archives or deletes processed rows after the retention window.
- [ ] Metrics expose publish lag, unpublished depth, attempts, and publisher errors.
- [ ] Alerts fire when publish lag exceeds the SLO (typically > 60s) or unpublished depth grows unbounded.
- [ ] The system is documented as **at-least-once**, never claimed as exactly-once.

