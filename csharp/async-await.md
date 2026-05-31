# Async and Await

## Concept Explanation

Async and await let ASP.NET Core release threads while waiting on I/O such as database calls, HTTP APIs, queues, and storage.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Async and Await** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What are `async` and `await`?

**Answer:** `async` marks a method that can perform asynchronous work, and `await` pauses that method until the awaited operation completes. The thread is not blocked while waiting for I/O, so ASP.NET Core can use threads more efficiently.

**Example:** Awaiting a database query or HTTP call lets the server handle other requests while the external operation is pending.

### 2. What problem does async solve in web APIs?

**Answer:** Async helps with I/O-bound work such as database calls, HTTP APIs, storage, and queues. It improves scalability by avoiding blocked request threads. It does not automatically make CPU-heavy code faster.

**Example:** `await _dbContext.Orders.ToListAsync(ct)` is useful. Wrapping CPU-heavy price calculation in `Task.Run()` inside an API is usually not the right fix.

### 3. What should an async method return?

**Answer:** Most async methods return `Task` or `Task<T>`. Use `ValueTask<T>` only for specialized high-performance cases where the method often completes synchronously and the team understands the trade-offs.

**Example:** `Task<OrderDto> GetOrderAsync(Guid id, CancellationToken ct)` is the normal shape for an API query service.

### 4. Why should you avoid `.Result` and `.Wait()`?

**Answer:** `.Result` and `.Wait()` block the current thread and can cause thread starvation or deadlocks in some environments. In ASP.NET Core, the correct approach is usually async all the way through the call chain.

**Example:** A controller should `await _orderService.PlaceOrderAsync(request, ct)` instead of calling `_orderService.PlaceOrderAsync(request, ct).Result`.

### 5. How should cancellation be handled?

**Answer:** Pass `CancellationToken` through async calls so work can stop when the client disconnects, a timeout occurs, or the host is shutting down. Cancellation is especially important for database, HTTP, and queue operations.

**Example:** Pass `ct` from the controller to EF Core: `await _dbContext.SaveChangesAsync(ct)`.

### 6. What are common async mistakes?

**Answer:** Common mistakes include blocking on async calls, forgetting to await a task, ignoring cancellation, using async for CPU-bound work without reason, and using `async void` outside event handlers.

**Example:** If an email task is started but not awaited, the request may return success before the operation fails, and the exception can be lost.

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

An API endpoint places an order, saves it to SQL, and calls a payment provider. Database and payment operations are I/O-bound, so they should be awaited and passed the request `CancellationToken`. That keeps request threads available and lets the operation stop if the client disconnects.

## Common Mistakes

- Blocking async work with `.Result` or `.Wait()`.
- Using `async void` for application logic.
- Forgetting to pass `CancellationToken`.
- Starting tasks without awaiting or intentionally managing them.
- Assuming async makes CPU-bound work faster.

## Summary Checklist

- [ ] I can explain async for I/O-bound work.
- [ ] I can use `Task`, `Task<T>`, and `CancellationToken`.
- [ ] I can avoid blocking on async calls.
- [ ] I can explain why async does not automatically improve CPU-bound work.
