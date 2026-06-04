# HTTP Methods and Status Codes

## What It Is

HTTP methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) describe the **intent** of the request — read, create, replace, partially update, remove. HTTP status codes (`2xx`, `3xx`, `4xx`, `5xx`) describe the **outcome** so the client can react without parsing prose.

Together they form the wire-level contract of every REST API. A payment service that returns `200 OK` for both success and validation failure forces clients to inspect the body for hidden errors — and that is how duplicate charges happen.

```csharp
// Bad: everything is 200 OK, client cannot distinguish outcomes.
[HttpPost("payments")]
public IActionResult Capture(CapturePaymentRequest request)
    => Ok(new { success = false, message = "Card declined" });

// Good: status code carries the outcome; body carries the detail.
[HttpPost("payments")]
public IActionResult Capture(CapturePaymentRequest request)
    => UnprocessableEntity(new ProblemDetails { Title = "Card declined", Status = 422 });
```

## Why It Exists

Before HTTP semantics were enforced, APIs returned `200 OK` for everything and embedded `"status": "error"` in the body. That meant every client — mobile app, partner integration, internal worker — had to write custom parsers, retry the wrong requests, and miss real failures. The IETF defined method semantics in [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) so that any HTTP-aware infrastructure — proxies, load balancers, retry policies, caches, monitoring — can reason about traffic uniformly.

The key property is **idempotency**: a `GET`, `PUT`, or `DELETE` can be retried by Polly, Azure Front Door, or a mobile client without side effects. `POST` cannot.

## When To Use It

**Use HTTP methods and status codes properly when:**
- Building any REST or RPC-over-HTTP API consumed by clients you do not control.
- Exposing public partner APIs where retries, caching, and CDNs are involved.
- Designing webhooks where the sender expects a specific `2xx` to stop retrying.
- Working with API gateways (Azure API Management, YARP) that route by method.

**Do not stretch HTTP semantics when:**
- Doing pure RPC between trusted internal services — gRPC is cleaner.
- Building event-driven flows — use Service Bus or Event Grid, not `POST /events`.
- Streaming — use SignalR or WebSockets, not long-polling `GET`.

## Why It Is Important

Status codes drive **production behavior** far beyond the response body:

- **Polly retry policies** retry on `5xx` and `408`/`429`, never on `4xx`. Returning `500` for a validation failure causes thundering-herd retries.
- **Azure Front Door and CDNs** cache `GET` responses. A `GET` with side effects gets cached and never reaches your server again.
- **Application Insights** computes failure rate from `5xx`. Returning `200` for errors hides outages from your dashboards and alerts.
- **Load balancer health probes** use `200` to keep instances in rotation.
- **Mobile clients** retry idempotent methods on flaky networks — non-idempotent `POST` retries on payments create duplicate charges.

## How It's Used in C# / .NET

ASP.NET Core surfaces this through attribute routing on controllers and Minimal APIs:

```csharp
// Controller style
[ApiController]
[Route("api/v1/orders")]
public sealed class OrdersController : ControllerBase
{
    [HttpGet("{orderId:guid}")]                     // GET — safe, idempotent, cacheable
    [HttpPost]                                       // POST — non-idempotent, creates
    [HttpPut("{orderId:guid}")]                      // PUT — idempotent full replace
    [HttpPatch("{orderId:guid}")]                    // PATCH — partial update
    [HttpDelete("{orderId:guid}")]                   // DELETE — idempotent removal
    public IActionResult Method() => Ok();
}

// Minimal API style
app.MapGet("/orders/{id:guid}", GetOrder);
app.MapPost("/orders", CreateOrder);
```

Common result helpers:

