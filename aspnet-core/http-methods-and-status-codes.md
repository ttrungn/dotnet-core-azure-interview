# HTTP Methods and Status Codes

## Concept Explanation

HTTP methods and status codes communicate intent and outcome between clients and APIs.

For ASP.NET Core API work, focus on how this concept affects request handling, endpoint behavior, client contracts, security, validation, diagnostics, and operational reliability. A strong explanation should show where it fits in the pipeline or application boundary and how it changes the behavior seen by API consumers.

When discussing it in an interview, use a realistic backend example such as orders, payments, inventory, or user access. Explain the happy path, failure path, testing approach, and trade-off so the answer sounds like production experience rather than documentation recall.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **HTTP Methods and Status Codes** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

## Key Interview Questions

| # | Level | Question |
|---|---|---|
| 1 | Basic | How do you decide resource names and URL structure for an order API? |
| 2 | Basic | When should an API return 200, 201, 204, 400, 401, 403, 404, 409, and 500? |
| 3 | Intermediate | How do you model filtering, sorting, pagination, and idempotency in REST APIs? |
| 4 | Intermediate | How should validation errors and business rule errors be represented to clients? |
| 5 | Advanced | How would you evolve a public API without breaking mobile or partner clients? |
| 6 | Real-world scenario | A payment endpoint sometimes creates duplicate charges after client retries. How would you redesign the API contract? |

## Strong Sample Answers

1. **Definition answer:** HTTP Methods and Status Codes is useful when it improves the way a .NET service expresses business behavior, handles change, or protects runtime reliability. I would explain it with an example from an order, invoice, payment, inventory, or support workflow rather than only giving a definition.

2. **Practical value answer:** In a real ASP.NET Core application, HTTP Methods and Status Codes matters because it affects maintainability, testability, production diagnostics, performance, security, or the API contract. I look for the smallest implementation that solves the business problem without adding ceremony.

3. **Implementation answer:** I would start from the use case, define the boundary, keep dependencies explicit through DI, write tests around business behavior, and check the impact on API responses, persistence, logging, and deployment.

4. **Mistake answer:** A common mistake is applying HTTP Methods and Status Codes mechanically. I would avoid adding patterns or infrastructure unless they reduce real risk, duplication, or coupling in the codebase.

5. **Senior answer:** The trade-off is usually between simplicity now and flexibility later. I would consider team experience, operational cost, data consistency, failure handling, and whether the design is easy for another developer to review and support.

6. **Scenario answer:** If an order API is slow or hard to change, I would measure first, identify whether the issue is database access, coupling, deployment, unclear boundaries, or weak observability, then apply HTTP Methods and Status Codes where it directly addresses that bottleneck.

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

You are building an order management capability for a commerce platform.

The business requires:
- Customers can place orders and review order status.
- Inventory, payment, and notification workflows must stay reliable.
- Support staff need clear diagnostics when something fails.
- The system must be deployable without long downtime.

For **HTTP Methods and Status Codes**, a strong candidate should connect the concept to this business flow, explain the technical decision, call out the cost of the decision, and describe how they would verify it in production. The interviewer is usually looking for practical reasoning: not just what the concept means, but when it improves maintainability, reliability, performance, or team delivery.

## Common Mistakes

- Memorizing a definition of HTTP Methods and Status Codes but failing to connect it to a production problem.
- Adding unnecessary abstraction before there is a clear reason.
- Ignoring error handling, logging, validation, and testing around the implementation.
- Treating the concept as a rule instead of a design tool.
- Not explaining trade-offs such as complexity, performance, team familiarity, and operational support.

## Follow-Up Questions an Interviewer May Ask

- How would you test this?
- How would you monitor it in production?
- What would make you choose a simpler approach?
- How would this design behave during partial failure?
- How would you explain this decision in a code review?
- What would you change if the traffic increased by ten times?

## Senior-Level Explanation and Trade-Off Discussion

A senior explanation of **HTTP Methods and Status Codes** should balance correctness and cost. The best answer usually says, "I would use this when the business risk or code complexity justifies it." For a 3+ year .NET developer, interviewers expect awareness that every pattern adds maintenance work. The stronger answer describes how the decision affects testing, deployment, observability, data consistency, and future changes.

## Summary Checklist

- [ ] I can define HTTP Methods and Status Codes in simple English.
- [ ] I can give a backend business example using orders, payments, invoices, inventory, or support workflows.
- [ ] I can discuss implementation in ASP.NET Core or Azure when relevant.
- [ ] I can explain common mistakes and how to avoid them.
- [ ] I can describe trade-offs, testing strategy, and production monitoring.
