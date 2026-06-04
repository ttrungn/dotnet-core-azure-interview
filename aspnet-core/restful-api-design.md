# RESTful API Design

## What It Is

**REST** (Representational State Transfer) is an architectural style for HTTP APIs defined by Roy Fielding in 2000. A RESTful API exposes the **domain as a set of resources** identified by stable URIs, manipulated through a uniform set of **HTTP methods** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`), with semantics expressed through **status codes** and **media types**. It is stateless on the server, cacheable where possible, and layered (clients don't know whether they're hitting Cloudflare, Azure Front Door, App Service, or the origin).

In practical .NET API design, "RESTful" usually means:

- **Resource URIs** as nouns: `/api/orders/{id}`, not `/api/getOrder?id=...`.
- **HTTP methods carry intent**: `POST /orders` creates, `GET /orders/{id}` reads, `PUT` replaces, `PATCH` partially updates, `DELETE` removes.
- **Status codes match the outcome**: `201 Created` with `Location`, `204 No Content` on `DELETE`, `409 Conflict` on a duplicate, `422 Unprocessable Entity` on a domain rule violation.
- **A consistent error format** — typically RFC 7807 `application/problem+json` (Problem Details).
- **Versioned** so changes don't break partners.
- **Pageable, filterable, sortable** via query parameters with documented limits.
- **Documented** via OpenAPI for client SDK generation.

Richardson's **REST Maturity Model** is a useful checklist:

| Level | Description                                           | Example                            |
|-------|-------------------------------------------------------|------------------------------------|
| 0     | "RPC over HTTP" — one URL, `POST` for everything      | `POST /api/handler { op: "create" }` |
| 1     | Multiple resources                                    | `POST /api/orders`, `POST /api/invoices` |
| 2     | HTTP verbs + status codes                             | `GET`, `POST`, `PUT`, `DELETE` + `201`/`409` |
| 3     | Hypermedia (HATEOAS) — links in responses             | `{ "id": ..., "_links": { "cancel": "/orders/123/cancel" } }` |

Most production .NET APIs sit at Level 2. Level 3 is worth it only for genuinely discoverable APIs (rare).

## Why It Exists

Before REST, distributed systems used **SOAP, XML-RPC, CORBA, DCOM** — all heavy, verbose, transport-coupled, and hard to evolve. SOAP services had `.asmx` endpoints with one URL per action and XML envelopes; clients needed generated proxies that broke on minor schema changes.

REST emerged to solve concrete problems:

1. **Web-scale interoperability** — any HTTP client (curl, browser, Postman, mobile SDK, Bash script) can call a REST API; no IDL compilation step.
2. **Cacheability** — `GET` requests on resource URIs are inherently cacheable by CDNs, browsers, reverse proxies, and Azure Front Door.
3. **Statelessness for horizontal scale** — no session affinity, any pod can serve any request.
4. **Evolvable contracts** — add a field, deprecate an endpoint, version the route; clients survive.
5. **Tooling and documentation** — OpenAPI emerged because resources + methods are easy to describe machine-readably.

For .NET teams it also matters that **Azure API Management, Azure Front Door, App Gateway, and AKS Ingress** all speak HTTP natively. A RESTful API drops into that infrastructure with caching, throttling, WAF, and OpenAPI-driven testing for free.

## When To Use It

**Use REST for:**

- **Public APIs and partner integrations** — universal client support, no SDK required.
- **Web frontends and mobile apps** — JSON over HTTPS is the lingua franca.
- **CRUD-like resources** that map naturally to nouns (orders, invoices, products, users).
- **Anything cacheable** — REST's `GET` semantics + ETags + `Cache-Control` headers shine.
- **Microservice external boundaries** in heterogeneous estates.

**Do not use REST for:**

- **High-throughput service-to-service calls inside one estate** — gRPC is faster, smaller, and strongly typed.
- **Real-time bidirectional flows** — use SignalR, WebSockets, or server-sent events.
- **Event-driven workflows** — use Service Bus, Event Grid, or Kafka, not "REST callbacks."
- **Bulk data transfer** — REST + JSON is verbose; consider gRPC streams or Parquet over blob storage.
- **Operations that aren't naturally resources** — a "send password reset email" isn't a resource. Model it as `POST /password-reset-requests` (a resource) or accept a non-RESTful action endpoint (`POST /users/{id}/actions/send-password-reset`).

## Why It Is Important

REST design choices directly drive **client experience, operability, and cost**:

1. **Client integration time** — a partner team can integrate a well-designed REST API in days; a poorly designed one (RPC-style, inconsistent status codes, ad-hoc errors) can take weeks and produce ongoing support tickets.
2. **Cacheability and cost** — proper `GET` resources behind Azure Front Door cut origin traffic by 70%+ and reduce App Service plan size.
3. **Versioning safety** — choosing URI versioning (`/api/v2/orders`) up front avoids the "we shipped a breaking change and broke the mobile app" outage.
4. **Idempotency** — for `PUT`, `DELETE`, and `POST` operations like payment capture, idempotency keys prevent duplicate charges when networks retry.
5. **OpenAPI / SDK generation** — clean REST design produces clean generated clients; messy design produces clients no one wants to use.
6. **Operability** — distributed tracing, log correlation, rate limiting, and WAF rules all assume HTTP semantics — `POST` to create, `GET` for read, `5xx` for retryable failures.

## How It's Used in C# / .NET

### 1. Resource modeling: nouns over verbs

```csharp
// ❌ RPC-style (level 0)
[HttpPost("api/getOrder")]
public OrderDto GetOrder(GetOrderRequest req) { ... }

