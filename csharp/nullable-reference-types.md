# Nullable Reference Types

## Concept Explanation

Nullable reference types make null expectations explicit and help catch missing data handling before runtime.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Core Ideas and Examples

Nullable reference types let C# express whether a reference should be null.

- **Non-nullable:** `string Name` means code expects a value.
- **Nullable:** `string? Email` means null is allowed.
- **Compiler warnings:** The compiler warns when null might be used unsafely.
- **Null-forgiving operator:** `!` suppresses warnings but does not add runtime safety.
- **External input:** API requests and database values still need validation.

Example: `Task<Customer?> FindCustomerAsync(Guid id)` clearly tells callers that the customer may not exist.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Nullable Reference Types** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What are nullable reference types?

**Answer:** Nullable reference types let C# express whether a reference is expected to be null. `string` means the value should not be null, while `string?` means null is allowed.

**Example:** `Customer.Email` might be `string?` if email is optional, but `Customer.Name` should be `string` if every customer must have a name.

### 2. Do nullable reference types prevent all null exceptions?

**Answer:** No. They provide compiler warnings and better intent, but runtime nulls can still happen through external input, databases, serialization, reflection, or suppressed warnings.

**Example:** An API request can still send `null`, so validation is still needed even when the C# model uses non-nullable properties.

### 3. What does the `?` mean on a reference type?

**Answer:** `?` means the reference may be null, so callers must handle that possibility. It is a signal in the contract.

**Example:** `Task<Customer?> FindCustomerAsync(Guid id)` tells callers the customer may not exist, so they should handle the not-found case.

### 4. What does the null-forgiving operator `!` do?

**Answer:** The `!` operator suppresses a compiler warning. It does not add a runtime check and does not make the value non-null. Use it sparingly when you know more than the compiler.

**Example:** `UserName = user.Name!` hides a warning, but if `Name` is actually null, the code can still fail later.

### 5. How do nullable reference types help API design?

**Answer:** They make required and optional fields clearer for developers and reviewers. They also force better handling of not-found data and optional values.

**Example:** A method returning `OrderDto?` communicates "not found" more clearly than returning `OrderDto` and sometimes returning null silently.

### 6. How would you introduce nullable reference types into an existing project?

**Answer:** I would enable them gradually, fix warnings in high-value areas first, and avoid suppressing warnings broadly. DTOs, domain models, and service contracts are good starting points.

**Example:** Start with the order creation flow, mark optional request fields as nullable, add validation, and fix warnings around customer lookup and payment data.

## Coding Example

```csharp
public async Task<OrderDto?> FindOrderAsync(Guid id, CancellationToken ct)
{
    var order = await _dbContext.Orders
        .AsNoTracking()
        .Where(order => order.Id == id)
        .Select(order => new OrderDto(order.Id, order.CustomerEmail, order.Total))
        .SingleOrDefaultAsync(ct);

    return order;
}

public sealed record OrderDto(Guid Id, string? CustomerEmail, decimal Total);
```

## Real-World Scenario

An order lookup may return no data, and a customer email may be optional. Nullable annotations make both facts visible in the method signature, so callers must handle missing orders and optional email addresses deliberately.

## Common Mistakes

- Suppressing warnings with `!` instead of fixing the model.
- Marking everything nullable to silence the compiler.
- Forgetting validation for external input.
- Assuming database columns and C# nullability always match automatically.
- Returning null from methods whose signatures say non-null.

## Summary Checklist

- [ ] I can explain `string` versus `string?`.
- [ ] I can use nullable returns for not-found cases.
- [ ] I can avoid careless use of `!`.
- [ ] I can combine nullability with validation.
