# Environment Configuration

## What It Is

**Environment configuration** is the practice of building a single binary or container image once and changing its runtime behavior — connection strings, feature flags, log levels, endpoints, secrets — through external configuration that varies per environment (Development, QA, Staging, Production).

The same `OrderApi:1.42.0` image runs in Dev pointing at `sql-dev`, in Staging pointing at `sql-stg`, and in Production pointing at `sql-prod-westus2` — without recompilation. The artifact is immutable; only its inputs change.

In .NET, this is implemented through the layered `IConfiguration` system, `IHostEnvironment`, environment-specific `appsettings.{Environment}.json` files, environment variables, command-line arguments, and external providers like Azure App Configuration and Azure Key Vault.

## Why It Exists

Before disciplined environment configuration, teams routinely shipped one of these anti-patterns:

- **Per-environment builds:** A `Debug-Dev` build and a `Release-Prod` build, compiled separately. The Prod binary was untested because Dev tested a different binary.
- **Hardcoded connection strings in `web.config` transforms:** A misconfigured XSLT replaced the wrong node and Production silently pointed at the QA database for three days.
- **Secrets in Git:** A leaked `appsettings.Production.json` with a SQL admin password committed to a public fork — a real incident class that drove the creation of Key Vault and GitHub secret scanning.
- **"Works on my machine":** Developers ran with one config; CI used another; Prod used a third. Bugs surfaced only in Prod.

