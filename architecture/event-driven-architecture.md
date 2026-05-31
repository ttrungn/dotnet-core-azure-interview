# Event-Driven Architecture

## Concept Explanation

Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency.

For architecture and design work, focus on the boundary this concept creates, the dependency direction it encourages, and the business problem it protects. The important part is not naming a pattern; it is explaining how the pattern keeps rules, data flow, and change isolated enough for the team to maintain the system.

When discussing it in an interview, connect the concept to a realistic service design decision. Describe what code would live where, how the design would be tested, what operational or delivery risk it reduces, and when the extra structure would become unnecessary ceremony.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Event-Driven Architecture** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is Event-Driven Architecture, and where have you used it in a .NET backend project?

**Answer:** Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency. The value is clearer ownership of rules and change. In an interview, explain the boundary it creates and the risk it reduces, not only the pattern name.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 2. What problem does Event-Driven Architecture solve for an API or business application?

**Answer:** Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency. The value is clearer ownership of rules and change. In an interview, explain the boundary it creates and the risk it reduces, not only the pattern name.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 3. How would you implement or apply Event-Driven Architecture in an ASP.NET Core service?

**Answer:** Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 4. What are common mistakes developers make with Event-Driven Architecture?

**Answer:** Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 5. What trade-offs should a senior developer consider before using Event-Driven Architecture?

**Answer:** Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 6. An order API is slow, hard to test, and risky to deploy. How could Event-Driven Architecture help, and what would you check first?

**Answer:** Event-driven architecture lets services publish and react to facts asynchronously, reducing direct coupling at the cost of eventual consistency. In practice, show who owns the business rules, how dependencies flow, how the design is tested, and when the added structure is worth the cost.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 7. How would you explain Event-Driven Architecture to a product owner without using unnecessary jargon?

**Answer:** I would explain Event-Driven Architecture as a way to reduce release risk by keeping important business behavior easier to change and test.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 8. How would you migrate an existing production feature toward better use of Event-Driven Architecture without stopping delivery?

**Answer:** Migrate one workflow at a time. Keep the external API stable, introduce the new structure behind it, add tests around the workflow, and monitor production behavior after release.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

## Coding Example

```csharp
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total);

public sealed class OrderPlacedPublisher
{
    private readonly ServiceBusSender _sender;

    public OrderPlacedPublisher(ServiceBusClient client)
    {
        _sender = client.CreateSender("order-events");
    }

    public async Task PublishAsync(OrderPlaced message, CancellationToken cancellationToken)
    {
        var body = BinaryData.FromObjectAsJson(message);
        await _sender.SendMessageAsync(new ServiceBusMessage(body)
        {
            MessageId = message.OrderId.ToString(),
            Subject = nameof(OrderPlaced)
        }, cancellationToken);
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
