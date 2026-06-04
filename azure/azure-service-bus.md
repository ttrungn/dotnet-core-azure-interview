# Azure Service Bus

## What It Is

Azure Service Bus is a fully managed enterprise message broker offering two messaging primitives:

- **Queues** — point-to-point. One message → one competing consumer. Used for work distribution.
- **Topics and subscriptions** — publish/subscribe. One message → fan-out to N subscriptions, each with its own filter.

Both support transactional sessions, dead-lettering, scheduled messages, duplicate detection, and AMQP 1.0. It's the broker of choice for **decoupling .NET microservices** where you need ordering, exactly-once-ish semantics, and operational maturity.

```csharp
// Publishing an OrderPlaced event so Billing, Inventory, and Notification can each consume it
await using var client = new ServiceBusClient(
    "contoso-prod.servicebus.windows.net",
    new DefaultAzureCredential());

var sender = client.CreateSender("order-events");
var message = new ServiceBusMessage(BinaryData.FromObjectAsJson(orderPlaced))
{
    MessageId = orderPlaced.OrderId.ToString(),    // duplicate detection key
    Subject = nameof(OrderPlaced),                 // subscription filter
    ContentType = "application/json"
};
await sender.SendMessageAsync(message, ct);
```

## Why It Exists

Direct HTTP calls between services break under load and during partial outages:

1. **Tight coupling** — if Billing is down, Checkout returns 500 to the customer.
2. **Backpressure** — a slow downstream service drags the whole chain.
3. **No retry semantics** — every caller reinvents retry logic, with different bugs.
4. **No ordering or once-per-customer guarantees** — concurrent processing of the same customer's orders causes balance drift.
5. **No durable buffering** — a deploy that takes 90 seconds drops 90 seconds of events.
6. **Poison messages** — one malformed payload poisons the whole consumer.

Service Bus gives durable buffering, broker-managed retries, dead-lettering, session-based ordering, FIFO per session, transactions, and the operational backbone (private endpoints, Geo-DR, premium isolation) that enterprise workloads require. It's heavier than Storage Queues and Event Hubs but solves problems neither can.

## When To Use It

**Use Service Bus for:**

- Decoupling microservices (Order → Billing, Inventory, Notification).
- Background work offloaded from HTTP requests.
- Per-tenant/per-customer ordered processing via **sessions**.
- Transactional outbox consumption with **duplicate detection**.
- Scheduled delivery (`ScheduledEnqueueTimeUtc`) — e.g., "send reminder in 24 hours".
- Workflows that need **exactly-once-ish** delivery (peek-lock + idempotent consumer).
- Integration with on-premise / legacy systems via AMQP.

**Do not use Service Bus for:**

- High-volume telemetry / event streams at millions per second → use **Event Hubs**.
- Lightweight task queues with no ordering or transaction needs → use **Storage Queues** (cheaper, simpler).
- Real-time fan-out to browsers → use **SignalR / Web PubSub**.
- Replayable event sourcing log → use **Event Hubs** or **Kafka**.
- Sub-millisecond messaging → Service Bus latency is ~20–80ms; not designed for HFT.

## Why It Is Important

In any real production microservice topology, Service Bus is the load-bearing decoupler. It directly enables:

- **Reliability** — a Billing outage doesn't break checkout; orders queue and process when Billing recovers.
- **Scalability** — consumers scale horizontally based on queue depth.
- **Compliance** — broker-side audit, AMQP over TLS, Private Endpoint, Managed Identity.
- **Operational maturity** — dead-letter queues capture poison messages for human review, message TTL prevents queue buildup, scheduled delivery handles retries and delays.
- **Geo-resilience** — Premium tier supports **Geo-DR** with paired primary/secondary namespaces and a single alias.

If an interviewer asks "how do you publish an event to be consumed by multiple services", Service Bus topics + subscriptions is the textbook answer.

## How It's Used in C# / .NET

NuGet packages:

- `Azure.Messaging.ServiceBus` — the modern SDK (replaces the deprecated `Microsoft.Azure.ServiceBus`).
- `Microsoft.Extensions.Azure` — `AddAzureClients` for DI registration with shared connection and retry policy.
- `Azure.Identity` — Managed Identity authentication.

### Queue vs. topic + subscription