The modern principle — derived from the [12-Factor App](https://12factor.net/config) — is that **config lives in the environment, not the codebase**, and the **same artifact promotes through environments unchanged**.

## When To Use It

**Use environment-based configuration for:**

- Any non-trivial backend service deployed to more than one environment.
- Database connection strings, API base URLs, message bus connection strings.
- Feature flags that differ between environments (e.g., new checkout flow ON in Staging, OFF in Prod).
- Log levels (Debug in Dev, Information in Prod, Warning in high-volume hot paths).
- Secrets — always pulled from a secret store, never from a config file.
- CORS allowed origins, rate-limit thresholds, retry counts, timeouts.

**Do not use it for:**

- **Business logic branching.** If your code says `if (env.IsProduction()) ApplyDiscount();` you have a bug, not a configuration. Business rules belong in code, possibly behind a feature flag, but never gated by environment name.
- **Values that never change.** Constants like `const int MaxRetries = 3` for a stable algorithm don't need configuration overhead.
- **Per-tenant settings in a multi-tenant SaaS.** Those belong in a tenant configuration store (database or Azure App Configuration with tenant labels), not in `appsettings.json`.

## Why It Is Important

In production this single discipline drives:

- **Release safety.** The exact binary that passed Staging smoke tests is the binary that runs in Production. No surprise recompilation.
- **Compliance & auditability.** SOC 2 / ISO 27001 auditors ask "how do you ensure the Production artifact matches the tested artifact?" The answer is: one build, immutable image tag, config injected at runtime.
- **Secret hygiene.** Secrets never appear in source control, in build logs, or in container layers. They are pulled from Key Vault via managed identity at startup.
- **Fast recovery.** A bad config push is reverted by editing one App Configuration key — no redeploy, no pipeline run.
- **Multi-region & multi-tenant scale.** The same image runs in 12 regions; only the `Region` and `KeyVaultUri` env vars differ.
- **Developer onboarding.** A new engineer clones the repo, sets `ASPNETCORE_ENVIRONMENT=Development`, and the app uses local SQL + a local secrets file — no production credentials touch their laptop.

## How It's Used in C# / .NET

### The configuration hierarchy

ASP.NET Core builds `IConfiguration` from multiple **providers**, applied in order. Later providers **override** earlier ones:

1. `appsettings.json` (base, checked into Git)
2. `appsettings.{Environment}.json` (e.g., `appsettings.Staging.json`)
3. User Secrets (Development only, stored outside the repo)
4. Environment variables
5. Command-line arguments
6. Custom providers (Azure App Configuration, Azure Key Vault, AWS Parameter Store)

The environment name comes from the `ASPNETCORE_ENVIRONMENT` environment variable (or `DOTNET_ENVIRONMENT` for non-web hosts). Default is `Production` when unset — a deliberate fail-safe.

### Reading the environment

```csharp
var builder = WebApplication.CreateBuilder(args);

if (builder.Environment.IsDevelopment())
{
    builder.Services.AddSwaggerGen();
}

if (builder.Environment.IsProduction())
{
    builder.Services.AddApplicationInsightsTelemetry();
}

// Custom environment name — not just Dev/Staging/Prod
if (builder.Environment.IsEnvironment("QA"))
{
    builder.Services.AddSingleton<ITestDataSeeder, QaSeeder>();
}
```

### Strongly-typed config with `IOptions<T>`

Never read raw strings from `IConfiguration` in business code. Bind a class and inject `IOptions<T>` or `IOptionsSnapshot<T>`:

```csharp
public sealed class StripeOptions
{
    public const string Section = "Payments:Stripe";

    [Required]
    public string ApiKey { get; init; } = string.Empty;

    [Range(1, 60)]
    public int TimeoutSeconds { get; init; } = 30;

    public Uri WebhookEndpoint { get; init; } = null!;
}

builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection(StripeOptions.Section))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // Fail fast at boot, not on first request
```

### Environment variables for nested keys

On Linux, container, and shell environments, JSON nesting is expressed with a **double underscore**:

```bash
# appsettings.json:  { "Payments": { "Stripe": { "ApiKey": "..." } } }
export Payments__Stripe__ApiKey="sk_live_..."
export Payments__Stripe__TimeoutSeconds="45"
export ConnectionStrings__OrdersDb="Server=..."
```

This is non-negotiable on Linux because `:` is reserved in many shells. Windows accepts both `:` and `__`.

### Azure App Service deployment slots

Azure App Service offers **deployment slots** — secondary endpoints (e.g., `myapi-staging.azurewebsites.net`) sharing the same App Service Plan. Each slot has its own configuration. App settings can be marked **"deployment slot setting"** (a.k.a. "sticky") so they **do not move during a slot swap**:

- `ASPNETCORE_ENVIRONMENT` → sticky (Staging slot stays `Staging`, even after swap)
- `SqlConnection__ProductionDb` → sticky (each slot points to its own DB if needed)
- `Image_Tag` → not sticky (moves with the swap, so the new code goes live)

After a swap, the slot that was Staging is now Production. The app reads its `ASPNETCORE_ENVIRONMENT` and behaves accordingly — same image, different config.

### GitHub Actions environments

GitHub Actions exposes three configuration scopes:

| Scope | Visible in logs? | Per-environment? | Use for |
|---|---|---|---|
| **Repository variable** | Yes | No | Non-secret defaults (e.g., `DOTNET_VERSION: 8.0.x`) |
| **Repository secret** | No (masked) | No | Shared secrets across all envs (rare; usually wrong) |
| **Environment variable** | Yes | Yes | Per-env non-secret (e.g., `API_BASE_URL` for Staging vs Prod) |
| **Environment secret** | No (masked) | Yes | Per-env secrets (e.g., `AZURE_CREDENTIALS_PROD` with required reviewer approval) |

Environment-scoped secrets can require **manual approval** and **branch restrictions** before a job can read them — the gate that prevents a PR build from deploying to Prod.

```yaml
jobs:
  deploy-prod:
    environment: production   # triggers approval gate
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS_PROD }}
      - uses: azure/webapps-deploy@v3
        with:
          app-name: orders-api
          images: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### Bicep parameter files (`*.bicepparam`)

Infrastructure-as-Code is itself environment-configured. One Bicep template, three parameter files:

```bicep
// main.bicep — shared template
param sqlSku string
param appServicePlanSku string
param environment string
```

```bicep
// main.staging.bicepparam
using './main.bicep'
param sqlSku = 'S1'
param appServicePlanSku = 'P1v3'
param environment = 'Staging'
```

```bicep
// main.prod.bicepparam
using './main.bicep'
param sqlSku = 'BC_Gen5_4'
param appServicePlanSku = 'P3v3'
param environment = 'Production'
```

The pipeline runs `az deployment group create --template-file main.bicep --parameters main.${{ inputs.env }}.bicepparam`.

### Azure App Configuration with labels

Azure App Configuration is a managed key-value store. **Labels** scope a key to an environment:

| Key | Label | Value |
|---|---|---|
| `Payments:Stripe:ApiKey` | `Staging` | `{vault:stripe-staging}` |
| `Payments:Stripe:ApiKey` | `Production` | `{vault:stripe-prod}` |
| `Features:UseNewCheckout` | `Staging` | `true` |
| `Features:UseNewCheckout` | `Production` | `false` |

```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options
        .Connect(new Uri(builder.Configuration["AppConfig:Endpoint"]!), new DefaultAzureCredential())
        .Select(KeyFilter.Any, LabelFilter.Null)                          // shared defaults
        .Select(KeyFilter.Any, builder.Environment.EnvironmentName)        // env override
        .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))
        .ConfigureRefresh(refresh => refresh.Register("Sentinel", refreshAll: true)
                                            .SetCacheExpiration(TimeSpan.FromSeconds(30)))
        .UseFeatureFlags();
});
```

Flip `Features:UseNewCheckout` from `false` to `true` in the portal — within 30 seconds, every Production instance picks up the change. **No redeploy.**

## Advantages

- **One artifact, many environments.** The image SHA tested in QA is the image SHA running in Prod.
- **Secrets stay out of source control.** Key Vault references, managed identities, and `DefaultAzureCredential` remove the temptation to commit secrets.
- **Fast config changes.** Toggle a feature flag or change a rate limit without a deploy.
- **Environment parity.** Dev, Staging, and Prod differ only in their inputs, not their behavior.
- **Audit trail.** Azure App Configuration logs every change with the principal that made it.
- **Testability.** Tests inject an `IOptions<T>` mock — no static configuration to fight.

## Disadvantages

- **Config drift.** Staging and Prod silently diverge over months as engineers tweak Staging "just to test" and forget to mirror Prod. Mitigation: IaC + drift detection.
- **Distributed config debugging.** "Which provider supplied this value?" requires understanding provider ordering. The `IConfigurationRoot.GetDebugView()` extension helps.
- **Latency on startup.** Pulling from Key Vault and App Configuration at boot can add 2–5 seconds. Use sentinel-based refresh, not boot-only loads, for frequently-changing values.
- **Cost.** Azure App Configuration and Key Vault have per-request pricing; chatty apps doing thousands of reads per second need a local cache (the SDKs cache by default).
- **Operational discipline required.** Without good naming conventions (`Payments:Stripe:ApiKey`), keys explode into chaos.

## Common Mistakes

### 1. Committing `appsettings.Production.json` with real secrets

```jsonc
// ❌ checked into Git — leaked to anyone with repo access
{
  "ConnectionStrings": {
    "OrdersDb": "Server=sql-prod;User Id=sa;Password=Sup3rS3cret!;"
  }
}
```

**Fix:** Reference Key Vault. The config file holds only the *reference*, not the value.

```jsonc
// appsettings.Production.json — safe to commit
{
  "ConnectionStrings": {
    "OrdersDb": "@Microsoft.KeyVault(SecretUri=https://kv-prod.vault.azure.net/secrets/orders-db-conn/)"
  }
}
```

The App Service resolves the reference at runtime using its managed identity.

### 2. Branching business logic on `IsProduction()`

```csharp
// ❌ Production silently behaves differently from Staging — untestable
public decimal CalculateTotal(Order order)
{
    if (_env.IsProduction())
        return order.Items.Sum(i => i.Price);
    else
        return order.Items.Sum(i => i.Price) * 0.5m; // "test mode"
}
```

**Fix:** Use a feature flag or `IOptions<PricingOptions>` so the same code path runs everywhere.

```csharp
public decimal CalculateTotal(Order order) =>
    order.Items.Sum(i => i.Price) * _pricing.Value.DiscountMultiplier;
