# Azure App Service

## What It Is

Azure App Service is a **fully managed Platform-as-a-Service (PaaS)** for hosting web apps, REST APIs, and mobile backends. You upload your compiled ASP.NET Core artifact (a `.zip`, a Docker image, or a Git push), and Azure runs it on a managed fleet of VMs that you never have to log into, patch, or scale by hand.

Three pieces make up the service:

1. **App Service Plan** — the compute (CPU, RAM, instance count) that one or more apps run on. You pick the SKU (`F1`, `B1`, `S1`, `P1v3`, `I1v2` etc.) and the OS (Linux or Windows).
2. **Web App / App Service** — the actual application bound to a plan. Has its own URL (`https://orders-api.azurewebsites.net`), TLS cert, app settings, and slots.
3. **Deployment Slots** — staging copies of the app that share the plan, used for blue-green releases via slot swap.

```text
┌─────────────────── App Service Plan (P1v3 Linux, 2 instances) ──────────────────┐
│                                                                                  │
│   ┌─── orders-api (production slot) ───┐  ┌─── orders-api (staging slot) ────┐  │
│   │   ASP.NET Core 8, Managed Identity │  │   ASP.NET Core 8, same image     │  │
│   │   Hits prod SQL, prod Key Vault    │  │   Hits prod SQL, prod Key Vault  │  │
│   └────────────────────────────────────┘  └──────────────────────────────────┘  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

You write the .NET code; Azure handles the OS, runtime upgrades, load balancer, TLS termination, autoscaling, and traffic routing between slots.

## Why It Exists

Before PaaS, every team running a .NET API in production had to:

- Provision Windows or Linux VMs, install the .NET runtime, configure IIS or Kestrel.
- Set up a load balancer manually, terminate TLS on a reverse proxy.
- Patch the OS monthly, reboot in maintenance windows, monitor disk space.
- Write custom deployment scripts that copied files, recycled app pools, and prayed nothing was in flight.
- Build their own blue-green deployment system to avoid downtime during releases.

That overhead consumed weeks of every release cycle and was a recurring source of outages — "the deployment failed because someone patched the VM yesterday."

App Service exists to make hosting a web application a **commodity**. You declare the SKU and region, push your code, and Azure handles everything from the load balancer down. Compared to running on Kubernetes (AKS) or raw VMs, App Service is the lowest-operational-overhead way to host an ASP.NET Core API in Azure.

## When To Use It

**Use App Service for:**

- **HTTP-based ASP.NET Core APIs and web apps** — REST APIs for an order/checkout system, internal admin portals, customer-facing storefronts.
- **Background workers tied to HTTP triggers** — webhook receivers (Stripe, GitHub, payment gateways), scheduled cleanup jobs that you trigger via WebJobs.
- **Multi-tenant SaaS** — host one app per tenant on Isolated tier, or one shared app on Premium with per-tenant routing.
- **Lift-and-shift of legacy ASP.NET 4.x apps** — Windows plans support full .NET Framework with classic IIS modules.
- **Teams without dedicated platform engineers** — when nobody on the team wants to babysit Kubernetes.

**Do not use App Service for:**

- **Event-driven, bursty workloads** — use [azure-functions.md](azure-functions.md) on Consumption plan. App Service plans are billed for the instance whether idle or busy.
- **Containerized workloads needing custom networking, sidecars, or service mesh** — use Azure Container Apps or AKS.
- **Long-running CPU-bound jobs (>30 min per request)** — App Service times out long requests at 230 seconds for the front-end gateway. Use a queue + worker (Service Bus + Functions or Container Apps).
- **Stateful workloads needing local SSD** — App Service file system is slow network storage. Use Cosmos DB, Azure SQL, or Storage instead.
- **Workloads requiring SSH, kernel modules, or root access** — use VMs or AKS.

## Why It Is Important

App Service is the most common Azure compute service you will be asked about in a .NET interview because it sits at the intersection of three production concerns every senior engineer must understand:

1. **Operations cost** — App Service costs are dominated by the **plan SKU**, not the number of apps. Picking `P1v3` when `B1` would work wastes thousands of dollars per year; picking `B1` when `P1v3` is needed causes throttling and lost revenue.
2. **Release safety** — Slot swaps are the standard zero-downtime deployment pattern in Azure. Knowing how to configure slot-sticky settings, warm up the staging slot, and roll back via a reverse swap is a daily operational skill.
3. **Identity and secrets** — App Service has first-class **Managed Identity** support. Hard-coded connection strings in `appsettings.json` are an anti-pattern; Managed Identity + Key Vault references is the accepted standard.

If you understand App Service plans, slots, autoscale rules, and Managed Identity, you can deploy and operate the vast majority of .NET workloads in Azure.

## How It's Used in C# / .NET

A real order API typically uses App Service like this:

### 1. Configure the host with `IConfiguration` providers

App Service exposes **App Settings** and **Connection Strings** as environment variables that automatically flow into `IConfiguration`. You add Key Vault and App Configuration as additional providers.

```csharp
using Azure.Identity;
using Microsoft.Extensions.Azure;

