# Input Validation

## Concept Explanation

Input validation rejects malformed or invalid requests before they corrupt business flow or persistence.

For ASP.NET Core API work, focus on how this concept affects request handling, endpoint behavior, client contracts, security, validation, diagnostics, and operational reliability. A strong explanation should show where it fits in the pipeline or application boundary and how it changes the behavior seen by API consumers.

When discussing it in an interview, use a realistic backend example such as orders, payments, inventory, or user access. Explain the happy path, failure path, testing approach, and trade-off so the answer sounds like production experience rather than documentation recall.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Input Validation** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Keep endpoint contracts explicit: request DTOs, response DTOs, validation, status codes, and error shape.
- Do not expose database entities directly from public APIs.
- Use middleware for cross-cutting behavior such as authentication, correlation IDs, exception handling, and logging.
- Treat API behavior as a contract that client teams depend on.

## Interview Questions and Answers

### 1. What is Input Validation, and where have you used it in a .NET backend project?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 2. What problem does Input Validation solve for an API or business application?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 3. How would you implement or apply Input Validation in an ASP.NET Core service?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 4. What are common mistakes developers make with Input Validation?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 5. What trade-offs should a senior developer consider before using Input Validation?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 6. An order API is slow, hard to test, and risky to deploy. How could Input Validation help, and what would you check first?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

## Coding Example

```csharp
public sealed record RegisterCustomerRequest(string Email, string DisplayName);

public sealed class RegisterCustomerValidator : AbstractValidator<RegisterCustomerRequest>
{
    public RegisterCustomerValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.DisplayName).NotEmpty().MaximumLength(120);
    }
}
```

## Real-World Scenario

Use an order or payment API as the reference point. Explain the request contract, the normal response, the failure response, and the behavior clients can rely on. A strong answer also mentions where the behavior belongs: middleware, endpoint, filter, service, or configuration.

## Common Mistakes

- Explaining the feature without describing the client-visible API behavior.
- Mixing transport concerns, business rules, and persistence in one controller method.
- Returning inconsistent status codes or error shapes.
- Ignoring validation, authentication, authorization, logging, and versioning impact.
- Treating framework defaults as production design decisions.

## Summary Checklist

- [ ] I can explain the API behavior from request to response.
- [ ] I can give status-code and error-response examples.
- [ ] I can say where the implementation belongs in ASP.NET Core.
- [ ] I can describe testing and client compatibility concerns.
