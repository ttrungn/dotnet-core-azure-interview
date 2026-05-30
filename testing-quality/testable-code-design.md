# Testable Code Design

## Concept Overview

Testable code design keeps dependencies explicit, side effects isolated, and business decisions easy to exercise.

In interview answers, keep the explanation practical: name the problem, show how the concept helps, and mention the cost or limitation.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Testable Code Design** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

## Key Interview Questions

| # | Level | Question |
|---|---|---|
| 1 | Basic | What does Testable Code Design improve in a .NET codebase? |
| 2 | Basic | What should be covered by unit tests versus integration tests? |
| 3 | Intermediate | How would you test an order submission workflow that uses a database and a payment gateway? |
| 4 | Intermediate | How do you avoid brittle tests that only verify implementation details? |
| 5 | Advanced | How do you balance code quality work with delivery pressure? |
| 6 | Real-world scenario | A pull request changes billing behavior but has weak tests. What feedback would you give? |

## Strong Sample Answers

1. **Definition answer:** Testable Code Design is useful when it improves the way a .NET service expresses business behavior, handles change, or protects runtime reliability. I would explain it with an example from an order, invoice, payment, inventory, or support workflow rather than only giving a definition.

2. **Practical value answer:** In a real ASP.NET Core application, Testable Code Design matters because it affects maintainability, testability, production diagnostics, performance, security, or the API contract. I look for the smallest implementation that solves the business problem without adding ceremony.

3. **Implementation answer:** I would start from the use case, define the boundary, keep dependencies explicit through DI, write tests around business behavior, and check the impact on API responses, persistence, logging, and deployment.

4. **Mistake answer:** A common mistake is applying Testable Code Design mechanically. I would avoid adding patterns or infrastructure unless they reduce real risk, duplication, or coupling in the codebase.

5. **Senior answer:** The trade-off is usually between simplicity now and flexibility later. I would consider team experience, operational cost, data consistency, failure handling, and whether the design is easy for another developer to review and support.

6. **Scenario answer:** If an order API is slow or hard to change, I would measure first, identify whether the issue is database access, coupling, deployment, unclear boundaries, or weak observability, then apply Testable Code Design where it directly addresses that bottleneck.

## Coding Example

```csharp
[Fact]
public async Task SubmitAsync_rejects_invoice_when_customer_credit_limit_is_exceeded()
{
    var credit = Substitute.For<ICreditLimitService>();
    credit.HasAvailableCreditAsync(customerId, 1200m, Arg.Any<CancellationToken>())
        .Returns(false);

    var service = new InvoiceSubmissionService(credit);

    var result = await service.SubmitAsync(customerId, 1200m, CancellationToken.None);

    result.Status.Should().Be(InvoiceStatus.Rejected);
    result.Reason.Should().Be("Credit limit exceeded");
}
```

## Real-World Scenario

You are building an order management capability for a commerce platform.

The business requires:
- Customers can place orders and review order status.
- Inventory, payment, and notification workflows must stay reliable.
- Support staff need clear diagnostics when something fails.
- The system must be deployable without long downtime.

For **Testable Code Design**, a strong candidate should connect the concept to this business flow, explain the technical decision, call out the cost of the decision, and describe how they would verify it in production. The interviewer is usually looking for practical reasoning: not just what the concept means, but when it improves maintainability, reliability, performance, or team delivery.

## Common Mistakes

- Memorizing a definition of Testable Code Design but failing to connect it to a production problem.
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

A senior explanation of **Testable Code Design** should balance correctness and cost. The best answer usually says, "I would use this when the business risk or code complexity justifies it." For a 3+ year .NET developer, interviewers expect awareness that every pattern adds maintenance work. The stronger answer describes how the decision affects testing, deployment, observability, data consistency, and future changes.

## Summary Checklist

- [ ] I can define Testable Code Design in simple English.
- [ ] I can give a backend business example using orders, payments, invoices, inventory, or support workflows.
- [ ] I can discuss implementation in ASP.NET Core or Azure when relevant.
- [ ] I can explain common mistakes and how to avoid them.
- [ ] I can describe trade-offs, testing strategy, and production monitoring.