| Primitive | Use when |
|---|---|
| **Queue** | Work distribution to one consumer pool. Example: `payment-capture-requests`. |
| **Topic + Subscription** | Fan-out — multiple independent consumers. Example: `order-events` topic, with `billing-orders`, `inventory-orders`, `notification-orders` subscriptions. |

### Peek-lock vs. receive-and-delete

| Mode | Semantics | Use when |
|---|---|---|
| **Peek-lock** (default) | Message locked for N seconds; consumer must `Complete` or it returns to queue | Always, for business workflows |
| **Receive-and-delete** | Message removed on receive | Only for fire-and-forget telemetry where loss is acceptable |

### Modern consumer with `ServiceBusProcessor`

```csharp
public sealed class OrderPlacedConsumer : BackgroundService
{
    private readonly ServiceBusProcessor _processor;
    private readonly IBillingService _billing;
    private readonly ILogger<OrderPlacedConsumer> _log;

    public OrderPlacedConsumer(
        ServiceBusClient client,
        IBillingService billing,
        ILogger<OrderPlacedConsumer> log)
    {
        _processor = client.CreateProcessor(
            topicName: "order-events",
            subscriptionName: "billing-orders",
            new ServiceBusProcessorOptions
            {
                MaxConcurrentCalls = 16,
                AutoCompleteMessages = false,        // we control completion
                MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5),
                PrefetchCount = 32
            });
        _billing = billing;
        _log = log;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _processor.ProcessMessageAsync += HandleMessageAsync;
        _processor.ProcessErrorAsync += HandleErrorAsync;
        await _processor.StartProcessingAsync(ct);
        await Task.Delay(Timeout.Infinite, ct);
    }

    private async Task HandleMessageAsync(ProcessMessageEventArgs args)
    {
        var orderPlaced = args.Message.Body.ToObjectFromJson<OrderPlaced>();
        try
        {
            // Idempotent: use args.Message.MessageId as the natural dedupe key
            await _billing.ChargeAsync(orderPlaced, args.Message.MessageId, args.CancellationToken);
            await args.CompleteMessageAsync(args.Message);
        }
        catch (TransientException ex)
        {
            _log.LogWarning(ex, "Transient failure for {MessageId}, will retry", args.Message.MessageId);
            await args.AbandonMessageAsync(args.Message);   // returns to queue for retry
        }
        catch (PoisonMessageException ex)
        {
            _log.LogError(ex, "Poison message {MessageId}, dead-lettering", args.Message.MessageId);
            await args.DeadLetterMessageAsync(args.Message, ex.Reason, ex.Message);
        }
    }

    private Task HandleErrorAsync(ProcessErrorEventArgs args)
    {
        _log.LogError(args.Exception, "Processor error in {Source}", args.ErrorSource);
        return Task.CompletedTask;
    }

    public override async Task StopAsync(CancellationToken ct)
    {
        await _processor.StopProcessingAsync(ct);
        await _processor.DisposeAsync();
        await base.StopAsync(ct);
    }
}
```

### Sessions for per-key ordering

```csharp
// Producer assigns a session ID
new ServiceBusMessage(payload) { SessionId = customerId.ToString() };

// Consumer processes all messages for a session in FIFO order, one at a time per session
var processor = client.CreateSessionProcessor("customer-events", new ServiceBusSessionProcessorOptions
{
    MaxConcurrentSessions = 8,    // 8 customers processed in parallel, each strictly ordered
    AutoCompleteMessages = false
});
```

### Scheduled messages

```csharp
// Send a "follow-up email" message to be enqueued in 24 hours
var msg = new ServiceBusMessage(payload)
{
    ScheduledEnqueueTime = DateTimeOffset.UtcNow.AddHours(24)
};
await sender.SendMessageAsync(msg);
```

### DI registration with Managed Identity

```csharp
builder.Services.AddAzureClients(clients =>
{
    clients.AddServiceBusClientWithNamespace("contoso-prod.servicebus.windows.net");
    clients.UseCredential(new DefaultAzureCredential());
});

builder.Services.AddHostedService<OrderPlacedConsumer>();
```

### Bicep — namespace, topic, subscription with RBAC

