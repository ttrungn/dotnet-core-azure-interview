# Authentication and Authorization

## What It Is

**Authentication (AuthN)** answers "who are you?" — verifying the caller's identity from a credential (password, token, certificate, managed identity). **Authorization (AuthZ)** answers "what may you do?" — deciding whether an authenticated caller is allowed to perform a specific action on a specific resource. They are two distinct middleware stages in ASP.NET Core, in that order.

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(/* ... */);

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanApproveRefunds", p => p.RequireClaim("permission", "refund.approve"));
});

app.UseAuthentication(); // first — populates HttpContext.User
app.UseAuthorization();  // second — enforces policies
```

The output of authentication is a `ClaimsPrincipal` on `HttpContext.User`. Authorization reads claims and applies rules. Failing AuthN returns `401 Unauthorized`. Failing AuthZ returns `403 Forbidden`. Mixing these up is the single most common security bug.

## Why It Exists

Before ASP.NET Core's unified authentication/authorization stack, .NET apps had `[Authorize]` + `web.config` `<authorization>` rules + Membership/Identity (or hand-rolled session lookups). The problems:

- **Tight coupling to roles** — every "if user is admin" check meant a role; permissions exploded into hundreds of roles like `OrderAdmin_Region1_Tier2`.
- **No clean way to express "user can edit *their* order"** — role checks don't know about resource ownership.
- **Inconsistent identity model** — Forms auth, Windows auth, custom tokens each populated `IPrincipal` differently.
- **Hard to test** — authorization rules were intertwined with HTTP and database state.

The modern stack — `Authentication` schemes + `Authorization` policies + `IAuthorizationRequirement`/`AuthorizationHandler<TRequirement, TResource>` — gives a clean separation: schemes know how to verify credentials, policies declare rules, handlers implement them, and resource-based authorization handles "user can edit this specific order."

## When To Use It

**Every public endpoint and message handler must answer both questions.** There is no "I don't need auth for this" except for genuinely anonymous endpoints (health checks, public landing pages, login endpoints).

Use **roles** for coarse-grained, slow-changing groupings (`Admin`, `Support`, `Customer`).

Use **claims** for boolean attributes carried in the token (`tenant`, `country`, `mfa_completed`).

Use **policies** when the decision is more complex than a single claim — composing multiple claims, calling a service, checking a feature flag, evaluating time-of-day.

Use **resource-based authorization** (`IAuthorizationService.AuthorizeAsync(user, resource, policy)`) when the decision depends on the specific entity ("user can read order X iff order.CustomerId == user.Id").

Use **`[AllowAnonymous]`** sparingly and explicitly — and only after the global default policy requires authentication.

## Why It Is Important

This is the single highest-leverage area for security and compliance:

1. **Data isolation** — multi-tenant SaaS lives or dies by correct authorization. A cross-tenant leak is front-page news.
2. **Compliance** — SOC 2, PCI-DSS, HIPAA all require role-based access controls, audit trails, and least privilege.
3. **Blast radius** — a leaked token for an over-privileged account compromises far more than one for a least-privileged account.
4. **Auditability** — every action must be traceable to an identified caller, with the policy that allowed it.

In Azure, this layer integrates with Entra ID (formerly Azure AD), Managed Identities, RBAC roles on Azure resources, and Conditional Access policies — your API's authorization is a piece of a larger zero-trust posture.

## How It's Used in C# / .NET

### 1. AddAuthentication — schemes

A scheme is a way of verifying credentials. You can register many; one is the default.

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenantId}/v2.0";
        options.Audience = "api://orders-api";
    })
    .AddCookie() // for the admin portal hosted in the same app
    .AddPolicyScheme("Smart", "Smart", options =>
    {
        // Choose scheme per request: bearer for API, cookie for browser
        options.ForwardDefaultSelector = ctx =>
            ctx.Request.Headers.Authorization.ToString().StartsWith("Bearer ")
                ? JwtBearerDefaults.AuthenticationScheme
                : CookieAuthenticationDefaults.AuthenticationScheme;
    });
```