```

### 3. Forgetting `ValidateOnStart()` — invalid config crashes the first request, not the deploy

```csharp
// ❌ App boots fine. First /api/payments call throws NullReferenceException.
builder.Services.Configure<StripeOptions>(
    builder.Configuration.GetSection("Payments:Stripe"));
```

**Fix:** Validate at startup so a misconfigured pod **fails its readiness probe** and never receives traffic.

```csharp
builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Payments:Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### 4. Using `:` separator in container env vars on Linux

```bash
# ❌ Some shells & orchestrators strip or reinterpret ':'
Payments:Stripe:ApiKey=sk_live_...
```

**Fix:** Always use double-underscore in container environments.

```yaml
# docker-compose.yml
services:
  api:
    environment:
      - Payments__Stripe__ApiKey=sk_live_...
      - ConnectionStrings__OrdersDb=Server=...
```

### 5. Reading `IConfiguration` in hot paths instead of `IOptionsSnapshot<T>`

```csharp
// ❌ Re-parses config on every call, no validation, returns string
public class CheckoutController(IConfiguration config) : ControllerBase
{
    public IActionResult Pay() =>
        Ok(new { ApiKey = config["Payments:Stripe:ApiKey"] });
}
```

**Fix:** Inject a typed `IOptionsSnapshot<StripeOptions>` (per-request) or `IOptionsMonitor<StripeOptions>` (long-lived singletons).