```bicep
resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'contoso-prod-sb'
  location: location
  sku: { name: 'Premium', tier: 'Premium', capacity: 1 }
  properties: {
    minimumTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'   // private endpoint only
    zoneRedundant: true
  }
}

resource topic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: sb
  name: 'order-events'
  properties: {
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    enablePartitioning: false
    maxMessageSizeInKilobytes: 1024   // Premium supports up to 100MB
  }
}

resource subscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: topic
  name: 'billing-orders'
  properties: {
    lockDuration: 'PT1M'
    maxDeliveryCount: 5     // after 5 failures, goes to DLQ
    deadLetteringOnMessageExpiration: true
  }
}

// Grant the Billing service Managed Identity receiver rights only on this subscription
resource receiver 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: subscription
  name: guid(subscription.id, billingPrincipalId, 'ServiceBusDataReceiver')
  properties: {
    principalId: billingPrincipalId
    principalType: 'ServicePrincipal'
    // Azure Service Bus Data Receiver
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0')
  }
}
```

## Advantages

- **Decoupling** — producers and consumers scale and deploy independently.
- **Durable buffering** survives consumer downtime and deploys.
- **Ordering via sessions** at per-key granularity (per-customer, per-tenant).
- **Dead-letter queue** isolates poison messages without blocking the queue.
- **Duplicate detection** with `MessageId` reduces idempotency burden.
- **Scheduled delivery** without an external scheduler.
- **Transactions** can group sends across topics/queues atomically.
- **Premium tier** offers VNet integration, Private Endpoint, JMS 2.0, Geo-DR, dedicated capacity.
- **Managed Identity** support — no shared-access-signature secret to rotate.

## Disadvantages

- **Cost** — Premium starts around $700/month per messaging unit; Standard is per-operation but limited.
- **Latency** — 20–80ms per send/receive; not for sub-millisecond use cases.
- **Throughput ceiling** — even Premium tops out far below Event Hubs (think 1–10K msgs/sec, not millions).
- **Idempotency is still the consumer's job** — duplicate detection only covers the broker's send window (max 7 days).
- **Session processing serializes per session** — high contention on a hot session limits throughput.
- **Complex SDK surface** — `ServiceBusClient`, `ServiceBusSender`, `ServiceBusProcessor`, `ServiceBusReceiver`, sessions, partitions — getting it wrong wastes capacity.
- **Geo-DR is alias-based with manual failover** — not active-active.

## Common Mistakes

### 1. Using `receive-and-delete` for business workflows

```csharp
// ❌ Crash mid-processing = message lost forever
var receiver = client.CreateReceiver("order-events", "billing", new ServiceBusReceiverOptions
{
    ReceiveMode = ServiceBusReceiveMode.ReceiveAndDelete
});
```

**Fix:** Use peek-lock (default) and complete explicitly after successful processing:

```csharp
// ✅ Crash = lock expires, message returns to queue for retry
var processor = client.CreateProcessor("order-events", "billing",
    new ServiceBusProcessorOptions { AutoCompleteMessages = false });
```

### 2. Not implementing idempotency in the consumer

```csharp
// ❌ Same message redelivered = duplicate charge on customer's card
public async Task HandleAsync(OrderPlaced msg)
{
    await _stripe.ChargeAsync(msg.CustomerId, msg.Total);
}
```

**Fix:** Use the `MessageId` (or a domain key) as an idempotency key against a database table:

```csharp
// ✅
public async Task HandleAsync(OrderPlaced msg, string messageId, CancellationToken ct)
{
    if (await _processedRepo.ExistsAsync(messageId, ct)) return;   // already done
    using var tx = await _db.BeginTransactionAsync(ct);
    await _stripe.ChargeAsync(msg.CustomerId, msg.Total, ct);
    await _processedRepo.RecordAsync(messageId, ct);
    await tx.CommitAsync(ct);
}
```

### 3. Long-running handler exceeding lock duration

```csharp
// ❌ Lock is 1 minute, handler runs 3 minutes — broker re-delivers, you process twice
public async Task HandleAsync(...) { await Task.Delay(TimeSpan.FromMinutes(3), ct); }
```

**Fix:** Enable auto-lock-renewal in the processor options:

```csharp
// ✅
new ServiceBusProcessorOptions
{
    MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(10)
}
```

For genuinely long workflows, hand off to a Durable Function or split into steps.

### 4. Using SAS connection strings instead of Managed Identity

```csharp
// ❌ Connection string with shared key sits in config; needs rotation
var client = new ServiceBusClient("Endpoint=sb://...;SharedAccessKey=abc...");
```

**Fix:**

```csharp
// ✅ No secret; least-privilege RBAC role per consumer
var client = new ServiceBusClient(
    "contoso-prod.servicebus.windows.net",
    new DefaultAzureCredential());
```