| Helper | Status | When |
| --- | --- | --- |
| `Ok(value)` | 200 | Successful read with body |
| `CreatedAtAction(...)` | 201 | Created — sets `Location` header |
| `NoContent()` | 204 | Successful command with no body |
| `BadRequest(problem)` | 400 | Malformed request |
| `Unauthorized()` | 401 | Missing/invalid token |
| `Forbid()` | 403 | Authenticated but not allowed |
| `NotFound()` | 404 | Resource does not exist |
| `Conflict(problem)` | 409 | State conflict — duplicate, version mismatch |
| `UnprocessableEntity(problem)` | 422 | Syntactically valid but semantically invalid |
| `StatusCode(429)` | 429 | Rate limited |
| `Problem(...)` | 500 | Unhandled server failure |

### Full Status Code Reference

| Code | Name | Meaning | Backend Example |
| --- | --- | --- | --- |
| **200** | OK | Success with body | `GET /orders/{id}` returns order JSON |
| **201** | Created | New resource created | `POST /orders` returns new order + `Location` header |
| **202** | Accepted | Queued for async processing | `POST /reports` queued to Azure Service Bus |
| **204** | No Content | Success, nothing to return | `DELETE /carts/{id}` |
| **301** | Moved Permanently | Permanent redirect | Old `/v1/users` → `/v2/users` |
| **304** | Not Modified | Client cache still valid (`ETag` matched) | `GET` with `If-None-Match` |
| **400** | Bad Request | Malformed JSON, missing required field | Body fails model binding |
| **401** | Unauthorized | No/invalid JWT — *not authenticated* | Token expired |
| **403** | Forbidden | Authenticated, but not allowed | Customer trying to access admin endpoint |
| **404** | Not Found | Resource ID does not exist | `GET /orders/{unknownId}` |
| **405** | Method Not Allowed | Wrong verb for the route | `DELETE /reports` when only `GET` exists |
| **409** | Conflict | State conflict | Duplicate idempotency key, ETag version mismatch |
| **410** | Gone | Resource permanently removed | Deprecated API endpoint |
| **415** | Unsupported Media Type | Wrong `Content-Type` | Client sent XML, API only accepts JSON |
| **422** | Unprocessable Entity | Valid syntax, invalid semantics | "Order total cannot be negative" |
| **429** | Too Many Requests | Rate limit exceeded | Per-tenant throttle hit; include `Retry-After` |
| **500** | Internal Server Error | Unhandled exception | NullReferenceException leaked through middleware |
| **502** | Bad Gateway | Upstream gave invalid response | Payment gateway returned malformed JSON |
| **503** | Service Unavailable | Temporarily down for maintenance | Deployment in progress; include `Retry-After` |
| **504** | Gateway Timeout | Upstream did not respond | Stripe call exceeded 30s timeout |

### Idempotency and Safety

| Method | Safe (no side effects) | Idempotent (same result on retry) |
| --- | --- | --- |
| GET | Yes | Yes |
| HEAD | Yes | Yes |
| OPTIONS | Yes | Yes |
| PUT | No | Yes |
| DELETE | No | Yes |
| POST | No | **No** — unless you implement an idempotency key |
| PATCH | No | **No** — unless the patch is absolute, not delta |

For payments and order creation, add an `Idempotency-Key` header and a request-deduplication table (Stripe pattern) so retries are safe.

## Advantages

- **Universal contract** — every HTTP client, proxy, gateway, and monitoring tool understands the codes.
- **Free behavior** — retries, caching, conditional requests, and health checks work without custom logic.
- **Observability** — dashboards group by 2xx/4xx/5xx automatically; SLO/SLI math is trivial.
- **Security clarity** — `401` vs `403` lets clients show the right UX (re-login vs "not allowed").

## Disadvantages

- **Ambiguity in the middle** — `400` vs `422`, `401` vs `403`, `409` vs `422` are debated; teams must pick conventions.
- **PATCH is under-specified** — JSON Merge Patch (RFC 7396) and JSON Patch (RFC 6902) behave differently.
- **Cannot represent partial success** — bulk endpoints need bespoke body shapes.
- **Status codes are coarse** — clients usually still need an error code in the body for branching logic.

## Common Mistakes

### 1. Returning 200 for business failures