var builder = WebApplication.CreateBuilder(args);

// Managed Identity: works on App Service via the IDENTITY_ENDPOINT env var,
// locally via az login / Visual Studio. Never pass keys.
var credential = new DefaultAzureCredential();

// Pull secrets from Key Vault (production override of appsettings.json)
var keyVaultUri = builder.Configuration["KeyVault:Uri"]
    ?? throw new InvalidOperationException("KeyVault:Uri is required");
builder.Configuration.AddAzureKeyVault(new Uri(keyVaultUri), credential);

// Pull feature flags & tenant config from App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(builder.Configuration["AppConfig:Endpoint"]!), credential)
           .UseFeatureFlags();
});
```

### 2. Register Azure SDK clients via `Microsoft.Extensions.Azure`

```csharp
builder.Services.AddAzureClients(clients =>
{
    clients.UseCredential(credential);

    clients.AddBlobServiceClient(new Uri(builder.Configuration["Storage:BlobUri"]!));
    clients.AddServiceBusClientWithNamespace(builder.Configuration["ServiceBus:Namespace"]!);
    clients.AddSecretClient(new Uri(builder.Configuration["KeyVault:Uri"]!));
});
```

No connection strings, no SAS tokens — just URIs and Managed Identity.

### 3. Wire up health checks (App Service uses them for instance health)

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("OrdersDb")!,
                  name: "sql", tags: new[] { "ready" })
    .AddAzureServiceBusQueue(
        fullyQualifiedNamespace: builder.Configuration["ServiceBus:Namespace"]!,
        queueName: "order-events",
        tokenCredential: credential,
        tags: new[] { "ready" });

var app = builder.Build();

app.MapHealthChecks("/health/live");                            // is the process running?
app.MapHealthChecks("/health/ready", new HealthCheckOptions     // can we serve traffic?
{
    Predicate = check => check.Tags.Contains("ready")
});
```

Configure App Service Health Check to `/health/ready` so unhealthy instances are removed from rotation automatically.

### 4. Add Application Insights for distributed tracing

```csharp
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    options.EnableAdaptiveSampling = true;
});
```

When deployed to App Service, you can also enable **codeless instrumentation** via the portal — no NuGet package required.

### 5. Deploy with GitHub Actions + Managed Identity (OIDC, no secrets)

```yaml
# .github/workflows/deploy.yml
name: deploy-orders-api
on:
  push:
    branches: [main]

permissions:
  id-token: write    # required for OIDC federated credential
  contents: read

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }

      - run: dotnet publish src/Orders.Api -c Release -o ./publish

      - uses: azure/login@v2
        with:
          client-id:     ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:     ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: orders-api
          slot-name: staging                # deploy to staging slot
          package: ./publish

      - run: az webapp deployment slot swap -g orders-rg -n orders-api --slot staging
```

### 6. Configure slot-sticky settings (per-slot connection strings, etc.)

