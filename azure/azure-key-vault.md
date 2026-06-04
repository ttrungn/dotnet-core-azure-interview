# Azure Key Vault

## What It Is

Azure Key Vault is a managed secret store that holds three kinds of sensitive material:

- **Secrets** — arbitrary strings (connection strings, API keys, webhook signing keys).
- **Keys** — RSA/EC cryptographic keys used for encryption, signing, or wrapping. Can be HSM-backed (Premium tier, FIPS 140-2 Level 2/3).
- **Certificates** — X.509 certificates (TLS for App Gateway, mTLS client certs) with automated renewal hooks for integrated issuers.

A vault has a DNS endpoint like `https://contoso-prod-kv.vault.azure.net/`. Apps authenticate to it with **Microsoft Entra (Azure AD) identities** — almost always a **Managed Identity** for code running in Azure, never a stored credential.

```csharp
// Resolve a payment provider API key at startup using Managed Identity
var client = new SecretClient(
    new Uri("https://contoso-prod-kv.vault.azure.net/"),
    new DefaultAzureCredential());

KeyVaultSecret secret = await client.GetSecretAsync("Stripe--ApiKey", cancellationToken: ct);
string stripeKey = secret.Value;
```

The app never sees a vault password. Entra issues a short-lived OAuth token, Key Vault validates it, RBAC authorizes the read, and the secret comes back over TLS.

## Why It Exists

Before Key Vault, secrets lived in:

1. `web.config` / `appsettings.json` checked into Git — leaked the moment the repo went public or was cloned to a laptop.
2. CI/CD variable groups — visible to anyone with pipeline edit rights, no audit of who read what.
3. `.env` files copied between developer machines over chat.
4. Bare environment variables on VMs — invisible to security review, never rotated.

The pain that drove Key Vault:

- A leaked Stripe key in a public commit costs real money within minutes.
- PCI-DSS, SOC 2, HIPAA, and ISO 27001 all require **centralized secret storage, rotation, and audit trails**. A `.config` file fails every one.
- Rotating a database password meant editing config across 12 services and redeploying.
- No one could answer "who read the production payment key last Tuesday?".

Key Vault gives one tenant-wide, audited, RBAC-controlled store with automatic rotation hooks and 90-day soft-delete safety.

## When To Use It

**Use Key Vault for:**

- Database connection strings (SQL, Cosmos, PostgreSQL).
- Third-party API keys (Stripe, SendGrid, Twilio, Adyen).
- Symmetric signing keys (JWT issuer keys, webhook HMAC secrets).
- TLS certificates for App Gateway, Front Door, or custom App Service domains.
- Service Bus / Storage / Event Hubs connection strings *if* you must use them (prefer Managed Identity to the resource itself).
- Encryption keys for **customer-managed keys (CMK)** on Storage, SQL TDE, Cosmos.
- Anything a security scanner would flag as a "credential".

**Do not use Key Vault for:**

- Non-sensitive configuration (feature flags, retry counts, base URLs) — use **Azure App Configuration** or `appsettings.json`. Key Vault is throttled (~2,000 requests / 10 seconds per vault) and costs per operation.
- High-volume per-request data lookups — cache or use App Configuration.
- Storing customer secrets at scale (millions of users) — use a customer secrets vault pattern with Cosmos + envelope encryption.
- Large binary blobs — use Blob Storage with encryption.

## Why It Is Important

In production .NET systems on Azure, Key Vault is the single load-bearing component for **zero-trust secret handling**. It directly enables:

- **Compliance** — SOC 2 CC6.1, PCI-DSS 3.5, ISO 27001 A.10.1 all map cleanly to "secrets in Key Vault + RBAC + audit log".
- **Rotation without redeploy** — change a secret value, services pick up the new version on next read or via Event Grid notification.
- **Blast radius containment** — a compromised App Service can only access what its Managed Identity is RBAC-granted; it cannot dump every secret in the tenant.
- **Audit forensics** — every `GetSecret` is logged to a Log Analytics workspace with caller identity, IP, and version.
- **Soft-delete + purge protection** — accidental deletion is recoverable for 90 days, and even a tenant admin cannot permanently destroy a secret without satisfying purge protection delay.