[HttpPost("api/cancelOrder")]
public void CancelOrder(CancelOrderRequest req) { ... }

// ✅ Resource-style (level 2)
[HttpGet("api/orders/{id:guid}")]
public Task<ActionResult<OrderDto>> GetById(Guid id, CancellationToken ct) { ... }

[HttpPost("api/orders/{id:guid}/cancellations")]
public Task<ActionResult<CancellationDto>> Cancel(Guid id, CancelOrderRequest req, CancellationToken ct) { ... }
```

Notice "cancel" became `cancellations` — a sub-resource. You can also `POST /orders/{id}/actions/cancel` if that reads cleaner; both are accepted.

### 2. URI conventions

| Pattern                             | Meaning                                              |
|-------------------------------------|------------------------------------------------------|
| `/api/orders`                       | The collection of orders                             |
| `/api/orders/{id:guid}`             | A single order                                       |
| `/api/orders/{id:guid}/lines`       | Lines belonging to that order                        |
| `/api/orders/{id:guid}/lines/{lineId:int}` | A specific line                                |
| `/api/orders/{id:guid}/cancellations` | Cancellation as a sub-resource                     |
| `/api/orders?status=pending&page=2` | Filtering, pagination via query parameters           |

**Rules:**
- Lowercase, hyphenated multi-word segments (`/sales-orders/`, not `/SalesOrders/`).
- Plural collection names (`/orders`, not `/order`).
- IDs in path; everything else (filters, paging, sorting) in query string.
- No file extensions (`.json`); negotiate content via `Accept` header.

### 3. HTTP methods and idempotency

```csharp
// GET — safe, idempotent, cacheable
[HttpGet("{id:guid}")]
public Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken ct);

// POST — create or non-idempotent action (returns 201 + Location)
[HttpPost]
public async Task<IActionResult> Create(CreateOrderRequest req, CancellationToken ct)
{
    var id = await _orders.PlaceAsync(req, ct);
    return CreatedAtAction(nameof(Get), new { id }, new { id });
}

// PUT — full replace, idempotent
[HttpPut("{id:guid}")]
public Task<IActionResult> Replace(Guid id, ReplaceOrderRequest req, CancellationToken ct);

// PATCH — partial update (JSON Patch RFC 6902 or JSON Merge Patch RFC 7396)
[HttpPatch("{id:guid}")]
public Task<IActionResult> Update(Guid id, JsonPatchDocument<OrderDto> patch, CancellationToken ct);