See [jwt-authentication.md](jwt-authentication.md) for JwtBearer specifics.

### 2. AddAuthorization — policies and defaults

```csharp
builder.Services.AddAuthorization(options =>
{
    // Secure by default — every endpoint requires authentication unless [AllowAnonymous]
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    options.AddPolicy("Orders.Read",  p => p.RequireScope("Orders.Read"));
    options.AddPolicy("Orders.Write", p => p.RequireScope("Orders.Write"));

    options.AddPolicy("CanApproveRefunds", p =>
        p.RequireClaim("permission", "refund.approve")
         .RequireClaim("mfa_completed", "true"));

    options.AddPolicy("InternalOnly", p =>
        p.RequireAssertion(ctx =>
            ctx.User.FindFirst("email")?.Value?.EndsWith("@contoso.com") == true));
});
```

### 3. Roles vs claims vs policies — when to use which

| Mechanism | When to use                                | Example                                       |
|-----------|--------------------------------------------|-----------------------------------------------|
| **Role**  | Coarse, persona-based, slow to change      | `[Authorize(Roles = "Admin")]`                |
| **Claim** | Single attribute the token already carries | `RequireClaim("country", "US")`               |
| **Policy**| Composite or computed rule                 | `[Authorize(Policy = "CanApproveRefunds")]`   |
| **Resource-based** | Decision depends on the specific entity | `_authz.AuthorizeAsync(user, order, "Owner")` |

The progression as systems grow: roles → claims → policies → resource-based. Most production code converges on policies for declarative rules and resource-based for "is this *my* thing?" checks.

### 4. Requirements and handlers — custom authorization logic

```csharp
// Requirement — a marker
public sealed class MinimumPurchaseRequirement(decimal amount) : IAuthorizationRequirement
{
    public decimal Amount { get; } = amount;
}

// Handler — actual rule
public sealed class MinimumPurchaseHandler(ICustomerLifetimeValue ltv)
    : AuthorizationHandler<MinimumPurchaseRequirement>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumPurchaseRequirement requirement)
    {
        var userId = context.User.FindFirstValue("oid");
        if (userId is null) return;

        var total = await ltv.GetLifetimeTotalAsync(Guid.Parse(userId));
        if (total >= requirement.Amount)
            context.Succeed(requirement);
    }
}

// Registration + policy
builder.Services.AddScoped<IAuthorizationHandler, MinimumPurchaseHandler>();
builder.Services.AddAuthorization(opt =>
    opt.AddPolicy("LoyaltyTier", p => p.Requirements.Add(new MinimumPurchaseRequirement(10_000m))));
```

### 5. Resource-based authorization

```csharp
public sealed class OrderOwnerRequirement : IAuthorizationRequirement;

public sealed class OrderOwnerHandler : AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOwnerRequirement requirement,
        Order order)
    {
        var userId = context.User.FindFirstValue("oid");
        if (userId == order.CustomerId.ToString() ||
            context.User.IsInRole("SupportAgent"))
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

// In the endpoint
[HttpGet("/api/v1/orders/{id:guid}")]
public async Task<IActionResult> Get(Guid id, IAuthorizationService authz, CancellationToken ct)
{
    var order = await _orders.GetAsync(id, ct);
    if (order is null) return NotFound();

    var result = await authz.AuthorizeAsync(User, order, new OrderOwnerRequirement());
    if (!result.Succeeded) return Forbid();

    return Ok(_mapper.Map<OrderDto>(order));
}
```

This is the **only** correct way to express "user can read their own orders but support agents can read any." Putting it in a SQL `WHERE` clause works for queries but not for single-resource GETs.

### 6. Minimal API style