Some settings should swap with the slot (the new code), others should stay with the slot (the database the slot points to). Mark the ones that must **not** swap with `--slot-settings`:

```bash
az webapp config appsettings set -g orders-rg -n orders-api --slot staging \
  --slot-settings "ConnectionStrings__OrdersDb=Server=tcp:orders-staging.database.windows.net;..." \
                  "ApplicationInsights__ConnectionString=InstrumentationKey=staging-key"
```

Now when you swap staging → production, the **code** moves but staging keeps pointing at the staging DB and staging Application Insights — preventing the dreaded "new code wrote to prod DB during warm-up" outage.

### 7. Autoscale rules (azure CLI)

```bash
az monitor autoscale create -g orders-rg --resource orders-plan \
  --resource-type Microsoft.Web/serverfarms \
  --name orders-autoscale --min-count 2 --max-count 10 --count 2

az monitor autoscale rule create -g orders-rg --autoscale-name orders-autoscale \
  --condition "CpuPercentage > 70 avg 5m" --scale out 1
az monitor autoscale rule create -g orders-rg --autoscale-name orders-autoscale \
  --condition "CpuPercentage < 25 avg 10m" --scale in 1
```

### App Service Plan SKU cheat sheet

| SKU                 | Use case                            | Per-month (rough)     | Slots | VNet | Always On |
|---------------------|-------------------------------------|------------------------|-------|------|-----------|
| `F1` Free           | Hello-world demos                   | $0                    | 0     | No   | No        |
| `B1` Basic          | Dev/test                            | ~$13                  | 0     | No   | Yes       |
| `S1` Standard       | Low-traffic prod                    | ~$70                  | 5     | No   | Yes       |
| `P1v3` Premium v3   | Real prod APIs, autoscale, VNet     | ~$140                 | 20    | Yes  | Yes       |
| `I1v2` Isolated v2  | Compliance/PCI/ASE, dedicated VNet  | ~$400+                | 20    | Yes  | Yes       |

## Advantages

- **Zero VM/OS management** — Azure patches the OS and runtime.
- **First-class Managed Identity** — no connection strings in code.
- **Deployment slots** — blue-green deploys are a single CLI command.
- **Autoscale** — scale-out rules based on CPU, memory, queue length, or custom metrics.
- **Native CI/CD** — GitHub Actions, Azure DevOps, and `azd up` all have App Service deploy targets.
- **Hybrid Connections** — securely call on-premises SQL/AD/file shares without a full VPN.
- **Built-in TLS, custom domains, and free managed certificates** for `*.azurewebsites.net`.
- **Linux and Windows support** — same product surface; pick Linux for cost and parity with containers.

## Disadvantages

- **Pay-per-plan, not per-request** — even an idle `P1v3` costs ~$140/month per instance. Use Functions Consumption for bursty workloads.
- **Vendor lock-in to Azure** — slots, Hybrid Connections, easyAuth, and Key Vault references don't transfer to other clouds.
- **230-second request timeout at the front-end load balancer** — long-running jobs need a queue + worker pattern.
- **Limited file system** — `D:\home` on Windows / `/home` on Linux is slow shared storage, not local SSD.
- **Cold start on scale-out** — adding a new instance under load takes 30-90 seconds; pre-warm with `Always On` and `WEBSITE_ADD_SITENAME_BINDINGS_IN_APPHOST_CONFIG`.
- **Regional caps** — Premium V3 isn't available in every region; new regions take months to get it.
- **No GPU, no FPGA, no kernel access** — wrong platform for ML inference or video transcoding.

## Common Mistakes

### 1. Hard-coding connection strings in `appsettings.json`

```csharp
// BUG: secret in source control, no rotation, same value for all environments
{
  "ConnectionStrings": {
    "OrdersDb": "Server=tcp:prod-sql.database.windows.net;User Id=admin;Password=P@ssw0rd123;"
  }
}
```

**Fix**: use Managed Identity + Key Vault reference. The App Service setting becomes:

```text
ConnectionStrings__OrdersDb = @Microsoft.KeyVault(VaultName=orders-kv;SecretName=OrdersDbConnString)
```