### 5. Treating sessions as a global throughput knob

```csharp
// ❌ One session ID for everything = serial processing, zero parallelism
new ServiceBusMessage(payload) { SessionId = "default" };
```

**Fix:** Use a meaningful per-entity key — customer ID, order ID — so different entities can be processed in parallel:

```csharp
// ✅ Different customers processed concurrently; one customer's events strictly ordered
new ServiceBusMessage(payload) { SessionId = customerId.ToString() };
```

### 6. Not configuring the dead-letter queue

A subscription with `maxDeliveryCount = 5` and no DLQ monitoring silently buries failures.

**Fix:** Set `maxDeliveryCount` to a sane value (5–10), and alert on DLQ depth:

```kql
// Application Insights / Log Analytics alert
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where MetricName == "DeadletteredMessages"
| where Total > 0
```

Plus a dead-letter consumer or human-driven inspection runbook.

### 7. Sending huge payloads

Standard tier caps at 256KB. Apps that send 5MB JSON blobs get `MessageSizeExceededException`.

**Fix:** Either upgrade to Premium (up to 100MB) or — better — send a small reference message pointing at a blob in Storage (claim-check pattern):

```csharp
// ✅ Upload payload to blob, send message with the URI
await blobClient.UploadAsync(largePayload, ct);
var msg = new ServiceBusMessage(BinaryData.FromObjectAsJson(new { BlobUri = blobClient.Uri }));
```

## Best Practices

- **Use the modern `Azure.Messaging.ServiceBus` SDK**; the old `Microsoft.Azure.ServiceBus` is deprecated.
- **Always Managed Identity**, never SAS keys, in production.
- **Always peek-lock + explicit complete**; never receive-and-delete for business events.
- **Always implement idempotent consumers** — broker-side duplicate detection is a complement, not a substitute.
- **Use `MessageId` as a domain-meaningful idempotency key** (order ID, payment intent ID).
- **Enable duplicate detection** on producer-side topics where event IDs are stable.
- **Set `MaxDeliveryCount`** to 5–10 with **DLQ monitoring and alerting**.
- **Use sessions for per-entity ordering**; never use a global session ID.
- **Enable auto-lock-renewal** in the processor for handlers that can exceed lock duration.
- **Tune `PrefetchCount`** to ~3x `MaxConcurrentCalls` for throughput.
- **Use Premium tier** for VNet integration, Private Endpoint, and Geo-DR in regulated environments.
- **Provision via Bicep** with RBAC scoped per-queue or per-subscription, not namespace-wide.
- **Apply the claim-check pattern** for payloads > 256KB.
- **Alert on `ActiveMessageCount`** rising (consumer lag) and `DeadletteredMessages` (poison).

## Related Concepts

- [architecture/event-driven-architecture.md](architecture/event-driven-architecture.md) — Service Bus is the canonical broker for event-driven .NET.
- [architecture/outbox-pattern.md](architecture/outbox-pattern.md) — reliably publish events via the transactional outbox; Service Bus is the destination.
- [architecture/saga-pattern.md](architecture/saga-pattern.md) — multi-step distributed workflows use Service Bus for choreography.
- [azure/azure-key-vault.md](azure/azure-key-vault.md) — store any remaining shared keys in Key Vault.
- [azure/application-insights.md](azure/application-insights.md) — distributed tracing follows messages across queues automatically.
- [azure/azure-functions.md](azure/azure-functions.md) — Service Bus triggers for serverless consumers.

## Real-World Usage

### Order processing

Checkout publishes `OrderPlaced` to the `order-events` topic. Three subscriptions consume it: `billing-orders` (Stripe capture), `inventory-orders` (stock decrement), `notification-orders` (email + push). Each subscription scales independently. If Stripe is degraded, billing's queue depth grows; checkout is unaffected.

### Per-customer ordered processing

A subscription wallet service processes `WalletDebit`, `WalletCredit`, `WalletReversal` events. Ordering matters per customer. Producers set `SessionId = customerId`, consumer uses `CreateSessionProcessor` with `MaxConcurrentSessions = 16`. Sixteen customers process in parallel; each customer's events are strict FIFO.

### Multi-region failover

