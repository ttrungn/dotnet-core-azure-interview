# API Versioning

## What It Is

API versioning is the practice of evolving an HTTP (or gRPC) contract so that **existing clients keep working** while you ship new behavior. A version is essentially a stable shape promise: "if you call `/api/v1/orders`, you will keep getting the v1 request and response contract until we formally deprecate it."

In .NET, the standard library is **`Asp.Versioning.Http`** (the package formerly known as `Microsoft.AspNetCore.Mvc.Versioning`, donated to the .NET Foundation and now maintained by the [dotnet/aspnet-api-versioning](https://github.com/dotnet/aspnet-api-versioning) project). It supports four version-selection strategies — URL segment, query string, custom header, and media-type — that you can use individually or combined.

```csharp
// Versioned endpoint — v1 and v2 coexist
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public sealed class OrdersController : ControllerBase
{
    [HttpGet("{id:guid}"), MapToApiVersion("1.0")]
    public Task<ActionResult<OrderV1Dto>> GetV1(Guid id) => /* ... */;

    [HttpGet("{id:guid}"), MapToApiVersion("2.0")]
    public Task<ActionResult<OrderV2Dto>> GetV2(Guid id) => /* ... */;
}
```

Versioning is not just about URLs — it's about lifecycle: introduce, support, deprecate, sunset, remove. Without a deprecation policy, "versioning" is just URL prefixes.

## Why It Exists

In a server-rendered web app, you control both ends — you ship the front-end and the back-end together, no version drift. APIs are different:

- **Mobile apps** sit on user phones, updated when (or if) the user feels like it.
- **Partner integrations** are written once by another team and rarely touched.
- **SPAs** are usually deployed independently of the API.
- **Background workers and Logic Apps** call your API on schedules.

Without versioning, a single "improvement" — renaming a field, tightening a validation rule, changing a status code — silently breaks every old client. The result is either (a) you stop changing the API (bad for the business) or (b) you change it and field angry support tickets for weeks (worse).

API versioning gives you a contract you can evolve **safely**: add v2 alongside v1, announce a deprecation date, give clients time to migrate, monitor usage, and finally remove v1 when usage drops below a threshold.

## When To Use It

**Use API versioning when:**

- The API is consumed by clients you do not control (mobile, partners, SPAs deployed independently).
- You expect to make breaking changes over the API's lifetime.
- The API has multiple long-lived consumers with different release cadences.
- Compliance requires explicit contract management (financial APIs, healthcare).

**You can skip explicit versioning when:**

- The API is purely internal between services you own, deploy together, and version-pin via NuGet packages or gRPC contracts.
- The API is short-lived (proof of concept, internal tool).
- You can guarantee backward compatibility forever (rare, requires discipline).

**Avoid this anti-pattern:** versioning everything from day one "just in case." Routes become noisy, every endpoint has a v1, and nobody ever ships a v2 because the cost is high. Start without versioning; add it the first time you need a breaking change.

## Why It Is Important

API versioning is what separates a hobby project from a production platform. It drives:

1. **Customer retention** — partners and mobile users do not see their integrations break overnight.
2. **Velocity** — the team can ship improvements without waiting for every client to be ready.
3. **Cost control** — old endpoints can be deprecated and removed on a schedule, so operational cost doesn't grow forever.
4. **Auditability** — `Sunset` and `Deprecation` headers ([RFC 8594](https://datatracker.ietf.org/doc/html/rfc8594), [draft-ietf-httpapi-deprecation-header](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-deprecation-header)) make the contract lifecycle machine-readable.

In Azure, API Management (APIM) layers on top of this — APIM versions, revisions, and products map to your service's versions and let you expose v1 to a "Legacy Partner" product while v2 is published to "All Customers."

## How It's Used in C# / .NET

### 1. Install and register

```xml
<PackageReference Include="Asp.Versioning.Http" Version="8.1.0" />
<PackageReference Include="Asp.Versioning.Mvc" Version="8.1.0" />
<PackageReference Include="Asp.Versioning.Mvc.ApiExplorer" Version="8.1.0" />
```

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = false; // force clients to be explicit
    options.ReportApiVersions = true;                    // adds api-supported-versions header
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("x-api-version"),
        new MediaTypeApiVersionReader("v"));
})
.AddMvc()        // for controllers
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});
```

### 2. Four version-selection strategies

| Strategy           | Looks like                                          | Pros                                          | Cons                                           |
|--------------------|-----------------------------------------------------|-----------------------------------------------|------------------------------------------------|
| **URL segment**    | `GET /api/v2/orders`                                | Visible, cacheable, easy to test in a browser | Versions are part of the resource identifier   |
| **Query string**   | `GET /api/orders?api-version=2.0`                   | Resource URL is stable                        | Often stripped by caches/proxies, easy to omit |
| **Custom header**  | `x-api-version: 2.0`                                | Clean URLs, RESTful                           | Invisible in browser address bar, harder debug |
| **Media type**     | `Accept: application/json; v=2.0`                   | Most "RESTful" per Fielding                   | Hardest for non-expert clients                 |

**Practical default:** URL segment for public APIs (most discoverable), header for internal microservices (cleaner URLs).

### 3. Minimal API style

```csharp
var versionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .ReportApiVersions()
    .Build();