```csharp
// Bad
[HttpPost("payments")]
public IActionResult Capture(...) =>
    Ok(new { success = false, error = "card_declined" });
```

```csharp
// Fix — status code carries the outcome
[HttpPost("payments")]
public IActionResult Capture(...) =>
    UnprocessableEntity(new ProblemDetails
    {
        Type = "https://api.contoso.com/errors/card-declined",
        Title = "Card declined",
        Status = StatusCodes.Status422UnprocessableEntity
    });
```

### 2. Confusing 401 and 403

```csharp
// Bad — returning 401 when user is logged in but lacks role
if (!User.IsInRole("Admin")) return Unauthorized();
```

```csharp
// Fix — 403 means authenticated but not permitted
if (!User.IsInRole("Admin")) return Forbid();
```

`401` means "I do not know who you are — present credentials." `403` means "I know who you are, and you cannot do this."

### 3. Using POST for retryable money movement without idempotency

```csharp
// Bad — mobile retry creates duplicate charges
[HttpPost("transfers")]
public async Task<IActionResult> Transfer(TransferRequest req, CancellationToken ct)
{
    await _bank.TransferAsync(req, ct);
    return Ok();
}
```

```csharp
// Fix — require Idempotency-Key, dedupe at the boundary
[HttpPost("transfers")]
public async Task<IActionResult> Transfer(
    [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
    TransferRequest req,
    CancellationToken ct)
{
    var existing = await _idempotency.TryGetAsync(idempotencyKey, ct);
    if (existing is not null) return Ok(existing); // safe replay

    var result = await _bank.TransferAsync(req, ct);
    await _idempotency.StoreAsync(idempotencyKey, result, ct);
    return Ok(result);
}
```

### 4. Returning 500 for validation failures

```csharp
// Bad — Polly will retry, dashboards will page the on-call
if (request.Amount < 0) throw new Exception("Negative amount");
```

```csharp
// Fix — 422 is a client error; not retryable
if (request.Amount < 0)
    return UnprocessableEntity(new ProblemDetails
    {
        Title = "Amount must be positive",
        Status = StatusCodes.Status422UnprocessableEntity
    });
```

### 5. PUT used for partial update

```csharp
// Bad — client sends only { "email": "..." }, server wipes other fields
[HttpPut("{id:guid}")]
public IActionResult Update(Guid id, UserDto dto) => _users.Replace(id, dto);
```

```csharp
// Fix — PATCH for partial, PUT for full replacement
[HttpPatch("{id:guid}")]
public IActionResult Patch(Guid id, JsonPatchDocument<UserDto> patch) { ... }
```

### 6. Forgetting Retry-After on 429 / 503

```csharp
// Bad — client retries immediately and gets throttled again
return StatusCode(429);
```

```csharp
// Fix — tell the client when to retry
Response.Headers.RetryAfter = "30";
return StatusCode(StatusCodes.Status429TooManyRequests);
```

## Best Practices

- Pick a status-code policy per service and document it in the OpenAPI spec.
- Return `ProblemDetails` (RFC 7807) for every `4xx`/`5xx` — see [aspnet-core/error-handling-and-problem-details.md](error-handling-and-problem-details.md).
- Use `400` for **shape** errors (bad JSON, missing field) and `422` for **rule** errors (negative amount, expired card).
- Always include `Location` on `201 Created`.
- Set `Retry-After` on every `429` and `503`.
- Use `ETag` + `If-Match` for optimistic concurrency on `PUT`/`PATCH` → `412 Precondition Failed`.
- Require `Idempotency-Key` on `POST` for money movement, order creation, and webhook senders.
- Never include stack traces or SQL in error bodies — log internally, return safe text externally.
- Treat `5xx` rate as an SLI; alert when it exceeds your error budget.

## Related Concepts

- [aspnet-core/restful-api-design.md](restful-api-design.md)
- [aspnet-core/error-handling-and-problem-details.md](error-handling-and-problem-details.md)
- [aspnet-core/input-validation.md](input-validation.md)
- [aspnet-core/api-versioning.md](api-versioning.md)
- [aspnet-core/jwt-authentication.md](jwt-authentication.md)
- [architecture/reliability-design.md](../architecture/reliability-design.md)