### 6. Putting feature flags in `appsettings.json` instead of App Configuration

A feature flag in `appsettings.Production.json` requires a redeploy to flip. The whole point of a flag is **instant, no-deploy toggling**. Use Azure App Configuration's feature management or LaunchDarkly.

### 7. Leaking secrets through `/healthz` or diagnostic endpoints

The `GetDebugView()` extension dumps the entire configuration. **Never** expose this on a public endpoint. If you need it, gate behind authentication and only enable in Development.

## Best Practices

- **Default to Production.** When `ASPNETCORE_ENVIRONMENT` is unset, the runtime assumes Production. Make sure that fail-safe behaves correctly.
- **Validate at startup.** Use `ValidateOnStart()` + `ValidateDataAnnotations()`. Crash early.
- **Use managed identity** instead of connection strings everywhere it is supported (SQL, Storage, Key Vault, Service Bus, App Configuration).
- **One source of truth per setting.** Decide whether a value lives in App Configuration *or* the App Service settings — never both.
- **Document the schema.** A `appsettings.Schema.json` checked into the repo lists every key, its type, and required/optional status.
- **Prefer `IOptionsSnapshot<T>`** in scoped services so config refreshes pick up per request.
- **Use sentinel keys** in App Configuration to trigger refresh: one well-known key changes; the SDK refreshes everything.
- **Pin sticky settings** in App Service slots: `ASPNETCORE_ENVIRONMENT`, region-specific endpoints, slot-specific log sinks.
- **Audit App Configuration changes** with diagnostic settings sent to Log Analytics.
- **Use environment-scoped secrets in GitHub** with required reviewers for Production deploys.

## Related Concepts

- [azure/configuration-and-secrets-management.md](../azure/configuration-and-secrets-management.md) — Key Vault, App Configuration, managed identity
- [azure/azure-key-vault.md](../azure/azure-key-vault.md)
- [aspnet-core/request-pipeline.md](../aspnet-core/request-pipeline.md) — where the configuration is built
- [devops/ci-cd-pipelines.md](ci-cd-pipelines.md) — promoting one artifact through environments
- [devops/blue-green-deployment.md](blue-green-deployment.md) — slot configuration during swap
- [devops/rollback-strategy.md](rollback-strategy.md) — config as the fastest rollback lever
- [devops/health-checks.md](health-checks.md) — fail readiness when config is invalid