app.MapGet("/api/v{version:apiVersion}/orders/{id:guid}", GetOrderV1Async)
   .WithApiVersionSet(versionSet)
   .MapToApiVersion(1.0);

app.MapGet("/api/v{version:apiVersion}/orders/{id:guid}", GetOrderV2Async)
   .WithApiVersionSet(versionSet)
   .MapToApiVersion(2.0);
```

### 4. Deprecation

```csharp
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase { }
```

Combined with `ReportApiVersions = true`, the response gets:

```
api-supported-versions: 2.0
api-deprecated-versions: 1.0
```

Add `Sunset` and `Deprecation` headers (a custom middleware or `IEndpointFilter`):

```csharp
app.Use(async (ctx, next) =>
{
    await next();
    if (ctx.GetEndpoint()?.Metadata.GetMetadata<ApiVersionMetadata>() is { } meta
        && meta.IsApiVersionNeutral == false
        && meta.Map(ApiVersionMapping.Explicit).DeclaredApiVersions.Any(v => v == new ApiVersion(1, 0)))
    {
        ctx.Response.Headers.Append("Deprecation", "true");
        ctx.Response.Headers.Append("Sunset", "Wed, 01 Jan 2026 00:00:00 GMT");
        ctx.Response.Headers.Append("Link",
            "</docs/migration/v1-to-v2>; rel=\"deprecation\"");
    }
});
```

### 5. Swagger / OpenAPI integration

Each version gets its own OpenAPI document:

```csharp
var provider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
app.UseSwagger();
app.UseSwaggerUI(options =>
{
    foreach (var desc in provider.ApiVersionDescriptions)
        options.SwaggerEndpoint($"/swagger/{desc.GroupName}/swagger.json", desc.GroupName);
});
```

## Advantages

- Lets you ship breaking changes without breaking existing clients.
- Standard library, mature, supports controllers and Minimal APIs.
- Multiple reader strategies — pick what fits your consumers.
- Built-in deprecation reporting via `api-supported-versions` / `api-deprecated-versions` headers.
- Plays well with Swagger, APIM, and OpenAPI tooling.

## Disadvantages

- Real cost: every active version is a parallel code path to maintain, test, and monitor.
- URL segment versioning bakes the version into resource identifiers ("REST purists" object).
- Easy to accumulate dead versions ("we still have v1, v2, v3, v4-beta, v5...").
- Headers/media-type versioning is invisible to non-expert tools (`curl`, browser).
- DTOs must be duplicated across versions, which feels redundant.

## Common Mistakes

### 1. Versioning everything from day one with no plan to deprecate

```csharp
[ApiVersion("1.0")] // every endpoint, forever
```

**Fix:** introduce versioning the first time you have a real breaking change. Until then, an unversioned route is fine. Always pair version creation with a deprecation calendar.

### 2. Treating any change as a breaking change

```csharp
// Adding an optional field — NOT a breaking change
public record OrderDto(Guid Id, decimal Total, string? Notes); // Notes added in v1.1
```

**Fix:** know the difference. **Non-breaking:** adding optional response fields, adding optional request fields, adding endpoints, relaxing validation. **Breaking:** removing/renaming fields, changing types, tightening validation, changing status codes, changing default behavior. Only the second category needs a new major version.

### 3. Sharing DTOs across versions

```csharp
// Shared OrderDto — change it for v2, you broke v1
public record OrderDto(...) { }
public class OrdersV1Controller { Task<OrderDto> Get(); }
public class OrdersV2Controller { Task<OrderDto> Get(); }
```

**Fix:** separate DTOs per version (`Contracts/V1/OrderDto.cs`, `Contracts/V2/OrderDto.cs`). They look duplicated; that's the point — each version is frozen.

### 4. Sharing controllers/handlers across versions

```csharp
public class OrdersController
{
    public Task<IActionResult> Get(Guid id, ApiVersion version)
    {
        if (version.MajorVersion == 1) return GetV1(id);
        return GetV2(id);
    }
}
```

**Fix:** separate controllers or `[MapToApiVersion]` per action. The branching style turns into a `switch` with 10 versions in three years.

### 5. No deprecation strategy, no telemetry on per-version usage

You add v2 in March, in October you still have v1 traffic, you have no idea who's still calling it.

**Fix:** emit a metric per request tagged with the resolved API version. Set a deprecation date when you ship v2. Email partners. Watch the metric drop to zero. **Then** remove v1.

### 6. Returning the same `application/json` content type for different shapes

Two callers asking for `application/json` against v1 vs v2 should not get different shapes for the same URL — that violates HTTP caching semantics.

**Fix:** use distinct URLs (URL segment), distinct headers, or distinct media types per version. Don't silently switch.

### 7. Versioning at the wrong granularity

Versioning the whole API as one ("v1 → v2 of everything") forces you to bump every endpoint when one changes. Versioning every endpoint independently is chaos.

**Fix:** version per resource family (`/api/v2/orders` can coexist with `/api/v1/customers`). Use `[ApiVersionNeutral]` on truly stable endpoints like health checks.

## Best Practices

- Use `Asp.Versioning.Http` — don't roll your own.
- URL segment for public APIs, header for internal microservices.
- Default to "explicit version required" (`AssumeDefaultVersionWhenUnspecified = false`).
- Set `ReportApiVersions = true` so clients see supported/deprecated versions.
- Emit `Deprecation` and `Sunset` headers on deprecated endpoints with a `Link` to the migration doc.
- Version per resource family, not the whole API.
- Separate DTOs per version; never share them.
- Track per-version request counts as a metric — your "is it safe to delete v1?" signal.
- Document the deprecation policy publicly (typical: 6–12 months notice).
- Test old versions in CI for as long as they're supported.

## Related Concepts

- [restful-api-design.md](restful-api-design.md) — URL design, status codes, resource modeling.
- [swagger-openapi.md](swagger-openapi.md) — one OpenAPI document per version.
- [controllers-and-minimal-apis.md](controllers-and-minimal-apis.md) — both styles support versioning.
- [error-handling-and-problem-details.md](error-handling-and-problem-details.md) — error shape may itself need versioning.
- [../devops/blue-green-deployment.md](../devops/blue-green-deployment.md) — multi-version deploy strategies.
- [../azure/app-service.md](../azure/app-service.md) — APIM revisions and versions sit in front.

## Real-World Usage

### Mobile-backed APIs

A retail order API supports a mobile app where 30% of users haven't updated in 6 months. v1 stays alive for at least 12 months after v2 ships. Telemetry tracks `request_count{api_version="1.0"}` by client app build number — when 95% of installs are on a build that uses v2, v1 is deprecated; six months later, removed.

### B2B partner integrations

Each partner is on a different version. v1 has 4 partners, v2 has 12. Removing v1 requires written sign-off from each v1 partner. The deprecation date in the contract is real. APIM enforces — when a v1 partner's token hits the v2-only endpoint, APIM returns 410 Gone with a Link header to their account manager.

### Internal microservices

The orders service calls the catalog service. Both are owned by adjacent teams. They version with header (`x-api-version: 2.0`). When catalog ships v2, the orders team upgrades within the same sprint — no long-lived multi-version support.

### Azure API Management

APIM sits in front of N microservices. Each backend has versions; APIM exposes a unified API to consumers and translates. Versioned products in APIM (`/external/v1`, `/external/v2`) map to internal versions. Sunset headers propagate automatically.

## Code Example — Before and After

### Before — single unversioned API, accidental breaking change

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken ct)
    {
        var order = await _mediator.Send(new GetOrderQuery(id), ct);
        return order is null ? NotFound() : Ok(order);
    }
}

public record OrderDto(Guid Id, decimal Total, string Status);
```

