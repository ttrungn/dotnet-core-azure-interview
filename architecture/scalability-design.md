# Scalability Design

## Concept Explanation

Scalability design is the practice of making a system handle increased load without unacceptable latency, cost, or operational risk. It includes application, database, infrastructure, and traffic-management choices.

It exists because systems grow unevenly. Reads may grow faster than writes, one tenant may become noisy, or a marketing campaign may create sudden spikes.

## Core Ideas and Examples

- **Vertical scaling:** Use a larger machine or database tier.
- **Horizontal scaling:** Add more instances.
- **Load balancing:** Spread traffic across healthy instances.
- **Caching:** Avoid repeated expensive work.
- **Sharding:** Split data across partitions or databases.
- **Read replicas:** Scale read-heavy workloads.
- **Partitioning:** Split tables or data by key or range.
- **CDN:** Cache static or edge-friendly content near users.
- **Queue-based load leveling:** Use queues to smooth bursts.
- **Rate limiting:** Protect the system from excessive traffic.

Example: an order history endpoint may scale by adding indexes, pagination, caching, read replicas, and eventually partitioning by customer or date.

## Why This Matters in a .NET Developer Interview

Senior backend interviews use scalability design to test whether you can identify bottlenecks, choose the simplest effective scaling strategy, and avoid premature distributed complexity.

## Architecture

```text
Users
  |
  v
CDN / WAF / Rate Limiter
  |
  v
Load Balancer
  |
  +--> API instance 1
  +--> API instance 2
  +--> API instance N
          |
          +--> Cache
          +--> Primary Database
          +--> Read Replicas
          +--> Queue --> Workers
```

### Internal Workflow

1. Measure current bottleneck.
2. Reduce unnecessary work with better algorithms, query design, and caching.
3. Scale stateless application instances horizontally.
4. Protect dependencies with rate limits and backpressure.
5. Split read and write paths when needed.
6. Partition or shard only when simpler database scaling is insufficient.
7. Monitor latency, throughput, saturation, cost, and error rate.

## Related Concepts

- **Horizontal Scaling:** Add instances behind a load balancer.
- **Vertical Scaling:** Increase CPU, memory, or database capacity.
- **Load Balancing:** Route requests to healthy instances.
- **Caching:** In-memory, distributed, HTTP, CDN, and database cache strategies.
- **Sharding:** Split data by tenant, customer, region, or key range.
- **Read Replicas:** Offload read queries from primary database.
- **Database Partitioning:** Split large tables for manageability and performance.
- **CDN:** Cache static and edge-cacheable content globally.
- **Queue-Based Load Leveling:** Buffer spikes and process asynchronously.
- **Rate Limiting:** Protect service and dependencies from overload.

## Trade-Off Analysis

**Advantages**
- Handles growth and traffic spikes.
- Improves latency when bottlenecks are addressed correctly.
- Supports workload isolation and operational control.

**Disadvantages**
- Adds infrastructure and operational complexity.
- Caching introduces invalidation problems.
- Sharding complicates queries and transactions.
- Read replicas may be stale.

**Alternatives**
- Optimize queries and indexes before adding architecture.
- Use vertical scaling for simpler workloads.
- Use managed platform autoscaling.
- Reduce product requirements that create expensive queries.

**Cost and performance**
- More instances and replicas increase cost.
- Caching can reduce database cost.
- Sharding can improve capacity but increases engineering cost.
- Queues improve throughput but add latency and eventual consistency.

## Failure Scenarios

- **Hot partition:** One shard or tenant gets too much traffic; rebalance or isolate tenant.
- **Cache stampede:** Many requests miss cache at once; use locking, jittered TTLs, or stale-while-revalidate.
- **Read replica lag:** Reads return stale data; route critical reads to primary or expose consistency expectations.
- **Database bottleneck:** Detect high CPU, locks, slow queries, or connection saturation; optimize before scaling blindly.
- **Autoscaling too slow:** Pre-scale for known events and use queues for bursts.
- **Rate limit too strict:** Legitimate users fail; use per-user, per-tenant, or adaptive limits.
- **Queue backlog:** Detect age and depth; scale workers or shed lower-priority work.

