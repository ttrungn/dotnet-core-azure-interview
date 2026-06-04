# Configuration and Secrets Management

## What It Is

Configuration and secrets management is the discipline of keeping environment-specific values **out of compiled code** and **out of source control**, then composing them at startup (and sometimes refreshing them at runtime) so the same artifact runs correctly in dev, test, staging, and production.

In .NET, configuration flows through `IConfiguration` — a layered key-value tree composed from many providers in a defined order. Secrets are a subset of configuration that require encryption at rest, RBAC, audit, and rotation; they live in **Azure Key Vault** or **Azure App Configuration with Key Vault references**.

```csharp
// Same code, three environments, zero recompilation
var builder = WebApplication.CreateBuilder(args);
// appsettings.json + appsettings.{Environment}.json + env vars + Key Vault
// are all merged into builder.Configuration automatically
string connStr = builder.Configuration.GetConnectionString("OrdersDb")!;
```

## Why It Exists

Early .NET apps shipped with `web.config` containing connection strings, API keys, and environment-specific settings. The pain that followed:

1. **Per-environment recompilation** — different `web.config` per environment meant separate build outputs, breaking the "build once, deploy many" principle.
2. **Secret leakage** — production passwords in Git history, visible to every contributor and every fork.
3. **Configuration drift** — production settings edited in-place on the server, never reflected in source control.
4. **No layering** — overriding one setting per developer required copy-pasting the entire config file.
5. **Restart-required changes** — toggling a feature flag meant a restart, which meant downtime.
6. **No audit** — who changed the connection string at 2am Friday and broke checkout?

`IConfiguration` (introduced in .NET Core) solves layering. **User Secrets**, **Key Vault**, **App Configuration**, and the **Options pattern** solve secrets, runtime refresh, and strongly-typed binding.

## When To Use It

**Always use the configuration system for:**

- Connection strings.
- Feature flags.
- Endpoint URLs (downstream APIs, message brokers).
- Limits and thresholds (retry counts, timeouts, page sizes).
- Per-environment behavior toggles.
- Secrets (via Key Vault).

**Always use the Options pattern for:**

- Strongly-typed access to grouped settings (`IOptions<StripeOptions>` rather than `_config["Stripe:ApiKey"]` scattered through the code).
- Settings that need validation at startup.
- Settings consumed by injected services (testability).

**Use App Configuration for:**

- Feature flags with percentage rollout, time windows, or audience targeting.
- Dynamic settings that must refresh without redeploy.
- Shared configuration across many microservices.

**Do not use this pattern for:**

- Sensitive values in plain `appsettings.json` — always Key Vault.
- Per-request data (use a real cache or database).
- Application logic dressed up as "feature flags" that never get cleaned up — that's tech debt with a switch.

## Why It Is Important

Configuration management is the boundary between "the artifact works on my machine" and "the artifact works in every environment without modification". Get it right and:

- **One build artifact** flows from dev → staging → prod with environment-specific behavior driven by configuration.
- **Secrets stay out of Git** and out of build logs.
- **Rotation is operational, not a deployment**.
- **Feature flags decouple deploy from release**, enabling progressive rollout and instant rollback.
- **Audit and compliance** (SOC 2, PCI-DSS) requirements for centralized configuration and secret access are satisfied by Key Vault + App Configuration diagnostic logs.

Get it wrong and you ship secrets to GitHub, redeploy to flip a flag, or discover at 3am that the staging connection string was pointed at production.

## How It's Used in C# / .NET

NuGet packages:

- `Microsoft.Extensions.Configuration` — core abstractions.
- `Microsoft.Extensions.Configuration.Json` / `EnvironmentVariables` / `CommandLine` / `UserSecrets` — built-in providers.
- `Microsoft.Extensions.Configuration.AzureAppConfiguration` — App Configuration provider.
- `Azure.Extensions.AspNetCore.Configuration.Secrets` — Key Vault provider.
- `Microsoft.FeatureManagement.AspNetCore` — feature flag evaluation.
- `Microsoft.Extensions.Options` — Options pattern.
- `Microsoft.Extensions.Options.DataAnnotations` — `ValidateDataAnnotations()` for settings classes.

### The provider chain (order matters — last wins)

`WebApplication.CreateBuilder` registers providers in this order:

| Order | Provider | Source |
|---|---|---|
| 1 | `ChainedConfigurationProvider` | Host configuration |
| 2 | `JsonConfigurationProvider` | `appsettings.json` |
| 3 | `JsonConfigurationProvider` | `appsettings.{Environment}.json` |
| 4 | `JsonConfigurationProvider` | User Secrets (Development only) |
| 5 | `EnvironmentVariablesConfigurationProvider` | Process env vars |
| 6 | `CommandLineConfigurationProvider` | `--key=value` args |

You append Key Vault and App Configuration **after** this so they override file-based defaults but can themselves be overridden in unit tests via in-memory configuration.

```csharp
builder.Configuration
    .AddAzureAppConfiguration(options =>
    {
        options.Connect(new Uri(appConfigEndpoint), new DefaultAzureCredential())
               .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
               .Select(KeyFilter.Any, LabelFilter.Null)
               .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
               .ConfigureRefresh(refresh =>
               {
                   refresh.Register("Sentinel", refreshAll: true)
                          .SetRefreshInterval(TimeSpan.FromSeconds(30));
               })
               .UseFeatureFlags();
    });
```

### Options pattern — three flavours

| Interface | Lifetime | Refresh behavior | Use when |
|---|---|---|---|
| `IOptions<T>` | Singleton, snapshot at startup | Never refreshes | Settings that don't change after startup (DI registrations) |
| `IOptionsSnapshot<T>` | Scoped, recomputed per request | Refreshes per request | Per-request config (multi-tenant, feature toggles) |
| `IOptionsMonitor<T>` | Singleton with change notifications | Refreshes on provider change + `OnChange` callback | Background services, hot reload |

```csharp
public sealed class StripeOptions
{
    [Required] public required string ApiKey { get; init; }
    [Required, Url] public required string WebhookEndpoint { get; init; }
    [Range(1, 60)] public int TimeoutSeconds { get; init; } = 30;
}

builder.Services.AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();   // fail fast at startup, not on first request

// Consumed in a service
public sealed class StripeClient(IOptionsMonitor<StripeOptions> options)
{
    public Task ChargeAsync(decimal amount, CancellationToken ct)
    {
        var current = options.CurrentValue;  // always the latest version
        // ...
    }
}
```

### User Secrets for local dev

```bash
dotnet user-secrets init
dotnet user-secrets set "Stripe:ApiKey" "sk_test_local_xxx"
```

Stored in `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` on Windows. Never deployed, never in Git, loaded only when `Environment == "Development"`.

## Advantages

- **Build once, deploy many** — one artifact runs in every environment.
- **Layered overrides** — environment files override base, env vars override files, command line overrides env vars. Each layer is intentional.
- **Strongly-typed access** via Options pattern with validation.
- **Hot reload** via `IOptionsMonitor<T>` and App Configuration sentinel keys.
- **Feature flag platform** (App Configuration + `Microsoft.FeatureManagement`) supports progressive rollout, time windows, audience targeting.
- **Secret separation** — Key Vault references mean App Configuration entries can point at secrets without containing them.
- **Audit and rotation** — central control plane in App Configuration / Key Vault.

## Disadvantages

- **Provider order is implicit** — the wrong order silently makes the wrong value win, with no error.
- **String-keyed configuration is fragile** — typos in `"Stripe:ApyKey"` return null without warning. The Options pattern mitigates but doesn't fully eliminate this.
- **App Configuration adds a network dependency** at startup; a misconfigured Managed Identity blocks the app from booting.
- **Feature flag sprawl** — flags that should have been removed years ago accumulate as permanent tech debt.
- **Hot reload semantics are subtle** — `IOptions<T>` does not refresh; `IOptionsSnapshot<T>` only refreshes per request scope; `IOptionsMonitor<T>` may invoke callbacks on a different thread.
- **Local dev parity** — making User Secrets, env vars, and Key Vault all coherent across team members takes discipline.

## Common Mistakes

### 1. Putting secrets in `appsettings.json`

```json
// ❌ appsettings.Production.json checked into Git
{
  "ConnectionStrings": {
    "OrdersDb": "Server=...;User Id=admin;Password=P@ssw0rd!"
  }
}
```

**Fix:** Store the value in Key Vault, reference it through the configuration provider chain:

