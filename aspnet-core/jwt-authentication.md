# JWT Authentication

## What It Is

JWT (JSON Web Token, [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)) is a compact, URL-safe token format used to carry identity and claims between an identity provider (Microsoft Entra ID, Auth0, Okta, IdentityServer, Keycloak) and an API. The token is three Base64Url segments separated by dots:

```
eyJhbGciOi...  .  eyJzdWIiOi...  .  X9F8eY...
   header           payload            signature
```

The **header** declares the signing algorithm (`RS256`, `ES256`, rarely `HS256`). The **payload** holds claims: `sub` (subject = user id), `iss` (issuer), `aud` (audience), `exp` (expiry), `iat` (issued-at), plus app-specific claims like `roles`, `scope`, `tid` (tenant), `oid` (object id), or `permission`. The **signature** lets the API prove the token was issued by a trusted authority and was not modified.

```csharp
// API only needs to validate the token — it does not need to call the IdP per request.
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";
        options.Audience = "api://orders-api";
    });
```

JWT is **bearer** authentication: whoever holds the token can use it. Treat it like a password.

## Why It Exists

Before JWT and OAuth 2.0 / OpenID Connect became standard, .NET APIs typically used:

- **Forms / cookie auth** — fine for server-rendered web apps, useless for mobile, SPAs, and machine-to-machine calls.
- **WS-Security / SAML** — heavyweight XML, painful for SPAs and partner integrations.
- **Custom API keys** — opaque, hard to scope, hard to revoke, often hard-coded in client apps.

The pain points were sharp: every team rolled their own identity, sessions were stateful (couldn't scale horizontally without sticky sessions or Redis), token revocation was inconsistent, and there was no standard way to express "this token can do X on resource Y."

JWT + OAuth 2.0 solved this:

- **Stateless** — the API verifies the signature with a public key (JWKS); no database lookup per request.
- **Self-describing** — claims travel with the token.
- **Cross-platform** — every language has libraries.
- **Federated** — one IdP can issue tokens for many APIs in many languages.

In .NET, `Microsoft.AspNetCore.Authentication.JwtBearer` made this a one-line config since ASP.NET Core 2.0.

## When To Use It

**Use JWT bearer authentication for:**

- Public APIs consumed by SPAs (React, Angular, Blazor WASM), mobile apps, or partner services.
- Microservices that need to propagate caller identity (`Authorization: Bearer <token>` flows through).
- Machine-to-machine calls using OAuth 2.0 client credentials flow.
- Multi-tenant SaaS where the `tid` claim identifies the tenant.

**Do not use JWT for:**

- **Server-rendered web apps with the same origin** — cookie auth (`AddCookie`) is simpler, integrates with anti-forgery, and gets `HttpOnly`/`Secure`/`SameSite` for free.
- **Sessions you need to revoke instantly** — JWTs are valid until they expire. Either accept short expiry + refresh tokens, maintain a server-side denylist (kills statelessness), or use reference tokens.
- **Storing sensitive data** — payload is Base64-encoded, not encrypted. Anyone who reads the token reads the claims. Use JWE (encrypted JWT) only if you have a hard requirement.
- **Authorization decisions by themselves** — the token says *who* the caller is; *what* they can do is a policy decision (see [authentication-and-authorization.md](authentication-and-authorization.md)).

## Why It Is Important

JWT bearer auth is the default identity mechanism for any non-trivial .NET API today. Getting it right drives:

1. **Scalability** — stateless verification means an API replica can handle any request without sticky sessions or distributed cache.
2. **Security posture** — short-lived access tokens + refresh tokens + key rotation via JWKS = minimal blast radius if a token leaks.
3. **Interoperability** — your API speaks OAuth, so any IdP, any SDK, any partner can integrate.
4. **Audit and compliance** — every request carries `sub`, `tid`, and `oid`, so logs and traces have real identity, not just IPs.

In Azure, this is how App Service / Container Apps / AKS-hosted APIs authenticate calls from Functions, Logic Apps, APIM, and front ends backed by Entra ID — all without ever holding a password.

## How It's Used in C# / .NET

### 1. The basic JwtBearer setup

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"]; // e.g. https://login.microsoftonline.com/<tenant>/v2.0
        options.Audience = builder.Configuration["Auth:Audience"];   // e.g. api://orders-api
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30), // default is 5 minutes — too generous
            NameClaimType = "name",
            RoleClaimType = "roles"
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                ctx.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>()
                    .LogWarning(ctx.Exception, "JWT validation failed");
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

