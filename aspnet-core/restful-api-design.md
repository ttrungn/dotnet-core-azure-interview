# RESTful API Design

## Concept Explanation

RESTful API design uses resources, HTTP semantics, predictable URLs, status codes, and representations to create APIs that clients can use reliably.

For ASP.NET Core API work, focus on how this concept affects request handling, endpoint behavior, client contracts, security, validation, diagnostics, and operational reliability. A strong explanation should show where it fits in the pipeline or application boundary and how it changes the behavior seen by API consumers.

When discussing it in an interview, use a realistic backend example such as orders, payments, inventory, or user access. Explain the happy path, failure path, testing approach, and trade-off so the answer sounds like production experience rather than documentation recall.

## Core Ideas and Examples

RESTful API design models API behavior around resources and HTTP semantics.

- **Resources:** Use nouns such as orders, invoices, payments, and shipments.
- **URLs:** Keep routes predictable, such as `/api/v1/orders/{orderId}`.
- **Methods:** Use HTTP methods to express intent.
- **Representations:** Return DTOs designed for clients, not EF entities.
- **Errors:** Use consistent error bodies, such as Problem Details.
- **Idempotency:** Protect retryable operations such as payment capture.

Example: `POST /api/v1/orders` creates an order; `GET /api/v1/orders/{id}` reads it; `POST /api/v1/payments/{id}/capture` may require an idempotency key.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **RESTful API Design** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. How do you decide resource names and URL structure for an order API?

**Answer:** Start from business resources, not controller actions. Use stable nouns such as orders, invoices, payments, and shipments; keep identifiers in the path; and put filtering or paging in query parameters. The URL should describe what the client is working with, while HTTP methods describe the action.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 2. When should an API return 200, 201, 204, 400, 401, 403, 404, 409, and 500?

**Answer:** Status codes should communicate the result without forcing the client to parse text. Use success codes for successful operations, `400` for invalid input, `401` for unauthenticated callers, `403` for forbidden actions, `404` for missing resources, `409` for conflicts, and `500` for unexpected server failures.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 3. How do you model filtering, sorting, pagination, and idempotency in REST APIs?

**Answer:** Filtering, sorting, and pagination belong in explicit query parameters with documented defaults and limits. Idempotency matters for retryable operations, especially payments or order submission, so the client can retry without creating duplicate business transactions.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 4. How should validation errors and business rule errors be represented to clients?

**Answer:** Validation errors should tell the client which input is invalid. Business rule errors should explain the conflict in client-safe language. A consistent Problem Details response helps clients handle failures predictably.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 5. How would you evolve a public API without breaking mobile or partner clients?

**Answer:** Prefer additive changes when possible. If a breaking change is unavoidable, introduce a new version, keep the old contract running for existing clients, and communicate the migration path clearly.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 6. A payment endpoint sometimes creates duplicate charges after client retries. How would you redesign the API contract?

**Answer:** I would make the operation idempotent, require an idempotency key, store the result of the first request, and return the same outcome for safe retries. For payments, this protects customers from duplicate charges.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 7. How would you explain RESTful API Design to a product owner without using unnecessary jargon?

**Answer:** I would explain RESTful API Design as making the API predictable for client teams: fewer integration bugs, clearer errors, safer retries, and lower release risk.

**Example:** For example, `POST /api/v1/orders` creates an order, returns `201 Created`, and uses Problem Details for validation errors so client teams can handle failures consistently.

### 8. How would you migrate an existing production feature toward better use of RESTful API Design without stopping delivery?

**Answer:** I would improve one endpoint at a time, preserve the current contract, add integration tests for current behavior, introduce the improved route or response shape, and monitor client errors after release.

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