Or skip the password entirely with Azure AD authentication:

```text
Server=tcp:prod-sql.database.windows.net;Database=Orders;Authentication=Active Directory Default;
```

### 2. Forgetting to mark connection strings as slot-sticky

```bash
# BUG: this setting swaps with the slot. After a swap, prod code points at staging DB.
az webapp config appsettings set --settings "ConnectionStrings__OrdersDb=..."
```

**Fix**: use `--slot-settings` so the setting stays bound to the slot, not the code:

```bash
az webapp config appsettings set --slot-settings "ConnectionStrings__OrdersDb=..."
```

### 3. Picking the wrong tier and discovering it under load

```text
"We're on B1 because it's cheap. Why does the API throttle at 50 RPS?"
```

`B1` has 1.75 GB RAM, 1 vCPU, and no autoscale. **Fix**: start at `P1v3` for any production workload that needs autoscale, slots, or VNet integration.

### 4. Not enabling Always On

By default, App Service unloads idle apps to save memory. The first request after idle takes 5-15 seconds while the runtime warms up.

```bash
# Fix
az webapp config set -g orders-rg -n orders-api --always-on true
```

`Always On` is **not available** on Free/Shared tiers — another reason to start at Basic or higher in production.

### 5. Deploying directly to the production slot

```yaml
# BUG: traffic served during warm-up sees errors
- uses: azure/webapps-deploy@v3
  with:
    app-name: orders-api
    package: ./publish
```

**Fix**: always deploy to a staging slot, run smoke tests, then swap:

```yaml
- uses: azure/webapps-deploy@v3
  with: { app-name: orders-api, slot-name: staging, package: ./publish }

- run: |
    curl -fsS https://orders-api-staging.azurewebsites.net/health/ready
    az webapp deployment slot swap -g orders-rg -n orders-api --slot staging
```

### 6. Missing retries when calling Azure dependencies

```csharp
// BUG: a transient 503 from Service Bus crashes the request
await _serviceBus.SendMessageAsync(message);
```

**Fix**: the Azure SDKs already retry transient failures by default; configure timeouts explicitly via `ClientOptions.Retry`:

```csharp
clients.AddServiceBusClientWithNamespace(...).ConfigureOptions(o =>
{
    o.RetryOptions.MaxRetries = 5;
    o.RetryOptions.Mode = ServiceBusRetryMode.Exponential;
    o.RetryOptions.Delay = TimeSpan.FromMilliseconds(200);
});
```

### 7. Using the deprecated `WEBSITES_PORT` for Linux containers without setting the listener

```csharp
// BUG: ASP.NET Core listens on http://localhost:5000, App Service maps to port 80, app returns 502
```

**Fix**: tell Kestrel to listen on `$PORT` (Linux) or `%HTTP_PLATFORM_PORT%` (Windows):

```csharp
builder.WebHost.UseUrls($"http://0.0.0.0:{Environment.GetEnvironmentVariable("PORT") ?? "8080"}");
```

Or simpler: leave `WEBSITES_PORT=8080` as an app setting and let Kestrel pick its default.

## Best Practices

- **Use Managed Identity for every Azure dependency.** Connection strings with passwords are legacy.
- **Use Premium V3 (`P1v3`+) for production.** It is the only tier with reliable autoscale, VNet integration, and slot warm-up.
- **Always have at least 2 instances in production** — single-instance is a single point of failure during patching.
- **Deploy via slot swap with health check warm-up.** Configure `WEBSITE_SWAP_WARMUP_PING_PATH=/health/ready` so the swap waits for readiness.
- **Mark all environment-specific settings as `--slot-settings`** (DB connection strings, Application Insights connection, Key Vault URIs).
- **Set `ASPNETCORE_ENVIRONMENT` as a slot-sticky app setting** so staging doesn't accidentally run prod migrations.
- **Enable diagnostic settings to Log Analytics** — App Service logs, HTTP logs, and Application Insights together give end-to-end traces.
- **Lock down the SCM site** (`*.scm.azurewebsites.net`) with IP restrictions or Private Link in production.
- **Use Bicep or Terraform**, not portal clicks — every plan/app/slot configuration should be reproducible.
- **Set `WEBSITE_RUN_FROM_PACKAGE=1`** so deploys are atomic (mount a single ZIP rather than copying files).

