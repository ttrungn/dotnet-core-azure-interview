# Saga Pattern

## Concept Explanation

The Saga Pattern manages a business transaction that spans multiple services without using one global database transaction. Each step commits locally, then the system either continues to the next step or runs compensating actions when a later step fails.

It exists because microservices usually own separate databases. A checkout flow may touch Order, Payment, Inventory, Shipping, and Notification services. A single ACID transaction across all of them is usually impractical, slow, and operationally fragile.

## Core Ideas and Examples

- **Saga:** A sequence of local transactions that together complete one business process.
- **Local transaction:** A normal transaction inside one service database.
- **Compensation transaction:** A business action that semantically reverses a previous step, such as refunding a payment or releasing reserved inventory.
- **Eventual consistency:** The system may be temporarily inconsistent while the saga is running.
- **Choreography:** Services react to events without a central coordinator.
- **Orchestration:** A coordinator explicitly commands each step.
- **Idempotency:** Repeated commands or events must not create duplicate effects.
- **Correlation id:** Identifier used to connect all messages and logs for one saga instance.

Example: checkout creates an order, reserves inventory, captures payment, creates shipment, and sends confirmation. If payment capture fails after inventory reservation, the saga releases inventory and marks the order as failed.

## Why This Matters in a .NET Developer Interview

Senior backend interviews use sagas to test distributed-system judgment. Interviewers want to know whether you understand transactions, failure recovery, idempotency, eventual consistency, messaging, and operational trade-offs beyond simple CRUD.

Interviewers will listen for:
- Awareness that distributed transactions are expensive and often avoided.
- Clear explanation of compensation instead of rollback.
- Handling of duplicate messages, retries, timeouts, and partial failure.
- Ability to compare orchestration and choreography.
- Practical production monitoring and recovery strategy.

## Architecture

### Orchestrated Saga

```text
Client
  |
  v
Order API
  |
  v
Saga Orchestrator
  |-- ReserveInventory --> Inventory Service
  |-- CapturePayment ----> Payment Service
  |-- CreateShipment ----> Shipping Service
  |-- ConfirmOrder ------> Order Service
```

The orchestrator stores saga state and decides the next command. This is easier to reason about for complex workflows because one component owns the process.

### Choreographed Saga

```text
Order Service -- OrderCreated event --> Inventory Service
Inventory Service -- InventoryReserved event --> Payment Service
Payment Service -- PaymentCaptured event --> Shipping Service
Shipping Service -- ShipmentCreated event --> Notification Service
```

Each service reacts to events and publishes the next event. This reduces central coordination but makes the overall workflow harder to see.

### Typical Data Flow

1. Client starts a business operation, such as checkout.
2. First service saves local state and publishes a command or event.
3. Next service performs its local transaction.
4. Each successful step advances the saga.
5. If a step fails, the saga triggers compensating actions.
6. Saga state ends as completed, failed, compensated, or requiring manual intervention.

## Related Concepts

- **Distributed Transactions:** Transactions across multiple resources. They provide strong consistency but are difficult across independently deployed services.
- **2PC:** Two-Phase Commit coordinates prepare and commit phases across participants. It can block participants and creates coordinator dependency, so it is uncommon in high-scale microservice workflows.
- **Compensation Transactions:** Business reversal actions. Refund payment, release inventory, cancel shipment, or issue credit.
- **Message Queues:** Provide durable asynchronous communication between services.
- **Outbox Pattern:** Often used with sagas so local state changes and event publishing do not get out of sync.
- **Inbox Pattern:** Helps consumers deduplicate messages and process each message safely.
- **Eventual Consistency:** The system converges after all steps and compensations finish.

## Trade-Off Analysis

**Advantages**
- Avoids long distributed locks.
- Fits services with separate databases.
- Makes long-running business workflows possible.
- Supports retries, compensation, and manual recovery.

**Disadvantages**
- Harder than a local transaction.
- Requires careful idempotency and state tracking.
- Temporary inconsistency is visible to users and support teams.
- Compensation may not perfectly reverse real-world actions.

**Alternatives**
- Single database transaction in a modular monolith.
- 2PC for limited enterprise scenarios.
- Manual reconciliation.
- Redesign workflow to avoid cross-service writes.

**Operational complexity**
- Requires durable messages, saga state, monitoring, replay tools, dead-letter handling, and support dashboards.

**Cost and performance**
- Adds message broker cost and more network calls.
- Improves availability by avoiding synchronous dependency chains.
- Increases end-to-end completion latency.

## Failure Scenarios