The `Authority` URL is fetched at startup; the framework downloads the OpenID Connect discovery document (`/.well-known/openid-configuration`) and the signing keys (JWKS). Keys are cached and refreshed automatically — no app restart needed when the IdP rotates keys.

### 2. Microsoft Entra ID (formerly Azure AD)

Use the higher-level `Microsoft.Identity.Web` package — it wraps `JwtBearer` with sensible defaults, downstream token acquisition, and on-behalf-of flow:

```csharp
// NuGet: Microsoft.Identity.Web
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));
```

```json
// appsettings.json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "common",
  "ClientId": "api://orders-api"
}
```

This gives you `[Authorize]`, `[RequiredScope("Orders.Write")]`, and `ITokenAcquisition` for calling downstream APIs as the user.

### 3. Reading claims in your code

```csharp
[Authorize]
[HttpPost("/api/v1/orders")]
public async Task<IActionResult> Create(CreateOrderRequest request, CancellationToken ct)
{
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier)
              ?? User.FindFirstValue("sub")
              ?? User.FindFirstValue("oid");
    var tenantId = User.FindFirstValue("tid");
    // ...
}
```

Wrap this in an `ICurrentUser` service so handlers don't take a hard dependency on `HttpContext`:

```csharp
public interface ICurrentUser
{
    Guid UserId { get; }
    Guid TenantId { get; }
    bool HasPermission(string permission);
}

internal sealed class HttpCurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    private ClaimsPrincipal Principal => accessor.HttpContext!.User;
    public Guid UserId => Guid.Parse(Principal.FindFirstValue("oid")!);
    public Guid TenantId => Guid.Parse(Principal.FindFirstValue("tid")!);
    public bool HasPermission(string p) =>
        Principal.FindAll("permission").Any(c => c.Value == p);
}
```

### 4. Refresh tokens

The API itself never sees refresh tokens — they are held by the client and exchanged at the IdP's `/token` endpoint for a new access token. Your job:

- Issue **short-lived** access tokens (10–60 minutes typical).
- Configure the IdP to issue refresh tokens with rotation (each use returns a new refresh token, old one revoked).
- For SPAs, use the Authorization Code flow with PKCE; store the refresh token in a backend-for-frontend (BFF), not in localStorage.

### 5. JwtSecurityTokenHandler vs JsonWebTokenHandler

The legacy `System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler` has been superseded by `Microsoft.IdentityModel.JsonWebTokens.JsonWebTokenHandler`. The newer one is faster, allocates less, and is the default in modern `JwtBearer`. It also renames claims more predictably — `JwtSecurityTokenHandler` mapped `sub` → `ClaimTypes.NameIdentifier`, which surprised many devs. To disable that mapping on the legacy handler:

```csharp
JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();
```

For new code, prefer `JsonWebTokenHandler` — it is the default starting with `Microsoft.AspNetCore.Authentication.JwtBearer` 7.0+ when you set `options.UseSecurityTokenValidators = false` (which is the default in 8.0+).

## Advantages

- Stateless — scales horizontally without a session store.
- Standardized — every language and IdP speaks JWT.
- Self-contained — claims travel with the token, no extra round-trip.
- Federation-friendly — one IdP, many APIs.
- First-class .NET support — `JwtBearer` and `Microsoft.Identity.Web` cover most cases.

## Disadvantages