## Related Concepts

- **Azure Functions** ([azure-functions.md](azure-functions.md)) — sibling PaaS, billed per execution, better for bursty workloads.
- **Container Apps / AKS** — when you need sidecars, KEDA scaling, or service mesh.
- **App Service Environment (ASE) v3** — dedicated, single-tenant App Service for compliance (PCI, FedRAMP).
- **Hybrid Connections** — tunnel TCP from App Service to on-prem servers without a VPN.
- **Key Vault references** ([azure-key-vault.md](azure-key-vault.md)) — `@Microsoft.KeyVault(...)` syntax in app settings.
- **App Configuration** ([configuration-and-secrets-management.md](configuration-and-secrets-management.md)) — centralized feature flags and tenant settings.
- **Application Insights** ([application-insights.md](application-insights.md)) — telemetry, distributed traces, KQL queries.
- **Deployment slots** — the App Service implementation of blue-green deployment ([../devops/blue-green-deployment.md](../devops/blue-green-deployment.md)).
- **Managed Identity** — the credential type `DefaultAzureCredential` resolves to on Azure-hosted apps.

## Real-World Usage

### Order API for an e-commerce platform

- **Plan**: `P1v3` Linux, 2 instances, autoscale 2-10 on CPU > 70%.
- **Slots**: `production` (live traffic) and `staging` (next release).
- **Identity**: System-assigned Managed Identity with `Storage Blob Data Contributor` on the invoice container and `Azure Service Bus Data Sender` on the `order-events` queue.
- **Secrets**: SQL connection string and Stripe webhook secret in Key Vault, referenced via `@Microsoft.KeyVault(...)` app settings.
- **Telemetry**: Application Insights with adaptive sampling at 30%; alerts on failure rate > 1% over 5 min.
- **Release**: GitHub Actions pushes to staging slot → runs smoke tests → swaps to production.

### Webhook receiver for payment gateways

- **Plan**: `S1` Standard (1 instance), no autoscale needed (low, predictable traffic).
- **Restriction**: IP allow-list for Stripe's published webhook IPs only.
- **Identity**: Posts received events to a Service Bus queue, then returns `202`. Heavy processing is in [azure-functions.md](azure-functions.md).

### Internal admin portal

- **Plan**: `B1` Basic (single instance, no slots — internal users tolerate 30 sec downtime during deploy).
- **Auth**: App Service Authentication (Easy Auth) with Microsoft Entra ID, no auth code in the app.

### Multi-tenant SaaS

- **Plan**: `I1v2` Isolated v2 in an App Service Environment for VNet isolation and dedicated compute.
- **Apps**: One App Service per tenant, all on the same plan, distinguished by custom domain (`acme.contoso-saas.com`).

## Code Example — Before and After

### Before: VM-hosted, hand-rolled, leaky

```csharp
// Program.cs running on an Azure VM behind a manually configured nginx
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<OrdersDb>(o =>
    o.UseSqlServer("Server=tcp:orders-sql.database.windows.net;User Id=sa;Password=PlainPass!;"));

// New HttpClient per request — leaks sockets
public class StripeClient
{
    public async Task<Charge> Charge(decimal amount)
    {
        var http = new HttpClient { BaseAddress = new Uri("https://api.stripe.com") };
        http.DefaultRequestHeaders.Add("Authorization", "Bearer sk_live_HARDCODED");
        var resp = await http.PostAsJsonAsync("/v1/charges", new { amount });
        return await resp.Content.ReadFromJsonAsync<Charge>();
    }
}

var app = builder.Build();
app.UseHttpsRedirection();
app.MapGet("/", () => "Orders API");
app.Run("http://0.0.0.0:5000");
```

