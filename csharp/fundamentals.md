# C# Fundamentals for Backend Development

## Concept Explanation

Core C# knowledge for backend work includes types, collections, control flow, access modifiers, exceptions, async programming, and how C# features map to maintainable services.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **C# Fundamentals for Backend Development** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What C# fundamentals matter most for backend development?

**Answer:** The most important fundamentals are value types and reference types, null handling, collections, exceptions, async programming, access modifiers, interfaces, and basic object-oriented design. In backend work, these fundamentals affect API correctness, memory behavior, testability, and how clearly business rules are expressed.

**Example:** In an order API, choosing `decimal` for money, `Guid` for identifiers, `DateTimeOffset` for timestamps, and explicit null checks for optional fields prevents many production bugs before architecture patterns are even involved.

### 2. What is the difference between a value type and a reference type?

**Answer:** A value type stores the value directly, while a reference type stores a reference to an object. Structs, `int`, `bool`, and `decimal` are common value types. Classes, strings, arrays, and most service objects are reference types. This matters because assignment, mutation, and nullability behave differently.

**Example:** If two variables point to the same `Customer` object, changing the object through one variable is visible through the other. If two variables hold a `decimal amount`, changing one does not change the other.

### 3. Why is `decimal` usually preferred over `double` for money?

**Answer:** `decimal` is designed for base-10 financial calculations and avoids many binary floating-point rounding surprises. `double` is better for scientific or approximate numeric work, not business amounts that must be exact to cents.

**Example:** A payment service should use `decimal amount` for invoice totals, discounts, and taxes. Using `double` can create rounding differences that are difficult to explain to users and auditors.

### 4. How do access modifiers help maintainable code?

**Answer:** Access modifiers control what other code can use or change. Good use of `private`, `internal`, and `public` keeps implementation details hidden and makes the public contract smaller. A smaller contract is easier to test, review, and safely change.

**Example:** An `Order` class can expose `Confirm()` publicly but keep `Status` setters private, so callers cannot accidentally mark an invalid order as confirmed.

### 5. How would you explain exceptions in basic C# terms?

**Answer:** Exceptions represent failures that interrupt normal execution. Backend code should throw exceptions for unexpected or invalid states, catch them at meaningful boundaries, log enough context, and return safe API responses.

**Example:** A domain service may throw `InvalidOperationException` when confirming an empty order. ASP.NET Core middleware can translate that failure into a controlled error response without exposing stack traces.

### 6. How do you apply C# fundamentals in a real API feature?

**Answer:** Start with clear types, explicit method names, simple control flow, and small classes. Use language features only when they make behavior clearer or safer. Strong fundamentals reduce the need for complicated fixes later.

**Example:** For `PlaceOrder`, define request DTOs, validate required fields, use `decimal` for totals, handle cancellation with `CancellationToken`, and return a result that clearly describes success or failure.

## Coding Example

```csharp
public sealed class CreateOrderRequest
{
    public Guid CustomerId { get; init; }
    public IReadOnlyList<CreateOrderLineRequest> Lines { get; init; } = [];
}

public sealed class CreateOrderLineRequest
{
    public Guid ProductId { get; init; }
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

public static decimal CalculateTotal(CreateOrderRequest request)
{
    if (request.Lines.Count == 0)
        throw new ArgumentException("An order must contain at least one line.");

    return request.Lines.Sum(line => line.Quantity * line.UnitPrice);
}
```

## Real-World Scenario

You are implementing order creation. Good C# fundamentals show up in small decisions: `Guid` for ids, `decimal` for money, `IReadOnlyList<T>` for input that should not be mutated by consumers, and explicit validation before calculation. These choices make the feature easier to review before any framework-specific code is involved.

## Common Mistakes

- Using vague types such as `string` for values that need stronger meaning.
- Ignoring nullability and relying on runtime failures.
- Using public setters everywhere, which allows invalid object state.
- Catching exceptions too broadly and hiding useful diagnostics.
- Choosing clever syntax when simple code would be easier to maintain.

## Summary Checklist

- [ ] I can explain value types, reference types, nullability, collections, and exceptions.
- [ ] I can choose appropriate types for money, identifiers, dates, and optional data.
- [ ] I can use access modifiers to protect object state.
- [ ] I can connect basic C# choices to API correctness and maintainability.