## Real-World Usage

### ASP.NET Core multi-environment SaaS

A B2B billing SaaS runs the same `BillingApi:v3.18.0` image in five environments: `Dev`, `QA`, `Staging`, `ProductionEU`, `ProductionUS`. Each environment is a separate Azure App Service with its own App Configuration store and Key Vault. The image never knows where it runs — only `ASPNETCORE_ENVIRONMENT` and `AppConfig__Endpoint` differ.

### Azure Functions with App Configuration

A Service Bus-triggered Function processes invoice events. Connection strings and the `Features:RetryOnDuplicate` flag live in App Configuration. Flipping the flag during a vendor outage stops duplicate-retry behavior in 30 seconds without a deploy.

### GitHub Actions deploying to AKS across three regions

The pipeline builds one image, pushes to ACR, then runs three parallel deploy jobs — each in a GitHub environment (`prod-westeurope`, `prod-eastus`, `prod-southeastasia`). Each environment has its own approval reviewer, its own kubeconfig secret, and its own Bicep parameter file. The Helm values file is templated per region from environment-scoped variables.

### Multi-tenant configuration

For a per-tenant override (e.g., "tenant Acme uses a custom email template"), the team stores those overrides in App Configuration with a label like `tenant:acme-corp` and uses `IOptionsSnapshot<TenantSettings>` with a tenant-aware `IConfiguration` decorator. Environment configuration handles infrastructure; tenant configuration handles business variation.

## Code Example — Before and After

### Before — secrets in source, environment branching in code

```csharp
public sealed class StripeClient
{
    private const string DevKey  = "sk_test_<REDACTED_EXAMPLE_KEY>";
    private const string ProdKey = "sk_live_<REDACTED_EXAMPLE_KEY>"; // ❌ committed to Git

    private readonly HttpClient _http;
    private readonly IWebHostEnvironment _env;

    public StripeClient(HttpClient http, IWebHostEnvironment env)
    {
        _http = http;
        _env = env;
    }

    public async Task<string> ChargeAsync(decimal amount, CancellationToken ct)
    {
        var key = _env.IsProduction() ? ProdKey : DevKey;          // ❌ env branching
        _http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", key);

        var url = _env.IsProduction()
            ? "https://api.stripe.com/v1/charges"
            : "https://api.stripe.com/v1/charges";                  // ❌ duplicated

        // ... no timeout, no retry config, hardcoded
        var resp = await _http.PostAsync(url, BuildBody(amount), ct);
        return await resp.Content.ReadAsStringAsync(ct);
    }
}
```

Problems:
- Production key in source control — audit failure, rotation requires a rebuild.
- `IsProduction()` branching makes Staging behave differently from Prod.
- No timeout or retry configuration — values are buried in code.
- Cannot rotate the key without a deploy.

### After — typed options, Key Vault reference, identical behavior across environments