```csharp
app.MapGet("/api/v1/orders/{id:guid}", GetOrderAsync)
   .RequireAuthorization("Orders.Read");

app.MapPost("/api/v1/refunds/{id:guid}/approve", ApproveRefundAsync)
   .RequireAuthorization("CanApproveRefunds");
```

## Advantages

- Clean separation: AuthN (who) vs AuthZ (what).
- Declarative — `[Authorize(Policy = "X")]` is readable, testable, and discoverable.
- Multiple schemes coexist in one app (JWT for API, cookie for admin portal).
- Resource-based authorization handles ownership cleanly.
- First-class support in .NET — works with controllers, Minimal APIs, gRPC, SignalR.

## Disadvantages

- Easy to get wrong silently — forgetting `[Authorize]` on one endpoint creates a hole.
- Resource-based authorization adds a service call per request — perf cost on hot paths.
- Policy explosion — too many fine-grained policies become hard to reason about.
- Migration from role-based to policy-based is painful in large codebases.
- The `[Authorize]`/`[AllowAnonymous]` ordering rules in inheritance are subtle.

## Common Mistakes

### 1. Forgetting `[Authorize]` on a single endpoint

```csharp
// BAD — controller has [Authorize] but this one is missing it after a copy-paste
[HttpPost("/refunds/{id:guid}/approve")]
public async Task<IActionResult> Approve(Guid id) { /* ... */ }
```

**Fix:** set `FallbackPolicy` to require authentication. Then anonymous endpoints must opt in with `[AllowAnonymous]`.

```csharp
options.FallbackPolicy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build();
```

### 2. Treating authentication as authorization

```csharp
// BAD — every signed-in user can approve refunds
[Authorize]
public async Task<IActionResult> ApproveRefund(Guid id) { /* ... */ }
```

**Fix:** require a specific policy.

```csharp
[Authorize(Policy = "CanApproveRefunds")]
public async Task<IActionResult> ApproveRefund(Guid id) { /* ... */ }
```

### 3. Reusing role names across apps

`Admin` in the orders service shouldn't mean `Admin` in the payments service.

**Fix:** scope role names per app (`Orders.Admin`, `Payments.Admin`), or use scopes (`api://orders-api/Orders.Write`). With Entra ID, define App Roles on the API's app registration so the `roles` claim only contains roles from that API.

### 4. Resource ownership check in code instead of the database

```csharp
// BAD — TOCTOU: the order can be reassigned between Get and Update
var order = await _db.Orders.FindAsync(id);
if (order.CustomerId != userId) return Forbid();
order.Notes = newNotes;
await _db.SaveChangesAsync();
```

**Fix:** combine ownership into the query.

```csharp
var rows = await _db.Orders
    .Where(o => o.Id == id && o.CustomerId == userId)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Notes, newNotes));
if (rows == 0) return Forbid(); // or NotFound — pick one and be consistent
```

For complex rules, still use `IAuthorizationService.AuthorizeAsync` on the loaded entity inside the same transaction.

### 5. Returning 404 when 403 is more honest (or the reverse)

- Returning `403` when the resource doesn't exist leaks existence.
- Returning `404` when the user just lacks permission can confuse legitimate UI.