- Hard to revoke before expiry (the core trade-off of statelessness).
- Tokens are visible to anyone who intercepts them — TLS everywhere is mandatory.
- Payload is **not encrypted**, only signed. Don't put PII or secrets in claims.
- Token size grows with claims — large tokens hit header-size limits in some proxies.
- Clock skew between IdP and API can cause spurious `IDX10223` failures.

## Common Mistakes

### 1. Putting sensitive data in the token

```csharp
// BAD — credit card last 4, internal employee ID, PHI
new Claim("payment_card", "4242"); 
```

**Fix:** the token payload is Base64, not encrypted. Put identifiers only. Look the rest up server-side. Use JWE if you genuinely need an encrypted token.

### 2. Disabling signature validation "for testing"

```csharp
// BAD — never ship this
options.TokenValidationParameters.ValidateIssuerSigningKey = false;
options.TokenValidationParameters.SignatureValidator = (token, _) => new JwtSecurityToken(token);
```

**Fix:** test against a real IdP (Entra ID, IdentityServer, Auth0 dev tenant) or a local one (`dotnet user-jwts`, IdentityServer Duende).

### 3. Long-lived access tokens, no refresh strategy

A 7-day access token means a stolen token works for 7 days. **Fix:** access token ≤ 1 hour; refresh token rotates; revoke refresh tokens server-side on logout or anomaly.

### 4. Trusting the `role` claim without scoping it to your API

```csharp
// BAD — accepts ANY role claim from ANY app the user is in
[Authorize(Roles = "Admin")]
```

**Fix:** scope roles per API. In Entra ID, define App Roles on **your** app registration. Validate `aud` matches your API. Use policies that check `scope` or `roles` together with audience.

### 5. Reading `HttpContext.User` deep in the domain

```csharp
// BAD — domain layer now depends on HTTP
public class OrderService
{
    public OrderService(IHttpContextAccessor accessor) { ... }
}
```

**Fix:** wrap identity in an `ICurrentUser` interface; pass it (or just the IDs) explicitly into handlers.

### 6. Wide `ClockSkew`

```csharp
options.TokenValidationParameters.ClockSkew = TimeSpan.FromMinutes(5); // default
```

**Fix:** five minutes lets an expired token live five extra minutes. Tighten to 30 seconds; sync your servers with NTP.

### 7. Logging the token

```csharp
logger.LogInformation("Request from token {Token}", Request.Headers.Authorization);
```

**Fix:** never. The token is a credential. Log claims (`sub`, `oid`, `tid`) instead.

## Best Practices

- Use `Microsoft.Identity.Web` if you're on Entra ID.
- Validate **all four**: issuer, audience, lifetime, signature.
- Keep access tokens short; use refresh tokens with rotation.
- Set `RequireHttpsMetadata = true` in production.
- Tighten `ClockSkew` to ≤ 1 minute; sync clocks via NTP.
- Wrap claim access in `ICurrentUser`; never sprinkle `HttpContext` through the domain.
- Store secrets (client secrets, signing keys for HS256) in Key Vault, never in `appsettings.json`.
- Log auth failures at `Warning`, never log the token itself.
- For multi-tenant SaaS, always check `tid` matches the tenant in the URL/route.
- Pair with [authentication-and-authorization.md](authentication-and-authorization.md) — authentication is half the story.

## Related Concepts

- [authentication-and-authorization.md](authentication-and-authorization.md) — what to do *after* you've identified the caller.
- [error-handling-and-problem-details.md](error-handling-and-problem-details.md) — 401 vs 403 response shape.
- [../azure/azure-key-vault.md](../azure/azure-key-vault.md) — store signing keys and client secrets.
- [../azure/configuration-and-secrets-management.md](../azure/configuration-and-secrets-management.md) — bind `AzureAd` config from Key Vault.
- [request-pipeline.md](request-pipeline.md) — order of `UseAuthentication`/`UseAuthorization`.
- [logging-and-monitoring.md](logging-and-monitoring.md) — log identity claims, not tokens.

