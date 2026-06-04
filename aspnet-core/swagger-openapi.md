# Swagger and OpenAPI

## What It Is

**OpenAPI** (formerly Swagger Specification) is a vendor-neutral, language-agnostic description format for HTTP APIs. An OpenAPI document is a JSON or YAML file that describes every route, parameter, request body, response, status code, and security scheme an API exposes. The current version is OpenAPI 3.1, which aligns with JSON Schema 2020-12.

**Swagger** is the name for the original tooling ecosystem around the format — Swagger UI (interactive browser docs), Swagger Codegen (client generation), and Swagger Editor. The spec was renamed "OpenAPI" in 2015 when it was donated to the Linux Foundation; the tools kept the Swagger brand.

In .NET there are three competing libraries to produce the document from your code:

| Library                              | NuGet                                  | Status                                    |
|--------------------------------------|----------------------------------------|-------------------------------------------|
| **Swashbuckle.AspNetCore**           | `Swashbuckle.AspNetCore`               | Most common, mature, supports OpenAPI 3.0 |
| **NSwag**                            | `NSwag.AspNetCore`                     | Also generates clients, OpenAPI 3.0       |
| **Microsoft.AspNetCore.OpenApi**     | Built-in since .NET 9                  | First-party, OpenAPI 3.0, leaner          |

```csharp
// .NET 9 minimal — first-party OpenAPI document generator
builder.Services.AddOpenApi();
app.MapOpenApi(); // serves /openapi/v1.json
```

The document is the **contract** — it's what front-end teams build against, what partners integrate with, what client SDKs are generated from, and what API gateways (Azure APIM) import.

## Why It Exists

Before OpenAPI, API documentation was a Word doc, a Confluence page, or a hand-written README. Three predictable problems:

- **Doc drift** — the API shipped a new field, nobody updated the doc, the front-end team integrated against stale docs and shipped a bug.
- **No tooling** — humans had to read prose and copy-paste request examples.
- **No machine consumers** — you couldn't generate typed clients, validate responses against a schema, or auto-import into an API gateway.

OpenAPI solved this by making the contract **executable**: generated from code (so it can't drift), readable by humans (via Swagger UI), and consumable by tools (codegen, contract testing, mock servers, gateways, security scanners). The same JSON file feeds your docs site, your TypeScript client SDK, your Postman collection, and your APIM import.

## When To Use It

**Always generate an OpenAPI document for:**

- Any public HTTP API consumed by clients outside your team.
- Internal microservices where another team writes the consumer.
- APIs imported into Azure API Management, AWS API Gateway, or Kong.
- APIs that need typed client SDKs (NSwag/`openapi-generator` to TypeScript, Swift, Kotlin, etc.).
- APIs where you want contract tests.

**Skip Swagger UI in production for:**

- Pure machine-to-machine APIs where humans never browse.
- High-security internal APIs where exposing the schema gives attackers a roadmap.

The document is cheap to ship; the **UI** is what you optionally gate or remove.

## Why It Is Important

OpenAPI drives:

1. **Onboarding speed** — a new front-end dev opens Swagger UI and tries `POST /orders` in 30 seconds, no Postman setup, no docs hunt.
2. **Cross-team coordination** — back-end ships a schema change, front-end's generated client breaks at `npm install`, conversation happens **before** prod.
3. **Gateway integration** — APIM imports OpenAPI directly; policies and products align to operations.
4. **Security review** — security scanners (Stackhawk, ZAP) consume OpenAPI to fuzz every documented endpoint.
5. **Client SDKs** — typed SDKs for 10 languages in CI, no manual handwriting.

In Azure, OpenAPI is the cross-service contract glue: APIM, Functions, Logic Apps, and Power Platform all consume it.

## How It's Used in C# / .NET

### 1. Swashbuckle (most common, OpenAPI 3.0)

```xml
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.7.0" />
```

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Orders API",
        Version = "v1",
        Description = "Order management for the storefront platform.",
        Contact = new OpenApiContact { Name = "Orders Team", Email = "orders@contoso.com" }
    });

    // XML doc comments for richer descriptions
    var xmlPath = Path.Combine(AppContext.BaseDirectory, "OrdersApi.xml");
    options.IncludeXmlComments(xmlPath);

    // JWT bearer security scheme
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        Description = "Paste your access token."
    });
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        [new OpenApiSecurityScheme
        {
            Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
        }] = Array.Empty<string>()
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment() || app.Environment.IsStaging())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Enable XML comments in the csproj:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);CS1591</NoWarn>
</PropertyGroup>
```

### 2. .NET 9 built-in OpenAPI

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((doc, ctx, ct) =>
    {
        doc.Info = new OpenApiInfo
        {
            Title = "Orders API",
            Version = "v1",
            Description = "Order management for the storefront platform."
        };
        return Task.CompletedTask;
    });
});

var app = builder.Build();
app.MapOpenApi(); // GET /openapi/v1.json

// Pair with a UI (Scalar is a modern alternative to Swagger UI)
if (app.Environment.IsDevelopment())
    app.MapScalarApiReference(); // NuGet: Scalar.AspNetCore
```

