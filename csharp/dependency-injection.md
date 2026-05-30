# Dependency Injection

## Concept Overview

Dependency Injection moves object creation to composition roots so services depend on contracts and can be configured, replaced, and tested cleanly.

In interview answers, keep the explanation practical: name the problem, show how the concept helps, and mention the cost or limitation.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Dependency Injection** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Prefer readable, intention-revealing C# over clever syntax.
- Use modern C# features when they reduce bugs: nullable reference types, records for immutable data, pattern matching, and collection expressions where the team supports them.
- Keep business rules in services or domain models, not hidden inside controllers or EF queries.
- For interviews, explain how the language feature helps reliability, testability, or maintainability.

## Key Interview Questions

| # | Level | Question |
|---|---|---|
| 1 | Basic | What is Dependency Injection, and where have you used it in a .NET backend project? |
| 2 | Basic | What problem does Dependency Injection solve for an API or business application? |
| 3 | Intermediate | How would you implement or apply Dependency Injection in an ASP.NET Core service? |
| 4 | Intermediate | What are common mistakes developers make with Dependency Injection? |
| 5 | Advanced | What trade-offs should a senior developer consider before using Dependency Injection? |
| 6 | Real-world scenario | An order API is slow, hard to test, and risky to deploy. How could Dependency Injection help, and what would you check first? |
| 7 | Advanced | How would you explain Dependency Injection to a product owner without using unnecessary jargon? |
| 8 | Real-world scenario | How would you migrate an existing production feature toward better use of Dependency Injection without stopping delivery? |

## Strong Sample Answers

1. **Definition answer:** DI means a class receives dependencies instead of creating them. This keeps services testable and avoids hard-coded infrastructure.

2. **Lifetime answer:** In ASP.NET Core, transient is for lightweight stateless services, scoped is usually per request and fits EF Core DbContext, and singleton must not depend on scoped services.

3. **Testing answer:** I can replace a payment gateway or notification sender with a fake in tests while keeping the business service unchanged.

4. **Design answer:** DI is not a reason to inject twenty services into one class. Too many dependencies usually means the class has too many responsibilities.

5. **Runtime answer:** I validate service registration at startup and watch for lifetime bugs, especially singleton services capturing request-specific state.

6. **Scenario answer:** If invoice submission directly creates an SMTP client and SQL connection, I would inject abstractions and move configuration to the composition root.

7. **Communication answer:** I would describe Dependency Injection in business terms: it either lowers release risk, makes customer-facing behavior more predictable, or makes failures easier to recover from.

8. **Migration answer:** I would not rewrite everything. I would choose one high-value workflow, add tests, introduce the improved design behind the existing API contract, release incrementally, and monitor behavior after deployment.

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

You are building an order management capability for a commerce platform.

The business requires:
- Customers can place orders and review order status.
- Inventory, payment, and notification workflows must stay reliable.
- Support staff need clear diagnostics when something fails.
- The system must be deployable without long downtime.

For **Dependency Injection**, a strong candidate should connect the concept to this business flow, explain the technical decision, call out the cost of the decision, and describe how they would verify it in production. The interviewer is usually looking for practical reasoning: not just what the concept means, but when it improves maintainability, reliability, performance, or team delivery.

## Common Mistakes

- Memorizing a definition of Dependency Injection but failing to connect it to a production problem.
- Adding unnecessary abstraction before there is a clear reason.
- Ignoring error handling, logging, validation, and testing around the implementation.
- Treating the concept as a rule instead of a design tool.
- Not explaining trade-offs such as complexity, performance, team familiarity, and operational support.

## Follow-Up Questions an Interviewer May Ask

- How would you test this?
- How would you monitor it in production?
- What would make you choose a simpler approach?
- How would this design behave during partial failure?
- How would you explain this decision in a code review?
- What would you change if the traffic increased by ten times?

## Senior-Level Explanation and Trade-Off Discussion

A senior explanation of **Dependency Injection** should balance correctness and cost. The best answer usually says, "I would use this when the business risk or code complexity justifies it." For a 3+ year .NET developer, interviewers expect awareness that every pattern adds maintenance work. The stronger answer describes how the decision affects testing, deployment, observability, data consistency, and future changes.

## Summary Checklist

- [ ] I can define Dependency Injection in simple English.
- [ ] I can give a backend business example using orders, payments, invoices, inventory, or support workflows.
- [ ] I can discuss implementation in ASP.NET Core or Azure when relevant.
- [ ] I can explain common mistakes and how to avoid them.
- [ ] I can describe trade-offs, testing strategy, and production monitoring.