A payments platform deploys Premium namespaces in West Europe (primary) and North Europe (secondary) with **Geo-DR alias**. Producers always write to `contoso-payments.servicebus.windows.net` (the alias). On primary loss, ops triggers manual failover; the alias flips to the secondary within minutes. Consumer Managed Identities have RBAC on both regions.

### Scheduled retries

A notification worker that fails to send an email schedules a retry message with `ScheduledEnqueueTime = now + 5 minutes`, attempt count in headers. After 5 failed attempts, the message is dead-lettered. No external scheduler required.

### Outbox pattern

Order service writes the order and an outbox row in the same SQL transaction. A relay Function polls the outbox, publishes to Service Bus with `MessageId = outboxRowId`, and marks the row dispatched. Duplicate detection on the topic absorbs the relay's at-least-once retries.

### CI/CD

A pipeline provisions ephemeral Service Bus namespaces per PR for integration testing. Each PR deploys an isolated namespace; integration tests publish and consume real messages. Cleanup happens automatically when the PR closes.

## Code Example — Before and After

### Before — HTTP coupling, no idempotency, secrets in config

```csharp
// CheckoutController.cs
[HttpPost]
public async Task<IActionResult> Checkout(CheckoutRequest req, CancellationToken ct)
{
    await _orders.SaveAsync(req, ct);
    // ❌ Synchronous call — if Billing is down, checkout fails
    await _billingHttp.PostAsJsonAsync("/charge", req, ct);
    // ❌ Same for Inventory and Notification
    await _inventoryHttp.PostAsJsonAsync("/decrement", req, ct);
    await _notificationHttp.PostAsJsonAsync("/email", req, ct);
    return Ok();
}

// BillingController.cs
[HttpPost("charge")]
public async Task<IActionResult> Charge(ChargeRequest req, CancellationToken ct)
{
    // ❌ Caller retried? You charge twice
    await _stripe.ChargeAsync(req.CustomerId, req.Amount, ct);
    return Ok();
}
```

**Problems:**
- One downstream failure breaks the whole checkout.
- Retries cause duplicate charges.
- No durable buffering during deploys.
- No ordering guarantee for sequential customer events.

### After — Service Bus topic, idempotent consumer, Managed Identity, DLQ

```csharp
// Program.cs — producer side
builder.Services.AddAzureClients(c =>
{
    c.AddServiceBusClientWithNamespace("contoso-prod.servicebus.windows.net");
    c.UseCredential(new DefaultAzureCredential());
});

// CheckoutController.cs
public sealed class CheckoutController : ControllerBase
{
    private readonly ServiceBusSender _sender;
    private readonly IOrdersRepository _orders;

    public CheckoutController(ServiceBusClient sb, IOrdersRepository orders)
    {
        _sender = sb.CreateSender("order-events");
        _orders = orders;
    }

    [HttpPost]
    public async Task<IActionResult> Checkout(CheckoutRequest req, CancellationToken ct)
    {
        var order = await _orders.SaveAsync(req, ct);

        var evt = new OrderPlaced(order.Id, order.CustomerId, order.Total);
        var msg = new ServiceBusMessage(BinaryData.FromObjectAsJson(evt))
        {
            MessageId = order.Id.ToString(),       // duplicate detection key
            SessionId = order.CustomerId.ToString(),  // per-customer ordering
            Subject = nameof(OrderPlaced),
            ContentType = "application/json"
        };
        await _sender.SendMessageAsync(msg, ct);
        return Accepted();
    }
}

// BillingConsumer.cs — Background service
public sealed class BillingConsumer : BackgroundService
{
    private readonly ServiceBusSessionProcessor _processor;
    private readonly IStripeClient _stripe;
    private readonly IProcessedMessages _processed;
    private readonly ILogger<BillingConsumer> _log;

    public BillingConsumer(
        ServiceBusClient client,
        IStripeClient stripe,
        IProcessedMessages processed,
        ILogger<BillingConsumer> log)
    {
        _processor = client.CreateSessionProcessor(
            topicName: "order-events",
            subscriptionName: "billing-orders",
            new ServiceBusSessionProcessorOptions
            {
                MaxConcurrentSessions = 16,
                AutoCompleteMessages = false,
                MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5),
                PrefetchCount = 32
            });
        _stripe = stripe;
        _processed = processed;
        _log = log;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _processor.ProcessMessageAsync += HandleAsync;
        _processor.ProcessErrorAsync += args =>
        {
            _log.LogError(args.Exception, "Processor error: {Source}", args.ErrorSource);
            return Task.CompletedTask;
        };
        await _processor.StartProcessingAsync(ct);
        await Task.Delay(Timeout.Infinite, ct);
    }

    private async Task HandleAsync(ProcessSessionMessageEventArgs args)
    {
        var messageId = args.Message.MessageId;
        if (await _processed.ExistsAsync(messageId, args.CancellationToken))
        {
            // Already charged; just complete the redelivery
            await args.CompleteMessageAsync(args.Message);
            return;
        }

        var evt = args.Message.Body.ToObjectFromJson<OrderPlaced>();

        try
        {
            // ✅ Idempotency key passed to Stripe so even Stripe-side retries are safe
            await _stripe.ChargeAsync(evt.CustomerId, evt.Total, messageId, args.CancellationToken);
            await _processed.RecordAsync(messageId, args.CancellationToken);
            await args.CompleteMessageAsync(args.Message);
            _log.LogInformation("Charged order {OrderId}", evt.OrderId);
        }
        catch (StripeDeclineException ex)
        {
            // Business failure → DLQ for human review, do not retry
            await args.DeadLetterMessageAsync(args.Message, "Declined", ex.DeclineCode);
            _log.LogWarning(ex, "Order {OrderId} declined", evt.OrderId);
        }
        catch (Exception ex)
        {
            // Transient → abandon for retry (up to MaxDeliveryCount)
            _log.LogWarning(ex, "Transient failure for {OrderId}", evt.OrderId);
            await args.AbandonMessageAsync(args.Message);
        }
    }

    public override async Task StopAsync(CancellationToken ct)
    {
        await _processor.StopProcessingAsync(ct);
        await _processor.DisposeAsync();
        await base.StopAsync(ct);
    }
}
```

