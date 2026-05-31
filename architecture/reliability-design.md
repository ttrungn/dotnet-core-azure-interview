# Reliability Design

## Concept Explanation

Reliability design is the practice of making systems continue to behave acceptably when components are slow, unavailable, overloaded, or failing. It is not one pattern; it is a set of design, operational, and observability decisions.

Reliability exists because production systems fail. Networks timeout, databases slow down, message brokers throttle, dependencies deploy bugs, and users retry requests.

## Core Ideas and Examples

- **Availability:** System is usable when needed.
- **Fault tolerance:** System continues operating despite failures.
- **Graceful degradation:** System offers reduced functionality instead of total outage.
- **Timeout:** Stop waiting after a bounded time.
- **Retry:** Try again for transient failures.
- **Circuit breaker:** Stop calling a failing dependency temporarily.
- **Bulkhead:** Isolate resources so one failure does not exhaust the whole system.
- **Health check:** Report whether the system is alive or ready.
- **Observability:** Logs, metrics, traces, and alerts that explain production behavior.

Example: if the recommendation service fails, checkout should still work. The page can hide recommendations while core ordering remains available.

## Why This Matters in a .NET Developer Interview

Senior backend interviews use reliability design to test production maturity. Interviewers want evidence that you can design beyond the happy path and protect users, data, and operations when dependencies fail.

## Architecture

```text
Client
  |
  v
API Gateway / Load Balancer
  |
  v
Order API
  |-- timeout + retry + circuit breaker --> Payment Provider
  |-- connection pool + health checks ----> Database
  |-- async queue + dead-letter ----------> Background Workers
  |
  v
Observability: logs, metrics, traces, alerts
```

### Internal Workflow

1. Define critical user journeys.
2. Identify dependencies and failure modes.
3. Add timeouts to every remote call.
4. Retry only safe transient failures.
5. Use circuit breakers and bulkheads to prevent cascading failure.
6. Add fallback or graceful degradation where business permits.
7. Monitor SLOs, error budgets, latency, saturation, and dependency failures.
8. Practice incident response and recovery.

## Related Concepts

- **Retry Pattern:** Reattempt transient failures with backoff and jitter.
- **Circuit Breaker:** Opens after repeated failures to protect the system.
- **Timeout Pattern:** Prevents requests from waiting forever.
- **Bulkhead Pattern:** Isolates resources by dependency, tenant, or workload.
- **Health Checks:** Liveness and readiness probes for platforms and load balancers.
- **Failover:** Move traffic to another instance, region, or dependency.
- **Disaster Recovery:** Restore service after major outage or data loss.
- **Fault Tolerance:** Continue operating despite component failure.
- **Monitoring and Tracing:** Detect and diagnose reliability problems.

## Trade-Off Analysis

**Advantages**
- Reduces outage impact.
- Prevents cascading failures.
- Improves customer trust and support visibility.
- Makes incident response faster.

**Disadvantages**
- Adds complexity and configuration.
- Retries can amplify load if poorly designed.
- Fallbacks can hide real failures.
- Multi-region and failover designs increase cost.

**Alternatives**
- Simpler single-region design for low-criticality systems.
- Manual recovery for internal tools.
- Stronger upstream dependency contracts instead of complex local resilience.

**Cost and performance**
- Timeouts and circuit breakers can fail fast and reduce latency during incidents.
- Redundant infrastructure increases cost.
- Observability storage and alerting have ongoing cost.

## Failure Scenarios

- **Dependency timeout:** Detect with latency metrics and traces; recover with timeout, retry, fallback, or queue.
- **Retry storm:** Detect high request volume and dependency saturation; recover with backoff, jitter, and circuit breaker.
- **Database connection exhaustion:** Detect pool saturation; recover by limiting concurrency and optimizing queries.
- **Message backlog:** Detect queue depth and age; scale consumers or pause producers.
- **Region outage:** Fail over if architecture supports it; otherwise communicate and recover from backups.
- **Bad deployment:** Detect error-rate spike; rollback, disable feature flag, or roll forward.
- **Silent failure:** Missing telemetry hides impact; add business metrics and synthetic checks.

