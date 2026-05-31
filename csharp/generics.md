# Generics

## Concept Explanation

Generics allow reusable type-safe code for common infrastructure such as repositories, result wrappers, validators, and handlers.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Core Ideas and Examples

Generics let you write reusable code while preserving type safety.

- **Generic type:** `List<T>` can be `List<Order>` or `List<Customer>`.
- **Generic method:** A method can operate on different types without casting.
- **Constraint:** `where T : IEntity` limits allowed types.
- **Type safety:** The compiler prevents mixing incompatible types.

Example: `Result<T>` can represent success or failure for `OrderDto`, `PaymentResult`, or `InvoiceDto` without duplicating the wrapper type.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Generics** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What are generics in C#?

**Answer:** Generics let you write reusable code that works with different types while keeping compile-time type safety. They avoid repeated code and reduce casting.

**Example:** `List<Order>` and `List<Customer>` use the same generic collection type, but the compiler still prevents adding a `Customer` to a list of orders.

### 2. Why are generics useful in backend code?

**Answer:** Generics are useful for common structures such as result wrappers, validators, repositories, handlers, and collections where the behavior is shared but the data type changes.

**Example:** `Result<T>` can represent success or failure for `OrderDto`, `InvoiceDto`, or `PaymentResult` without creating separate result classes.

### 3. What is a generic constraint?

**Answer:** A generic constraint limits what types can be used for a generic parameter. Constraints allow the generic code to rely on specific capabilities, such as requiring a class, a parameterless constructor, or an interface.

**Example:** `where TEntity : IEntity` lets a repository access `entity.Id` because every allowed type must implement `IEntity`.

### 4. When should you avoid generics?

**Answer:** Avoid generics when they make the code harder to understand or hide important business differences. Reuse is not valuable if the shared abstraction becomes vague.

**Example:** A generic `Process<T>()` method for orders, refunds, and shipments may hide three different business workflows that deserve separate names.

### 5. What is the risk of a generic repository?

**Answer:** A generic repository can hide useful EF Core features and force every aggregate into the same persistence shape. It is helpful only when the shared operations are truly meaningful.

**Example:** `IRepository<T>.GetAll()` can encourage loading too much data. A specific `IOrderRepository.GetPendingOrdersForCustomer()` expresses intent better.

### 6. How would you explain generics in an interview?

**Answer:** I would say generics provide reusable, type-safe building blocks. Then I would give a practical example and mention that generic abstractions should not erase important business meaning.

**Example:** `IValidator<TRequest>` works well because validation has a shared shape, but each request still has its own validator rules.

## Coding Example

```csharp
public sealed record Result<T>(bool Succeeded, T? Value, string? Error)
{
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}

public async Task<Result<OrderDto>> GetOrderAsync(Guid id, CancellationToken ct)
{
    var order = await _orders.FindAsync(id, ct);

    if (order is null)
    {
        return Result<OrderDto>.Failure("Order was not found.");
    }

    return Result<OrderDto>.Success(new OrderDto(order.Id, order.Total));
}

public sealed record OrderDto(Guid Id, decimal Total);
```

## Real-World Scenario

Several query methods need to return either data or a failure reason. A generic `Result<T>` avoids repeating the same success/failure structure for every DTO while keeping the actual returned type specific.

## Common Mistakes

- Creating generic code before duplication is real.
- Using vague names like `TData` and `TThing`.
- Hiding domain-specific behavior behind generic methods.
- Building generic repositories that fight EF Core.
- Adding constraints without understanding why they are needed.

## Summary Checklist

- [ ] I can explain type-safe reuse.
- [ ] I can use common generic types such as `List<T>` and `Task<T>`.
- [ ] I can explain generic constraints.
- [ ] I can recognize when generics hide business meaning.
