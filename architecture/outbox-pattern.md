# Outbox Pattern

## Concept Explanation

The Outbox Pattern reliably publishes messages by storing them in the same database transaction as the business change. A separate publisher reads the outbox table and sends messages to a broker.

It exists because saving database state and publishing a message are two separate operations. If one succeeds and the other fails, services become inconsistent.

## Core Ideas and Examples

- **Outbox table:** Database table that stores messages waiting to be published.
- **Transactional messaging:** Business data and message record are committed atomically.
- **Publisher/relay:** Background process that reads outbox rows and sends them to the broker.
- **At-least-once delivery:** Messages may be sent more than once, so consumers must be idempotent.
- **Inbox pattern:** Consumer-side table that records processed message ids to prevent duplicate effects.
- **CDC:** Change Data Capture can stream outbox rows without polling.

Example: Order service saves a new order and an `OrderPlaced` outbox message in the same transaction. If the process crashes after commit, the publisher can still send the message later.

## Why This Matters in a .NET Developer Interview

Senior interviewers use outbox questions to evaluate transactional thinking. They want to know whether you understand the gap between database commits and message publishing, and why "just publish an event after SaveChanges" is not reliable enough.

## Architecture

```text
API Request
   |
   v
Order Service
   |
   | same DB transaction
   v
+-----------------------+
| Orders table          |
| OutboxMessages table  |
+-----------------------+
   |
   v
Outbox Publisher
   |
   v
Message Broker
   |
   v
Consumers
```

### Internal Workflow

1. Application handles a command such as `PlaceOrder`.
2. It writes business data and an outbox row in the same database transaction.
3. Transaction commits.
4. Background publisher reads unpublished outbox rows.
5. Publisher sends messages to the broker.
6. Publisher marks rows as published or records retry metadata.
7. Consumers process messages idempotently.

## Related Concepts

- **Transactional Messaging:** Keeps state changes and messages synchronized.
- **Inbox Pattern:** Deduplicates received messages.
- **Message Deduplication:** Uses message ids or idempotency keys.
- **Exactly Once vs At Least Once:** Outbox usually gives at-least-once publishing, not true exactly-once end-to-end behavior.
- **CDC:** Debezium, SQL Server CDC, or managed streaming can detect outbox inserts and publish them.
- **Event Publishing:** Outbox is usually used for integration events such as `OrderPlaced`.
- **Sagas:** Outbox often advances saga steps reliably.

## Trade-Off Analysis

**Advantages**
- Prevents losing messages after database commit.
- Prevents publishing events for rolled-back transactions.
- Works well with microservices and event-driven systems.
- Gives retry and audit trail for message publishing.

**Disadvantages**
- Requires extra table, publisher, cleanup, and monitoring.
- Consumers must handle duplicates.
- Outbox table can grow quickly.
- Adds publishing latency.

**Alternatives**
- Publish directly after commit, simpler but less reliable.
- Use distributed transactions, often operationally heavy.
- Use broker transactions when supported, but still may not cover database and broker atomically.
- CDC-based outbox for high-throughput systems.

**Cost and performance**
- Extra writes to the database.
- Background polling can increase load.
- CDC reduces polling cost but adds infrastructure complexity.

## Failure Scenarios

- **Crash after DB commit before publish:** Outbox publisher sends later.
- **Publish succeeds but marking row published fails:** Message may be sent again; consumer must deduplicate.
- **Broker unavailable:** Publisher retries and alerts when backlog grows.
- **Poison message:** Move to failed state or dead-letter workflow after retry threshold.
- **Outbox table grows:** Archive or delete published rows after retention.
- **Schema change breaks consumers:** Version event contracts and support old consumers during migration.

## Real-World Examples