```csharp
// ✅ appsettings.json holds only non-secret defaults
builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!),
    new DefaultAzureCredential());
// Secret "ConnectionStrings--OrdersDb" in the vault overrides any file value
```

### 2. Reading `IConfiguration` directly all over the codebase

```csharp
// ❌ Scattered string keys, no type safety, no validation
public sealed class StripeClient(IConfiguration config)
{
    public Task ChargeAsync(decimal amount) {
        var key = config["Stripe:ApiKey"];
        var timeout = int.Parse(config["Stripe:TimeoutSeconds"] ?? "30");
        // ...
    }
}
```

**Fix:** Bind to a strongly-typed options class with validation:

```csharp
// ✅
builder.Services.AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

public sealed class StripeClient(IOptions<StripeOptions> options)
{
    private readonly StripeOptions _opts = options.Value;
    public Task ChargeAsync(decimal amount) { /* use _opts.ApiKey, _opts.TimeoutSeconds */ }
}
```

### 3. Using `IOptions<T>` when you need hot-reload

```csharp
// ❌ Snapshot at startup; sentinel-key rotation in App Configuration has no effect
public sealed class FeatureGate(IOptions<FeatureOptions> options) { }
```

**Fix:** Use `IOptionsMonitor<T>` for background services or `IOptionsSnapshot<T>` for per-request:

```csharp
// ✅
public sealed class FeatureGate(IOptionsMonitor<FeatureOptions> options)
{
    public bool IsEnabled(string name) => options.CurrentValue.Flags[name];
}
```

### 4. Forgetting `.ValidateOnStart()`

Without it, a missing or invalid setting throws on the first request that touches the options class — at 3am, in production, after a deploy that "passed" smoke tests.

```csharp
// ✅ Fails the host startup if validation fails
builder.Services.AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### 5. Using `appsettings.Production.json` to hold real production values

```csharp
// ❌ The file is in the artifact and visible to anyone with build access
{ "Stripe": { "ApiKey": "sk_live_..." } }
```

**Fix:** Production values live in **App Configuration with the `Production` label**, or in App Service settings. `appsettings.{Environment}.json` is for *non-sensitive* environment-specific defaults — log levels, feature toggle baselines, base URLs.

### 6. Not using a sentinel key for App Configuration refresh

```csharp
// ❌ Polls every key on every refresh interval — expensive and slow
options.ConfigureRefresh(r => r.SetRefreshInterval(TimeSpan.FromSeconds(30)));
```

**Fix:** Register a single `Sentinel` key with `refreshAll: true`. The provider polls only the sentinel; if it changed, it pulls the full key set once:

```csharp
// ✅ One key polled, full refresh only when sentinel bumps
options.ConfigureRefresh(refresh =>
{
    refresh.Register("Sentinel", refreshAll: true)
           .SetRefreshInterval(TimeSpan.FromSeconds(30));
});
```

Bump the sentinel value (any string change) in App Configuration when you want subscribers to refresh.

### 7. Per-environment labels misused for secrets

Storing `Stripe:ApiKey` under both `Development` and `Production` labels in App Configuration directly puts secret values in App Configuration. App Configuration is *not* encrypted as strongly as Key Vault and has different RBAC granularity. Always use **Key Vault references** from App Configuration for secrets:

```
Key: Stripe:ApiKey
Label: Production
Value: {"uri":"https://contoso-prod-kv.vault.azure.net/secrets/Stripe--ApiKey"}
Content-Type: application/vnd.microsoft.appconfig.keyvaultref+json;charset=utf-8
```

## Best Practices

- **Layer providers in increasing specificity**: defaults in `appsettings.json`, env-specific in `appsettings.{Env}.json`, secrets in Key Vault (last so it wins).
- **Use the Options pattern with validation** for every settings group. Always `.ValidateOnStart()`.
- **Use `IOptionsMonitor<T>` for singletons** that need to react to refresh; `IOptionsSnapshot<T>` for per-request scopes.
- **Never commit secrets** to source control, ever. Use User Secrets locally, Key Vault elsewhere.
- **Use App Configuration for shared configuration** across many microservices.
- **Use Key Vault references** in App Configuration for secret values.
- **Use a sentinel key** for App Configuration refresh to keep polling cheap.
- **Label keys per environment** (`Development`, `Staging`, `Production`) and filter on `builder.Environment.EnvironmentName`.
- **Disable production settings in lower environments** at the configuration layer, not via runtime `if` checks.
- **Audit configuration changes** via App Configuration / Key Vault diagnostic logs.
- **Clean up stale feature flags** during sprint retros — flags with no flips for 6 months are removal candidates.

## Related Concepts

- [azure/azure-key-vault.md](azure/azure-key-vault.md) — secret storage with RBAC, soft delete, rotation.
- [azure/app-service.md](azure/app-service.md) — App Service settings and Key Vault references.
- [azure/application-insights.md](azure/application-insights.md) — observability for config refresh and feature-flag evaluation.
- [aspnet-core/request-pipeline.md](aspnet-core/request-pipeline.md) — host configuration vs. application configuration.
- [csharp/dependency-injection.md](csharp/dependency-injection.md) — Options pattern is DI-driven.
- [devops/environment-configuration.md](devops/environment-configuration.md) — environment-specific deployment patterns.

## Real-World Usage

### Payment APIs

A payments microservice loads `Stripe:ApiKey` from Key Vault via the configuration provider, binds `Stripe:WebhookSecret` and `Stripe:TimeoutSeconds` via `IOptions<StripeOptions>`, and validates at startup. A configuration drift in non-prod gets caught before a single request is served.

### Multi-tenant SaaS

A multi-tenant invoicing platform uses `IOptionsSnapshot<TenantBillingOptions>` so each request resolves its tenant's billing tier from App Configuration. Switching a tenant from Standard to Premium is a single App Configuration change — no deploy.

### Feature flags with progressive rollout

A checkout API gates a new shipping calculator behind a feature flag in App Configuration. The flag rolls out to 5% of users, then 25%, then 100% over a week. The team can disable instantly if conversion drops, without a deploy.

### Multi-region

Each region's App Service points at its regional App Configuration store (`contoso-weu-appconfig`, `contoso-neu-appconfig`). A pipeline writes config to both stores. If one region fails, the surviving region keeps reading its local store with zero cross-region dependency.

### CI/CD

GitHub Actions workflows use OIDC federated credentials to read the integration test connection string from `contoso-ci-kv` at pipeline runtime. No long-lived secret stored as a GitHub secret.

## Code Example — Before and After

### Before — string keys, secrets in files, restart-to-toggle

```csharp
// appsettings.json (committed)
{
  "Stripe": {
    "ApiKey": "sk_live_51HxYz...",        // ❌ secret in Git
    "TimeoutSeconds": "30"                 // ❌ string, parsed at every use
  },
  "Features": { "NewShippingCalculator": "false" }
}