## Real-World Usage

### ASP.NET Core API behind Entra ID

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Orders.Read", p => p.RequireScope("Orders.Read"));
    options.AddPolicy("Orders.Write", p => p.RequireScope("Orders.Write"));
});

app.MapGet("/api/v1/orders/{id:guid}", GetOrder).RequireAuthorization("Orders.Read");
app.MapPost("/api/v1/orders",         CreateOrder).RequireAuthorization("Orders.Write");
```

### Azure Functions (isolated worker)

Add `Microsoft.Identity.Web` to the worker:

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices((ctx, services) =>
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddMicrosoftIdentityWebApi(ctx.Configuration.GetSection("AzureAd"));
        services.AddAuthorization();
    })
    .Build();
```

### Service-to-service with managed identity

A worker calling another API uses **client credentials** with a managed identity — no client secret to leak:

```csharp
var credential = new DefaultAzureCredential();
var token = await credential.GetTokenAsync(
    new TokenRequestContext(new[] { "api://orders-api/.default" }));
httpClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token.Token);
```

### Testing

Issue local dev JWTs without an IdP: `dotnet user-jwts create --scope Orders.Write`. For integration tests, replace `JwtBearer` with a test scheme that accepts any token and produces a fixed `ClaimsPrincipal`.

## Code Example — Before and After

### Before — cookie session, sticky, no machine-to-machine

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(opts =>
    {
        opts.LoginPath = "/login";
        opts.ExpireTimeSpan = TimeSpan.FromDays(30);
    });

[HttpPost("/login")]
public async Task<IActionResult> Login(string email, string password)
{
    var user = await _users.GetByEmailAsync(email);
    if (user == null || !_hasher.Verify(password, user.PasswordHash))
        return Unauthorized();

    var principal = new ClaimsPrincipal(
        new ClaimsIdentity(new[] { new Claim("sub", user.Id.ToString()) }, "Cookies"));
    await HttpContext.SignInAsync(principal);
    return Ok();
}
```

Problems: requires sticky sessions (or a shared cache) to scale; useless for mobile/SPA; no standard way for a partner to call the API; password lives in the app.

### After — JWT bearer with Entra ID, stateless, multi-client

```csharp
// Program.cs
using Microsoft.Identity.Web;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Orders.Read",  p => p.RequireScope("Orders.Read"));
    options.AddPolicy("Orders.Write", p => p.RequireScope("Orders.Write"));
    options.AddPolicy("Refunds.Approve", p =>
        p.RequireClaim("permission", "refund.approve"));
});

builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUser, HttpCurrentUser>();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();

app.MapPost("/api/v1/orders", CreateOrderAsync).RequireAuthorization("Orders.Write");
app.MapPost("/api/v1/refunds/{id:guid}/approve", ApproveRefundAsync)
   .RequireAuthorization("Refunds.Approve");

