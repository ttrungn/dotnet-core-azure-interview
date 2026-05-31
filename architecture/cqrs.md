# CQRS

## Concept Explanation

CQRS separates commands that change state from queries that read state, allowing each side to use the model that fits its job.

For architecture and design work, focus on the boundary this concept creates, the dependency direction it encourages, and the business problem it protects. The important part is not naming a pattern; it is explaining how the pattern keeps rules, data flow, and change isolated enough for the team to maintain the system.

When discussing it in an interview, connect the concept to a realistic service design decision. Describe what code would live where, how the design would be tested, what operational or delivery risk it reduces, and when the extra structure would become unnecessary ceremony.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **CQRS** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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
| 1 | Basic | What is CQRS, and what problem does it solve in a .NET service? |
| 2 | Basic | Where should business rules live in an order management system? |
| 3 | Intermediate | How do controllers, application services, domain models, and infrastructure depend on each other? |
| 4 | Intermediate | How would you keep Entity Framework Core, MediatR, and Azure SDK usage from leaking everywhere? |
| 5 | Advanced | When does this design become unnecessary ceremony for a small team? |
| 6 | Real-world scenario | A single controller validates payment rules, updates inventory, writes SQL, and sends email. How would you refactor it safely? |
| 7 | Advanced | How would you explain CQRS to a product owner without using unnecessary jargon? |
| 8 | Real-world scenario | How would you migrate an existing production feature toward better use of CQRS without stopping delivery? |

## Strong Sample Answers

1. **Definition answer:** CQRS separates write operations from read operations. In an order system, `CreateOrderCommand` enforces rules, while `GetOrderSummaryQuery` can return a read-optimized DTO.

2. **When useful:** I use it when reads and writes have different needs, such as complex order creation rules but simple dashboard queries.

3. **Implementation answer:** In .NET I commonly use MediatR handlers, validators, explicit command/query records, and EF Core. I keep handlers small and focused on one use case.

4. **Not required:** CQRS does not require separate databases or event sourcing. A single database with separate command and query models is often enough.

5. **Trade-off:** It adds more classes and structure. I would avoid it for simple CRUD screens where controllers and services are already clear.

6. **Scenario answer:** If support queries are slow, I might add a read model for order summaries while keeping the write model strict for payment and inventory rules.

7. **Communication answer:** I would describe CQRS in business terms: it either lowers release risk, makes customer-facing behavior more predictable, or makes failures easier to recover from.

8. **Migration answer:** I would not rewrite everything. I would choose one high-value workflow, add tests, introduce the improved design behind the existing API contract, release incrementally, and monitor behavior after deployment.

## Coding Example

```csharp
public sealed record CreateOrderCommand(Guid CustomerId, IReadOnlyList<OrderLineDto> Lines)
    : IRequest<Guid>;

public sealed class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly OrdersDbContext _db;

    public CreateOrderHandler(OrdersDbContext db) => _db = db;

    public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId, command.Lines);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        return order.Id;
    }
}

public sealed record GetOrderSummaryQuery(Guid OrderId) : IRequest<OrderSummaryDto?>;
```

## Real-World Scenario

You are building an order management capability for a commerce platform.

The business requires:
- Customers can place orders and review order status.
- Inventory, payment, and notification workflows must stay reliable.
- Support staff need clear diagnostics when something fails.
- The system must be deployable without long downtime.

For **CQRS**, a strong candidate should connect the concept to this business flow, explain the technical decision, call out the cost of the decision, and describe how they would verify it in production. The interviewer is usually looking for practical reasoning: not just what the concept means, but when it improves maintainability, reliability, performance, or team delivery.

## Common Mistakes

- Memorizing a definition of CQRS but failing to connect it to a production problem.
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

A senior explanation of **CQRS** should balance correctness and cost. The best answer usually says, "I would use this when the business risk or code complexity justifies it." For a 3+ year .NET developer, interviewers expect awareness that every pattern adds maintenance work. The stronger answer describes how the decision affects testing, deployment, observability, data consistency, and future changes.

## Summary Checklist

- [ ] I can define CQRS in simple English.
- [ ] I can give a backend business example using orders, payments, invoices, inventory, or support workflows.
- [ ] I can discuss implementation in ASP.NET Core or Azure when relevant.
- [ ] I can explain common mistakes and how to avoid them.
- [ ] I can describe trade-offs, testing strategy, and production monitoring.