```csharp
public sealed class StripeOptions
{
    public const string Section = "Payments:Stripe";

    [Required, MinLength(20)]
    public string ApiKey { get; init; } = string.Empty;

    [Required]
    public Uri BaseAddress { get; init; } = null!;

    [Range(1, 60)]
    public int TimeoutSeconds { get; init; } = 15;

    [Range(0, 5)]
    public int MaxRetries { get; init; } = 3;
}

public sealed class StripeClient
{
    private readonly HttpClient _http;
    private readonly IOptionsSnapshot<StripeOptions> _options;
    private readonly ILogger<StripeClient> _logger;

    public StripeClient(
        HttpClient http,
        IOptionsSnapshot<StripeOptions> options,
        ILogger<StripeClient> logger)
    {
        _http = http;
        _options = options;
        _logger = logger;
    }

    public async Task<ChargeResult> ChargeAsync(
        Guid invoiceId, decimal amount, CancellationToken ct)
    {
        var opts = _options.Value;
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(opts.TimeoutSeconds));

        using var req = new HttpRequestMessage(HttpMethod.Post, new Uri(opts.BaseAddress, "/v1/charges"));
        req.Headers.Authorization = new AuthenticationHeaderValue("Bearer", opts.ApiKey);
        req.Content = BuildBody(amount);

        _logger.LogInformation("Charging invoice {InvoiceId} for {Amount}", invoiceId, amount);
        using var resp = await _http.SendAsync(req, cts.Token);
        resp.EnsureSuccessStatusCode();

        return await resp.Content.ReadFromJsonAsync<ChargeResult>(cancellationToken: cts.Token)
            ?? throw new InvalidOperationException("Stripe returned empty body");
    }

    private static StringContent BuildBody(decimal amount) =>
        new(JsonSerializer.Serialize(new { amount, currency = "usd" }),
            Encoding.UTF8, "application/json");
}

// Program.cs
builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection(StripeOptions.Section))
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddHttpClient<StripeClient>();
```

```jsonc
// appsettings.json — committed, no secrets
{
  "Payments": {
    "Stripe": {
      "BaseAddress": "https://api.stripe.com",
      "TimeoutSeconds": 15,
      "MaxRetries": 3
    }
  }
}

// appsettings.Production.json — committed, Key Vault reference
{
  "Payments": {
    "Stripe": {
      "ApiKey": "@Microsoft.KeyVault(SecretUri=https://kv-prod-payments.vault.azure.net/secrets/stripe-api-key/)"
    }
  }
}
```

Now:
- Rotating the Stripe key is a Key Vault update — no deploy.
- Staging and Production execute the identical code path.
- A misconfigured environment fails at startup with a clear error message.
- The HTTP timeout and retry count are visible and tunable per environment.

## Interview Questions and Answers

### 1. We have one Docker image but three environments. How does it know which database to use?

**Why this matters:** Tests the candidate's grasp of the build-once-deploy-many principle and the .NET configuration pipeline.

**Answer:** The image is identical across environments. At container start, the orchestrator (App Service, AKS, ECS) injects environment variables such as `ASPNETCORE_ENVIRONMENT=Staging` and `ConnectionStrings__OrdersDb=Server=sql-stg;...`. The .NET host loads `appsettings.json`, then `appsettings.Staging.json`, then environment variables — each layer overriding the previous. The application code calls `IConfiguration["ConnectionStrings:OrdersDb"]` and gets the staging value without knowing it's running in Staging.

**Trade-off:** This adds a small startup cost (loading providers, validating options) but eliminates a class of "wrong-binary-in-Prod" incidents.

**Real project:** On a payments service, we shipped the same image tag through Dev → QA → Staging → Prod. The Prod App Service had a sticky app setting `ASPNETCORE_ENVIRONMENT=Production` plus a Key Vault reference for the SQL connection. The Staging slot had `ASPNETCORE_ENVIRONMENT=Staging` and pointed at a Staging Azure SQL. Swap moved the image, not the environment names — exactly the intended behavior.

### 2. A developer committed a production connection string. How do you remediate?

**Why this matters:** Tests incident response and secret hygiene.

**Answer:** Three immediate steps: (1) rotate the credential — the leaked value is permanently compromised regardless of `git rm`; (2) purge it from history with `git filter-repo` or BFG; (3) audit access logs on the resource (SQL audit logs, Key Vault access logs) for the period the secret was exposed. Prevention: enable GitHub push protection and secret scanning, enforce branch protection requiring secret scan, and move the secret behind a Key Vault reference so the config file holds only the reference URI.

**Trade-off:** Rotation may briefly break running services. Coordinate the rotation with a deploy that picks up the new secret.