If an interviewer asks "how do you store the Stripe API key?", the answer "Key Vault, accessed via Managed Identity, RBAC-scoped to the API's identity" is the bar.

## How It's Used in C# / .NET

NuGet packages:

- `Azure.Security.KeyVault.Secrets` — direct secret access (`SecretClient`).
- `Azure.Security.KeyVault.Keys` / `Azure.Security.KeyVault.Certificates` — keys and certs.
- `Azure.Identity` — provides `DefaultAzureCredential`, `ManagedIdentityCredential`, `ChainedTokenCredential`.
- `Azure.Extensions.AspNetCore.Configuration.Secrets` — loads vault secrets as `IConfiguration` entries.
- `Microsoft.Extensions.Azure` — `AddAzureClients` for DI registration of `SecretClient` with shared credential and retry policy.

### Pattern 1 — Configuration provider (preferred for app settings)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

var vaultUri = new Uri(builder.Configuration["KeyVault:Uri"]
    ?? throw new InvalidOperationException("KeyVault:Uri missing"));

builder.Configuration.AddAzureKeyVault(
    vaultUri,
    new DefaultAzureCredential(),
    new AzureKeyVaultConfigurationOptions
    {
        ReloadInterval = TimeSpan.FromMinutes(15) // poll for rotated secrets
    });

// Secret named "ConnectionStrings--OrdersDb" in the vault becomes
// builder.Configuration.GetConnectionString("OrdersDb")
```

The `--` separator in the secret name maps to `:` in `IConfiguration`, matching .NET's hierarchical key convention.

### Pattern 2 — Direct `SecretClient` (for runtime fetch / rotation-aware code)

```csharp
builder.Services.AddAzureClients(clients =>
{
    clients.AddSecretClient(new Uri(builder.Configuration["KeyVault:Uri"]!));
    clients.UseCredential(new DefaultAzureCredential());
});

// In a service
public sealed class StripeClientFactory
{
    private readonly SecretClient _secrets;
    private readonly IMemoryCache _cache;

    public StripeClientFactory(SecretClient secrets, IMemoryCache cache)
    {
        _secrets = secrets;
        _cache = cache;
    }

    public async Task<string> GetApiKeyAsync(CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync("stripe-key", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            KeyVaultSecret secret = await _secrets.GetSecretAsync("Stripe--ApiKey", cancellationToken: ct);
            return secret.Value;
        }) ?? throw new InvalidOperationException("Stripe key unavailable");
    }
}
```

### Pattern 3 — Key Vault references in App Service (no SDK code at all)

App Service and Functions can resolve `@Microsoft.KeyVault(...)` references in app settings automatically using the site's Managed Identity:

```
StripeApiKey = @Microsoft.KeyVault(VaultName=contoso-prod-kv;SecretName=Stripe--ApiKey)
```

The platform fetches the secret, injects the value as an environment variable, and refreshes every 24 hours. Best when the app cannot or should not take a direct SDK dependency.

### Bicep — provisioning with RBAC

```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'contoso-prod-kv'
  location: location
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true        // RBAC, not legacy access policies
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true          // cannot be undone, required for compliance
    publicNetworkAccess: 'Disabled'      // private endpoint only
  }
}