app.Run();
```

```csharp
// Endpoint handler — never touches HttpContext directly
internal static async Task<IResult> CreateOrderAsync(
    CreateOrderRequest request,
    IMediator mediator,
    ICurrentUser user,
    CancellationToken ct)
{
    var result = await mediator.Send(
        new CreateOrderCommand(user.UserId, user.TenantId, request.Lines), ct);
    return result.IsSuccess
        ? Results.Created($"/api/v1/orders/{result.Value}", new { id = result.Value })
        : Results.Problem(result.Error.Message, statusCode: result.Error.StatusCode);
}
```

```json
// appsettings.json — non-secret config
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "72f988bf-86f1-41af-91ab-2d7cd011db47",
  "ClientId": "api://orders-api"
}
```

Now the API is stateless, scales horizontally, accepts tokens from SPAs, mobile apps, partner backends, and other microservices using managed identity. Passwords never touch the API.

## Interview Questions and Answers

### 1. Walk me through what happens when a JWT-protected endpoint receives a request.

**Why this matters:** Tests whether the candidate actually understands the pipeline vs. having copy-pasted setup code.

**Answer:** (1) `AuthenticationMiddleware` inspects the `Authorization` header, extracts the bearer token. (2) `JwtBearerHandler` validates the token against `TokenValidationParameters`: signature (using public keys from JWKS, cached from the discovery doc), issuer (`iss` claim matches `Authority`), audience (`aud` matches `Audience`), and lifetime (`nbf` ≤ now ≤ `exp`, within `ClockSkew`). (3) On success, a `ClaimsPrincipal` is built from the token claims and assigned to `HttpContext.User`. (4) `AuthorizationMiddleware` runs the policies on the endpoint. (5) Endpoint runs. No call to the IdP per request — that's the whole point.

**Trade-off:** Statelessness vs. instant revocation.

**Real project:** A team confused by `401` errors learned that their proxy was stripping the `Authorization` header on long URLs — pipeline-level diagnosis only made sense once they understood the flow.

### 2. Why does `[Authorize(Roles = "Admin")]` sometimes silently fail with Entra ID tokens?

**Why this matters:** Common, frustrating, and exposes claim mapping issues.

**Answer:** Entra ID emits role claims under the claim type `roles` (plural). The default ASP.NET Core `Roles` check looks at `ClaimTypes.Role` (`http://schemas.microsoft.com/ws/2008/06/identity/claims/role`). They don't match. Fix: set `TokenValidationParameters.RoleClaimType = "roles"` in `JwtBearer` options. The `Microsoft.Identity.Web` package does this for you.

**Trade-off:** Implicit defaults bite when you mix authentication libraries.

**Real project:** A team spent half a day debugging this exact issue when migrating from `AddJwtBearer` to `AddMicrosoftIdentityWebApi` — and the other way around.

### 3. A user logs out but their JWT is still valid for 50 more minutes. How do you handle that?

**Why this matters:** Reveals understanding of stateless trade-offs.

**Answer:** Three real options. **(a)** Accept it — short access tokens (10–15 min) make this acceptable for most apps; logout clears the refresh token so they can't get a new access token. **(b)** Server-side denylist — on logout, write the token's `jti` to Redis with a TTL equal to `exp - now`; check on each request. Kills statelessness, hurts scale, but provides instant revocation. **(c)** Reference tokens via the IdP introspection endpoint (`/connect/introspect`) — every request calls the IdP. Most secure, slowest. Real systems usually pick (a) plus refresh-token revocation; (b) only for high-risk actions like banking.

**Trade-off:** Latency / scale vs. revocation immediacy.

**Real project:** A fintech app used short access tokens (5 min) and immediately invalidated refresh tokens on suspicious activity — got most of the benefit of (b) without the per-request lookup.

### 4. How would you secure a multi-tenant SaaS API with JWT?

**Why this matters:** Single-tenant auth is easy; tenant isolation is where breaches happen.

**Answer:** The token's `tid` (or a custom tenant claim) identifies the tenant. Three layers: **(1)** Validate `tid` is present and matches the tenant identified in the route or subdomain. **(2)** Every database query MUST filter by `TenantId` — enforce via EF Core global query filters and unit tests that fail if any query omits the filter. **(3)** Authorization policies that combine scope/role + tenant ownership ("user can read order X iff order.TenantId == user.TenantId"). Never trust the URL alone — always cross-check the token.

**Trade-off:** Extra plumbing in repositories vs. the catastrophic cost of a cross-tenant leak.

**Real project:** A team using EF global query filters caught a cross-tenant bug at code review when a `IgnoreQueryFilters()` was accidentally added; their integration tests asserting tenant isolation immediately failed.

### 5. The JwtBearer keys are downloaded at startup. What happens when Entra ID rotates them?

**Why this matters:** Many engineers think key rotation requires an app restart. It doesn't, but only if configured correctly.