**Real project:** A junior committed a Service Bus connection string. We rotated within 15 minutes, verified no anomalous access in the Service Bus audit log, force-pushed cleaned history, and added pre-commit `gitleaks` to the repo. The repo now uses Key Vault references for every secret.

### 3. How do you validate configuration so a typo doesn't reach production?

**Why this matters:** Distinguishes engineers who treat config as code from those who treat it as text.

**Answer:** Use `AddOptions<T>().Bind(...).ValidateDataAnnotations().ValidateOnStart()`. The `[Required]`, `[Range]`, and `[Url]` attributes (or a custom `IValidateOptions<T>`) run at host startup. A missing or out-of-range value throws `OptionsValidationException` before the host opens its listening socket — Kubernetes marks the pod `CrashLoopBackOff`, the readiness probe never passes, and no traffic reaches the bad config. Combine this with a CI job that runs `dotnet build` against the production parameter file to catch schema drift before deploy.

**Trade-off:** Strict validation can block deploys for legitimate gradual rollouts of new keys. Default values + nullable annotations help.

**Real project:** An invoicing service required `Smtp:Host`. A misconfigured Staging slot had `Smtp__Hos` (typo). With `ValidateOnStart`, the slot failed to start, the swap aborted, and Production stayed healthy. Without it, Staging would have started and then 500'd on the first email send.

### 4. Explain the difference between GitHub repository secrets, environment secrets, and variables.

**Why this matters:** Most pipeline incidents come from putting secrets in the wrong scope.

**Answer:** **Repository secrets** are available to every workflow in the repo with no gate — appropriate only for non-production credentials. **Environment secrets** are scoped to a named environment (e.g., `production`) and can require reviewer approval, branch protection, and a wait timer — the right place for Production deploy credentials. **Variables** (repository or environment) are non-secret values visible in logs — appropriate for `DOTNET_VERSION`, region names, image tag patterns. The deploy-to-prod job declares `environment: production`, which forces GitHub to evaluate the gates before exposing the environment secrets to the job.

**Trade-off:** Approvals slow down emergency hotfixes. Many teams allow a designated on-call group to self-approve while still requiring two-person review for normal releases.

**Real project:** Our `deploy-prod.yml` job uses `environment: production` with two required reviewers and a 5-minute wait timer. The `AZURE_CREDENTIALS_PROD` secret is environment-scoped — a feature-branch PR cannot trigger a Prod deploy because the gates block the job before the secret is available.

### 5. Why use Azure App Configuration when App Service already has app settings?

**Why this matters:** Tests architectural reasoning about centralized config.

**Answer:** App Service settings are per-resource — changing one means logging into the portal or running an `az webapp config` command for each of 12 regional App Services. Azure App Configuration is **centralized**: one store, labeled per environment, consumed by Functions, App Services, AKS pods, and Container Apps alike. It supports **feature flags** with percentage rollout, **dynamic refresh** via sentinel keys (no app restart needed), **point-in-time snapshots** for audit, and **Key Vault integration**. App settings remain useful for boot-critical values like the App Configuration endpoint itself.

**Trade-off:** App Configuration adds a dependency in the boot path. Always have local fallback defaults so a transient App Configuration outage doesn't take down the service.

**Real project:** A 12-region order API uses App Configuration to share connection strings, feature flags, and rate limits. The App Service-level setting holds only `AppConfig__Endpoint` and the managed-identity bootstrap. Adding a new region took 4 minutes; previously, updating 12 separate App Service config blobs took an hour and was error-prone.

### 6. What goes in `appsettings.json` vs `appsettings.Production.json` vs environment variables vs Key Vault?

**Why this matters:** Tests judgment about layering.