// Grant API's Managed Identity read-only access to secrets
resource secretsUser 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: kv
  name: guid(kv.id, apiPrincipalId, 'KeyVaultSecretsUser')
  properties: {
    principalId: apiPrincipalId
    principalType: 'ServicePrincipal'
    // Key Vault Secrets User (read only)
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '4633458b-17de-408a-b874-0445c86b69e6')
  }
}
```

## Advantages

- **Centralized secret storage** with one audit trail across all environments.
- **Managed Identity integration** eliminates the "how do we authenticate to the secret store" chicken-and-egg problem.
- **Soft delete + purge protection** prevents accidental and malicious destruction.
- **Versioning** — every secret update keeps history; apps can pin to a specific version during rollback.
- **Rotation hooks** — Event Grid emits `Microsoft.KeyVault.SecretNewVersionCreated`, letting consumers refresh caches.
- **HSM-backed keys** (Premium / Managed HSM) for FIPS 140-2 Level 3 compliance.
- **Private Endpoint** support — vault traffic stays on the Microsoft backbone, never the public internet.
- **RBAC at the secret level** (Premium) for fine-grained least privilege.

## Disadvantages

- **Throttling** — 2,000 requests per 10 seconds per vault. Apps that read secrets per request will get HTTP 429.
- **Latency** — every secret fetch is a network round-trip (~30–100 ms). Always cache.
- **Cost per operation** — at high volume the operations bill can dominate.
- **Cold start penalty** — first `GetSecret` on a new pod adds latency to the first request.
- **Region failure** — a single vault is regional. Multi-region apps need replicated vaults or read-through cache.
- **Complex RBAC migration** — legacy access policies and new RBAC mode coexist confusingly; mixing them causes silent permission failures.
- **Purge protection is irreversible** — turning it on locks you in for the retention period.

## Common Mistakes

### 1. Storing secrets in `appsettings.json` and "fixing it later"

```csharp
// ❌ Committed to Git, visible to every developer
{
  "Stripe": { "ApiKey": "sk_live_51HxYz..." }
}
```

**Fix:** Put the value in Key Vault, reference it via the configuration provider:

```csharp
// ✅ appsettings.json holds only the vault URI
{ "KeyVault": { "Uri": "https://contoso-prod-kv.vault.azure.net/" } }

builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!),
    new DefaultAzureCredential());

// Resolves "Stripe--ApiKey" from the vault
var key = builder.Configuration["Stripe:ApiKey"];
```

### 2. Using a service principal with a client secret instead of Managed Identity

```csharp
// ❌ The client secret is itself a credential that must be stored somewhere
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
```

**Fix:** Use the platform Managed Identity — no secret to store, rotate, or leak:

```csharp
// ✅ Works in App Service, Functions, Container Apps, AKS (workload identity), VMs
var credential = new DefaultAzureCredential();
```

### 3. Fetching the secret on every request

```csharp
// ❌ Hits the vault on every HTTP call, will hit 429 under load
public async Task<IActionResult> Charge(ChargeRequest req)
{
    var key = (await _secrets.GetSecretAsync("Stripe--ApiKey")).Value;
    return Ok(await _stripe.ChargeAsync(req, key));
}
```

**Fix:** Cache the secret in memory; refresh on rotation event or on a timer:

```csharp
// ✅ Cached for 10 minutes; refreshes via Event Grid SecretNewVersionCreated
var key = await _cache.GetOrCreateAsync("stripe-key", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    return (await _secrets.GetSecretAsync("Stripe--ApiKey")).Value;
});
```

### 4. Mixing RBAC and access policies on the same vault

When `enableRbacAuthorization = true` is set, **all access policies are ignored**. Teams flip the switch, then wonder why production breaks.

**Fix:** Pick one model per vault. For new vaults, always RBAC. Document the choice in the Bicep file.

### 5. Disabling purge protection "for now" in production

```bicep
// ❌ A compromised admin can permanently destroy production secrets in seconds
enablePurgeProtection: false
```

**Fix:** Purge protection is mandatory in production. Combined with soft delete it gives a 90-day undo window:

```bicep
// ✅ Cannot be turned off; required for SOC 2 / PCI compliance
enableSoftDelete: true
softDeleteRetentionInDays: 90
enablePurgeProtection: true
```

### 6. Hard-coding the vault URI per environment in code

```csharp
// ❌ Code must be recompiled to point at a different vault
var uri = new Uri("https://contoso-prod-kv.vault.azure.net/");
```

**Fix:** Read the URI from `IConfiguration` and override per environment via App Service settings or environment variables.

### 7. Not handling `RequestFailedException` with status 403

When a Managed Identity loses its role assignment, every read fails with 403. Apps that swallow the exception silently serve stale or default values.

**Fix:** Fail fast at startup with a clear error, and add a health check that probes the vault:

```csharp
builder.Services.AddHealthChecks()
    .AddAzureKeyVault(
        vaultUri,
        new DefaultAzureCredential(),
        options => options.AddSecret("HealthProbe"),
        name: "keyvault",
        tags: new[] { "ready" });
