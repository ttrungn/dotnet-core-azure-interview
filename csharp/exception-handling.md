# Exception Handling

## Concept Explanation

Exception handling protects application boundaries, keeps diagnostics useful, and prevents internal details from leaking to API consumers.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Core Ideas and Examples

Exception handling controls how unexpected failures are reported, logged, translated, and recovered from.

- **Throw:** Use when the current operation cannot continue safely.
- **Catch:** Catch where you can add context, recover, retry, or translate the error.
- **Rethrow:** Use `throw;` to preserve stack trace.
- **Global handling:** ASP.NET Core middleware can translate unhandled exceptions to safe API responses.
- **Validation:** Expected invalid input is often better represented as validation results, not exceptions.

Example: throw when an order is in an impossible state, but return validation errors when a request is missing required fields.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Exception Handling** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is exception handling?

**Answer:** Exception handling is how C# reports and manages unexpected failures. Code can throw an exception when it cannot continue normally, and higher-level code can catch it where there is enough context to handle, log, or translate it.

**Example:** A service may throw when a payment provider is unavailable. API middleware can log the exception and return a safe `500` response.

### 2. When should you throw an exception?

**Answer:** Throw an exception for unexpected states, programming errors, or failures that stop the current operation. Do not use exceptions for normal control flow that happens often and can be represented clearly.

**Example:** Throwing when an order total is negative is reasonable. Returning a validation result for a missing required field is often better than throwing.

### 3. Where should exceptions be caught in an ASP.NET Core app?

**Answer:** Catch exceptions at meaningful boundaries, such as middleware, background worker loops, or integration adapters. Avoid catching exceptions too low unless you can add context, retry, compensate, or translate the error.

**Example:** A repository should usually let a database exception bubble up. Global exception middleware can log it with request context and return Problem Details.

### 4. What is the difference between logging and handling an exception?

**Answer:** Logging records the failure. Handling means the code takes a meaningful action, such as retrying, returning a controlled response, using a fallback, or stopping safely. Catching only to log and rethrow can create duplicate logs.

**Example:** A message consumer may catch an exception, log the message id, and let the queue retry or move the message to a dead-letter queue.

### 5. How should custom exceptions be used?

**Answer:** Use custom exceptions when they represent a meaningful category that callers can handle differently. Avoid creating custom exceptions for every small failure.

**Example:** `PaymentDeclinedException` may be useful if the API should return a specific business response. `OrderServiceException` for every failure is usually too vague.

### 6. What should you avoid exposing to API clients?

**Answer:** Do not expose stack traces, connection strings, SQL details, secrets, or internal service names. Return a clear client-safe error and keep detailed diagnostics in logs and telemetry.

**Example:** Return `400` for validation errors, `404` for missing resources, and a generic `500` for unexpected failures while logging the exception internally.

## Coding Example

```csharp
public sealed class OrdersController : ControllerBase
{
    private readonly IOrderService _orders;

    public OrdersController(IOrderService orders)
    {
        _orders = orders;
    }

    [HttpPost("{id:guid}/confirm")]
    public async Task<IActionResult> ConfirmAsync(Guid id, CancellationToken ct)
    {
        try
        {
            await _orders.ConfirmAsync(id, ct);
            return NoContent();
        }
        catch (OrderNotFoundException)
        {
            return NotFound();
        }
    }
}
```

## Real-World Scenario

Order confirmation can fail because the order does not exist, the state is invalid, or the database is unavailable. Expected business failures should become clear client responses. Unexpected infrastructure failures should be logged with context and returned as safe generic errors.

## Common Mistakes

- Swallowing exceptions and hiding failures.
- Catching `Exception` everywhere without a recovery plan.
- Exposing internal exception details to clients.
- Using exceptions for expected validation flow.
- Logging the same exception repeatedly at multiple layers.

## Summary Checklist

- [ ] I can explain throw, catch, and rethrow behavior.
- [ ] I can choose where exceptions should be translated to API responses.
- [ ] I can avoid leaking internal details.
- [ ] I can distinguish validation results from unexpected exceptions.