PM asks: "Can we split `Total` into `Subtotal` and `Tax` and rename `Status` to `OrderStatus`?" The "yes, just deploy it" answer breaks the iOS app v3.2, the Android app v2.8, and the partner ETL job. Everyone is angry by 9 AM.

### After — explicit versioning with deprecation

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = false;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddMvc()
.AddApiExplorer(opt => { opt.GroupNameFormat = "'v'VVV"; opt.SubstituteApiVersionInUrl = true; });
```

```csharp
// Contracts/V1/OrderDto.cs — frozen
namespace OrdersApi.Contracts.V1;

public sealed record OrderDto(Guid Id, decimal Total, string Status);
```

```csharp
// Contracts/V2/OrderDto.cs — new shape
namespace OrdersApi.Contracts.V2;

public sealed record OrderDto(
    Guid Id,
    decimal Subtotal,
    decimal Tax,
    decimal Total,
    OrderStatus Status);
```

```csharp
// Controllers/V1/OrdersController.cs
namespace OrdersApi.Controllers.V1;

[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/orders")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<Contracts.V1.OrderDto>> Get(Guid id, CancellationToken ct)
    {
        var order = await mediator.Send(new GetOrderV1Query(id), ct);
        return order is null ? NotFound() : Ok(order);
    }
}
```

```csharp
// Controllers/V2/OrdersController.cs
namespace OrdersApi.Controllers.V2;

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<Contracts.V2.OrderDto>> Get(Guid id, CancellationToken ct)
    {
        var order = await mediator.Send(new GetOrderV2Query(id), ct);
        return order is null ? NotFound() : Ok(order);
    }
}
```

```csharp
// Middleware/SunsetHeaderMiddleware.cs
public sealed class SunsetHeaderMiddleware(RequestDelegate next)
{
    private static readonly DateTimeOffset V1Sunset =
        new(2026, 6, 1, 0, 0, 0, TimeSpan.Zero);