**Fix:** decide a policy and apply it consistently. Common choice: `404` for unowned/unknown resources (don't leak existence to attackers), `403` only when the user *knows* about the resource but can't act on it (e.g., the resource is linked from their own dashboard).

### 6. Mixing `[Authorize(Roles = "X")]` and `[Authorize(Policy = "Y")]` carelessly

Multiple `[Authorize]` attributes combine with **AND**, not OR. So `[Authorize(Roles="Admin")] [Authorize(Policy="MfaCompleted")]` requires both. If you want OR semantics, write a single policy that does the OR.

### 7. Putting authorization logic in the domain

```csharp
// BAD — domain entity depends on HTTP user
public class Order
{
    public void Cancel(ClaimsPrincipal user) { /* ... */ }
}
```

**Fix:** the domain enforces invariants (`order.Status == Pending`), the application layer enforces authorization (`authz.AuthorizeAsync(user, order, "CanCancel")`). The domain stays pure.

## Best Practices

- Set a global `FallbackPolicy` requiring authentication. Make endpoints opt out, not opt in.
- Prefer **policies** over roles for new code; roles are fine for personas.
- Use **resource-based authorization** for ownership and per-entity decisions.
- Scope role and scope names per API (no shared `Admin`).
- Keep policies in one file (`AuthorizationPolicies.cs`) so they're easy to audit.
- Test policies as unit tests — they're handler classes.
- Log authorization decisions (success and failure) for high-value actions.
- Pair with [jwt-authentication.md](jwt-authentication.md) — authentication is the prerequisite.
- Never put `ClaimsPrincipal` or `HttpContext` in the domain layer.
- Decide and document your `404` vs `403` policy.

## Related Concepts

- [jwt-authentication.md](jwt-authentication.md) — how the `ClaimsPrincipal` arrives.
- [request-pipeline.md](request-pipeline.md) — `UseAuthentication` then `UseAuthorization` ordering.
- [error-handling-and-problem-details.md](error-handling-and-problem-details.md) — 401 vs 403 response shape.
- [../azure/azure-key-vault.md](../azure/azure-key-vault.md) — secrets for app-only auth.
- [../azure/app-service.md](../azure/app-service.md) — managed identity to access Azure resources.
- [../architecture/clean-architecture.md](../architecture/clean-architecture.md) — authorization lives in the application layer.
- [../testing-quality/unit-testing.md](../testing-quality/unit-testing.md) — authorization handlers are unit-test targets.

## Real-World Usage

### Multi-tenant SaaS

Every request carries `tid` (tenant id) in the JWT. The authorization layer has a `SameTenantRequirement` checked on every resource access: `resource.TenantId == user.TenantId`. EF Core global query filters enforce tenant isolation at the database level. Both layers must agree — defense in depth.

### Microservices with shared identity

All services trust the same Entra ID tenant. Each service registers as a separate app registration with its own audience (`api://orders-api`, `api://payments-api`). Scopes (`Orders.Write`, `Payments.Refund`) are defined per app. Calling services request a token with the specific scopes they need via `ITokenAcquisition`.

### Admin portal + customer API in one app

The admin portal uses cookie auth (browser flow, anti-forgery tokens), the API uses JWT bearer. A `PolicyScheme` selects based on the `Authorization` header. Both end up populating `HttpContext.User` with claims; downstream policies don't care which scheme produced them.

### Resource-based authorization for documents

A document management system has `Read`, `Write`, `Share`, `Delete` requirements per `Document`. The handler queries an `IDocumentAclService` that looks up ACL entries (cached per request). Each endpoint calls `authz.AuthorizeAsync(User, document, new ReadRequirement())` before returning content.

### Audit logging

Every action that changes state writes an audit event: `{actor: oid, action: 'refund.approve', resource: orderId, policy: 'CanApproveRefunds', result: 'Allowed', timestamp}`. Stored in append-only Azure Table Storage. Compliance reviewers can answer "who approved this refund and under which policy?" in seconds.

## Code Example — Before and After

### Before — role spaghetti, anonymous holes, ownership in code

```csharp
[ApiController]
[Route("api/orders")]
[Authorize(Roles = "Customer,Admin,Support")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var order = await _db.Orders.FindAsync(id);
        if (order == null) return NotFound();
        var userId = User.FindFirst("sub")?.Value;
        if (User.IsInRole("Customer") && order.CustomerId.ToString() != userId)
            return Forbid();
        return Ok(order);
    }

    [HttpPost("{id:guid}/refund")]
    public async Task<IActionResult> Refund(Guid id) // forgot [Authorize] override — accidentally any role allowed
    {
        // ...
    }
}
```

Problems: role list duplicated everywhere, ownership check in code (TOCTOU risk), one endpoint missing the proper authorization, entity returned directly.

### After — fallback policy, named policies, resource-based ownership

```csharp
// Program.cs
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    // Secure by default
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    options.AddPolicy("Orders.Read",  p => p.RequireScope("Orders.Read"));
    options.AddPolicy("Orders.Write", p => p.RequireScope("Orders.Write"));

    options.AddPolicy("CanApproveRefunds", p =>
        p.RequireClaim("permission", "refund.approve")
         .RequireClaim("mfa_completed", "true"));
});

builder.Services.AddScoped<IAuthorizationHandler, OrderOwnerHandler>();
```

```csharp
// Authorization/OrderOwnerHandler.cs
namespace OrdersApi.Authorization;

public sealed class OrderOwnerRequirement : IAuthorizationRequirement;

public sealed class OrderOwnerHandler : AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderOwnerRequirement requirement,
        Order order)
    {
        var userId = context.User.FindFirstValue("oid");
        var isOwner = userId == order.CustomerId.ToString();
        var isSupport = context.User.IsInRole("SupportAgent");
        if (isOwner || isSupport)
            context.Succeed(requirement);
        return Task.CompletedTask;
    }
}
```

```csharp
// Controllers/V1/OrdersController.cs
namespace OrdersApi.Controllers.V1;

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public sealed class OrdersController(
    IOrderRepository orders,
    IAuthorizationService authz,
    ILogger<OrdersController> logger) : ControllerBase
{
    [HttpGet("{id:guid}")]
    [Authorize(Policy = "Orders.Read")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken ct)
    {
        var order = await orders.GetAsync(id, ct);
        if (order is null) return NotFound();

        var result = await authz.AuthorizeAsync(User, order, new OrderOwnerRequirement());
        if (!result.Succeeded)
        {
            logger.LogInformation("AuthZ denied: user {User} order {Order}",
                User.FindFirstValue("oid"), id);
            return NotFound(); // don't leak existence to non-owners
        }

        return Ok(order.ToDto());
    }

    [HttpPost("{id:guid}/refund")]
    [Authorize(Policy = "CanApproveRefunds")]
    [ProducesResponseType(StatusCodes.Status202Accepted)]
    public async Task<IActionResult> Refund(
        Guid id,
        [FromBody] RefundRequest request,
        IMediator mediator,
        CancellationToken ct)
    {
        await mediator.Send(new ApproveRefundCommand(id, request.Amount,
            User.FindFirstValue("oid")!), ct);
        return Accepted();
    }
}
```

Now: the fallback policy makes accidents impossible, each endpoint declares the exact policy it needs, ownership is enforced by a testable handler, refund approval requires both permission + MFA, and the response doesn't leak resource existence.

## Interview Questions and Answers

### 1. What's the difference between 401 and 403, and which should you return when?

**Why this matters:** Confusing them is the most common security UX bug.

**Answer:** **401 Unauthorized** means "I don't know who you are" — token missing, expired, or invalid. The response should include a `WWW-Authenticate` header (e.g., `Bearer realm="api"`). **403 Forbidden** means "I know who you are, but you can't do this." The two are not interchangeable. A signed-in user who lacks permission gets `403`. An anonymous user who hits a protected endpoint gets `401`. In ASP.NET Core, `Unauthorized()` returns 401, `Forbid()` returns 403. Optional refinement for security: return `404` instead of `403` when the resource exists but the user has no relation to it — this avoids leaking resource existence to enumeration attacks.

**Trade-off:** Returning `404` instead of `403` is more secure but slightly less helpful to legitimate users.

**Real project:** A team's mobile app showed "you don't have permission" instead of "please log in" for 30% of errors — turned out the server returned `403` for both cases. Fixing it to 401 vs 403 dropped login-confusion support tickets significantly.

### 2. When would you use a policy instead of `[Authorize(Roles = "...")]`?

**Why this matters:** Tests modern AuthZ design.

**Answer:** Almost always for non-trivial rules. Roles are fine for coarse personas ("admin sees everything"). Policies cover everything else: claim combinations, computed rules, dependencies on services. Example: "must have `permission=refund.approve` AND `mfa_completed=true` AND the refund amount is below their approval limit." That's a single policy with one custom `IAuthorizationRequirement` and a handler that reads claims and calls an approval-limit service. Roles can't express that. The bonus: policies are unit-testable handler classes; `[Authorize(Roles="X")]` is just a string.

**Trade-off:** Policies are more verbose for simple cases; if the rule really is "user has role X," `[Authorize(Roles="X")]` is fine.

**Real project:** A payments service replaced 14 role-based attributes with 5 policies; new requirements ("require MFA for refunds > $1k") became a one-line policy change.

### 3. How do you authorize "user can read their own orders but support agents can read any"?

**Why this matters:** Classic resource-based authorization question.

**Answer:** **Resource-based authorization.** Define an `OrderOwnerRequirement` and a handler that takes `(AuthorizationHandlerContext, OrderOwnerRequirement, Order)`. The handler checks `order.CustomerId == user.oid` OR `user.IsInRole("SupportAgent")` and calls `context.Succeed(requirement)`. The endpoint loads the order and calls `await _authz.AuthorizeAsync(User, order, new OrderOwnerRequirement())`. For list endpoints, push the predicate into the query (`Where(o => o.CustomerId == userId || isSupport)`) so you don't materialize unauthorized rows.

**Trade-off:** Adds a service call per resource access; cache hot resources.

**Real project:** A B2B portal had hundreds of endpoints with hand-rolled "if owner or admin" checks. Migrating to one `ResourceOwnerHandler` deleted ~2000 lines and caught three accidental bypasses in the process.

### 4. How do you secure an endpoint by default so a missing `[Authorize]` doesn't create a hole?

**Why this matters:** This is the single most effective AuthZ hardening change.

**Answer:** Set a `FallbackPolicy` that requires authentication. Every endpoint without an explicit `[Authorize]` or `[AllowAnonymous]` gets the fallback applied. Anonymous endpoints must opt out with `[AllowAnonymous]`, which is grep-able and code-review-able. Combine with `DefaultPolicy` to make `[Authorize]` (with no policy name) also require a specific scope. Result: forgetting `[Authorize]` no longer creates a hole; forgetting `[AllowAnonymous]` on a public endpoint produces a 401, which someone notices immediately.

**Trade-off:** A few public endpoints need `[AllowAnonymous]` you wouldn't have otherwise written.

**Real project:** A team's security audit flagged 6 endpoints missing `[Authorize]`. Adding `FallbackPolicy` made future occurrences impossible.

### 5. How do you test authorization policies?

**Why this matters:** Tests testability discipline.

**Answer:** Three layers. **(1) Unit-test the handler** — instantiate it, build a `ClaimsPrincipal` with the relevant claims, build the resource, build `AuthorizationHandlerContext`, call `HandleAsync`, assert `context.HasSucceeded`. **(2) Integration tests via `WebApplicationFactory<Program>`** — replace `JwtBearer` with a test scheme that produces any principal you want, then hit the endpoint and assert 200 vs 403. **(3) Contract tests** — for each endpoint, enumerate the matrix of (anonymous, authenticated-no-permission, authenticated-with-permission, owner, support) and assert the expected status. Skipping any of these means a regression ships.

**Trade-off:** Test surface grows linearly with endpoints; worth it for security-critical paths.

**Real project:** A team added the integration-test matrix and immediately caught two endpoints that allowed anonymous access in a refactor PR.

### 6. How do you handle authorization when calling another microservice?

**Why this matters:** Microservices auth is its own discipline.

**Answer:** Two patterns. **(a) On-behalf-of (OBO)** — the calling service exchanges the user's token for a new token scoped to the downstream API, preserving user identity through the chain. Use `Microsoft.Identity.Web`'s `ITokenAcquisition.GetAccessTokenForUserAsync` with the downstream scope. **(b) Client credentials with managed identity** — the calling service authenticates as itself with a managed identity, and the downstream service authorizes "this service is allowed to do X." Use `DefaultAzureCredential.GetTokenAsync` with `api://downstream/.default`. Pick OBO when the action represents the user's intent (e.g., reading user data); pick client credentials when the action is system-to-system (e.g., a background reconciliation job).

**Trade-off:** OBO preserves audit identity but adds token-exchange latency; client credentials is faster but loses the user context.

**Real project:** An orders service called payments service via OBO for user-initiated refunds (auditable as the user) but used managed identity for nightly reconciliation jobs (auditable as the service).

### 7. How do you migrate from role-based auth (10+ roles) to policy-based auth without downtime?

**Why this matters:** Real systems migrate, they don't rewrite.

**Answer:** Incremental. **Step 1:** define the target policies and wire them as additional checks alongside existing role checks (`[Authorize(Roles = "Admin")] [Authorize(Policy = "CanX")]` — both pass). **Step 2:** issue tokens with the new claims/permissions from the IdP, alongside existing roles. **Step 3:** verify in telemetry that every request passing the role check also passes the new policy. **Step 4:** remove the role attribute, keep the policy. **Step 5:** stop issuing the deprecated role claim from the IdP. Each step is reversible. Never do a big-bang switch — there will be edge cases your audit didn't catch.

**Trade-off:** Months of dual-running; protects against missed cases.

**Real project:** A retail platform migrated 60+ endpoints over a quarter using this pattern; only two cases needed code fixes when the policy was stricter than the role.

### 8. A user complains they get 403 sometimes and 200 other times for the same URL. What's likely wrong?

**Why this matters:** Real-world AuthZ debugging.

**Answer:** Several likely causes. **(1) Stale token** — token issued before the user's permissions changed; missing the new claim. Fix: shorter access-token lifetimes + refresh. **(2) Caching** — a CDN or response cache is returning a 403 from another user's request. Fix: never cache authenticated responses, or include user-id in the cache key. **(3) Per-request resource state** — the authorization handler depends on data that changes (e.g., "user must be in the workspace" — they got removed). Fix: log the resolved decision with claims so you can replay. **(4) Multiple instances with inconsistent state** — different replicas of an authorization service. Fix: shared cache or sticky routing. Diagnostic: log every AuthZ failure with `oid`, `policy`, `resource`, and the relevant claim values.

**Trade-off:** More logging vs. ability to actually debug intermittent issues.

**Real project:** A team chasing intermittent 403s discovered a CDN was caching authenticated responses for 60 seconds because the `Cache-Control: private` header was missing. One-line fix.

## Summary Checklist

- [ ] I distinguish authentication (who) from authorization (what) and return 401 vs 403 correctly.
- [ ] I set a global `FallbackPolicy` requiring authentication; endpoints opt out with `[AllowAnonymous]`.
- [ ] I prefer **policies** over inline role checks for non-trivial rules.
- [ ] I use **resource-based authorization** (`IAuthorizationService.AuthorizeAsync(user, resource, requirement)`) for ownership.
- [ ] I scope role and scope names per API (no shared `Admin` across services).
- [ ] I keep authorization out of the domain layer.
- [ ] I unit-test authorization handlers and integration-test the (anonymous, user, owner, admin) matrix.
- [ ] I log authorization decisions on high-value actions with actor, policy, and resource.
- [ ] I decide a 404-vs-403 convention for "user doesn't own this resource" and apply it consistently.
- [ ] I use OBO for user-initiated cross-service calls and managed identity for system-to-system.
