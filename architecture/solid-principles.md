# SOLID Principles

## Concept Explanation

SOLID principles guide object and service design so code can change without spreading unnecessary impact across the system.

For architecture and design work, focus on the boundary this concept creates, the dependency direction it encourages, and the business problem it protects. The important part is not naming a pattern; it is explaining how the pattern keeps rules, data flow, and change isolated enough for the team to maintain the system.

When discussing it in an interview, connect the concept to a realistic service design decision. Describe what code would live where, how the design would be tested, what operational or delivery risk it reduces, and when the extra structure would become unnecessary ceremony.

## Core Ideas and Examples

SOLID is a set of five object-oriented design principles. The goal is not to memorize the letters; the goal is to design classes and services that can change without breaking unrelated behavior.

- **S - Single Responsibility Principle:** A class should have one main reason to change. Example: `OrderPricingService` calculates totals, while `OrderEmailService` sends confirmation emails. Combining pricing, persistence, and email in one class makes every change risky.
- **O - Open/Closed Principle:** Code should be open for extension but closed for modification. Example: add a new `IPaymentMethod` implementation for bank transfer instead of editing a large `switch` statement every time a payment type is added.
- **L - Liskov Substitution Principle:** A derived type should be usable anywhere its base type is expected without surprising behavior. Example: if `CachedProductRepository` implements `IProductRepository`, callers should not need special rules to use it safely instead of `SqlProductRepository`.
- **I - Interface Segregation Principle:** Prefer small, focused interfaces over large interfaces that force classes to implement methods they do not need. Example: split `IOrderOperations` into `IOrderReader`, `IOrderWriter`, and `IOrderCanceller` when consumers only need part of the behavior.
- **D - Dependency Inversion Principle:** High-level business code should depend on abstractions, not concrete infrastructure. Example: `CheckoutService` depends on `IPaymentGateway`, while `StripePaymentGateway` implements the actual provider call.

In a .NET API, SOLID usually appears in controllers, services, domain models, validators, repositories, and integrations. Good SOLID design should make code easier to test and change; bad SOLID usage creates unnecessary interfaces and layers with no practical benefit.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **SOLID Principles** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is SOLID Principles, and what problem does it solve in a .NET service?

**Answer:** SOLID principles guide object and service design so code can change without spreading unnecessary impact across the system. The value is clearer ownership of rules and change. In an interview, explain the boundary it creates and the risk it reduces, not only the pattern name.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 2. Where should business rules live in an order management system?

**Answer:** Business rules should live in domain models or application use cases where they can be tested without HTTP, SQL, or cloud dependencies. Controllers should translate requests, not own rules.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 3. How do controllers, application services, domain models, and infrastructure depend on each other?

**Answer:** Controllers call application services or handlers. Application code uses domain models and abstractions. Infrastructure implements those abstractions. Dependencies should point inward toward business rules, not outward toward frameworks.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 4. How would you keep Entity Framework Core, MediatR, and Azure SDK usage from leaking everywhere?

**Answer:** Keep framework-specific code at the edges. EF Core belongs in persistence infrastructure, Azure SDK calls belong in adapters, and application code should depend on clear contracts that express business needs.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 5. When does this design become unnecessary ceremony for a small team?

**Answer:** The design is too heavy when the extra layers cost more than the change risk they reduce. For simple CRUD, a clear controller and service may be more maintainable than a full pattern stack.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 6. A single controller validates payment rules, updates inventory, writes SQL, and sends email. How would you refactor it safely?

**Answer:** First add tests around the current behavior. Then move business rules into a domain or application service, isolate persistence and external calls, and release the change without changing the public API contract.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 7. How would you explain SOLID Principles to a product owner without using unnecessary jargon?

**Answer:** I would explain SOLID Principles as a way to reduce release risk by keeping important business behavior easier to change and test.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

### 8. How would you migrate an existing production feature toward better use of SOLID Principles without stopping delivery?

**Answer:** Migrate one workflow at a time. Keep the external API stable, introduce the new structure behind it, add tests around the workflow, and monitor production behavior after release.

**Example:** For example, an order controller calls `ConfirmOrderHandler`, the handler loads the order, the domain object enforces `Confirm()`, and infrastructure saves changes or sends messages.

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