Problems:
- Passwords and Stripe key in source.
- No telemetry, no health checks.
- `new HttpClient()` per request.
- Manual TLS, manual patching of the VM.
- No staged release path.

### After: App Service, Managed Identity, slot deploy

```csharp
using Azure.Identity;
using Microsoft.Extensions.Azure;

var builder = WebApplication.CreateBuilder(args);
var credential = new DefaultAzureCredential();

// Key Vault is the source of truth for secrets
builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!), credential);

// Azure AD auth to SQL — no password
builder.Services.AddDbContext<OrdersDb>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("OrdersDb")));

// Typed HttpClient with pooling + retry policies
builder.Services.AddHttpClient<IStripeClient, StripeClient>(c =>
{
    c.BaseAddress = new Uri("https://api.stripe.com");
    c.DefaultRequestHeaders.Add("Authorization",
        $"Bearer {builder.Configuration["Stripe:ApiKey"]}");
})
.AddStandardResilienceHandler();

// Azure SDK clients via Managed Identity
builder.Services.AddAzureClients(c =>
{
    c.UseCredential(credential);
    c.AddServiceBusClientWithNamespace(builder.Configuration["ServiceBus:Namespace"]!);
});

// Telemetry + health
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddHealthChecks()
    .AddDbContextCheck<OrdersDb>(tags: new[] { "ready" });

var app = builder.Build();
app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready",
    new HealthCheckOptions { Predicate = c => c.Tags.Contains("ready") });
app.MapControllers();
app.Run();
```

Bicep snippet for the plan + app + slot:

```bicep
resource plan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'orders-plan'
  location: location
  sku: { name: 'P1v3', tier: 'PremiumV3' }
  kind: 'linux'
  properties: { reserved: true }
}

resource app 'Microsoft.Web/sites@2023-01-01' = {
  name: 'orders-api'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: plan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: true
      healthCheckPath: '/health/ready'
    }
  }
}

resource staging 'Microsoft.Web/sites/slots@2023-01-01' = {
  parent: app
  name: 'staging'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: { serverFarmId: plan.id }
}
```

## Interview Questions and Answers

### 1. When would you choose App Service over Azure Functions or Container Apps?

**Why this matters**: PaaS selection is a decision every Azure architect makes weekly. Picking the wrong service wastes money or causes outages.

**Answer**: App Service for **long-running HTTP workloads with predictable traffic** — internal APIs, customer-facing storefronts, admin portals. Functions Consumption for **bursty event-driven workloads** (webhooks that fire 0 to 1000 RPS unpredictably). Container Apps when you need **sidecars, KEDA scaling on non-HTTP triggers, or service mesh** but don't want a full AKS cluster.

**Trade-off**: App Service is billed per plan instance even when idle. If your API handles 100 requests per day, you pay full price for an idle `P1v3`; Functions Consumption would cost cents. But Functions has a cold start and a 10-minute (Consumption) request limit.

**Real project**: An order API serving 50-500 RPS continuously runs on `P1v3` App Service. The webhook receiver that processes 0-200 RPS spiky Stripe events runs on Functions Premium. Both write to the same Service Bus queue, which is drained by a single worker.

### 2. Walk me through a zero-downtime deployment using deployment slots.

**Answer**: 

