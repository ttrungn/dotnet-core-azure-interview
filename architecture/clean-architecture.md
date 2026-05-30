# Clean Architecture

## Concept Overview

Clean Architecture organizes code around business rules and use cases, keeping frameworks and infrastructure at the edges.

In interview answers, keep the explanation practical: name the problem, show how the concept helps, and mention the cost or limitation.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Clean Architecture** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Start from business use cases and consistency boundaries, not from patterns.
- Keep framework dependencies at the edges when business rules are important.
- Use abstractions to protect meaningful boundaries, not to hide every concrete class.
- Make design decisions easy to test and easy to explain in a code review.

## Key Interview Questions

| # | Level | Question |
|---|---|---|
| 1 | Basic | What is Clean Architecture, and what problem does it solve in a .NET service? |
| 2 | Basic | Where should business rules live in an order management system? |
| 3 | Intermediate | How do controllers, application services, domain models, and infrastructure depend on each other? |
| 4 | Intermediate | How would you keep Entity Framework Core, MediatR, and Azure SDK usage from leaking everywhere? |
| 5 | Advanced | When does this design become unnecessary ceremony for a small team? |
| 6 | Real-world scenario | A single controller validates payment rules, updates inventory, writes SQL, and sends email. How would you refactor it safely? |
| 7 | Advanced | How would you explain Clean Architecture to a product owner without using unnecessary jargon? |
| 8 | Real-world scenario | How would you migrate an existing production feature toward better use of Clean Architecture without stopping delivery? |

## Strong Sample Answers

1. **Definition answer:** Clean Architecture keeps business rules and use cases independent from frameworks. ASP.NET Core, EF Core, and Azure SDKs should sit at the edge.

2. **Dependency direction:** Controllers call application use cases; application code calls domain logic and abstractions; infrastructure implements those abstractions.

3. **Testing value:** Because business rules do not depend on HTTP or SQL, I can test order confirmation or invoice approval without starting the web host.

4. **Practical boundary:** I do not create many projects automatically. I separate boundaries when it improves clarity and protects important business rules.

5. **Trade-off:** Too many layers can slow delivery. For small features I still keep dependencies clean but avoid unnecessary wrappers.

6. **Refactoring answer:** I would move business rules out of controllers first, add tests, then isolate persistence and external service calls behind interfaces where they create coupling.

7. **Communication answer:** I would describe Clean Architecture in business terms: it either lowers release risk, makes customer-facing behavior more predictable, or makes failures easier to recover from.

8. **Migration answer:** I would not rewrite everything. I would choose one high-value workflow, add tests, introduce the improved design behind the existing API contract, release incrementally, and monitor behavior after deployment.

## Coding Example

```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];
    public Guid Id { get; private set; } = Guid.NewGuid();
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Only draft orders can be changed.");

        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Confirm()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Cannot confirm an empty order.");

        Status = OrderStatus.Confirmed;
    }
}

public sealed record Money(decimal Amount, string Currency);
```

## Real-World Scenario

You are building an order management capability for a commerce platform.

The business requires:
- Customers can place orders and review order status.
- Inventory, payment, and notification workflows must stay reliable.
- Support staff need clear diagnostics when something fails.
- The system must be deployable without long downtime.

For **Clean Architecture**, a strong candidate should connect the concept to this business flow, explain the technical decision, call out the cost of the decision, and describe how they would verify it in production. The interviewer is usually looking for practical reasoning: not just what the concept means, but when it improves maintainability, reliability, performance, or team delivery.

## Common Mistakes

- Memorizing a definition of Clean Architecture but failing to connect it to a production problem.
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

A senior explanation of **Clean Architecture** should balance correctness and cost. The best answer usually says, "I would use this when the business risk or code complexity justifies it." For a 3+ year .NET developer, interviewers expect awareness that every pattern adds maintenance work. The stronger answer describes how the decision affects testing, deployment, observability, data consistency, and future changes.

## Summary Checklist

- [ ] I can define Clean Architecture in simple English.
- [ ] I can give a backend business example using orders, payments, invoices, inventory, or support workflows.
- [ ] I can discuss implementation in ASP.NET Core or Azure when relevant.
- [ ] I can explain common mistakes and how to avoid them.
- [ ] I can describe trade-offs, testing strategy, and production monitoring.