```

## Best Practices

- **One vault per environment per workload boundary**: `contoso-dev-payments-kv`, `contoso-prod-payments-kv`. Never share a vault across dev and prod.
- **Always Managed Identity, never client secrets**, except for genuinely external systems.
- **Enable RBAC authorization mode** on every new vault. Assign least-privilege built-in roles: `Key Vault Secrets User` for readers, `Key Vault Secrets Officer` for writers.
- **Enable soft delete and purge protection** in non-dev environments.
- **Use Private Endpoint** in production. Disable public network access.
- **Cache secrets in-process** with a TTL between 5 and 30 minutes, plus Event Grid invalidation.
- **Rotate secrets on a schedule** — at least every 90 days for high-value secrets, 30 days for payment processors.
- **Use `--` in secret names** to map to `:` in `IConfiguration` (`Stripe--ApiKey` → `Stripe:ApiKey`).
- **Log every access** to a Log Analytics workspace via diagnostic settings; alert on unusual caller IPs or off-hours reads.
- **Pin to secret versions during rollback** — keep the previous version active until the deploy stabilizes.
- **Provision via Bicep / Terraform**, never the portal. Portal-created vaults drift from infrastructure as code.

## Related Concepts

- [azure/configuration-and-secrets-management.md](azure/configuration-and-secrets-management.md) — broader configuration story including App Configuration and Options pattern.
- [azure/app-service.md](azure/app-service.md) — Key Vault references work natively in App Service settings.
- [azure/application-insights.md](azure/application-insights.md) — monitor vault dependency calls and failures.
- [aspnet-core/authentication-and-authorization.md](aspnet-core/authentication-and-authorization.md) — JWT signing keys live in Key Vault.
- [devops/ci-cd-pipelines.md](devops/ci-cd-pipelines.md) — pipelines use Key Vault tasks to fetch deployment-time secrets.
- [azure/deployment-to-azure.md](azure/deployment-to-azure.md) — provisioning vaults with Bicep and OIDC federated credentials.

## Real-World Usage

### Payment APIs

A checkout API hosted on App Service uses Managed Identity to read the Stripe live key from `contoso-prod-payments-kv`. The key is cached for 10 minutes. When platform engineering rotates the key in the vault, Event Grid notifies a webhook that invalidates the cache. The rotation deploys zero code changes and zero downtime.

### Multi-region failover

For an order processing system running in West Europe and North Europe, two vaults (`contoso-weu-kv` and `contoso-neu-kv`) are kept in sync by a rotation Azure Function that writes to both. Each regional App Service reads from its local vault. If a region fails, Front Door reroutes; the surviving region keeps serving without secret-store latency penalties.

### Zero-trust internal services

An internal admin API in AKS uses **workload identity** (federated Managed Identity) to read API keys for internal microservices from Key Vault. No pod has a long-lived credential; tokens are issued per-pod and expire in 24 hours. Network policy locks vault traffic to a Private Endpoint inside the cluster's VNet.

### Customer-managed keys (CMK)

A healthcare app stores customer PHI in Azure SQL with Transparent Data Encryption backed by a CMK in Key Vault. The customer's compliance team holds the key permissions; revoking access in the vault renders the database unreadable within minutes — a contractual requirement.

### CI/CD

A GitHub Actions workflow uses an OIDC federated credential (no stored secret) to assume an Azure identity, then pulls deployment-time secrets (test database password, integration test API keys) from a dedicated `contoso-cicd-kv` vault. The pipeline identity has read access only to secrets prefixed with `Pipeline--`.

## Code Example — Before and After

### Before — secrets in config, scattered access, no rotation

```csharp
// appsettings.Production.json (committed to repo)
{
  "ConnectionStrings": {
    "OrdersDb": "Server=tcp:prod-sql.database.windows.net;User Id=admin;Password=P@ssw0rd123!;..."
  },
  "Stripe": { "ApiKey": "sk_live_51HxYz..." },
  "SendGrid": { "ApiKey": "SG.xK7..." }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<OrdersDbContext>(opts =>
    opts.UseSqlServer(builder.Configuration.GetConnectionString("OrdersDb")));

builder.Services.AddSingleton<IStripeClient>(_ =>
    new StripeClient(builder.Configuration["Stripe:ApiKey"]));
```

**Problems:**
- Production password in Git history forever.
- Rotating the Stripe key requires a code change + deploy.
- No audit trail of who read what.
- Local dev machines hold production credentials.

### After — Key Vault + Managed Identity + cached rotation-aware access

```csharp
// appsettings.json (no secrets, safe to commit)
{
  "KeyVault": { "Uri": "https://contoso-prod-payments-kv.vault.azure.net/" },
  "Stripe": { "CacheMinutes": 10 }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Load all secrets as configuration entries; "ConnectionStrings--OrdersDb" → ConnectionStrings:OrdersDb
builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!),
    new DefaultAzureCredential(),
    new AzureKeyVaultConfigurationOptions
    {
        ReloadInterval = TimeSpan.FromMinutes(15)
    });

