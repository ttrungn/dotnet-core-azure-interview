# Records and Immutability

## Concept Explanation

Records and immutable models reduce accidental mutation and make request, response, and value-object code easier to reason about.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Records and Immutability** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is a record in C#?

**Answer:** A record is a type designed for data-focused objects. Records provide value-based equality and concise syntax, which makes them useful for DTOs, messages, and value objects.

**Example:** `public sealed record OrderSummaryDto(Guid Id, string Status, decimal Total);` is clear for an API response.

### 2. What does immutability mean?

**Answer:** Immutability means an object cannot be changed after it is created. This reduces accidental mutation and makes data flow easier to reason about.

**Example:** A `Money` value object with `Amount` and `Currency` should not change after creation. If a calculation is needed, return a new `Money` value.

### 3. Why are records useful for API models?

**Answer:** Records reduce boilerplate and make request or response shapes explicit. They are especially useful when the object is just carrying data and does not need rich behavior.

**Example:** `CreateOrderRequest` can be a record with customer id and line items because it represents input data.

### 4. What is value-based equality?

**Answer:** Value-based equality means two records with the same values are considered equal, even if they are different instances. Classes use reference equality by default unless equality is overridden.

**Example:** Two `Address("Hanoi", "VN")` records compare as equal if their values match.

### 5. When should you avoid records?

**Answer:** Avoid records for entities that have identity and lifecycle. Entities usually need controlled mutation and identity-based equality, not value equality.

**Example:** `Order` should usually be a class because it changes state from draft to confirmed and has identity over time. `OrderLineDto` can be a record.

### 6. What is the `with` expression used for?

**Answer:** A `with` expression creates a copy of a record with selected values changed. It supports immutable update style.

**Example:** `var updated = request with { CouponCode = "SPRING" };` creates a new request value without changing the original.

## Coding Example

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add money with different currencies.");

        return this with { Amount = Amount + other.Amount };
    }
}

public sealed record OrderSummaryDto(Guid Id, Money Total, string Status);
```

## Real-World Scenario

Money is a good immutable value because changing an amount in place can create subtle bugs. Returning a new value after calculation makes pricing code easier to test and reason about.

## Common Mistakes

- Using records for mutable domain entities with identity.
- Assuming a record is deeply immutable when it contains mutable collections.
- Exposing mutable lists from immutable-looking types.
- Using `with` to bypass validation rules.
- Choosing records only for shorter syntax without considering equality behavior.

## Summary Checklist

- [ ] I can explain records and value-based equality.
- [ ] I can use records for DTOs and value objects.
- [ ] I can explain when a class is better than a record.
- [ ] I can avoid mutable collections inside immutable-looking types.