    public async Task Invoke(HttpContext ctx)
    {
        await next(ctx);
        var meta = ctx.GetEndpoint()?.Metadata.GetMetadata<ApiVersionMetadata>();
        if (meta?.Map(ApiVersionMapping.Explicit).DeprecatedApiVersions
                .Contains(new ApiVersion(1, 0)) == true)
        {
            ctx.Response.Headers["Deprecation"] = "true";
            ctx.Response.Headers["Sunset"] = V1Sunset.ToString("R");
            ctx.Response.Headers["Link"] =
                "</docs/orders/v1-to-v2>; rel=\"deprecation\"; type=\"text/html\"";
        }
    }
}
```

Now: v1 keeps working for existing clients with clear deprecation signaling, v2 ships immediately, the migration doc is linked in every v1 response, and per-version metrics tell the team when v1 is safe to retire.

## Interview Questions and Answers

### 1. URL versioning, header versioning, or media-type versioning — which would you pick?

**Why this matters:** Tests whether the candidate knows the trade-offs vs. parroting "REST says X."

**Answer:** For a public API consumed by partners and mobile apps, **URL segment** (`/api/v2/...`) — it's visible, easy to test in a browser, never stripped by proxies, and self-documenting. For internal microservice-to-microservice calls where you control both ends, **header** (`x-api-version: 2.0`) — keeps URLs clean. Media-type versioning is theoretically the most correct REST style, but real-world tooling (curl, browsers, basic load testers) handles it poorly. I'd combine URL + header so the framework accepts either.

**Trade-off:** URL bakes the version into the resource identifier; purists hate it. Reality always wins.

**Real project:** A retail API used URL versioning for the public endpoint and headers for the BFF-to-microservice hop. Best of both.

### 2. Is adding an optional field to a response a breaking change?

**Why this matters:** Wrong answer here causes either too many versions or breaking deployments.

**Answer:** **No**, for clients that follow the "tolerant reader" principle (ignore unknown fields). Yes, in practice, for clients that use strict deserializers configured to fail on unknown properties — e.g., `JsonSerializerOptions { UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow }`, or generated typed clients. The pragmatic rule: document "tolerant reader expected" in your API guidelines, and additive changes do not bump the version. **Removing** or **renaming** a field always bumps the major version.

**Trade-off:** You depend on client discipline. Some clients will break anyway; have a plan.

**Real project:** A team added a `notes` field to an order response. One partner's generated client rejected the response. Fix: partner regenerated their client. No version bump.

### 3. How long do you support an old version?

**Why this matters:** Wrong answer either burns the team out maintaining versions forever, or burns customers.

**Answer:** Depends on consumer type. **Mobile/SDK consumers**: 12–18 months minimum because users don't update. **Web SPAs**: 3–6 months because deploys are frequent. **Internal microservices**: as long as the calling team needs to ship the upgrade — often a single sprint. **Partner integrations**: whatever your contract says, typically 12 months with a written notice. Publish the policy on the API docs; communicate per-version sunset dates via `Sunset` headers and email.

**Trade-off:** Longer support = more parallel code paths = more cost.

**Real project:** A B2B API had 6 active major versions because nobody enforced sunset. Cleanup took two quarters; afterwards a written 12-month policy prevented recurrence.

### 4. Should you version DTOs, controllers, or just routes?

**Why this matters:** Determines whether the codebase stays manageable across versions.

**Answer:** **All three**, in different ways. Routes carry the version (`/api/v2/orders`). Controllers/endpoints are separated per version (`Controllers.V1.OrdersController`, `Controllers.V2.OrdersController`) so each version is its own readable file with no `if (version == 1)` branching. DTOs are separated per version (`Contracts.V1.OrderDto`, `Contracts.V2.OrderDto`) because they're contracts and contracts are frozen. Domain entities and handlers are **not** versioned — they're internal implementation, free to evolve. The mapping between versioned DTO and domain entity is the only "translation" layer.

**Trade-off:** Some duplication. Worth it.

**Real project:** A team that shared one `OrderDto` across versions ended up with `[JsonIgnore]` attributes conditionally applied per version — a mess. Splitting DTOs fixed it.

### 5. A client is hitting v1 and you removed v1 yesterday. What do they see and what should they see?

**Why this matters:** Tests deprecation hygiene.

**Answer:** Returning `404 Not Found` is wrong — it tells the client the URL was never valid. The correct status is **`410 Gone`** with a body that says "v1 was deprecated on YYYY-MM-DD and removed on YYYY-MM-DD; migrate to v2 at <link>." Include the `Link` header. Even better: leave a stub v1 endpoint that returns `410` for 30 days after removal, so clients get a clear signal instead of `404`-style confusion. Most importantly, this surprise shouldn't happen — your `Sunset` headers and partner emails should have warned them.

**Trade-off:** A stub endpoint is extra code, but the support cost it saves is huge.

**Real project:** A team that removed an endpoint without sunset headers caused a P1 with two partners. Adding `410` stubs + a 12-month sunset policy prevented repeats.

### 6. How does API versioning interact with database schema?

**Why this matters:** Versioning isn't just transport — data has to back the contract.

**Answer:** The database is the **single source of truth**; versions live above it. v1 and v2 must both project from the same schema. If v2 introduces `Subtotal` and `Tax`, they're either new columns (with backfill) or computed from existing columns. v1's response is now a projection ("Total = Subtotal + Tax"). You never duplicate tables per version. If the new shape requires data the schema doesn't have, you migrate the schema additively (add columns, don't drop), backfill, then ship v2. v1 keeps working because the new columns are optional in its projection.

**Trade-off:** Schema migrations need to be backward-compatible — that's a separate discipline (see [../data-access/migrations.md](../data-access/migrations.md)).

**Real project:** A team split `customer.name` into `first_name` + `last_name`. They added the two new columns, backfilled from `name`, shipped v2 with the new shape, kept `name` for v1 as a computed concat, deprecated v1, dropped `name` six months later.

### 7. How do you avoid the "we have 8 versions and nobody knows which one to use" mess?

**Why this matters:** Tests organizational maturity, not just code.

**Answer:** Four practices. **(1)** Publish a written deprecation policy — every version has a sunset date when it ships. **(2)** Make `ReportApiVersions = true` and check `Deprecation`/`Sunset` headers in CI integration tests so deprecated calls fail builds in client repos. **(3)** Per-version request metrics dashboard, reviewed monthly by the API owners. **(4)** Treat removing a version as a real piece of work in the backlog, not "we'll get to it." Removal day is the only way the cost stays bounded.

**Trade-off:** Removal is political — someone is using v3 and doesn't want to migrate. Hold the line.

**Real project:** A team that ran a quarterly "version review" meeting with product, ops, and partner support kept active versions to ≤ 3 indefinitely.

### 8. How do you test multiple API versions in CI?

**Why this matters:** Versioning that isn't tested breaks silently.

**Answer:** Two layers. **(1)** Per-version integration tests using `WebApplicationFactory<Program>` — each test sets the version explicitly via URL or header and asserts the response shape matches the frozen DTO. **(2)** Contract tests — JSON schema files per version, asserted against live responses in CI. When someone accidentally changes v1's shape, the contract test fails. For consumer-driven contracts (Pact), partner test suites run against the API in a staging environment. Together these catch both shape regressions and behavioral regressions.

**Trade-off:** More tests to maintain, but they catch the most expensive class of bug.

**Real project:** A team added JSON schema contract tests and immediately caught two accidental v1 shape changes during what they thought was a non-breaking refactor.

## Summary Checklist

- [ ] I use `Asp.Versioning.Http` (the official package), not custom routing.
- [ ] I pick a version strategy that fits the consumers (URL for public, header for internal).
- [ ] I set `AssumeDefaultVersionWhenUnspecified = false` to force explicit versions.
- [ ] I set `ReportApiVersions = true` so clients see `api-supported-versions`.
- [ ] I separate DTOs per version, never share them across versions.
- [ ] I separate controllers/endpoints per version, no `if (version == X)` branching.
- [ ] I emit `Deprecation` and `Sunset` headers on deprecated endpoints.
- [ ] I publish a written deprecation policy with notice periods.
- [ ] I track per-version request metrics and review them on a schedule.
- [ ] I have per-version contract tests in CI to catch silent breakage.