## Real-World Usage

### ASP.NET Core APIs

Controllers and Minimal APIs both expose `Results.*` and `TypedResults.*` helpers in .NET 8+. Prefer `TypedResults` for compile-time status code accuracy and better OpenAPI generation.

### Azure Functions

HTTP-triggered functions return `IActionResult` or `HttpResponseData`. The same conventions apply — APIM in front of Functions will retry `5xx` and short-circuit `4xx`.

### Azure API Management

APIM policies can rewrite status codes, attach `Retry-After`, and enforce rate limits — returning `429` consistently across all backends.

### Payment & Order Systems

Stripe, Adyen, and PayPal all require `Idempotency-Key` on `POST /charges`. Mirror that on internal payment APIs to survive client retries during network partitions.

### Webhook Endpoints

Service Bus, Event Grid, GitHub, and Stripe webhooks retry on `5xx` and stop on `2xx`. Return `200` immediately after persisting the event; do the work asynchronously.

## Code Example — Before and After

### Before — opaque, retry-unsafe payment endpoint

```csharp
[ApiController]
[Route("api/payments")]
public sealed class PaymentsController : ControllerBase
{
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentsController> _logger;

    public PaymentsController(IPaymentGateway gateway, ILogger<PaymentsController> logger)
    {
        _gateway = gateway;
        _logger = logger;
    }

    [HttpPost("capture")]
    public async Task<IActionResult> Capture(CaptureRequest request)
    {
        try
        {
            if (request.Amount <= 0)
                return Ok(new { success = false, message = "Invalid amount" });

            var result = await _gateway.CaptureAsync(request.InvoiceId, request.Amount);

            if (!result.Approved)
                return Ok(new { success = false, message = result.DeclineReason });

            return Ok(new { success = true, transactionId = result.TransactionId });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Capture failed");
            return Ok(new { success = false, message = ex.Message });
        }
    }
}
```

Problems: every outcome is `200`. Polly will not retry transient gateway failures. Dashboards show 100% success. Mobile retries cause duplicate charges.

### After — honest status codes, idempotent, observable

```csharp
namespace Contoso.Payments.Api.Controllers;

[ApiController]
[Route("api/v1/payments")]
[Produces("application/json")]
public sealed class PaymentsController : ControllerBase
{
    private readonly IPaymentGateway _gateway;
    private readonly IIdempotencyStore _idempotency;
    private readonly ILogger<PaymentsController> _logger;

    public PaymentsController(
        IPaymentGateway gateway,
        IIdempotencyStore idempotency,
        ILogger<PaymentsController> logger)
    {
        _gateway = gateway;
        _idempotency = idempotency;
        _logger = logger;
    }

    /// <summary>Capture a previously authorized payment.</summary>
    /// <response code="201">Capture succeeded.</response>
    /// <response code="409">Idempotency key reused with a different body.</response>
    /// <response code="422">Card declined or business rule failed.</response>
    /// <response code="502">Upstream payment gateway failure.</response>
    [HttpPost("capture")]
    [ProducesResponseType(typeof(CaptureResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status422UnprocessableEntity)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status502BadGateway)]
    public async Task<IActionResult> Capture(
        [FromHeader(Name = "Idempotency-Key"), Required] string idempotencyKey,
        [FromBody] CaptureRequest request,
        CancellationToken cancellationToken)
    {
        var cached = await _idempotency.TryGetAsync(idempotencyKey, request, cancellationToken);
        if (cached is { Conflict: true })
        {
            return Conflict(new ProblemDetails
            {
                Title = "Idempotency key reused with different payload",
                Status = StatusCodes.Status409Conflict
            });
        }
        if (cached is { Response: { } prior })
        {
            return CreatedAtAction(nameof(GetCapture), new { id = prior.TransactionId }, prior);
        }

        var result = await _gateway.CaptureAsync(request.InvoiceId, request.Amount, cancellationToken);

        return result.Outcome switch
        {
            CaptureOutcome.Approved => await StoreAndReturnCreated(idempotencyKey, request, result, cancellationToken),
            CaptureOutcome.Declined => UnprocessableEntity(new ProblemDetails
            {
                Type = "https://api.contoso.com/errors/card-declined",
                Title = "Card declined",
                Detail = result.DeclineReason,
                Status = StatusCodes.Status422UnprocessableEntity
            }),
            CaptureOutcome.GatewayTimeout => StatusCode(StatusCodes.Status504GatewayTimeout, new ProblemDetails
            {
                Title = "Payment gateway timed out",
                Status = StatusCodes.Status504GatewayTimeout
            }),
            _ => StatusCode(StatusCodes.Status502BadGateway, new ProblemDetails
            {
                Title = "Payment gateway error",
                Status = StatusCodes.Status502BadGateway
            })
        };
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(CaptureResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetCapture(Guid id, CancellationToken cancellationToken)
        => await _gateway.FindAsync(id, cancellationToken) is { } found
            ? Ok(found)
            : NotFound();

    private async Task<IActionResult> StoreAndReturnCreated(
        string key, CaptureRequest request, CaptureResult result, CancellationToken ct)
    {
        var response = new CaptureResponse(result.TransactionId, request.Amount);
        await _idempotency.StoreAsync(key, request, response, ct);
        _logger.LogInformation("Captured {Amount} for invoice {InvoiceId}", request.Amount, request.InvoiceId);
        return CreatedAtAction(nameof(GetCapture), new { id = result.TransactionId }, response);
    }
}
```

