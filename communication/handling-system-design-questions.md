# Handling System Design Questions

## Concept Explanation

System design answers show how you break down requirements, identify risks, choose architecture, and evolve a solution.

For communication topics, focus on how the answer is structured, what evidence supports it, and how clearly it connects technical choices to business impact. The concept matters because interviews often evaluate whether you can explain judgment, not just implementation details.

When practicing it, use a real project-style situation with context, options, decision, trade-off, and result. Keep the language specific enough for engineers while still understandable to product owners, managers, or non-specialist interviewers.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Handling System Design Questions** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Use a clear structure: context, decision, reason, trade-off, result.
- Give concrete examples from backend systems.
- Avoid memorized definitions when the interviewer asks for experience.
- Be honest about uncertainty and explain how you would verify assumptions.

## Interview Questions and Answers

### 1. How should you start a system design answer?

**Answer:** Start by clarifying requirements, constraints, users, traffic, data, and success criteria. Do not jump directly to services or diagrams before the problem is clear.

**Example:** For an order system, ask about order volume, payment flow, inventory consistency, reporting needs, and failure expectations.

### 2. What should you cover after requirements?

**Answer:** Describe the main flows, API boundaries, data model, storage choice, integration points, and failure handling. Keep the design tied to the requirements you clarified.

**Example:** For checkout, cover create order, reserve inventory, capture payment, publish events, and show order status.

### 3. How do you discuss scalability without guessing?

**Answer:** State assumptions and scale the parts that need it. Separate read traffic, write traffic, background processing, database bottlenecks, and external dependencies.

**Example:** If order history reads are heavy, add pagination, indexes, caching, or a read model before splitting the whole system into services.

### 4. How do you handle failure scenarios in system design?

**Answer:** Name the failure, describe user impact, and explain the recovery path. Include retries, idempotency, dead-lettering, timeouts, monitoring, and manual support when relevant.

**Example:** If payment succeeds but order update fails, use idempotency and reconciliation so support can see and repair the inconsistent state.

### 5. How do you explain trade-offs in a design?

**Answer:** Compare options using reliability, complexity, cost, team skill, delivery speed, and future change. Make clear why one option fits the current constraints better.

**Example:** A modular monolith may be better than microservices when one team owns the system and deployment independence is not yet the bottleneck.

### 6. What should you do when the interviewer challenges your design?

**Answer:** Treat the challenge as new input. Revisit assumptions, adjust the design, and explain what changes. Do not defend the first answer blindly.

**Example:** If the interviewer says traffic is ten times higher, discuss database indexing, read models, queues, caching, and autoscaling in that order of likely impact.

## Coding Example

```text
Interviewer: Why did you choose Azure Service Bus instead of calling the billing service directly?

Candidate:
The order API needs to respond quickly and billing can be temporarily unavailable.
I would publish an OrderPlaced event to Service Bus and let billing process it asynchronously.
The trade-off is eventual consistency, so I would expose order status clearly and add retries,
dead-letter monitoring, and idempotency in the billing consumer.
```

## Real-World Scenario

Use a real project-style story as the reference point. Explain context, options, decision, trade-off, and result. A strong answer is specific enough for engineers but clear enough for product owners and interviewers who are checking judgment.

## Common Mistakes

- Giving a memorized answer without context or evidence.
- Using buzzwords instead of explaining the decision and trade-off.
- Skipping the result, lesson, or impact on the team.
- Sounding defensive during code review or design questions.
- Failing to adapt the explanation for technical and non-technical listeners.

## Summary Checklist

- [ ] I can structure answers with context, options, decision, trade-off, and result.
- [ ] I can give concrete examples from backend work.
- [ ] I can explain technical choices in business language.
- [ ] I can answer follow-up questions without becoming vague or defensive.