`Microsoft.AspNetCore.OpenApi` is leaner than Swashbuckle, doesn't ship a UI, and uses `Microsoft.OpenApi`. Use it for new projects on .NET 9+ unless you need a Swashbuckle-specific feature.

### 3. Documenting endpoints

**Controllers:**

```csharp
/// <summary>Create a new order for an existing customer.</summary>
/// <param name="request">The order to create.</param>
/// <response code="201">Order created. Location header points to the new resource.</response>
/// <response code="400">Validation failure. Body is ValidationProblemDetails.</response>
/// <response code="409">Customer exists but is suspended.</response>
[HttpPost]
[ProducesResponseType(typeof(OrderCreatedResponse), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
public async Task<ActionResult<OrderCreatedResponse>> Create(
    CreateOrderRequest request, CancellationToken ct) { /* ... */ }
```

**Minimal API:**

```csharp
app.MapPost("/api/v1/orders", CreateOrderAsync)
   .WithName("CreateOrder")
   .WithSummary("Create a new order")
   .WithDescription("Creates an order for an existing customer and returns the new id.")
   .Produces<OrderCreatedResponse>(StatusCodes.Status201Created)
   .ProducesValidationProblem()
   .ProducesProblem(StatusCodes.Status409Conflict)
   .RequireAuthorization();
```

### 4. NSwag (when you want both server document and client SDK)

```xml
<PackageReference Include="NSwag.AspNetCore" Version="14.0.0" />
```

```csharp
builder.Services.AddOpenApiDocument(settings =>
{
    settings.Title = "Orders API";
    settings.Version = "v1";
});

app.UseOpenApi();      // /swagger/v1/swagger.json
app.UseSwaggerUi();    // /swagger
```

NSwag's killer feature is `NSwagStudio` / `nswag.json` — point it at the OpenAPI URL and it generates a C# or TypeScript client at build time. Useful when your front-end and back-end live in the same repo.

### 5. Code generation

Given an OpenAPI document, generate a typed client in any language. Common tools:

- **`openapi-generator`** (CLI, Java-based, 50+ languages) — `openapi-generator-cli generate -g typescript-axios -i ./openapi.json -o ./web/src/api`
- **NSwag** (C# / TypeScript) — built-in MSBuild integration.
- **Kiota** (Microsoft, OpenAPI 3.0, C# / TypeScript / Java / Python / Go) — designed for OData and Microsoft Graph patterns.

In CI, run codegen against the deployed OpenAPI URL of the staging environment. If the client SDK fails to build, the API made a breaking change.

## Advantages

- Machine-readable contract — feeds docs, codegen, gateways, scanners.
- Auto-generated from code, so it doesn't drift if you keep attributes accurate.
- Swagger UI gives instant interactive documentation.
- First-class support in Azure APIM, Logic Apps, Power Platform.
- Drives client SDK generation in any language.

## Disadvantages

- The generated document is only as accurate as the attributes you add — `[ProducesResponseType]` discipline is essential.
- OpenAPI 3.0 / 3.1 differences trip up tooling (Swashbuckle is still 3.0; Microsoft.AspNetCore.OpenApi is 3.0 too).
- Swagger UI in production is a discoverable attack surface — gate or remove it.
- Complex polymorphism (`oneOf`, `discriminator`) is awkward and inconsistently supported across tools.
- Doesn't replace human-readable narrative docs for workflows ("how do I do a refund?").

## Common Mistakes

### 1. Shipping Swagger UI to production with no auth

```csharp
app.UseSwagger();
app.UseSwaggerUI(); // always on, anyone on the internet can browse it
```

**Fix:** gate by environment, by IP allowlist, or behind `[Authorize]`. Or remove the UI entirely and serve only the JSON document.

### 2. Missing `[ProducesResponseType]` everywhere

Without it, Swagger says every endpoint returns `200 OK` and nothing else. Clients generate types that miss error responses.

```csharp
// BAD
[HttpGet("{id:guid}")]
public ActionResult<OrderDto> Get(Guid id) { /* ... */ }
```

**Fix:**

```csharp
[ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
public ActionResult<OrderDto> Get(Guid id) { /* ... */ }
```

### 3. Returning `IActionResult` and losing type info

```csharp
// BAD — schema for the body is gone
public async Task<IActionResult> Get(Guid id) => Ok(await _mediator.Send(...));
```

**Fix:** return `ActionResult<T>` or `Results<Ok<OrderDto>, NotFound>` (typed results in .NET 7+).

### 4. Forgetting to document the security scheme

JWT-protected endpoints show no "Authorize" button in Swagger UI; users can't try them.

**Fix:** call `AddSecurityDefinition` + `AddSecurityRequirement` as shown above, or use `.RequireAuthorization()` on Minimal API endpoints with the right metadata.

### 5. Exposing internal entities directly

```csharp
[HttpGet("{id:guid}")]
public Order Get(Guid id) => _db.Orders.Find(id); // leaks navigation properties, internal flags
```

**Fix:** always project to a DTO. The OpenAPI schema should match a contract, not a database table.

### 6. No version per OpenAPI document

When you have v1 and v2 of your API but a single Swagger doc, clients can't tell which operations belong to which version.

**Fix:** one document per version (`/swagger/v1/swagger.json`, `/swagger/v2/swagger.json`) via `IApiVersionDescriptionProvider` integration. See [api-versioning.md](api-versioning.md).

### 7. Treating Swagger UI as the primary doc

Swagger UI is an explorer, not a user guide. It doesn't tell you "how to do a payment workflow."

**Fix:** complement OpenAPI with a narrative doc (Markdown, Docusaurus, GitBook) that explains workflows, auth setup, and common scenarios. OpenAPI documents endpoints; narrative documents intent.

## Best Practices

- Pick one library and stick to it (Swashbuckle for mature, .NET 9 OpenAPI for greenfield).
- Add `[ProducesResponseType]` (or Minimal API equivalents) on every endpoint, every status code.
- Always project to DTOs, never expose internal entities.
- Document security schemes (`Bearer` for JWT, `OAuth2` for Entra ID flows).
- Generate XML doc comments and include them in the OpenAPI build.
- Gate Swagger UI by environment or auth in production.
- One OpenAPI document per API version.
- Generate clients in CI; treat client-build failure as breaking-change detection.
- Validate the OpenAPI document in CI with `spectral lint` or `openapi-cli validate`.
- Publish the document as a build artifact for downstream teams.

## Related Concepts

- [restful-api-design.md](restful-api-design.md) — the contract you're describing.
- [api-versioning.md](api-versioning.md) — one OpenAPI doc per version.
- [input-validation.md](input-validation.md) — validation rules appear in the schema.
- [error-handling-and-problem-details.md](error-handling-and-problem-details.md) — error responses in the doc.
- [controllers-and-minimal-apis.md](controllers-and-minimal-apis.md) — both styles produce OpenAPI.
- [../azure/app-service.md](../azure/app-service.md) — APIM imports OpenAPI.

## Real-World Usage

### Public partner API

Hosted Swagger UI behind a partner-portal login. OpenAPI JSON is downloadable as part of the partner SDK. APIM consumes the same JSON to provision new partner subscriptions. Sample requests in OpenAPI examples are kept up to date because they're generated from integration tests.

### Internal microservices

OpenAPI document published to a shared developer portal (Backstage, internal Hugo site) on every deploy. Consumer teams subscribe to a Slack channel that posts when any service publishes a new OpenAPI doc; they review breaking changes before client codegen breaks their CI.

### Azure API Management

A pipeline step in Azure DevOps takes the OpenAPI document from the deployed staging environment and imports it into APIM via `az apim api import`. Operations, schemas, and example responses all sync. Policies attached to operations (rate limit, IP filter, JWT validation) carry across deploys.

### Client SDK generation

The front-end repo has a CI step: `openapi-generator-cli generate -g typescript-axios -i $API_STAGING/openapi.json -o ./src/generated/api`. The generated client is committed (or generated fresh per CI run). When the API changes, the front-end TypeScript build fails until the front-end is updated. This catches breaking changes before they reach prod.

### Mock servers for front-end development

`prism mock ./openapi.json --port 4010` starts a mock server that returns example responses from the document. The front-end team can build against the contract before the back-end implements it. As the back-end ships, the team flips the base URL.

## Code Example — Before and After

### Before — endpoints with no documentation, Swagger UI in prod, leaking entities

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddSwaggerGen();
var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI();
app.MapControllers();
app.Run();

// Controller
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var order = await _db.Orders.FindAsync(id);
        return Ok(order); // raw EF entity, all fields including internal flags
    }
}
```

Problems: no descriptions, no status codes documented, raw entity leaks navigation properties, Swagger UI exposed publicly, no security scheme so the Try-It-Out button doesn't work.

### After — documented endpoints, secured UI, typed responses, version-aware

```csharp
// Program.cs
using Asp.Versioning;
using Asp.Versioning.ApiExplorer;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApiVersioning(o =>
    {
        o.DefaultApiVersion = new ApiVersion(1, 0);
        o.ReportApiVersions = true;
        o.AssumeDefaultVersionWhenUnspecified = false;
    })
    .AddApiExplorer(o => { o.GroupNameFormat = "'v'VVV"; o.SubstituteApiVersionInUrl = true; });

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, "OrdersApi.xml"));

    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT"
    });
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        [new OpenApiSecurityScheme
        {
            Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
        }] = Array.Empty<string>()
    });
});