Now: `5xx` triggers Polly retries on upstream failures only; `4xx` stops retries; idempotency keys make `POST` safe to replay; Application Insights paints an accurate failure-rate chart.

## Interview Questions and Answers

### 1. Why does returning 500 for a validation failure cause production incidents?

**Why this matters:** It tests whether the candidate understands how status codes feed retry policies, alerts, and SLOs — not just response shapes.

**Answer:** `5xx` is "server's fault, please retry." Polly, Azure Front Door, and mobile SDKs all retry on `5xx`. A validation bug returning `500` triggers a retry storm — the same invalid request hammers the API thousands of times per second, paging the on-call and burning the error budget. Validation failures are the **client's** fault and must be `4xx` (typically `400` for shape, `422` for rules).

**Trade-off:** Some teams collapse all client errors into `400` for simplicity. That's fine if clients only care about retry behavior, but loses signal for analytics ("how often is the card declined?").

**Real project:** A fintech API was returning `500` when KYC data was missing. After a mobile release, 18% of users hit it; Polly's three retries amplified to 54% extra load and a P1 incident.

### 2. When do you use 400 vs 422?

**Why this matters:** This is the most common point of disagreement on API teams; a clear answer shows the candidate has actually shipped public APIs.

**Answer:** `400 Bad Request` means the request is **malformed** — invalid JSON, missing required field, wrong type. The server cannot even parse it. `422 Unprocessable Entity` means the request is syntactically valid but **violates a business rule** — negative amount, expired card, order total below minimum. Clients react differently: `400` usually points to a bug in the caller; `422` points to user input.

**Trade-off:** REST purists insist on `422`; some frameworks (older ASP.NET Web API) only emit `400`. Pick one and document it; consistency beats theological correctness.

### 3. Why is the difference between 401 and 403 a security issue?

**Why this matters:** Mixing them up leaks information and breaks client UX.

**Answer:** `401 Unauthorized` means "I do not know who you are — present credentials." The client should redirect to login or refresh the token. `403 Forbidden` means "I know who you are, you just cannot do this." The client should show "Access denied," not retry login. Returning `404` instead of `403` is a deliberate pattern in some products to avoid revealing that a resource exists — that's a security choice, not a bug.

**Real project:** An admin dashboard returned `401` for non-admins, sending users into infinite login loops because their token was perfectly valid.

