# Dependency Injection

## Concept Explanation

Dependency Injection moves object creation to composition roots so services depend on contracts and can be configured, replaced, and tested cleanly.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Dependency Injection** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is dependency injection?

**Answer:** Dependency injection means a class receives the objects it needs instead of creating them itself. This keeps construction in one place, usually the ASP.NET Core composition root, and keeps business classes focused on behavior.

**Example:** `PaymentService` receives `IPaymentGateway` through its constructor instead of creating `new StripeClient()` inside the method.

### 2. What problem does DI solve?

**Answer:** DI reduces hard-coded dependencies. It makes services easier to test, configure, and replace because the class depends on a contract rather than a concrete implementation.

**Example:** In tests, `PaymentService` can receive a fake gateway that returns a declined payment without calling a real provider.

### 3. What are the common ASP.NET Core service lifetimes?

**Answer:** `Transient` creates a new instance each time, `Scoped` creates one instance per request, and `Singleton` creates one instance for the application lifetime. The lifetime must match the dependency's state and thread-safety needs.

**Example:** EF Core `DbContext` is usually scoped. A singleton service should not capture a scoped `DbContext`, because that can cause lifetime bugs and stale state.

### 4. How do you register dependencies in ASP.NET Core?

**Answer:** Register dependencies in `Program.cs` or extension methods called from startup. Keep registrations close to application composition, not scattered inside business logic.

**Example:** `builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();` tells ASP.NET Core how to build `IPaymentGateway` when a service asks for it.

### 5. What are common DI mistakes?

**Answer:** Common mistakes include injecting too many services into one class, using the wrong lifetime, injecting `IServiceProvider` everywhere, and hiding object creation so much that the dependency graph becomes hard to understand.

**Example:** If a constructor has twelve dependencies, the class probably has too many responsibilities and should be split by use case.

### 6. How would you apply DI to a legacy feature?

**Answer:** I would start with one painful dependency, such as email, payment, file storage, or time. I would introduce a small interface, register the production implementation, add tests with a fake, and avoid rewriting unrelated code.

**Example:** Replace direct `SmtpClient` creation inside `InvoiceService` with an injected `IEmailSender`, then test invoice behavior without sending real email.

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

A checkout workflow needs a payment gateway in production, a fake gateway in tests, and possibly a different gateway in another market. DI keeps that wiring outside the business service. The service says what it needs; startup configuration decides which implementation it receives.

## Common Mistakes

- Using DI to hide poor class design instead of reducing responsibilities.
- Registering scoped dependencies inside singletons.
- Injecting `IServiceProvider` instead of declaring real dependencies.
- Creating interfaces for every class without a testing or boundary reason.
- Forgetting to validate registrations before deployment.

## Summary Checklist

- [ ] I can explain constructor injection.
- [ ] I can compare transient, scoped, and singleton lifetimes.
- [ ] I can register a service in ASP.NET Core.
- [ ] I can identify lifetime and too-many-dependencies problems.