builder.Services.ConfigureOptions<ConfigureSwaggerOptions>(); // adds one SwaggerDoc per version

builder.Services.AddControllers();

var app = builder.Build();

if (app.Environment.IsDevelopment() || app.Environment.IsStaging())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        var provider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
        foreach (var desc in provider.ApiVersionDescriptions)
            options.SwaggerEndpoint($"/swagger/{desc.GroupName}/swagger.json", desc.GroupName);
    });
}

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

```csharp
// ConfigureSwaggerOptions.cs — one OpenApiInfo per API version
internal sealed class ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider)
    : IConfigureOptions<SwaggerGenOptions>
{
    public void Configure(SwaggerGenOptions options)
    {
        foreach (var desc in provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(desc.GroupName, new OpenApiInfo
            {
                Title = "Orders API",
                Version = desc.ApiVersion.ToString(),
                Description = desc.IsDeprecated
                    ? "This API version is deprecated. Please migrate to v2."
                    : "Order management for the storefront platform."
            });
        }
    }
}
```

```csharp
// Controllers/V1/OrdersController.cs
namespace OrdersApi.Controllers.V1;

/// <summary>Order endpoints (v1).</summary>
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
[Produces("application/json")]
public sealed class OrdersController(IMediator mediator) : ControllerBase
{
    /// <summary>Get a single order by id.</summary>
    /// <param name="id">The order id.</param>
    /// <param name="ct">Cancellation token.</param>
    /// <response code="200">Order found.</response>
    /// <response code="404">Order not found.</response>
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(Contracts.V1.OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<Contracts.V1.OrderDto>> Get(Guid id, CancellationToken ct)
    {
        var order = await mediator.Send(new GetOrderV1Query(id), ct);
        return order is null ? NotFound() : Ok(order);
    }
}
```

Now: every endpoint has a typed schema, documented status codes, an `Authorize` button that works with JWTs, one OpenAPI document per version, and Swagger UI is gated by environment. The OpenAPI JSON is published as a CI artifact and imported into APIM.