**Answer:** `JwtBearerOptions` uses `ConfigurationManager<OpenIdConnectConfiguration>` which refreshes the discovery document automatically (default: every 24h, on validation failure). When Entra rotates keys, the next token signed with the new key triggers a refresh and validation succeeds. You don't need to restart. But: if you set `options.Configuration = ...` manually with a fixed value, you've disabled refresh and yes, you will get an outage at the next rotation. Don't do that.

**Trade-off:** The 24h refresh window means a brief window of "key not yet known" failures during rotation; tightening `AutomaticRefreshInterval` helps.

**Real project:** A team that hard-coded signing keys in config had a P1 outage during an Entra rotation. The fix was deleting their custom config and trusting the discovery document.

### 6. JwtSecurityTokenHandler or JsonWebTokenHandler?

**Why this matters:** Tests whether the candidate is current.

**Answer:** `JsonWebTokenHandler` for new code. It's faster, allocates less, doesn't do the legacy claim-name remapping (`sub` → `ClaimTypes.NameIdentifier`) that surprised everyone, and it's the default in `JwtBearer` 8.0+. `JwtSecurityTokenHandler` is still around for back-compat — if you see it in code that also does `DefaultInboundClaimTypeMap.Clear()`, you're looking at the legacy stack.

**Trade-off:** Migration requires re-reading claims by their original names (`sub`, `oid`) instead of `ClaimTypes.NameIdentifier`. Mostly mechanical.

**Real project:** A migration from the legacy handler cut JWT validation CPU by ~30% on a high-throughput API.

### 7. How do you do JWT auth in integration tests without spinning up Entra ID?

**Why this matters:** Tests pragmatic testing instincts.

**Answer:** Replace the `JwtBearer` scheme with a custom `AuthenticationHandler<TestSchemeOptions>` that builds a `ClaimsPrincipal` from headers in the test request. Register it in the `WebApplicationFactory<Program>` overrides. Tests set `Authorization: Test <user-id>` and the handler reads it. For finer control, use `dotnet user-jwts` to issue real signed tokens against a local key. Never call the real IdP from CI tests — flaky, slow, and rate-limited.

**Trade-off:** Test scheme bypasses real signature validation, so you should still have a smoke test that hits a deployed env with a real token.

**Real project:** A team's CI dropped from 12 min to 3 min after replacing real Entra calls with a test auth scheme.

### 8. A penetration tester says they decoded your token and read all the claims. Is that a vulnerability?

**Why this matters:** Confirms basic JWT literacy.

**Answer:** No — the payload is Base64Url encoded, not encrypted. Anyone with the token can decode it. The **signature** is what proves authenticity and prevents tampering. The takeaway is: don't put secrets, PII, or PHI in claims; put identifiers and let the API look up the rest. If your business genuinely requires encrypted token contents (rare), use JWE (encrypted JWT). The pen tester's real finding (if any) is whether the token was intercepted on the wire — fix that with HTTPS + HSTS, not with encryption.

**Trade-off:** None — this is just how JWT works. Educate, don't panic.

**Real project:** A pen-test report flagged "JWT is readable" as a finding; the team explained the spec, moved an internal employee ID out of the claims (it was used as an identifier elsewhere), and the finding was withdrawn.

## Summary Checklist

- [ ] I configure `Authority`, `Audience`, and `ValidateLifetime/Issuer/Audience/SigningKey`.
- [ ] I use `Microsoft.Identity.Web` for Entra ID instead of raw `JwtBearer` when possible.
- [ ] I set a tight `ClockSkew` (≤ 60 s).
- [ ] I keep access tokens short and use refresh-token rotation.
- [ ] I wrap claim access in `ICurrentUser`, never use `HttpContext` in the domain.
- [ ] I store no secrets in the token; the payload is not encrypted.
- [ ] I validate `tid` for multi-tenant APIs and combine with policy checks.
- [ ] I never log the raw token; I log claims like `sub`, `oid`, `tid`.
- [ ] I use a test auth scheme in integration tests instead of calling the real IdP.
- [ ] I trust the JWKS discovery refresh; I do not hard-code signing keys.