builder.Services.AddDbContext<OrdersDbContext>(opts =>
    opts.UseSqlServer(builder.Configuration.GetConnectionString("OrdersDb")));

// Register SecretClient for runtime rotation-aware reads
builder.Services.AddAzureClients(clients =>
{
    clients.AddSecretClient(new Uri(builder.Configuration["KeyVault:Uri"]!));
    clients.UseCredential(new DefaultAzureCredential());
});

builder.Services.AddMemoryCache();
builder.Services.AddSingleton<IStripeKeyProvider, StripeKeyProvider>();
builder.Services.AddScoped<IStripeClient>(sp =>
{
    var key = sp.GetRequiredService<IStripeKeyProvider>().GetAsync(CancellationToken.None).GetAwaiter().GetResult();
    return new StripeClient(key);
});

builder.Services.AddHealthChecks()
    .AddAzureKeyVault(
        new Uri(builder.Configuration["KeyVault:Uri"]!),
        new DefaultAzureCredential(),
        opts => opts.AddSecret("HealthProbe"),
        name: "keyvault",
        tags: new[] { "ready" });

// StripeKeyProvider.cs
public interface IStripeKeyProvider
{
    Task<string> GetAsync(CancellationToken ct);
}

public sealed class StripeKeyProvider : IStripeKeyProvider
{
    private readonly SecretClient _secrets;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _ttl;
    private readonly ILogger<StripeKeyProvider> _logger;

    public StripeKeyProvider(
        SecretClient secrets,
        IMemoryCache cache,
        IConfiguration config,
        ILogger<StripeKeyProvider> logger)
    {
        _secrets = secrets;
        _cache = cache;
        _ttl = TimeSpan.FromMinutes(config.GetValue("Stripe:CacheMinutes", 10));
        _logger = logger;
    }