// Program.cs
builder.Services.AddSingleton<IStripeClient>(sp =>
{
    var key = builder.Configuration["Stripe:ApiKey"];   // ❌ scattered
    return new StripeClient(key);
});

// CheckoutController.cs
public sealed class CheckoutController(IConfiguration config) : ControllerBase
{
    [HttpPost]
    public IActionResult Checkout(CheckoutRequest req)
    {
        if (bool.Parse(config["Features:NewShippingCalculator"] ?? "false"))
        {
            // new logic
        }
        // ...
    }
}
```

**Problems:**
- Secret leaked to Git.
- Type errors at runtime, not compile time.
- Feature flag change = redeploy.
- No validation, missing key = silent default.

### After — Options pattern, Key Vault, App Configuration with feature flags

```csharp
// appsettings.json (no secrets, safe to commit)
{
  "AppConfig": { "Endpoint": "https://contoso-prod-appconfig.azconfig.io" },
  "Stripe": { "TimeoutSeconds": 30 },
  "FeatureManagement": { "NewShippingCalculator": false }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(builder.Configuration["AppConfig:Endpoint"]!), new DefaultAzureCredential())
           .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
           .Select(KeyFilter.Any, LabelFilter.Null)
           .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("Sentinel", refreshAll: true)
                      .SetRefreshInterval(TimeSpan.FromSeconds(30));
           })
           .UseFeatureFlags(ff => ff.SetRefreshInterval(TimeSpan.FromSeconds(30)));
});

builder.Services.AddAzureAppConfiguration();
builder.Services.AddFeatureManagement();

builder.Services.AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddSingleton<IStripeClient, StripeClient>();

var app = builder.Build();
app.UseAzureAppConfiguration();  // middleware that triggers refresh on requests

