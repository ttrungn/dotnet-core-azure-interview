# Unit of Work

## Concept Explanation

Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext.

For architecture and design work, focus on the boundary this concept creates, the dependency direction it encourages, and the business problem it protects. The important part is not naming a pattern; it is explaining how the pattern keeps rules, data flow, and change isolated enough for the team to maintain the system.

When discussing it in an interview, connect the concept to a realistic service design decision. Describe what code would live where, how the design would be tested, what operational or delivery risk it reduces, and when the extra structure would become unnecessary ceremony.

## Core Ideas and Examples

Unit of Work coordinates multiple changes and commits them as one operation. In EF Core, `DbContext` commonly acts as the unit of work.

- **Tracks changes:** EF Core tracks added, modified, and deleted entities.
- **Commits once:** `SaveChangesAsync()` persists tracked changes together.
- **Transaction behavior:** Multiple database changes can succeed or fail as a group.
- **Scope:** In ASP.NET Core, a unit of work is often scoped to one request or one application use case.

Example: creating an order and order lines should commit together. If saving an order line fails, the order should not be partially saved.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Unit of Work** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

## Interview Questions and Answers

### 1. What is Unit of Work, and where have you used it in a .NET backend project?

**Answer:** Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext. The value is clearer ownership of rules and change. In an interview, explain the boundary it creates and the risk it reduces, not only the pattern name.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 2. What problem does Unit of Work solve for an API or business application?

**Answer:** Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext. The value is clearer ownership of rules and change. In an interview, explain the boundary it creates and the risk it reduces, not only the pattern name.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 3. How would you implement or apply Unit of Work in an ASP.NET Core service?

**Answer:** Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 4. What are common mistakes developers make with Unit of Work?

**Answer:** Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 5. What trade-offs should a senior developer consider before using Unit of Work?

**Answer:** Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 6. An order API is slow, hard to test, and risky to deploy. How could Unit of Work help, and what would you check first?

**Answer:** Unit of Work coordinates changes as a single transactional operation, commonly represented by EF Core DbContext. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

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

Use an order workflow as the reference point. Explain which code owns business rules, which code orchestrates use cases, which code talks to infrastructure, and how dependencies flow. A strong answer also says when the design is too heavy for the problem.

## Common Mistakes

- Naming a pattern without explaining the boundary or dependency direction.
- Adding layers that do not protect business rules or reduce change risk.
- Putting business rules in controllers, EF queries, or infrastructure adapters.
- Ignoring data consistency, deployment, and operational trade-offs.
- Assuming a pattern is always required for every feature size.

## Summary Checklist

- [ ] I can explain the boundary created by the concept.
- [ ] I can connect the design to a real business workflow.
- [ ] I can describe dependency direction and test strategy.
- [ ] I can explain when the design is unnecessary ceremony.