## Interview Questions and Answers

### 1. Swashbuckle, NSwag, or the new .NET 9 `Microsoft.AspNetCore.OpenApi` — which would you pick?

**Why this matters:** Tests current knowledge and trade-off awareness.

**Answer:** **Greenfield on .NET 9+**: `Microsoft.AspNetCore.OpenApi`, paired with Scalar or Swagger UI for the UI layer. It's first-party, leaner, and stays in step with framework releases. **Existing service on .NET 8**: stay on Swashbuckle — it's stable and the ecosystem of operation filters/schema filters is huge. **Need client SDK generation in the same repo**: NSwag, because its MSBuild integration generates a typed client at build time. The three are interchangeable for the basic "produce an OpenAPI doc" use case; the choice comes down to ecosystem fit.

**Trade-off:** Microsoft.AspNetCore.OpenApi is newer and lacks some Swashbuckle extension points.

**Real project:** A team starting fresh on .NET 9 chose Microsoft.AspNetCore.OpenApi + Scalar; cut the Swagger UI bundle and reduced response time on `/openapi/v1.json` significantly.

### 2. Should you ship Swagger UI to production?

**Why this matters:** Common security blind spot.

**Answer:** **The OpenAPI JSON, often yes** — partners and tools need it. **The UI, usually no for public APIs** — it's a discovery surface for attackers and an unnecessary attack vector. For internal microservices, behind corporate auth is fine. Common patterns: gate with `app.Environment.IsDevelopment()`, put behind `[Authorize]` with an admin policy, restrict by IP, or serve the JSON only and host the UI on a separate internal docs site that pulls from staging.

**Trade-off:** Convenience for partners vs. surface-area minimization. Pick per audience.

**Real project:** A team that left Swagger UI on a public payments API discovered an automated scanner enumerating every operation. They moved UI behind Entra ID auth, kept the JSON publicly available.

### 3. The front-end says "the docs say it returns `OrderDto` but I'm getting null." What happened?

**Why this matters:** Classic doc-drift problem; tests whether the candidate knows how the schema is derived.

**Answer:** The most likely cause is an endpoint returning `IActionResult` or `Task<IActionResult>` instead of `ActionResult<OrderDto>` — Swashbuckle can't infer the response type from `IActionResult`, so it defaults to "anything" or whatever a stale `[ProducesResponseType]` says. Either the attribute lies or the code returns a different shape. Fix: change the signature to `ActionResult<OrderDto>` or `Results<Ok<OrderDto>, NotFound>` (typed results) so the type travels with the method. Add a contract test that asserts the response schema matches the OpenAPI declaration.

**Trade-off:** Typed results add slight verbosity but prevent exactly this.

**Real project:** A team caught dozens of "documented as X, actually returns Y" mismatches by adding a schema-validation step in their integration tests.

### 4. How do you document polymorphic responses (`OrderDto` vs `RefundDto` from the same endpoint)?

**Why this matters:** Polymorphism is a real-world pain point with OpenAPI.

**Answer:** OpenAPI supports `oneOf` and `discriminator`. In Swashbuckle, configure `UseOneOfForPolymorphism` and `UseAllOfForInheritance` in `AddSwaggerGen`. Mark the base class with `[JsonPolymorphic]` and the derived types with `[JsonDerivedType]` (System.Text.Json 7+) so serialization matches the schema. Generated TypeScript clients then expose a union type. Honestly, this is awkward in every tool — when possible, prefer separate endpoints (`/api/orders/{id}` vs `/api/refunds/{id}`) over one endpoint returning many shapes.

**Trade-off:** Cleaner API at the cost of more routes vs. polymorphism with tooling friction.

**Real project:** A team's payment endpoint returned `CardPayment | BankTransfer | DigitalWallet`. Each consumer client struggled with the union; splitting into `/payments/cards/{id}`, `/payments/bank/{id}`, `/payments/wallets/{id}` made the contract clearer.

### 5. How do you generate a typed TypeScript client from your .NET API in CI?

**Why this matters:** Tests cross-team workflow knowledge.

