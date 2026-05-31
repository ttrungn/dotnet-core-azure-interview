# Code Review Discussion Questions

## Concept Explanation

Code review discussions test whether you can critique constructively, prioritize risks, and defend design choices with evidence.

For communication topics, focus on how the answer is structured, what evidence supports it, and how clearly it connects technical choices to business impact. The concept matters because interviews often evaluate whether you can explain judgment, not just implementation details.

When practicing it, use a real project-style situation with context, options, decision, trade-off, and result. Keep the language specific enough for engineers while still understandable to product owners, managers, or non-specialist interviewers.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Code Review Discussion Questions** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What should you focus on first in a code review?

**Answer:** Prioritize correctness, security, data consistency, API contract changes, performance risk, and test coverage before style preferences.

**Example:** In a billing change, review money calculations, rounding, idempotency, failure handling, and regression tests before naming comments.

### 2. How do you give review feedback constructively?

**Answer:** Be specific, explain the risk, and suggest a concrete improvement. Ask questions when intent is unclear instead of assuming the author is wrong.

**Example:** Say: "Could this create duplicate payments on retry? Should we add an idempotency key test?"

### 3. How do you respond when someone challenges your code?

**Answer:** Clarify the concern, check the evidence, and update the code or explain the trade-off. A good response focuses on the code and risk, not personal defense.

**Example:** If someone questions a query, inspect generated SQL together and decide whether projection or indexing is needed.

### 4. What review comments are low value?

**Answer:** Low-value comments are vague, preference-only, or unrelated to the risk of the change. Style should be automated where possible.

**Example:** Instead of debating formatting manually, enforce it with analyzers or `dotnet format` and spend review time on behavior.

### 5. How do you review tests?

**Answer:** Check whether tests protect important behavior, include edge cases, avoid brittle implementation details, and run reliably in CI.

**Example:** For order discounts, tests should cover zero quantity, expired coupon, maximum discount, and rounding behavior.

### 6. How do you discuss design in a review without blocking delivery unnecessarily?

**Answer:** Separate must-fix risks from follow-up improvements. If the current design is safe enough, propose a smaller change now and create a follow-up task.

**Example:** Approve a localized service extraction for this release, then schedule a broader boundary cleanup after tests are in place.

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