- **Message lost:** Use outbox, durable broker, retries, and monitoring.
- **Duplicate message:** Use idempotency keys and inbox deduplication.
- **Participant down:** Retry with backoff, pause saga, alert if timeout is exceeded.
- **Compensation fails:** Retry compensation, escalate to manual repair, expose clear support state.
- **Payment captured but order failed:** Refund payment or create manual reconciliation workflow.
- **Inventory reserved but payment failed:** Release inventory.
- **Orchestrator down:** Persist saga state so another instance can resume.
- **Choreography loop:** Use event contracts, correlation ids, and clear state transitions.

## Real-World Examples

- **E-commerce:** Reserve inventory, capture payment, create shipment, send email.
- **Payment systems:** Authorize payment, capture funds, settle transaction, reverse on failure.
- **Booking systems:** Reserve seat, charge customer, issue ticket, release seat if payment fails.
- **Banking systems:** Create transfer request, debit source, credit destination, compensate or reconcile if one side fails.
- **Microservices:** Order, Billing, Inventory, Shipping, and Notification services coordinate through commands and events.

## Interview Questions and Answers

### Beginner Questions

### 1. What is the Saga Pattern?

**Answer:** A saga is a sequence of local transactions across services. Each service commits its own change. If a later step fails, the system runs compensating actions instead of rolling back one global transaction.

**Example:** If payment fails after inventory is reserved, the saga releases inventory and marks the order as payment failed.

**Common follow-up questions:** Why not use one transaction? What is compensation? What does eventual consistency mean?

### 2. Why does the Saga Pattern exist?

**Answer:** It exists because distributed systems usually cannot rely on one database transaction across multiple independently owned services and databases.

**Example:** Order and Payment services may use separate databases. A checkout workflow still needs reliable coordination between them.

**Common follow-up questions:** What problems does 2PC create? When is a local transaction enough?

### 3. What is a compensation transaction?

**Answer:** A compensation transaction is a business action that reverses or offsets a previous committed step.

**Example:** Refund a captured payment, release reserved stock, or cancel a shipment request.

**Common follow-up questions:** Can every action be compensated? What if compensation fails?

### Intermediate Questions

### 4. Choreography vs orchestration: which would you choose?

**Answer:** Use orchestration when the workflow is complex, has many branches, or needs central visibility. Use choreography when the workflow is simple and services can react independently to events.

**Example:** A checkout saga with payment, inventory, shipping, fraud checks, and manual review usually benefits from orchestration.

**Common follow-up questions:** What is the downside of a central orchestrator? How do you debug choreography?

### 5. How do you make a saga reliable?

**Answer:** Persist saga state, use durable messaging, make commands idempotent, use correlation ids, retry transient failures, dead-letter poison messages, and expose operational dashboards.

**Example:** `CapturePaymentCommand` carries an idempotency key so retrying does not charge the customer twice.

**Common follow-up questions:** Where do you store saga state? How do you handle duplicate events?

### 6. How does the Outbox Pattern relate to sagas?

**Answer:** The outbox pattern ensures a service saves its local state and the message that advances the saga in the same database transaction.

**Example:** Order service saves `OrderCreated` and an outbox row together. A publisher later sends that event to the broker.

**Common follow-up questions:** Does outbox guarantee exactly-once delivery? What does the consumer still need to do?

### Senior-Level Questions

### 7. What are the hardest production problems with sagas?

**Answer:** The hardest problems are partial completion, duplicate messages, unclear user state, failed compensation, manual repair, schema evolution, and debugging across services.

**Example:** A customer sees payment captured but order pending. The system needs support-visible saga state and reconciliation.

**Common follow-up questions:** What metrics would you monitor? How would support repair the workflow?

### 8. When should you avoid a saga?

**Answer:** Avoid a saga when a single local transaction or modular monolith boundary would solve the problem more simply. Also avoid it when the business cannot tolerate eventual consistency or compensation.

**Example:** Internal admin CRUD for one database should not use a saga.

**Common follow-up questions:** How would you simplify the design? When is strong consistency required?

### 9. How would you design a payment saga for double-charge safety?

**Answer:** Use idempotency keys, payment authorization before capture where possible, durable saga state, outbox for event publishing, inbox deduplication for consumers, and reconciliation jobs comparing internal state with provider state.

**Example:** A retry of `CapturePayment(orderId, paymentAttemptId)` returns the original result instead of capturing twice.

**Common follow-up questions:** What if the provider times out? What if the provider captured but your service did not receive the response?

## Senior Engineer Insights

A mid-level engineer often explains sagas as "events between services." A senior engineer explains failure states, compensation limits, idempotency, operational visibility, and when a saga is unnecessary.

Interviewers are evaluating whether you can protect customer money, inventory, and support workflows under partial failure. Common mistakes include assuming compensation is the same as rollback, ignoring duplicate messages, hiding eventual consistency from users, and designing a saga where a single transaction would be simpler.