**Answer:** Two common chains. **(a)** `openapi-generator-cli generate -g typescript-axios -i $API/openapi/v1.json -o ./src/api` — run in the front-end repo's CI against the staging API. **(b)** NSwag in the back-end repo, output committed to a NuGet/npm package consumed by the front-end. Either way, the front-end build runs after codegen; if the API changed, the front-end build fails with type errors, which is exactly the "breaking change detected" signal you want. Pin the API version in the URL (`/v1/openapi.json`) so a breaking change on the API doesn't surprise the front-end mid-build.

**Trade-off:** Generated code in the repo vs. ephemeral codegen — the second is cleaner but harder to debug.

**Real project:** A team that moved from hand-written TypeScript types to generated ones eliminated an entire class of "wrong field name" bugs.

### 6. How do you keep examples in OpenAPI accurate when the schema changes?

**Why this matters:** Examples are the most-read part of the doc and the most likely to be wrong.

**Answer:** Generate examples from real integration tests. Instead of `[SwaggerRequestExample]` attributes with hand-written JSON, run a Swashbuckle/NSwag operation filter or a custom CI step that captures the request/response of a passing integration test and writes it into the OpenAPI document as an `example`. If the test breaks, the example doesn't get updated, but at least nothing wrong gets shipped. Alternative: use Verify or Snapshooter on response bodies and reference the snapshots in your OpenAPI examples.

**Trade-off:** Extra build complexity; pays off when examples stop lying.

**Real project:** A team's docs site auto-published examples extracted from integration tests; partner-reported "your example doesn't work" tickets dropped to zero.

### 7. How does API versioning integrate with the OpenAPI document?

**Why this matters:** Mismatched versioning + docs = confused consumers.

**Answer:** One OpenAPI document per API version. With `Asp.Versioning.Mvc.ApiExplorer`, register an `IConfigureOptions<SwaggerGenOptions>` that iterates `IApiVersionDescriptionProvider.ApiVersionDescriptions` and calls `SwaggerDoc("v1", ...)` and `SwaggerDoc("v2", ...)`. Swagger UI then has a dropdown to switch between them. Each operation appears only in the version that hosts it. Deprecated versions can be marked in the `OpenApiInfo.Description`. See [api-versioning.md](api-versioning.md) for the routing side.

**Trade-off:** More documents to maintain, but each is clear.

**Real project:** A team with three live versions had a Swagger UI dropdown for each; new hires picked the latest version with zero training.

### 8. How do you validate the generated OpenAPI document itself?

**Why this matters:** A broken OpenAPI doc silently breaks every downstream tool.

**Answer:** In CI, run `spectral lint ./openapi.json` (Stoplight Spectral) against the generated document. Spectral catches missing `description`, inconsistent `operationId`, missing examples, unused schemas, and style violations defined in a `.spectral.yml`. Pair with `openapi-cli validate` for raw spec compliance. For breaking-change detection between versions, `oasdiff` compares two OpenAPI documents and reports breaking, non-breaking, and unclassified changes — gate PRs on no-breaking-changes-without-version-bump.

**Trade-off:** Adds 30–60 seconds to CI; worth it for the bugs caught.

**Real project:** A team added `spectral lint` and immediately caught 40 endpoints with missing summaries; the rule "every operation must have a summary" was enforced going forward.

## Summary Checklist

- [ ] I generate an OpenAPI document for every public API (Swashbuckle, NSwag, or .NET 9 OpenAPI).
- [ ] I add `[ProducesResponseType]` for every status code on every endpoint.
- [ ] I return `ActionResult<T>` or typed results, never raw `IActionResult`.
- [ ] I document security schemes (Bearer JWT, OAuth2) so Swagger UI's Try-It-Out works.
- [ ] I generate XML doc comments and include them in the OpenAPI build.
- [ ] I gate Swagger UI by environment or auth in production.
- [ ] I produce one OpenAPI document per API version.
- [ ] I generate typed clients in CI and treat build failure as breaking-change detection.
- [ ] I lint the OpenAPI document with Spectral or equivalent.
- [ ] I publish the OpenAPI JSON as a CI artifact for downstream teams and APIM import.
