# Code Review Practices

## Concept Explanation

Code review improves correctness, maintainability, security, and shared understanding before changes reach production.

For testing and quality work, focus on how this concept improves confidence in behavior, reduces regression risk, and keeps the codebase understandable as requirements change. A strong explanation should connect quality practices to real defects they prevent, not treat them as process rituals.

When discussing it in an interview, describe what you would test or inspect, what feedback the practice gives the team, and how it fits into local development, pull requests, and CI. Also mention the trade-off: quality work should reduce risk without slowing delivery through unnecessary ceremony.

## Core Ideas and Examples

Code review is a quality practice for catching problems and sharing understanding before production.

- **Correctness:** Does the code do the right thing?
- **Security:** Are auth, secrets, input, and data exposure handled safely?
- **Maintainability:** Can another developer understand and change it?
- **Testing:** Are important behaviors and edge cases covered?
- **Operations:** Are logging, monitoring, migration, and rollback concerns addressed?

Example: for a billing change, review rounding, idempotency, failure handling, and tests before discussing naming preferences.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Code Review Practices** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Test behavior, not private implementation details.
- Use unit tests for business decisions and integration tests for real boundaries.
- Keep reviews focused on correctness, maintainability, security, and operational risk.
- Refactor with tests around the behavior you need to preserve.

## Interview Questions and Answers

### 1. What does Code Review Practices improve in a .NET codebase?

**Answer:** Code review improves correctness, maintainability, security, and shared understanding before changes reach production. The value is fast feedback and lower regression risk. Explain what defect it catches and how it helps developers change code safely.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 2. What should be covered by unit tests versus integration tests?

**Answer:** Unit tests should cover isolated business rules and decision logic. Integration tests should cover real boundaries such as database mapping, authentication, HTTP clients, and messaging.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 3. How would you test an order submission workflow that uses a database and a payment gateway?

**Answer:** Test pure business decisions with unit tests, then use integration tests for the database and external boundary. Fake the payment gateway for business behavior, but test persistence with a realistic provider.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 4. How do you avoid brittle tests that only verify implementation details?

**Answer:** Assert observable behavior, not private method calls or internal ordering. Tests should survive refactoring when the external behavior remains correct.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 5. How do you balance code quality work with delivery pressure?

**Answer:** Prioritize tests and quality work around the highest risk: money, security, data consistency, and frequent regressions. Keep low-risk cleanup proportional to the benefit.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 6. A pull request changes billing behavior but has weak tests. What feedback would you give?

**Answer:** I would ask for tests that prove the billing rule, cover edge cases, and protect against regression. I would focus the review on correctness, money impact, and maintainability.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 7. How would you explain Code Review Practices to a product owner without using unnecessary jargon?

**Answer:** I would ask for tests that prove the billing rule, cover edge cases, and protect against regression. I would focus the review on correctness, money impact, and maintainability.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

### 8. How would you migrate an existing production feature toward better use of Code Review Practices without stopping delivery?

**Answer:** I would ask for tests that prove the billing rule, cover edge cases, and protect against regression. I would focus the review on correctness, money impact, and maintainability.

**Example:** For example, a discount calculation should have unit tests for edge cases, while order persistence should have an integration test against a real database provider.

## Coding Example

```csharp
public sealed class PaymentService
{
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(IPaymentGateway gateway, ILogger<PaymentService> logger)
    {
        _gateway = gateway;
        _logger = logger;
    }

    public async Task<PaymentResult> CaptureAsync(Guid invoiceId, decimal amount, CancellationToken ct)
    {
        _logger.LogInformation("Capturing payment for invoice {InvoiceId}", invoiceId);
        return await _gateway.CaptureAsync(invoiceId, amount, ct);
    }
}
```

## Real-World Scenario

Use a business rule or API behavior as the reference point. Explain what confidence the practice gives, what defect it would catch, how it fits local development or CI, and when the practice becomes too expensive for the risk.

## Common Mistakes

- Treating tests or quality practices as rituals instead of feedback mechanisms.
- Testing implementation details instead of observable behavior.
- Ignoring integration points such as database, authentication, queues, and HTTP clients.
- Adding brittle tests that make refactoring harder.
- Skipping quality gates in CI and relying only on manual review.

## Summary Checklist

- [ ] I can explain what risk the practice reduces.
- [ ] I can give a concrete defect or regression it would catch.
- [ ] I can describe how it fits pull requests and CI.
- [ ] I can balance quality confidence against maintenance cost.