// StripeOptions.cs
public sealed class StripeOptions
{
    [Required] public required string ApiKey { get; init; }    // sourced from Key Vault via App Config reference
    [Range(1, 60)] public int TimeoutSeconds { get; init; } = 30;
}

// StripeClient.cs
public sealed class StripeClient(IOptionsMonitor<StripeOptions> options, ILogger<StripeClient> log)
    : IStripeClient
{
    public async Task<ChargeResult> ChargeAsync(decimal amount, CancellationToken ct)
    {
        var opts = options.CurrentValue;   // always fresh after sentinel bump
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(opts.TimeoutSeconds));
        // call Stripe with opts.ApiKey ...
    }
}

// CheckoutController.cs
public sealed class CheckoutController(IFeatureManager features) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Checkout(CheckoutRequest req, CancellationToken ct)
    {
        if (await features.IsEnabledAsync("NewShippingCalculator"))
        {
            // progressive-rollout-controlled code path
        }
        // ...
    }
}
```

**Why this is better:**
- Zero secrets in source control — `Stripe:ApiKey` is a Key Vault reference resolved by App Configuration.
- Compile-time and startup-time validation catches missing/invalid settings.
- `IOptionsMonitor<T>` picks up sentinel-driven refreshes without restart.
- Feature flag is evaluated per request; rollout percentage is changed in App Configuration.
- One artifact, three environments, behavior driven entirely by configuration.

## Interview Questions and Answers

### 1. Walk me through the configuration provider order in a .NET 8 Web API. Why does the order matter?

**Why this matters:** A common production bug is "the env var override isn't taking effect" — almost always caused by misunderstanding provider order.

**Answer:** `WebApplication.CreateBuilder` registers, in order: host config, `appsettings.json`, `appsettings.{Environment}.json`, User Secrets (Dev only), environment variables, command line args. Later providers override earlier ones. I append Key Vault and App Configuration after this chain so production secrets and dynamic settings win, but tests can still override anything via `AddInMemoryCollection`. The wrong order silently produces wrong values — for example, a stale `appsettings.Production.json` that overrides a Key Vault refresh.

**Trade-off:** Adding Key Vault before env vars means env vars override Key Vault (useful for tests); after env vars means Key Vault wins (typical for production). Pick consciously per project.

**Real project:** On a checkout API a stale `appsettings.Staging.json` was overriding a corrected App Configuration value because the file provider was registered after App Configuration in a misordered `Program.cs`. The fix was a two-line provider order change.

### 2. When would you use `IOptions<T>` vs. `IOptionsSnapshot<T>` vs. `IOptionsMonitor<T>`?

**Why this matters:** Picking the wrong one is the most common Options-pattern bug.

**Answer:** `IOptions<T>` is a singleton bound at startup — perfect for settings that never change (DI registrations, fixed limits). `IOptionsSnapshot<T>` is scoped — it recomputes per request, so multi-tenant or per-request feature checks work but it adds work per request. `IOptionsMonitor<T>` is a singleton with change notifications — required for background services and singletons that need to react to App Configuration sentinel refresh. If you inject `IOptions<T>` into a singleton that consumes a hot-reload flag, your reload silently does nothing.

**Trade-off:** `IOptionsMonitor` change callbacks run on a background thread — guard mutable state with locks if necessary.

### 3. Why use App Configuration if you already have Key Vault?

**Why this matters:** Tests whether the candidate understands the *purpose* split, not the *what*.

**Answer:** Key Vault is for **secrets** — high RBAC, audit, soft-delete, rotation, throttled. App Configuration is for **everything else** — feature flags, dynamic settings, shared config across services, with sentinel-driven hot reload. Putting secrets in App Configuration breaks compliance; putting feature flags in Key Vault hits throttling limits and forces redeploy-to-toggle. The right architecture is App Configuration as the configuration plane with Key Vault references for secret values — apps get one provider, secrets stay in the right store.

**Real project:** On a multi-microservice platform we consolidated 12 microservices onto one App Configuration store with Key Vault references for all secrets. Cross-service config changes that used to require 12 redeploys became a single key edit.

### 4. How do you handle a feature flag rollout from 0% to 100%?

**Why this matters:** Decoupling deploy from release is a senior-level practice.

**Answer:** Deploy the code behind a flag defaulted off in App Configuration. Enable for internal users via audience targeting (Entra group). Move to 5% percentage rollout, monitor business and error metrics in App Insights for an hour. If healthy, 25%, then 50%, then 100%. If a regression appears, set to 0% — the rollback is a configuration edit, not a deploy. After 100% has been stable for two weeks, remove the flag and the dead branch from code.

**Trade-off:** Percentage rollout requires a deterministic targeting context (user ID, session ID) so the same user gets a consistent experience.

### 5. A developer reports that `IConfiguration["Stripe:ApiKey"]` returns null in production but works locally. What do you check?

**Why this matters:** Diagnostic skill across the full configuration chain.

**Answer:** In order: (1) is the User Secrets value masking the absence in deployed config locally? (2) is the App Configuration / Key Vault provider actually registered in `Program.cs`? (3) does the runtime identity have RBAC on the vault? (4) is the key naming convention right — `Stripe--ApiKey` in Key Vault becomes `Stripe:ApiKey` in `IConfiguration`? (5) is the App Configuration label filter selecting the right environment? (6) did a startup exception in the App Configuration provider get swallowed, causing the fallback to return nulls? `IConfiguration.GetDebugView()` enumerates the full key tree with their sources — invaluable.

**Trade-off:** Logging `GetDebugView()` in non-production helps; never log it in production because secret values appear.

### 6. How do you keep dev/prod parity without forcing every developer to read from production Key Vault?

**Why this matters:** Local developer experience and security balance.

**Answer:** Three layers: (1) per-developer User Secrets for true secrets like a personal Stripe test key; (2) a shared `contoso-dev-kv` Key Vault with non-production secrets that any dev can read via their Entra group; (3) `appsettings.Development.json` for non-sensitive dev defaults. Production Key Vault access is granted only to the prod App Service Managed Identity and break-glass admin groups — no developer reads it day-to-day.

**Trade-off:** Dev/prod parity is never perfect — Stripe test mode behaves differently from live. Keep the differences documented.

### 7. Walk me through `.ValidateOnStart()`.

**Why this matters:** Failing fast vs. slow.

**Answer:** Without it, options validation runs lazily on the first access to `IOptions<T>.Value`. That means a missing required setting throws on the first request that touches that code path — possibly hours after deploy, possibly in a niche endpoint. `.ValidateOnStart()` runs validation during host startup so the app refuses to start with bad config. Combined with `[Required]`, `[Range]`, `[Url]`, and custom `IValidateOptions<T>`, it gives a strong startup-time safety net.

**Real project:** On an internal billing API, a missing `Billing:VatRate` setting only failed during month-end invoice generation. After adding `.ValidateOnStart()` the misconfiguration was caught at deploy time, not on the 30th.

### 8. How do you provide configuration to integration tests without baking environment values into test code?

**Why this matters:** Testable, environment-agnostic configuration.

**Answer:** Use `WebApplicationFactory<T>` and override configuration via `ConfigureAppConfiguration` with `AddInMemoryCollection`. Tests pass in-memory overrides for connection strings (Testcontainers), feature flags, and external API base URLs. Real Key Vault and App Configuration providers are not invoked. This keeps tests fast, isolated, and free of production credentials.

```csharp
var factory = new WebApplicationFactory<Program>()
    .WithWebHostBuilder(b => b.ConfigureAppConfiguration(cfg =>
        cfg.AddInMemoryCollection(new Dictionary<string, string?>
        {
            ["Stripe:ApiKey"] = "sk_test_xxx",
            ["Stripe:TimeoutSeconds"] = "5"
        })));
```

## Summary Checklist

- [ ] I can list the default configuration provider order and explain why it matters.
- [ ] I bind settings to strongly-typed classes via the Options pattern with validation and `.ValidateOnStart()`.
- [ ] I know when to use `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`.
- [ ] I store secrets in Key Vault and reference them from App Configuration, never in `appsettings.json`.
- [ ] I use User Secrets for local development.
- [ ] I configure App Configuration with sentinel-key refresh and environment labels.
- [ ] I use feature flags for progressive rollout and decoupling deploy from release.
- [ ] I authenticate to Key Vault and App Configuration via Managed Identity.
- [ ] I provide in-memory configuration overrides for integration tests.
- [ ] I audit configuration changes via Log Analytics diagnostic logs.