1. Deploy the new build to the `staging` slot (different URL: `orders-api-staging.azurewebsites.net`).
2. The staging slot is configured with the **same** SQL, Key Vault, and Service Bus as production (via `--slot-settings` so they don't move).
3. App Service warms up the slot by pinging the path configured in `WEBSITE_SWAP_WARMUP_PING_PATH`, which I set to `/health/ready`.
4. Once `/health/ready` returns 200, run smoke tests against `staging`.
5. Trigger `az webapp deployment slot swap` — Azure swaps the **hostnames** atomically, so production traffic instantly hits the new code. Old code now serves any traffic on the staging hostname.
6. If something breaks in production, run the swap again (reverse swap) — rollback is one CLI call.

**Trade-off**: Slot swaps share the plan's compute, so the staging slot consumes CPU/RAM from the same `P1v3`. If your app is memory-heavy, scale up before swapping.

**Real project**: Our checkout API does 50+ deployments per week. The slot-swap-with-warm-up pattern means we have never had a release-related outage in production.

### 3. A connection string was accidentally committed to `appsettings.json` and pushed to main. What do you do?

**Why this matters**: Tests incident-response thinking, not just clean-code knowledge.

**Answer**: 

1. **Rotate immediately** — the secret is now in git history forever, even after deletion. For SQL, reset the admin password; for Service Bus, rotate the SAS key.
2. **Remove from main** — open a PR that replaces the value with a Key Vault reference. Squash-merge so the bad commit isn't in main's first-parent history.
3. **Force-revoke any cached credentials** — restart all instances so they pick up the new secret.
4. **Audit access logs** — check Key Vault, SQL audit, and Storage logs for any access from outside trusted IPs since the commit.
5. **Fix the root cause** — add `git-secrets` / `gitleaks` pre-commit hook, configure GitHub secret scanning, and migrate the value to Managed Identity so there is no secret to leak.

**Trade-off**: Force-pushing to rewrite history erases the leak from new clones but doesn't help anyone who already cloned. Rotation is the only real fix.

**Real project**: A junior committed a Service Bus key to a public sample repo. We rotated within 5 minutes, but bots had already started using the key for spam. The lesson: rotation alone isn't enough — we now use Managed Identity for all Azure resources so there's nothing to leak.

### 4. Your App Service is throttling at 70% CPU but autoscale isn't adding instances. Why?

**Answer**: Common causes:

- **The plan SKU doesn't support autoscale** — `B1` Basic has no autoscale; you must be on `S1`+ or `P1v3`+.
- **The autoscale rule has a cool-down window** — after a scale-out, the rule waits (default 5-10 minutes) before scaling again. Spike traffic finishes before the next instance is added.
- **Max instance count is too low** — check the autoscale profile's `max-count`.
- **CPU metric is averaged across instances** — if one instance is at 95% and another at 30%, the average is 62% and the rule doesn't fire. Use **per-instance** metrics.
- **App is bottlenecked on something other than CPU** — outbound SNAT ports, SQL DTU, Service Bus throughput. Autoscale won't help; you need to scale the dependency.

**Trade-off**: Aggressive autoscale (`> 50% CPU`) means more instances and higher bill. Conservative (`> 80%`) means traffic suffers during the warm-up window.

**Real project**: We discovered our Black Friday throttling was actually **SNAT port exhaustion** calling Stripe, not CPU. The fix was adding a VNet integration + NAT gateway, not more App Service instances.

### 5. How do you store the SQL connection string for an App Service securely?

**Answer**: Three options in increasing order of security:

1. **App Service app setting** — better than `appsettings.json` (not in source), but the value sits in the App Service config plain-text and any contributor can read it.
2. **Key Vault reference** — `@Microsoft.KeyVault(VaultName=orders-kv;SecretName=OrdersDbConnString)`. App Service uses its Managed Identity to fetch the secret at startup; the value never appears in the portal.
3. **Managed Identity to SQL directly** — no connection string secret at all. The connection string becomes `Server=tcp:...;Database=Orders;Authentication=Active Directory Default;`, and the App Service Managed Identity is granted a SQL user with `db_datareader/db_datawriter`.

**Trade-off**: Option 3 has the best security posture (nothing to leak) but requires SQL to use Azure AD auth and your DBA to grant the Managed Identity. Some legacy SQL setups don't support Azure AD; option 2 is the fallback.

**Real project**: All new services in our org use option 3. Legacy services still using SQL auth use option 2 with monthly secret rotation enforced by a Functions timer trigger.

### 6. What's the difference between `WEBSITE_RUN_FROM_PACKAGE`, ZIP deploy, and Git deploy?

**Answer**:

- **Git deploy** — `git push azure main`. Build happens on the App Service VM. Slow, fragile, opaque. Avoid in CI/CD.
- **ZIP deploy** — `az webapp deploy --src-path app.zip`. The ZIP is extracted into `wwwroot`. Files are copied one at a time; partial failures leave a broken site.
- **Run from package (`WEBSITE_RUN_FROM_PACKAGE=1`)** — the ZIP is **mounted read-only** as the file system. Atomic — either the new package is live or nothing changed. Recommended default.

**Trade-off**: Run-from-package means you can't write to `wwwroot` at runtime (good — your app shouldn't anyway). Use Blob Storage for runtime data.