**Why this is better:**
- Checkout is async — Billing/Inventory/Notification failures don't break the user-facing API.
- Duplicate detection on the broker + database idempotency on the consumer = no double-charges.
- Sessions guarantee per-customer FIFO without serializing the whole queue.
- Managed Identity — no SAS key to leak.
- DLQ isolates poison messages for inspection.
- Stripe idempotency key forwarded from `MessageId` — even Stripe-side retries are safe.

## Interview Questions and Answers

### 1. When would you choose Service Bus over Storage Queues or Event Hubs?

**Why this matters:** Tests fit-for-purpose judgment.

**Answer:** Storage Queues are cheaper and simpler — pick them for trivial work distribution with no ordering, no transactions, no sessions, no pub/sub. Event Hubs is for streaming telemetry at millions per second with replay — analytics, audit logs, IoT. Service Bus sits in the middle: enterprise messaging with sessions, transactions, duplicate detection, dead-lettering, scheduled delivery, and Geo-DR. For "I'm decoupling microservices with business semantics" the answer is almost always Service Bus.

**Trade-off:** Service Bus throughput tops out well below Event Hubs; if you need 100K msgs/sec, you're in Event Hubs territory.

### 2. How do you guarantee exactly-once processing of `OrderPlaced`?

**Why this matters:** This is the single hardest distributed-systems question; honest answers reveal seniority.

**Answer:** Exactly-once delivery doesn't exist in distributed systems — only at-least-once + idempotent consumer. The pattern: producer sets `MessageId = OrderId` (broker dedupe window absorbs publisher retries); broker delivers at-least-once with peek-lock; consumer checks a `processed_messages` table for the `MessageId` inside the same database transaction that performs the side effect (Stripe charge, inventory decrement). If the message is redelivered, the consumer sees the row exists and completes the message without re-doing work. Result: from the customer's perspective, effectively exactly-once.

