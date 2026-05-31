# Error Handling and Problem Details

## Concept Explanation

Problem Details gives APIs a consistent RFC 7807 style error shape that clients can parse reliably.

For ASP.NET Core API work, focus on how this concept affects request handling, endpoint behavior, client contracts, security, validation, diagnostics, and operational reliability. A strong explanation should show where it fits in the pipeline or application boundary and how it changes the behavior seen by API consumers.

When discussing it in an interview, use a realistic backend example such as orders, payments, inventory, or user access. Explain the happy path, failure path, testing approach, and trade-off so the answer sounds like production experience rather than documentation recall.

## Core Ideas and Examples

Problem Details is a standard JSON shape for API errors. It helps clients handle failures consistently.

- **Common fields:** `type`, `title`, `status`, `detail`, and `instance`.
- **Validation errors:** Include field-level information when input is invalid.
- **Business conflicts:** Use `409 Conflict` for state conflicts such as duplicate payment capture.
- **Unexpected errors:** Return safe generic messages while logging internal details.
- **Global handling:** Middleware can convert unhandled exceptions into consistent responses.

Example: if an order id does not exist, return `404` with a client-safe Problem Details body, not a stack trace.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Error Handling and Problem Details** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What is Error Handling and Problem Details, and where have you used it in a .NET backend project?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 2. What problem does Error Handling and Problem Details solve for an API or business application?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 3. How would you implement or apply Error Handling and Problem Details in an ASP.NET Core service?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 4. What are common mistakes developers make with Error Handling and Problem Details?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 5. What trade-offs should a senior developer consider before using Error Handling and Problem Details?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 6. An order API is slow, hard to test, and risky to deploy. How could Error Handling and Problem Details help, and what would you check first?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 7. How would you explain Error Handling and Problem Details to a product owner without using unnecessary jargon?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 8. How would you migrate an existing production feature toward better use of Error Handling and Problem Details without stopping delivery?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

## Coding Example

```csharp
[ApiController]
[Route("api/v1/orders")]
public sealed class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<OrderCreatedResponse>> Create(
        CreateOrderRequest request,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        var orderId = await mediator.Send(new CreateOrderCommand(request.CustomerId, request.Lines), cancellationToken);
        return CreatedAtAction(nameof(GetById), new { orderId }, new OrderCreatedResponse(orderId));
    }

    [HttpGet("{orderId:guid}")]
    public async Task<ActionResult<OrderDetailsDto>> GetById(Guid orderId, IMediator mediator)
        => await mediator.Send(new GetOrderDetailsQuery(orderId)) is { } order ? Ok(order) : NotFound();
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
