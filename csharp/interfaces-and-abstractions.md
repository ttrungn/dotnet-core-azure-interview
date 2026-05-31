# Interfaces and Abstractions

## Concept Explanation

Interfaces describe capabilities and contracts without tying application code to a concrete implementation.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Interfaces and Abstractions** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is an interface in C#?

**Answer:** An interface defines a contract: what operations a type must provide, without saying how they are implemented. It lets application code depend on behavior instead of concrete infrastructure.

**Example:** `IPaymentGateway` can expose `CaptureAsync()` while `StripePaymentGateway` contains the actual HTTP integration details.

### 2. What problem do interfaces solve in backend systems?

**Answer:** Interfaces reduce coupling at important boundaries. They make it easier to replace infrastructure, test business logic, and keep external service details out of application services.

**Example:** A test can use `FakePaymentGateway` to simulate a declined card without calling a real payment provider.

### 3. When should you create an interface?

**Answer:** Create an interface when there is a meaningful boundary, multiple implementations, a need to isolate infrastructure, or a testing benefit. Do not create an interface for every class automatically.

**Example:** An interface for an email sender is useful because production sends real email and tests should not. An interface for a simple `OrderCalculator` may be unnecessary if it has no external dependency and only one implementation.

### 4. What is the difference between abstraction and indirection?

**Answer:** A useful abstraction hides details behind a meaningful concept. Indirection only adds another layer without improving understanding or flexibility.

**Example:** `IInvoicePdfGenerator` is meaningful because callers care about generating invoice PDFs, not the PDF library. `IOrderServiceWrapper` around `OrderService` usually adds noise.

### 5. How can too many interfaces hurt a codebase?

**Answer:** Too many interfaces make navigation harder, hide simple logic, and create fake flexibility. They can also make developers mock everything instead of testing useful behavior.

**Example:** If every class has a matching `IClassName` interface with one implementation, the project has more files but not necessarily better design.

### 6. How would you explain an abstraction decision in an interview?

**Answer:** I would explain the boundary first. If the code talks to a database, message broker, payment provider, or clock, an abstraction can protect business logic from details that change or are hard to test.

**Example:** For order expiration, inject an `IClock` so tests can verify expiration rules without depending on the real current time.

## Coding Example

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> CaptureAsync(Guid orderId, decimal amount, CancellationToken ct);
}

public sealed class CheckoutService
{
    private readonly IPaymentGateway _paymentGateway;

    public CheckoutService(IPaymentGateway paymentGateway)
    {
        _paymentGateway = paymentGateway;
    }

    public Task<PaymentResult> PayAsync(Guid orderId, decimal total, CancellationToken ct) =>
        _paymentGateway.CaptureAsync(orderId, total, ct);
}

public sealed record PaymentResult(bool Succeeded, string? FailureReason);
```

## Real-World Scenario

Checkout should not depend directly on a vendor SDK. An interface gives the application a stable payment contract while the infrastructure layer handles Stripe, Adyen, or another provider. Tests can use a fake gateway to verify checkout behavior without network calls.

## Common Mistakes

- Creating one interface per class without a real boundary.
- Making interfaces too broad, such as one interface with ten unrelated methods.
- Hiding EF Core or HTTP behavior so much that performance and failure behavior become unclear.
- Mocking implementation details instead of testing business outcomes.

## Summary Checklist

- [ ] I can explain interfaces as contracts.
- [ ] I can identify boundaries where abstractions are useful.
- [ ] I can avoid unnecessary one-implementation interfaces.
- [ ] I can give a testing example using a fake dependency.