// DELETE — idempotent (deleting an already-deleted resource still returns 204)
[HttpDelete("{id:guid}")]
public Task<IActionResult> Delete(Guid id, CancellationToken ct);
```

See [http-methods-and-status-codes.md](http-methods-and-status-codes.md) for the full semantic table.

### 4. Idempotency keys (for non-idempotent `POST` that must be retryable)

```csharp
[HttpPost("captures")]
public async Task<ActionResult<CaptureDto>> Capture(
    [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
    CapturePaymentRequest req,
    CancellationToken ct)
{
    if (string.IsNullOrEmpty(idempotencyKey))
        return Problem(statusCode: 400, title: "Missing Idempotency-Key header");

    var existing = await _idempotency.TryGetResultAsync(idempotencyKey, ct);
    if (existing is not null) return Ok(existing);            // return the same response

    var result = await _payments.CaptureAsync(req, ct);
    await _idempotency.StoreAsync(idempotencyKey, result, ct);
    return Ok(result);
}
```

Stripe, Adyen, and most payment APIs use exactly this pattern.

### 5. Pagination, filtering, sorting

```csharp
public record OrdersQuery(
    int Page = 1,
    int PageSize = 25,
    string? Status = null,
    string? Sort = "-createdAt");          // - prefix = descending

[HttpGet]
public async Task<ActionResult<PagedResult<OrderDto>>> Search([FromQuery] OrdersQuery q, CancellationToken ct)
{
    if (q.PageSize > 100) return Problem(statusCode: 400, title: "PageSize must be ≤ 100");

    var (items, total) = await _orders.SearchAsync(q.Page, q.PageSize, q.Status, q.Sort, ct);
    Response.Headers["X-Total-Count"] = total.ToString();
    return Ok(new PagedResult<OrderDto>(items, q.Page, q.PageSize, total));
}

public record PagedResult<T>(IReadOnlyList<T> Items, int Page, int PageSize, int Total)
{
    public int TotalPages => (int)Math.Ceiling(Total / (double)PageSize);
}
```

Cursor-based pagination (`?cursor=eyJ...`) is preferred for very large or frequently changing datasets — offset pagination drifts when rows are inserted.

### 6. Consistent error shape with Problem Details

```csharp
builder.Services.AddProblemDetails();          // RFC 7807

// In controller
return Problem(
    statusCode: StatusCodes.Status409Conflict,
    title: "Order already cancelled",
    detail: $"Order {id} is in state Cancelled and cannot be cancelled again",
    type: "https://errors.contoso.com/orders/already-cancelled");
```

See [error-handling-and-problem-details.md](error-handling-and-problem-details.md) for the full pattern with `IExceptionHandler`.

### 7. Versioning hooks

```csharp
// Using Asp.Versioning.Mvc (NuGet: Asp.Versioning.Mvc.ApiExplorer)
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;
    o.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
}).AddMvc();

[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController { /* ... */ }
```

See [api-versioning.md](api-versioning.md).

### 8. HATEOAS — when (rarely) to use it

```csharp
public record OrderResource(
    Guid Id, string Status, decimal Total,
    Dictionary<string, string> Links);     // self, cancel, refund, lines

// Response
{
  "id": "...",
  "status": "Pending",
  "total": 199.99,
  "_links": {
    "self": "/api/orders/123",
    "cancel": "/api/orders/123/cancellations",
    "lines": "/api/orders/123/lines"
  }
}
```

HATEOAS is valuable when the client genuinely discovers next steps from the server's state machine (e.g., workflow APIs). For most CRUD APIs, partners ignore the links and hard-code URLs anyway — skip the ceremony.

### 9. Content negotiation and media types

```csharp
[HttpGet("{id:guid}")]
[Produces("application/json", "application/xml")]
public async Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken ct) { ... }

builder.Services.AddControllers(o => o.RespectBrowserAcceptHeader = true)
    .AddXmlSerializerFormatters();