    public async Task<string> GetAsync(CancellationToken ct)
    {
        return (await _cache.GetOrCreateAsync("stripe-key", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _ttl;
            try
            {
                KeyVaultSecret s = await _secrets.GetSecretAsync("Stripe--ApiKey", cancellationToken: ct);
                _logger.LogInformation("Fetched Stripe key version {Version}", s.Properties.Version);
                return s.Value;
            }
            catch (RequestFailedException ex) when (ex.Status == 403)
            {
                _logger.LogCritical(ex, "Managed Identity lacks Key Vault Secrets User role");
                throw;
            }
        }))!;
    }
}
```

**Why this is better:**
- Zero secrets in source control.
- Managed Identity — no credential to leak.
- 10-minute cache prevents throttling under load.
- Health check catches misconfigured RBAC at deploy time.
- Structured logs record every key fetch with the secret version, enabling rollback diagnosis.
- Rotation is a vault operation, not a deploy.

## Interview Questions and Answers

### 1. A teammate proposes storing the Stripe API key in an App Service application setting "because it's already encrypted at rest". How do you respond?

**Why this matters:** Shows whether you understand the difference between encryption at rest and proper secret management — audit, rotation, least privilege.

**Answer:** App Service settings *are* encrypted at rest, but the model breaks down on three fronts: any contributor on the resource group can read every setting in the portal; there is no central audit trail of who read what when; and rotation requires editing the setting in every slot of every environment. Key Vault solves all three with RBAC, diagnostic logs, and one rotation operation per secret.

**Trade-off:** App settings are simpler — no extra service, no Managed Identity to configure. Acceptable for non-sensitive config like base URLs.

**Real project:** On a checkout service we migrated 14 App Service settings to Key Vault references over a single sprint. Audit revealed three developers had read the live Stripe key in the prior month — none should have.

### 2. The API is hitting HTTP 429 from Key Vault during traffic spikes. Walk me through the fix.

**Why this matters:** Tests practical knowledge of throttling limits and caching strategy.

**Answer:** Key Vault throttles at 2,000 requests per 10 seconds per vault. The cause is almost always per-request secret reads. The fix is in-process caching with a TTL of 5–15 minutes. For rotation-awareness, subscribe a webhook to the `Microsoft.KeyVault.SecretNewVersionCreated` Event Grid topic and invalidate the cache when fired. If multiple workloads share one vault, split into per-workload vaults to multiply the budget.

**Trade-off:** A longer cache TTL reduces vault load but lengthens the window where rotated secrets keep using the old value. 10 minutes is the usual compromise.

**Real project:** During Black Friday an order API hit 429 within minutes of launch. We added `IMemoryCache` with a 5-minute TTL and the issue disappeared.

### 3. Why is Managed Identity preferred over a service principal with a client secret?

**Why this matters:** Probes whether you grasp the bootstrapping problem of "how do I authenticate to the secret store".

**Answer:** A service principal client secret is itself a secret — it must be stored somewhere, rotated, and protected. That just moves the problem one level. Managed Identity eliminates the bootstrap secret entirely: Azure provisions a federated identity bound to the resource, and the SDK exchanges resource metadata for a short-lived OAuth token. There is nothing to leak, rotate, or commit. The only valid reason to use a client secret is when the caller is outside Azure (and even then, prefer federated credentials via OIDC).

**Trade-off:** Managed Identity only works for code running on Azure compute. Cross-cloud or on-prem callers still need federated credentials or a client secret.

### 4. You're designing a multi-region active-active deployment for a payments service. How do you architect Key Vault?

**Why this matters:** Tests understanding of regional service boundaries and failure modes.

**Answer:** Key Vault is a regional service — a vault in West Europe is unreachable if the region is down. For active-active I'd deploy one vault per region (`-weu-kv`, `-neu-kv`), and run a rotation Function that writes secret updates to both. Each regional App Service reads only its local vault, so a region failure has zero cross-region dependency. Premium Managed HSM has a multi-region replication option if HSM-backed keys are required.

**Trade-off:** Two vaults double the audit-log surface and require disciplined rotation tooling. For lower RTO requirements, a single vault with a read-through cache and a documented manual failover is cheaper.

**Real project:** For a payments platform serving Europe we ran paired vaults and a rotation Function in a hub VNet. RPO was ~30 seconds, RTO essentially zero.

### 5. Walk me through what soft-delete and purge protection prevent.

**Why this matters:** Compliance and incident-response awareness.

**Answer:** Soft delete means a deleted secret/key/cert is recoverable for the configured retention period (7–90 days). Purge protection further blocks anyone — including subscription owners — from permanently destroying soft-deleted items before retention expires. Together they defend against accidental deletion by an engineer and malicious deletion by a compromised admin account. SOC 2 and PCI-DSS effectively require both in production. The cost is irreversibility: once purge protection is on, you cannot turn it off until retention elapses.

**Trade-off:** In dev environments where vaults are recreated frequently, purge protection causes "vault name already in soft-delete" deploy errors. Disable in dev, enable in staging and prod.

### 6. A developer's local machine is failing to read from Key Vault with a 403. Production works. What do you check?

**Why this matters:** Debugging Managed Identity vs. local auth flows.

**Answer:** `DefaultAzureCredential` walks a chain: environment variables → Managed Identity → Visual Studio → Azure CLI → Azure PowerShell → interactive browser. Locally it usually resolves to the developer's CLI identity (`az login`). The 403 means that CLI identity is not assigned the `Key Vault Secrets User` role on the dev vault. Fix by granting the developer's user or AAD group the role on the dev vault scope, not by adding a client secret to local config.

**Trade-off:** Granting individuals direct vault access doesn't scale. Use an AAD group like `payments-developers` and assign the role to the group.

### 7. How do you handle secret rotation for a database password used by 30 microservices?

**Why this matters:** Real operational scenario; tests coordination strategy.

**Answer:** Two-phase rotation: (1) provision a new password and store it as a new version of the secret in Key Vault — both old and new are valid on the database for an overlap window; (2) services pick up the new value on cache expiry; (3) after the overlap window, retire the old password. Event Grid notifications + a short cache TTL keep the overlap window small (10–15 minutes). For zero-downtime database password rotation specifically, Azure SQL with Entra-only authentication eliminates the password entirely — services authenticate with Managed Identity and the question disappears.

**Trade-off:** Entra-only auth requires every consumer to support token-based SQL connections; legacy ADO.NET code may need refactoring.

### 8. What's the difference between `Key Vault Secrets User` and `Key Vault Secrets Officer`?

**Why this matters:** Least-privilege RBAC literacy.

**Answer:** `Key Vault Secrets User` grants read-only access to secret values — the right role for application Managed Identities. `Key Vault Secrets Officer` adds create, update, and delete — the right role for rotation Functions or DevOps pipelines. Granting an application the Officer role is overreach: a compromised app could delete every secret. Always start at User and escalate per workflow if required.

## Summary Checklist

- [ ] I can explain why secrets belong in Key Vault rather than `appsettings.json` or App Service settings.
- [ ] I can authenticate to Key Vault using Managed Identity with `DefaultAzureCredential`.
- [ ] I know how to load Key Vault secrets as `IConfiguration` entries.
- [ ] I cache secrets in-process and refresh on Event Grid rotation events.
- [ ] I configure soft delete, purge protection, RBAC mode, and Private Endpoint on production vaults.
- [ ] I assign least-privilege roles (`Key Vault Secrets User` for readers).
- [ ] I can diagnose 403 (RBAC) and 429 (throttling) failures.
- [ ] I design multi-region deployments with paired regional vaults.
- [ ] I provision vaults via Bicep, not the portal.
- [ ] I monitor vault access with Log Analytics diagnostic logs and alerts.