## Real-World Examples

- **E-commerce:** Checkout remains available when recommendation or email service fails.
- **Payment systems:** Payment capture uses idempotency, provider timeouts, reconciliation, and alerts.
- **Booking systems:** Seat reservation uses short holds and expiration to recover from abandoned flows.
- **Banking systems:** Transfer systems use audit logs, reconciliation, failover, and strict monitoring.
- **Microservices:** Circuit breakers prevent one failing service from exhausting API threads.

## Interview Questions and Answers

### Beginner Questions

### 1. What is reliability design?

**Answer:** Reliability design makes a system continue to provide acceptable behavior when failures happen.

**Example:** If email sending fails, order creation still succeeds and email is retried later.

**Common follow-up questions:** What is the difference between reliability and availability? What does graceful degradation mean?

### 2. Why are timeouts important?

**Answer:** Timeouts prevent a service from waiting forever on a slow dependency and exhausting threads or connections.

**Example:** Checkout should not wait 60 seconds for a payment provider before failing or retrying safely.

**Common follow-up questions:** How do you choose timeout values? What should happen after timeout?

### 3. What is retry with backoff?

**Answer:** Retry with backoff waits longer between attempts, often with jitter, to avoid overwhelming a struggling dependency.

**Example:** Retry a transient Service Bus send failure after 200ms, then 1s, then 5s with jitter.

**Common follow-up questions:** What should not be retried? How does idempotency affect retries?

### Intermediate Questions

### 4. How does a circuit breaker work?

**Answer:** It tracks failures and temporarily stops calls to a failing dependency. After a cooldown, it allows limited test calls.

**Example:** If fraud service fails repeatedly, open the circuit and route orders to manual review instead of blocking checkout.

**Common follow-up questions:** What are closed, open, and half-open states? What metrics configure it?

### 5. What is the bulkhead pattern?

**Answer:** Bulkheads isolate resources so one failing dependency or workload does not consume everything.

**Example:** Use separate HTTP client connection pools for payment and notification providers so notification failure does not block payments.

**Common follow-up questions:** How do bulkheads apply to thread pools, queues, and tenants?

### 6. What should health checks verify?

**Answer:** Liveness should verify the process is alive. Readiness should verify the instance can serve traffic, including critical dependencies.

**Example:** Readiness fails if the API cannot reach Azure SQL, so the load balancer stops sending traffic to that instance.

**Common follow-up questions:** Should health checks call every dependency? How do you avoid health-check overload?

### Senior-Level Questions

### 7. How do you prevent cascading failure?

**Answer:** Use timeouts, circuit breakers, bulkheads, bounded queues, rate limits, backpressure, and graceful degradation.

**Example:** If Payment API slows down, checkout limits concurrency, fails fast after timeout, and prevents all request threads from blocking.

**Common follow-up questions:** How do you detect cascading failure? What dashboards matter?

### 8. How would you design disaster recovery?

**Answer:** Define RTO and RPO, backup strategy, restore testing, region failover, data replication, runbooks, and communication process.

**Example:** A banking ledger may require low RPO, tested backups, audit logs, and strict reconciliation before reopening writes.

**Common follow-up questions:** What is RTO vs RPO? How often should restore be tested?

### 9. What distinguishes useful observability from noisy monitoring?

**Answer:** Useful observability connects technical signals to user impact and business flows. Noisy monitoring alerts on symptoms that do not require action.

**Example:** Alert on checkout failure rate and payment dependency latency, not every isolated warning log.

**Common follow-up questions:** What SLOs would you define? What traces would you add?

## Senior Engineer Insights

A mid-level engineer often lists retry, circuit breaker, and health checks. A senior engineer explains when each one helps, when it hurts, and how it is monitored in production.

Interviewers are evaluating whether you can reduce blast radius, reason about failure modes, define operational signals, and make cost-aware reliability choices. Common mistakes include retrying non-idempotent operations, missing timeouts, making health checks too heavy, and adding alerts nobody owns.