```

Defaults of JSON + UTF-8 are fine for 99% of APIs. Only add XML / CSV / Protobuf when a partner needs it.

### 10. Caching

```csharp
[HttpGet("{id:guid}")]
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any, VaryByQueryKeys = new[] { "version" })]
public async Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken ct) { ... }
```

For mutation-aware caching, use **ETags + `If-None-Match`** to return `304 Not Modified`.

## Advantages

- **Universal client support** — every language and platform speaks HTTP + JSON.
- **Cacheable** — CDN, browser, reverse proxy, and Azure Front Door can cache `GET` responses.
- **Statelessness** scales horizontally and survives pod restarts.
- **Toolable** — OpenAPI generates docs, SDKs, mock servers, contract tests.
- **Evolvable** — additive fields, versioned routes, deprecation headers — partners survive change.
- **Composable with cloud infrastructure** — Azure API Management, Front Door, WAF, App Service, Container Apps, Functions all just work.
- **Debuggable with simple tools** — curl, Postman, browser DevTools.

## Disadvantages

- **Verbose for chatty workflows** — fetching an order, its lines, and customer takes three round trips unless you embed (or use GraphQL).
- **Overhead per request** — JSON parsing, HTTP headers, TLS handshakes add latency vs binary protocols like gRPC.
- **No native streaming** — you can do server-sent events or chunked transfer, but it's awkward vs WebSockets / gRPC streams.
- **Status code semantics are loose** — different teams disagree on `409 vs 422`, `200 vs 201`, `400 vs 422`. Document your conventions.
- **No built-in real-time** — for live updates you need SignalR/WebSockets in addition.
- **Easy to do wrong** — "RPC over HTTP" with `POST` for everything is technically REST but loses every benefit.

## Common Mistakes

### 1. Verbs in URIs

```csharp
// ❌ verb in URI, POST for everything
[HttpPost("api/createOrder")]
[HttpPost("api/deleteOrder")]
[HttpPost("api/getOrders")]
```

**Fix**: noun URIs, HTTP method conveys action.

```csharp
[HttpPost("api/orders")]
[HttpDelete("api/orders/{id:guid}")]
[HttpGet("api/orders")]
```

### 2. Returning `200 OK` for created resources

```csharp
// ❌
return Ok(new { id = newOrder.Id });
```

**Fix**: `201 Created` + `Location` header so clients can follow it.

```csharp
return CreatedAtAction(nameof(Get), new { id = newOrder.Id }, new { id = newOrder.Id });
```

### 3. Inconsistent error shapes

```csharp
// Endpoint A
return BadRequest("Order must have lines");
// Endpoint B
return BadRequest(new { error = "missing lines", code = 4001 });
// Endpoint C
return BadRequest(new[] { "Order must have lines" });
```

**Fix**: Use Problem Details everywhere via `AddProblemDetails()` + `IExceptionHandler`.

### 4. Exposing EF entities directly

```csharp
[HttpGet("{id:guid}")]
public Task<Order> Get(Guid id) => _db.Orders.Include(o => o.Lines).FirstAsync(o => o.Id == id);
```

This leaks navigation properties (and lazy loading), changes the public contract every time you tweak the schema, and may serialize circular references. **Fix**: map to a DTO.

### 5. Non-idempotent `POST` retried by clients → duplicate charges

```csharp
[HttpPost("captures")]
public Task Capture(CapturePaymentRequest req) { /* charges every call */ }
```

**Fix**: require an `Idempotency-Key` header and de-dupe at the application layer.

### 6. Pagination without limits → 1 GB JSON responses

```csharp
[HttpGet]
public Task<IEnumerable<OrderDto>> All() => _orders.GetAllAsync();
```

**Fix**: mandatory paging with documented `PageSize` cap (`100`, `500`). Return `400 Bad Request` if exceeded.

### 7. URI versioning vs no versioning at all

Shipping `/api/orders` v1 with no versioning, then adding `customerType` as a required field, breaks every existing client. Adopt versioning **on day one**.

### 8. Mixing 4xx vs 5xx semantics

Returning `500` for "order not found" pollutes monitoring and triggers paging on-call. Use `404`. Reserve `5xx` for actual server failures.

## Best Practices

- **Model your domain as resources**; use HTTP methods for actions.
- **Pick a versioning strategy on day one** (URL segment is the safest default).
- **Always return Problem Details** for errors. One shape for the whole API.
- **Document with OpenAPI**; commit the generated schema as a contract artifact.
- **Paginate every collection** with sensible defaults (25) and hard limits (100 or 500).
- **Require idempotency keys** for non-idempotent write operations (payments, sends).
- **Use `201 Created` + `Location`** on creation, `204 No Content` on update/delete with no body.
- **Use kebab-case URLs**, plural collection names, no trailing slashes.
- **Validate at the boundary** with DataAnnotations + FluentValidation; return `400` or `422`.
- **Set rate limits** at API Management or via .NET 7+ `UseRateLimiter`.
- **Make `GET` cacheable** with proper `Cache-Control` and `ETag` headers.
- **Test the contract** — generate the OpenAPI in CI and diff against the committed version to catch breaking changes.
- **Map domain exceptions to HTTP statuses** in one place via `IExceptionHandler`.

## Related Concepts

- **HTTP methods and status codes** — the vocabulary REST uses. See [http-methods-and-status-codes.md](http-methods-and-status-codes.md).
- **Controllers and Minimal APIs** — implementation styles. See [controllers-and-minimal-apis.md](controllers-and-minimal-apis.md).
- **API versioning** — evolving without breaking. See [api-versioning.md](api-versioning.md).
- **Input validation** — validating request DTOs at the boundary. See [input-validation.md](input-validation.md).
- **Error handling + Problem Details** — RFC 7807. See [error-handling-and-problem-details.md](error-handling-and-problem-details.md).
- **OpenAPI / Swagger** — machine-readable contract. See [swagger-openapi.md](swagger-openapi.md).
- **Authentication and authorization** — securing resources. See [authentication-and-authorization.md](authentication-and-authorization.md).
- **gRPC** — non-REST RPC for service-to-service calls.
- **HATEOAS** — Level 3 REST with discoverable hypermedia links.
- **GraphQL** — alternative for chatty UIs that need flexible projections.
- **Azure API Management** — gateway that adds throttling, caching, transformation, and developer portals on top of REST APIs.

## Real-World Usage

### ASP.NET Core API on Azure App Service

A public Orders API for a B2B SaaS sits behind Azure Front Door + API Management on App Service Premium V3. URLs are `/api/v{version}/orders`, errors are Problem Details, pagination is mandatory, and every write supports `Idempotency-Key`. The OpenAPI document is published to API Management's developer portal so partners self-serve onboarding.

### Azure Functions

A Stripe webhook receiver runs as an HTTP-triggered Function. It is technically REST (`POST /api/webhooks/stripe`) but the resource modeling is minimal — webhooks aren't queryable. The Function validates the signature, persists the event to Cosmos DB, and enqueues a Service Bus message; the Orders API exposes the queryable resources (`/api/orders`, `/api/payments`).

### Microservices

In an estate of 12 microservices, each one exposes a REST API at its external boundary (consumed by the SPA, mobile app, and partner gateway) and gRPC internally for high-throughput service-to-service. The Orders microservice exposes `/api/orders`; the internal `orders.proto` for service calls is a separate concern.

### Multi-tenant SaaS

Tenant is resolved from a JWT claim (not the URL — keeps URIs clean and prevents enumeration). The same `/api/orders/{id}` URL maps to a different database per tenant via a `TenantResolutionMiddleware`. Pagination, filtering, sorting are tenant-scoped automatically because the repository injects the tenant context.

### Testing

```csharp
public class OrdersApiContractTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Create_returns_201_with_Location_header()
    {
        var response = await _client.PostAsJsonAsync("/api/v1/orders", new { customerId = Guid.NewGuid(), lines = new[] { new { productId = Guid.NewGuid(), quantity = 1 } } });
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location!.ToString().Should().StartWith("/api/v1/orders/");
    }

    [Fact]
    public async Task Cancel_already_cancelled_order_returns_409_problem_details()
    {
        var response = await _client.PostAsJsonAsync($"/api/v1/orders/{KnownCancelledId}/cancellations", new {});
        response.StatusCode.Should().Be(HttpStatusCode.Conflict);
        var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>();
        problem!.Title.Should().Contain("already cancelled");
    }
}
```

## Code Example — Before and After

### Before: RPC-style endpoints, inconsistent shapes

```csharp
[ApiController]
[Route("api")]
public class OrdersApi : ControllerBase
{
    [HttpPost("createOrder")]
    public IActionResult Create(CreateOrderRequest req)
    {
        if (req.Lines.Count == 0)
            return BadRequest("Need lines");                  // string error
        var id = _orders.Place(req);
        return Ok(new { id });                                 // 200, no Location
    }