## Real-World Examples

- **E-commerce:** CDN product images, cache catalog reads, queue order emails, scale checkout APIs horizontally.
- **Payment systems:** Rate-limit payment attempts, isolate provider calls, shard by merchant or region at high scale.
- **Booking systems:** Use short-lived locks or holds, cache availability carefully, and protect hot events.
- **Banking systems:** Scale reads with replicas but keep ledger writes strongly controlled.
- **Microservices:** Scale stateless services independently and use queues to absorb spikes.

## Interview Questions and Answers

### Beginner Questions

### 1. What is scalability?

**Answer:** Scalability is the ability of a system to handle more load while keeping acceptable latency, reliability, and cost.

**Example:** An API handles more users by adding more stateless instances behind a load balancer.

**Common follow-up questions:** What is the difference between scalability and performance? What do you measure first?

### 2. Horizontal vs vertical scaling: what is the difference?

**Answer:** Vertical scaling makes one machine bigger. Horizontal scaling adds more machines or instances.

**Example:** Upgrade Azure SQL tier for vertical scaling; add more App Service instances for horizontal scaling.

**Common follow-up questions:** Which is simpler? Which has better limits?

### 3. Why is caching useful?

**Answer:** Caching reduces repeated expensive work and lowers latency, but it introduces invalidation and staleness concerns.

**Example:** Cache product catalog data for 5 minutes, but avoid caching payment status if users require immediate accuracy.

**Common follow-up questions:** What can go wrong with caching? What is cache stampede?

### Intermediate Questions

### 4. How do read replicas improve scalability?

**Answer:** They offload read-heavy traffic from the primary database. The trade-off is replication lag.

**Example:** Reporting queries use read replicas while checkout writes continue to the primary database.

**Common follow-up questions:** When should reads go to primary? How do you detect replica lag?

### 5. What is queue-based load leveling?

**Answer:** It buffers work during spikes so workers process at a controlled rate instead of overwhelming downstream systems.

**Example:** Order confirmation emails are queued and processed by workers after checkout succeeds.

**Common follow-up questions:** What happens when the queue backlog grows? Which work should not be queued?

### 6. How does rate limiting help scalability?

**Answer:** Rate limiting protects the system and dependencies from overload by controlling request volume.

**Example:** Limit password reset or payment attempts per user to protect both infrastructure and security.

**Common follow-up questions:** Should limits be per IP, user, tenant, or API key? How do you communicate limits to clients?

### Senior-Level Questions

### 7. When would you shard a database?

**Answer:** Shard only when simpler options such as indexing, query optimization, vertical scaling, read replicas, and partitioning are insufficient. Sharding adds major complexity.

**Example:** A SaaS platform may shard by tenant when one database can no longer handle tenant data volume or isolation requirements.

**Common follow-up questions:** What is a shard key? How do cross-shard queries work?

### 8. How would you handle a flash-sale traffic spike?

**Answer:** Use CDN, pre-scaling, rate limiting, queue-based load leveling, hot-key protection, cache warming, and graceful degradation of non-critical features.

**Example:** Queue purchase attempts, limit per-user requests, cache product details, and keep payment capacity protected.

**Common follow-up questions:** How do you keep inventory correct? How do you prevent overselling?

### 9. How do you choose what to scale first?

**Answer:** Measure bottlenecks first. Look at latency percentiles, throughput, database CPU, slow queries, cache hit rate, queue depth, and saturation before changing architecture.

**Example:** If p95 latency is dominated by one SQL query, adding API instances will not solve the actual bottleneck.

**Common follow-up questions:** What metrics would you inspect? How do you prove the fix worked?

## Senior Engineer Insights

A mid-level engineer often jumps to "add more servers" or "use cache." A senior engineer measures the bottleneck, understands workload shape, chooses the smallest effective scaling move, and explains the consistency and cost trade-offs.

Interviewers are evaluating whether you can scale a production system without creating unnecessary distributed complexity. Common mistakes include premature sharding, caching data with strict freshness requirements, ignoring database bottlenecks, and scaling APIs while the database is the real limit.