### 4. How do you make POST idempotent?

**Why this matters:** Critical for payments, order creation, and any retryable mutation.

**Answer:** Require an `Idempotency-Key` HTTP header (UUID generated by the client). Persist `{key → response}` in a deduplication table (Redis or SQL) with a TTL of 24h. On replay with the same key + same body, return the stored response. On replay with the same key + different body, return `409 Conflict` — this catches client bugs. Stripe popularized this pattern; copy it.

**Trade-off:** Adds a write and a read to every `POST`. For high-throughput endpoints, partition the dedupe store and keep TTLs short.

**Real project:** A payment gateway saw 0.3% duplicate captures during a mobile network outage. Adding `Idempotency-Key` drove duplicates to zero with ~2ms added latency.

### 5. When should an API return 202 Accepted instead of 200/201?

**Why this matters:** Tests understanding of async workflows.

**Answer:** `202 Accepted` means "I received your request, I will process it later." Use it when you publish to Azure Service Bus, Event Grid, or a background queue. Include a `Location` header pointing to a status endpoint the client can poll, or return a job ID. Never return `202` for synchronous work — clients will poll forever.

**Real project:** A report-generation API switched from synchronous `200` (with 90-second waits and timeouts) to `202 Accepted` + Service Bus + status polling. P99 latency dropped from 90s to 80ms.

### 6. Why must GET never have side effects?

**Why this matters:** A surprising number of "weird production bugs" trace back to side-effectful `GET`s.

**Answer:** Anything on the path of a `GET` may be cached, prefetched, or retried by browsers, CDNs, Azure Front Door, proxies, mobile SDKs, search engine crawlers, and link-preview bots. `GET /orders/{id}/cancel` will be cancelled by Slack's link unfurler when someone pastes the URL. Use `POST` or `DELETE` for state changes.

**Real project:** An e-commerce team built `GET /admin/refund/{orderId}` for convenience. Googlebot indexed the admin dashboard and refunded thousands of orders before anyone noticed.

### 7. How do you communicate rate limiting to clients?

**Why this matters:** Tests practical knowledge of production-grade APIs.

**Answer:** Return `429 Too Many Requests` with a `Retry-After` header (seconds or HTTP-date). Optionally include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` so well-behaved clients can self-throttle. ASP.NET Core 7+ has built-in `RateLimiter` middleware that emits these headers automatically.

**Trade-off:** `Retry-After` only works if clients honor it. Combine with per-tenant quotas at API Management for defense in depth.

### 8. How do you evolve a public API's status code contract without breaking clients?

**Why this matters:** Backward compatibility is a senior-level concern.

**Answer:** Status codes are part of the contract — changing `200` to `204` for the same endpoint breaks clients that check for a body. Add new codes only in additive ways: a new error path may return a new code, but existing paths must keep returning what they did. For larger changes, version the API (`/api/v2/...`) — see [aspnet-core/api-versioning.md](api-versioning.md).

**Real project:** A partner API team changed `200` with empty array to `204 No Content` for "no results." Three partner integrations broke overnight because they assumed `204 == "endpoint deprecated."`

## Summary Checklist

- [ ] I can name the safe and idempotent methods and explain why it matters for retries.
- [ ] I can choose between `400`, `401`, `403`, `404`, `409`, `422`, `429`, `500`, `502`, `503` for real scenarios.
- [ ] I implement `Idempotency-Key` for retryable `POST` endpoints.
- [ ] I return `ProblemDetails` for every `4xx`/`5xx`.
- [ ] I include `Location` on `201` and `Retry-After` on `429` / `503`.
- [ ] I never put side effects behind `GET`.
- [ ] I use `ETag` + `If-Match` for optimistic concurrency.
- [ ] I document status codes in OpenAPI via `[ProducesResponseType]`.
- [ ] I know how status codes drive Polly retries, App Insights failure rate, and CDN caching.
- [ ] I can evolve the contract without breaking existing clients.