    [HttpPost("getOrder")]                                     // POST for read
    public IActionResult Get(GetRequest req)
    {
        var o = _orders.Get(req.Id);
        if (o == null) return StatusCode(500);                 // 500 for not found
        return Ok(o);                                          // EF entity
    }

    [HttpPost("cancelOrder")]
    public IActionResult Cancel(CancelRequest req)
    {
        try { _orders.Cancel(req.Id); return Ok(); }
        catch (Exception ex) { return BadRequest(new { msg = ex.Message }); }
    }
}
```

Issues: not cacheable (`POST` for reads), no status code discipline, mixed error shapes, entity leakage, no versioning, no idempotency.

### After: REST level 2 with Problem Details and versioning

```csharp
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpGet("{id:guid}", Name = nameof(GetOrder))]
    [ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ResponseCache(Duration = 60, VaryByHeader = "Accept")]
    public async Task<ActionResult<OrderDto>> GetOrder(Guid id, CancellationToken ct)
        => await _mediator.Send(new GetOrderQuery(id), ct) is { } o ? Ok(o) : NotFound();

    [HttpGet]
    public async Task<ActionResult<PagedResult<OrderDto>>> Search([FromQuery] OrdersQuery q, CancellationToken ct)
    {
        if (q.PageSize > 100)
            return Problem(statusCode: 400, title: "PageSize must be ≤ 100");
        return Ok(await _mediator.Send(new SearchOrdersQuery(q), ct));
    }

    [HttpPost]
    [ProducesResponseType<OrderCreatedResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<OrderCreatedResponse>> Create(
        CreateOrderRequest req, CancellationToken ct)
    {
        var id = await _mediator.Send(new PlaceOrderCommand(req), ct);
        return CreatedAtRoute(nameof(GetOrder), new { id, version = "1.0" }, new OrderCreatedResponse(id));
    }

    [HttpPost("{id:guid}/cancellations")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Cancel(Guid id, [FromHeader(Name = "Idempotency-Key")] string idempotencyKey, CancellationToken ct)
    {
        await _mediator.Send(new CancelOrderCommand(id, idempotencyKey), ct);
        return NoContent();
    }
}
```

Now: every resource has a stable URI, methods have proper semantics, errors are Problem Details (via `IExceptionHandler`), pagination is enforced, the endpoint can be cached, and idempotency-safe cancellation is supported.

## Interview Questions and Answers

### 1. Walk me through how you'd design `/orders` and its sub-resources for a B2B commerce API.

**Why this matters**: Tests whether the candidate can model a real domain, not recite REST rules.

**Answer**: I'd model the core resources first: `/api/v1/orders` (collection), `/api/v1/orders/{id}` (single), and `/api/v1/orders/{id}/lines` (composition). Cancellation is a state transition, not a resource — I prefer `POST /api/v1/orders/{id}/cancellations` (a cancellation record) because it's queryable, auditable, and supports multiple attempts with an `Idempotency-Key`. Refunds become `POST /api/v1/orders/{id}/refunds`.

Reads have mandatory pagination (`?page=&pageSize=`), filtering (`?status=&customerId=`), and sorting (`?sort=-createdAt`). Errors are Problem Details. The OpenAPI is published to API Management so partners can generate SDKs.

**Trade-off**: Some people prefer `POST /orders/{id}/cancel` (action endpoint). It works, but loses auditability of "how many cancellation attempts and why." For payments-adjacent domains I always prefer the resource style.

**Real project**: A SaaS Orders API at a marketplace adopted exactly this shape; the auditing-as-resource pattern was instrumental during PCI compliance review.

### 2. When do you return `400` vs `409` vs `422`?

**Why this matters**: Inconsistent status codes are one of the most common API design mistakes.

**Answer**:

- **`400 Bad Request`** — the request is malformed: missing required field, wrong type, JSON parse error. Caller cannot retry without changing the request.
- **`409 Conflict`** — the request is well-formed but conflicts with current state: duplicate key, version mismatch (optimistic concurrency), cancelling an already-cancelled order.
- **`422 Unprocessable Entity`** — the request is well-formed and syntactically valid, but violates business rules: order total below minimum, customer over their credit limit. (Some teams collapse 422 into 400; both are defensible. Pick one and document it.)

Plus: **`401 Unauthorized`** = missing or invalid auth, **`403 Forbidden`** = authenticated but not allowed, **`404 Not Found`** = resource doesn't exist (don't use `404` for "auth failed", that's an information leak).

**Trade-off**: 422 isn't universally understood by all clients. If your audience is partners on legacy stacks, 400 with a clear Problem Details body is safer.

**Real project**: An Orders API on Stripe-style 409 for state conflicts and 422 for business rule failures made the Mobile team's error handling code dramatically simpler — they could branch on `409 → "retry with fresh data"` vs `422 → "show validation message"`.

### 3. How do you make a `POST` endpoint safe to retry?

**Answer**: Use **idempotency keys**. The client generates a unique key (typically a UUID) and sends it in an `Idempotency-Key` header. The server:

1. Looks up the key in a store (Redis, Cosmos DB, SQL).
2. If found, returns the stored response — no work is repeated.
3. If not found, performs the work, stores the result keyed by the idempotency key, and returns the response.
4. The key is typically retained 24 hours to a few days.

```csharp
public async Task<IActionResult> Capture(string idempotencyKey, CapturePaymentRequest req, CancellationToken ct)
{
    var cached = await _idem.GetAsync(idempotencyKey, ct);
    if (cached is not null) return Ok(cached);
    var result = await _payments.CaptureAsync(req, ct);
    await _idem.StoreAsync(idempotencyKey, result, ct);
    return Ok(result);
}
```

**Trade-off**: Storing every key adds infrastructure (Redis). For low-volume APIs you can store in the same SQL database; for high-volume, a TTL'd Redis cache.

**Real project**: Stripe and Adyen both require this pattern; we adopted it for our internal Payments API to handle mobile retries over flaky cellular networks — eliminated ~0.3% duplicate-charge incidents.

### 4. URL versioning vs header vs query — which do you pick and why?

**Answer**: I default to **URL segment versioning** (`/api/v1/orders`) because:

- **Visible** — partners and ops see the version in logs, traces, dashboards immediately.
- **Cacheable** — `/api/v2/orders/123` is a different cache key than `/api/v1/orders/123`. CDNs handle it correctly.
- **Toolable** — Postman collections, curl examples, OpenAPI docs all version cleanly.
- **Simple to reason about** — no magic header negotiation.

Header versioning (`X-Api-Version: 2`) is cleaner academically but caches less well and is invisible in logs. Query versioning (`?api-version=2`) is fine but messy when combined with other query parameters.

**Trade-off**: URL versioning means changing the URL when you upgrade — but that's exactly what makes it explicit. See [api-versioning.md](api-versioning.md).

**Real project**: An Orders API used URL versioning from day one. When v2 introduced a breaking change to the `OrderLine` schema, partners migrated over 6 months with both versions running side-by-side, no surprises.

### 5. How would you design pagination for `/orders` if customers can have millions of orders?

**Why this matters**: Offset pagination breaks at scale.

**Answer**: For small datasets, offset pagination (`?page=&pageSize=`) is fine. For millions of rows or rapidly changing data, switch to **cursor-based pagination**:

- The server returns an opaque cursor pointing to the next page (`?cursor=eyJpZCI6IjEyMyJ9`).
- The cursor encodes the last seen sort key (e.g., `(createdAt, id)`).
- Inserts/updates don't cause rows to "shift" across pages.
- It's faster (`WHERE (createdAt, id) > (lastSeenAt, lastSeenId)` uses an index seek, not an offset scan).

```json
{
  "items": [...],
  "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI2LTAzLTA0VDEwOjEzOjAwWiIsImlkIjoiYWJjIn0",
  "hasMore": true
}
```

**Trade-off**: No "jump to page 50" link — cursor pagination is forward-only. Most APIs are fine with that.

**Real project**: A Notifications API switched from `?page=` to cursor pagination after a partner started polling page 9000+ and timing out. Query time dropped from ~12s to <100ms.

### 6. A teammate proposes `POST /api/users/{id}/sendPasswordResetEmail`. What's wrong, and what's better?

**Answer**: It mixes a verb into a URI ("sendPasswordResetEmail"). REST recommends nouns. Two acceptable redesigns:

1. **Resource as a noun**: `POST /api/password-reset-requests` with `{ "email": "..." }`. Cleaner, queryable (admin can list pending requests).
2. **Action under a resource**: `POST /api/users/{id}/actions/send-password-reset`. Acceptable when the action is genuinely an operation on the resource and you don't need to query past actions.

Both return `202 Accepted` (the email send is async) with a Location header pointing to the request status.

**Trade-off**: Strict REST purists prefer option 1. Pragmatists accept option 2. Pick one and stay consistent across your API.

**Real project**: An auth service used option 1 because compliance required an audit trail of every reset request — "the resource" was the perfect place to store that record.

### 7. How do you prevent breaking changes when evolving a REST API?

**Answer**: Several layered practices:

1. **Make all additions backward compatible**: new optional fields, new endpoints, new optional headers don't need a version bump.
2. **Use OpenAPI diff in CI**: tools like `openapi-diff` compare the new spec against the committed baseline; any breaking change fails the build.
3. **Mark breaking changes by versioning the route**: `/api/v2/orders`.
4. **Use `Sunset` and `Deprecation` HTTP headers** to signal upcoming removal.
5. **Run both versions in parallel** for the agreed sunset window (commonly 6–12 months for partner APIs).
6. **Contract tests** with partners (Pact or equivalent) that fail before deployment if a breaking change slips through.

**Trade-off**: Two versions = two code paths to maintain. Be ruthless about sunset deadlines.

**Real project**: An Orders API ran v1 and v2 in parallel for 9 months while partners migrated. CI diff'd the OpenAPI on every PR and blocked merges with breaking changes to v1.

### 8. When is REST the wrong choice?

**Answer**:

- **Real-time bi-directional flows** (chat, live dashboards, multiplayer) — use SignalR or WebSockets.
- **Service-to-service inside one estate where latency and payload size matter** — gRPC + Protobuf.
- **Bulk data ingestion** — Service Bus / Event Hubs / blob upload; REST + JSON is wasteful.
- **Event-driven workflows** — publish to Service Bus / Event Grid; REST callbacks couple producers and consumers.
- **GraphQL is sometimes better** for UIs with many resource projections (mobile homepage that pulls user + recent orders + recommendations).

REST stays the right default for **public APIs, partner APIs, and CRUD-shaped microservice boundaries**.

**Trade-off**: Multiple protocols add complexity. Many teams keep REST as the universal "external" protocol and use gRPC/Service Bus only internally.

**Real project**: A marketplace exposed REST externally to partners, gRPC between Orders/Payments/Inventory, and Service Bus for async fulfillment. Each protocol's strength was used; partners only had to learn REST.

## Summary Checklist

- [ ] I can model a domain as a set of resources with stable URIs.
- [ ] I can pick the right HTTP method (`GET`/`POST`/`PUT`/`PATCH`/`DELETE`) for each operation.
- [ ] I can choose the correct status code (`201` vs `200`, `409` vs `422`, `401` vs `403`) for each outcome.
- [ ] I can design pagination, filtering, sorting with documented defaults and limits.
- [ ] I can apply idempotency keys to make non-idempotent `POST`s safe to retry.
- [ ] I can return Problem Details for all errors and map domain exceptions to HTTP statuses.
- [ ] I can version a REST API on day one and evolve it without breaking partners.
- [ ] I can describe REST's Richardson Maturity Model and when level 3 (HATEOAS) is worth the cost.
- [ ] I can publish an OpenAPI document and use it for SDK generation and contract tests.
- [ ] I can identify when REST is the wrong protocol (real-time, bulk transfer, event-driven, high-throughput service-to-service).