**Answer:** `appsettings.json` holds **shared defaults** that apply everywhere (default timeouts, MIME types). `appsettings.{Environment}.json` holds **environment-specific non-secrets** (BaseAddress per environment, log levels). **Environment variables** hold **deployment-specific** values that the orchestrator injects (region, instance id, image tag). **Key Vault references** hold **secrets** — connection strings, API keys, certificates. The hierarchy is intentional: later layers override earlier, and the most sensitive values come from the most controlled source.

**Trade-off:** More layers means more places to look when debugging. Standardize on the layout and document it.

**Real project:** A SaaS team had identical bugs in three environments traced to copy-pasted overrides between `appsettings.QA.json` and `appsettings.Staging.json`. We moved all environment differences to App Configuration labels and reduced `appsettings.*.json` to only the App Configuration endpoint. The number of "config drift" tickets dropped to near zero.

### 7. How would you implement zero-downtime config changes for a feature rollout?

**Why this matters:** Tests understanding of dynamic configuration.

**Answer:** Put the toggle in Azure App Configuration as a feature flag, use `Microsoft.FeatureManagement` with `IFeatureManager.IsEnabledAsync("NewCheckoutFlow")` in the code, and configure refresh with a sentinel key and a 30-second cache. Flipping the flag in the portal propagates to all instances within 30 seconds. For gradual rollouts use **TargetingFilter** with a percentage or specific user groups — e.g., 5% of traffic gets the new flow, ramping to 100% over a week. Rollback is flipping the flag back; no redeploy, no incident bridge.

**Trade-off:** Old code paths must remain in the codebase until the flag is fully retired — flag debt. Schedule cleanup tasks.

**Real project:** We rolled out a redesigned cart calculation. The flag started at 1% of orders, ramped to 25% over two days while we watched checkout error rates and revenue per session. At day three we caught a discount-edge-case bug, flipped the flag to 0% in 5 seconds, fixed the bug, and resumed the rollout.

### 8. How do you keep `appsettings.Development.json` from leaking developer-only behavior to Production?

**Why this matters:** Tests the candidate's grasp of the host environment defaulting to Production.

**Answer:** The ASP.NET Core host reads `ASPNETCORE_ENVIRONMENT`; if it's not set the runtime assumes Production and **never loads** `appsettings.Development.json`. The risk is that someone deploys the image with `ASPNETCORE_ENVIRONMENT=Development` set — the file is present in the build output. Mitigations: (1) include `appsettings.Development.json` only in the developer build via `<Content Include="appsettings.Development.json" CopyToOutputDirectory="Never" />` and User Secrets instead; (2) add a startup guard that throws if `env.IsDevelopment()` is true on a machine without a `IS_DEV_MACHINE=true` env var; (3) audit the deployment YAML to ensure `ASPNETCORE_ENVIRONMENT` is explicitly set per slot.

**Trade-off:** Stripping the Dev file from the published artifact means new developers cannot run the binary directly without setting User Secrets — a friction worth paying for.

**Real project:** A junior pushed a release where the App Service's `ASPNETCORE_ENVIRONMENT` was accidentally unset. The default-to-Production fail-safe saved us — Swagger and verbose logging stayed off. We then added an explicit `ASPNETCORE_ENVIRONMENT` to every Bicep parameter file so the value is never implicit.

## Summary Checklist

- [ ] I can explain how `IConfiguration` builds from layered providers.
- [ ] I can use `IHostEnvironment` and `IsDevelopment()` correctly without branching business logic.
- [ ] I can bind typed `IOptions<T>` with `ValidateDataAnnotations()` and `ValidateOnStart()`.
- [ ] I know to use `__` (double underscore) for nested keys in environment variables.
- [ ] I can configure Azure App Service deployment slots with sticky settings.
- [ ] I can use GitHub Actions environments with required reviewers for Production deploys.
- [ ] I can parameterize Bicep deployments with `.bicepparam` per environment.
- [ ] I can connect Azure App Configuration with labels and Key Vault references.
- [ ] I never commit secrets to Git and use managed identity for Azure resources.
- [ ] I can roll out a feature flag without a redeploy.