**Real project**: We migrated from ZIP deploy to `RUN_FROM_PACKAGE` after a deploy failure left half the app on v1 and half on v2 for 20 minutes. Mounted packages eliminate that failure mode entirely.

### 7. Production is down. The App Service is "Running" but every request returns 503. Where do you look?

**Answer**: My triage order:

1. **App Service Diagnose and Solve Problems** in the portal — Azure's AppLens diagnostics often spot the cause (SNAT exhaustion, instance crash loop, dependency timeout) in seconds.
2. **Application Insights Failures blade** — what's the exception? Distributed trace shows whether the failure is in the app, in SQL, in Key Vault, or in an outbound HTTP call.
3. **Kudu / `/api/logs/docker`** — check container start logs on Linux, stdout/stderr on Windows.
4. **Health check status** — if `/health/ready` is returning 503, App Service is correctly removing the instance from rotation. Find out why ready is failing.
5. **Recent changes** — diff App Service config and recent deployments. Most 503s I see are caused by a recent app setting change (typo in Key Vault reference, wrong env variable).

**Trade-off**: Restarting the app often "fixes" the symptom but loses the diagnostic state. Only restart after capturing logs.

**Real project**: A 503 outage turned out to be a Key Vault reference pointing to a deleted secret after a rotation script bug. App Service caches the resolved value, so a restart was the only way to re-resolve it. We added a Functions monitor that validates all `@Microsoft.KeyVault(...)` references every hour.

### 8. Walk me through cost optimization for an over-provisioned App Service environment.

**Answer**:

1. **Right-size the plan** — Azure Advisor recommendations show "this plan averages 12% CPU; consider downsizing." Use that as a starting hypothesis, validate with load tests.
2. **Consolidate apps** — multiple low-traffic apps can share a single plan. A `P1v3` can host 5-10 small apps comfortably.
3. **Use slots for environments instead of separate apps** — dev/test/staging slots share the production plan's compute.
4. **Switch dev/test plans to scheduled scale-down** — Logic Apps or `az` script to scale to `B1` evenings and weekends.
5. **Move bursty workloads to Functions Consumption** — webhooks, scheduled jobs, fan-out processors don't need an always-on plan.
6. **Reserve capacity** — 1-year or 3-year reservations on `P1v3` save 30-55% vs pay-as-you-go.

**Trade-off**: Consolidating apps means a noisy neighbor in one app can starve the others; only do this if you can monitor per-app metrics.

**Real project**: We dropped Azure spend by 40% in one quarter by moving 12 internal "tools" off their individual `S1` plans onto a single shared `P1v3` and migrating 4 webhook receivers to Functions Consumption.

## Summary Checklist

- [ ] I can explain App Service Plan vs Web App vs Deployment Slot.
- [ ] I can pick a plan SKU based on traffic, autoscale, VNet, and slot requirements.
- [ ] I can configure Managed Identity and Key Vault references so no secrets live in source.
- [ ] I can mark settings as slot-sticky and explain why some settings must not swap.
- [ ] I can deploy via GitHub Actions with OIDC federated credentials (no secrets).
- [ ] I can perform a zero-downtime release using slot swap with warm-up.
- [ ] I can write autoscale rules and recognize when CPU autoscale isn't the bottleneck.
- [ ] I can wire health checks (`/health/live`, `/health/ready`) and explain how App Service uses them.
- [ ] I can triage a 503 outage using Diagnose-and-Solve, App Insights, and Kudu logs.
- [ ] I can right-size, consolidate, and reserve plans to cut Azure cost without risking reliability.
