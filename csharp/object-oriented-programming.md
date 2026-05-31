# Object-Oriented Programming in C#

## Concept Explanation

OOP in C# uses classes, encapsulation, inheritance, and polymorphism to model business behavior while keeping code understandable and testable.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Core Ideas and Examples

Object-oriented programming organizes code around objects that combine state and behavior.

- **Encapsulation:** Hide internal state and expose safe operations. Example: `Order.Confirm()` validates before changing status.
- **Abstraction:** Expose what callers need without exposing implementation details. Example: `IPaymentGateway` hides the payment provider SDK.
- **Inheritance:** Share behavior through a base type when there is a true "is-a" relationship.
- **Polymorphism:** Call the same contract while different implementations behave differently.

Example: checkout can call `IPaymentProcessor.CaptureAsync()` without knowing whether the implementation uses Stripe, Adyen, or a fake test gateway.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Object-Oriented Programming in C#** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

## Interview Questions and Answers

### 1. What is object-oriented programming in C#?

**Answer:** Object-oriented programming organizes code around objects that hold state and behavior. In C#, the main ideas are encapsulation, abstraction, inheritance, and polymorphism. For backend systems, the most valuable part is usually encapsulation: keeping business rules close to the data they protect.

**Example:** An `Order` object should expose `AddLine()` and `Confirm()` methods instead of letting any caller freely modify `Status` and `Lines`.

### 2. What is encapsulation, and why does it matter?

**Answer:** Encapsulation means hiding internal state and exposing controlled operations. It matters because it prevents invalid object states and makes business rules easier to find.

**Example:** If an order cannot be confirmed without lines, the `Order.Confirm()` method can enforce that rule. Without encapsulation, many services may set `Status = Confirmed` directly and bypass the rule.

### 3. When should you use inheritance?

**Answer:** Use inheritance when there is a stable "is-a" relationship and shared behavior truly belongs in a base type. In business applications, composition is often safer because inheritance can create tight coupling and fragile hierarchies.

**Example:** A `CreditCardPaymentProcessor` and `BankTransferPaymentProcessor` may implement the same `IPaymentProcessor` interface instead of inheriting from a complex base class.

### 4. What is polymorphism in a backend service?

**Answer:** Polymorphism lets code work with a common contract while different implementations provide different behavior. It is useful when the caller should not care which concrete strategy is used.

**Example:** A checkout service can call `paymentProcessor.CaptureAsync()` whether the implementation uses Stripe, Adyen, or a test fake.

### 5. How can OOP make code worse?

**Answer:** OOP becomes harmful when developers create deep inheritance trees, over-model simple data, or hide behavior behind too many abstractions. The goal is not to use classes everywhere; the goal is to make rules easier to change safely.

**Example:** A simple request DTO should usually stay as a simple data shape. It does not need inheritance or business methods unless it owns real behavior.

### 6. How would you use OOP in an order workflow?

**Answer:** I would put order-specific rules inside an `Order` domain object and orchestration in an application service. The object protects invariants, while the service handles dependencies such as repositories, payment gateways, and notifications.

**Example:** `Order.Confirm()` validates the order state. `PlaceOrderService` loads inventory, saves the order, and publishes follow-up work.

## Coding Example

```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];

    public Guid Id { get; } = Guid.NewGuid();
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public void AddLine(Guid productId, int quantity, decimal unitPrice)
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

public sealed record OrderLine(Guid ProductId, int Quantity, decimal UnitPrice);

public enum OrderStatus { Draft, Confirmed }
```

## Real-World Scenario

An order should not be confirmed when it has no lines. With OOP, that rule belongs near the order state instead of being repeated across controllers, services, and background jobs. The object exposes useful behavior and protects itself from invalid changes.

## Common Mistakes

- Making every class mutable with public setters.
- Using inheritance when an interface or composition would be simpler.
- Creating "manager" classes that know every business rule.
- Treating domain objects as empty data containers when they should protect important invariants.
- Adding abstractions that do not reduce coupling or clarify behavior.

## Summary Checklist

- [ ] I can explain encapsulation, abstraction, inheritance, and polymorphism.
- [ ] I can show how an object protects business rules.
- [ ] I can explain when composition is better than inheritance.
- [ ] I can avoid over-engineering simple data models.