**Real project:** On a wallet service we caught two duplicate processings in the first week of launch — both prevented by the idempotency table, neither would have been caught by duplicate detection alone (the broker's window had expired).

### 3. A consumer is processing messages 10x slower than they're being published. Walk me through the response.

**Why this matters:** Production capacity and tuning skill.

**Answer:** First, confirm: check `ActiveMessageCount` and queue/subscription depth growing. Then diagnose the consumer: is it CPU-bound (scale out instances), I/O-bound on a downstream (database/HTTP — add concurrency or fix the bottleneck), or single-threaded (raise `MaxConcurrentCalls`/`MaxConcurrentSessions`)? Check `PrefetchCount` — too low and the consumer constantly waits on broker fetches. If the workload allows, increase consumer count via autoscale on queue depth (KEDA in AKS, App Service rules). If the work is genuinely too heavy for synchronous processing, split into a fast intake + slow worker tier.

**Trade-off:** Scaling consumers above ~32 per instance hits AMQP connection limits; prefer more instances over more concurrency per instance.

### 4. Why use sessions, and when do they hurt?

**Why this matters:** Sessions are powerful but easy to misuse.

**Answer:** Sessions guarantee FIFO and exclusive processing per session ID — critical for any per-entity ordering: wallet transactions per customer, state transitions per order, events per tenant. They hurt when (a) you misuse them for global ordering (one session ID = serialization, zero parallelism), or (b) you have a "hot" session that monopolizes a processor slot. The right granularity is per-entity, and the entity choice should match natural concurrency boundaries.

**Real project:** On a wallet system one VIP customer's events flooded one session and starved 15 others. We split into per-(customer, event-type) sessions, restoring parallelism.

### 5. The DLQ is full of messages from yesterday. What do you do?

**Why this matters:** Operational maturity.

**Answer:** DLQ is a runbook trigger, not a quiet alarm. Steps: (1) inspect a sample DLQ message to classify the failure — payload schema, downstream dependency, business rule violation; (2) if it's a transient downstream that's now recovered, re-enqueue the messages from DLQ to the main subscription (a small admin tool or AzCopy-style script); (3) if it's a schema or business issue, fix the consumer and re-process; (4) if it's poison data we'll never process, log and purge. Always document why DLQ happened in an incident review.

**Trade-off:** Auto-replaying DLQ is a footgun — you can re-introduce the same poison repeatedly.

### 6. Compare Standard and Premium tiers.

**Why this matters:** Cost vs. capability trade-off.

**Answer:** Standard is multi-tenant, per-operation billing, 256KB max message, no VNet/Private Endpoint, no Geo-DR pairing. Premium is single-tenant messaging units (1–16), predictable performance, 100MB max message, VNet integration, Private Endpoint, Geo-DR alias, JMS 2.0 support. The crossover is roughly: any regulated workload (PCI, HIPAA), any payload > 256KB, any latency-sensitive use case, or any need for network isolation goes Premium. Standard is fine for non-regulated decoupling of internal services.

**Trade-off:** Premium starts around $700/month per MU; Standard scales to zero idle cost.

### 7. How do you migrate consumers without losing messages during deploy?

**Why this matters:** Zero-downtime deploy literacy.

**Answer:** Service Bus peek-lock + `MaxDeliveryCount` already gives at-least-once delivery across consumer restarts. The new and old consumer versions can run concurrently — both pull from the same subscription, the broker distributes. To guarantee schema compatibility, version events: producers add fields without removing them, consumers ignore unknown fields. For breaking changes, route the new schema to a new subscription with a `Sql` filter (`Subject = 'OrderPlaced.v2'`), let v1 drain, then retire it.

### 8. How does distributed tracing flow through Service Bus?

**Why this matters:** Observability across async boundaries.

**Answer:** The modern `Azure.Messaging.ServiceBus` SDK propagates the W3C Trace Context (`traceparent`) automatically as a message property when an Activity is active at send time. The consumer SDK reads it and starts a child span linked to the producer's. With OpenTelemetry registered in both services, Application Insights shows one end-to-end trace from `POST /api/checkout` through the message hop into the Billing consumer's Stripe call. No manual propagation needed — but a custom client wrapper that skips Activities breaks the chain.

## Summary Checklist

- [ ] I choose Service Bus over Storage Queues or Event Hubs based on ordering, transactions, and throughput needs.
- [ ] I use the modern `Azure.Messaging.ServiceBus` SDK with Managed Identity.
- [ ] I use peek-lock + explicit complete, never receive-and-delete for business events.
- [ ] I implement idempotent consumers with `MessageId`-keyed dedup tables.
- [ ] I use sessions for per-entity FIFO, never a global session ID.
- [ ] I configure `MaxDeliveryCount`, DLQ monitoring, and dead-letter runbooks.
- [ ] I enable auto-lock-renewal and tune `MaxConcurrentCalls` / `PrefetchCount`.
- [ ] I provision via Bicep with per-entity RBAC role assignments.
- [ ] I apply the claim-check pattern for payloads exceeding the message size limit.
- [ ] I validate distributed tracing flows through Service Bus into downstream consumers.