- **E-commerce:** Publish `OrderPlaced` after saving order.
- **Payment systems:** Publish `PaymentCaptured` after recording provider result.
- **Booking systems:** Publish `SeatReserved` after reservation commit.
- **Banking systems:** Publish `TransferRecorded` for ledger downstream processing.
- **Microservices:** Keep Order, Billing, Shipping, and Reporting services synchronized asynchronously.

## Interview Questions and Answers

### Beginner Questions

### 1. What problem does the Outbox Pattern solve?

**Answer:** It solves the atomicity gap between saving data and publishing a message. Both must represent the same business fact.

**Example:** If an order is saved but `OrderPlaced` is not published, inventory and billing may never react.

**Common follow-up questions:** Why not publish after `SaveChanges`? What happens if the process crashes?

### 2. Does outbox guarantee exactly-once delivery?

**Answer:** Not end to end. It usually guarantees the message will be published at least once. Consumers must handle duplicates.

**Example:** If publishing succeeds but marking the outbox row published fails, the publisher may send the same event again.

**Common follow-up questions:** How do consumers deduplicate? What is the inbox pattern?

### 3. What goes into an outbox message?

**Answer:** Message id, type, payload, correlation id, occurred time, publish status, retry count, and error details.

**Example:** `OrderPlaced` includes order id, customer id, total, occurred time, and schema version.

**Common follow-up questions:** Should payload contain full data or just ids? How do you version messages?

### Intermediate Questions

### 4. How would you implement outbox with EF Core?

**Answer:** Add business entities and outbox entities to the same `DbContext`, save them in one transaction, then run a background worker that reads and publishes unpublished rows.

**Example:** `PlaceOrderHandler` adds `Order` and `OutboxMessage(OrderPlaced)` before calling `SaveChangesAsync()`.

**Common follow-up questions:** How do you avoid two workers publishing the same row? How do you batch?

### 5. Polling vs CDC: which is better?

**Answer:** Polling is simpler and often enough. CDC scales better and reduces polling load but adds infrastructure and operational complexity.

**Example:** A small order service can poll every few seconds. A high-volume payment platform may use CDC to stream outbox rows.

**Common follow-up questions:** How do you monitor lag? What happens during schema changes?

### 6. How do you clean up the outbox table?

**Answer:** Keep published messages for a retention period, archive if needed for audit, and delete old rows in batches to avoid database pressure.

**Example:** Keep 7 days of published messages and failed messages until reviewed.

**Common follow-up questions:** What if compliance requires longer retention? What indexes are needed?

### Senior-Level Questions

### 7. What are the hardest production problems with outbox?

**Answer:** Duplicate publishes, publisher concurrency, backlog growth, poison messages, schema evolution, and monitoring publish lag.

**Example:** If Service Bus is down for an hour, the outbox backlog grows and the system needs alerts and catch-up capacity.

**Common follow-up questions:** What metrics would you alert on? How would you throttle catch-up?

### 8. How does outbox interact with sagas?

**Answer:** Outbox reliably publishes the event or command that advances the saga after local state changes commit.

**Example:** Payment service records `PaymentCaptured` and writes an outbox event. The saga only advances when that event is published and consumed.

**Common follow-up questions:** Where is saga state stored? How do you handle duplicate saga events?

### 9. How would you design consumer idempotency?

**Answer:** Store processed message ids in an inbox table or make the business operation naturally idempotent using unique constraints and state checks.

**Example:** Billing consumer records message id before creating an invoice; retries of the same message do not create a second invoice.

**Common follow-up questions:** What if processing fails after inbox insert? Should inbox and business change be one transaction?

## Senior Engineer Insights

A mid-level engineer says outbox prevents lost events. A senior engineer explains that outbox shifts the system to at-least-once delivery and therefore requires deduplication, monitoring, cleanup, and operational repair.

Interviewers are evaluating whether you can reason across database, broker, consumer, and production operations. Common mistakes include claiming exactly-once delivery, ignoring duplicate consumers, leaving outbox rows unbounded, and failing to alert on publish lag.

