# Blue-Green Deployment

## What It Is

**Blue-green deployment** is a release strategy that runs two production-grade environments side by side. **Blue** is the version currently serving live traffic; **Green** is the candidate version, fully deployed and warmed but receiving no production load. The release is a near-instant **traffic switch** from Blue to Green. A rollback is the reverse switch.

On Azure App Service this is implemented as a **slot swap** between a `staging` slot (Green) and the `production` slot (Blue). On Kubernetes it is implemented by changing the `selector` of a `Service` (or the backend pool of an `Ingress`) from `app=orders,version=blue` to `app=orders,version=green`.

The key property: at the moment of switch, the new version is already running, healthy, and ready. The switch itself does not deploy, build, or restart anything customer-visible.

## Why It Exists

Classic in-place upgrades had two problems:

- **Downtime:** Stopping the service, deploying new bits, and restarting took 30–90 seconds where users saw 503s.
- **Slow rollback:** Reverting required re-running the build pipeline, re-deploying the previous artifact, and waiting for restart — often 15+ minutes during an incident.

The blue-green pattern, popularized by [Martin Fowler's 2010 article](https://martinfowler.com/bliki/BlueGreenDeployment.html) and embedded in cloud platforms a few years later, solves both:

- The new version is **fully warm** before any user touches it — JIT compilation done, connection pools filled, caches primed.
- The switch is a **routing change**, not a code change — milliseconds, not minutes.
- Rollback is **another routing change** — equally fast.

It also enabled mature engineering practices: smoke testing the candidate version on a real production-like environment using production secrets and production datastores before it sees traffic.

## When To Use It

**Use blue-green for:**

- Customer-facing APIs and web apps where a 30-second outage is unacceptable (checkout, payments, login).
- Services with slow warmup — JIT-heavy .NET apps, anything with large in-memory caches, Entity Framework model compilation.
- Releases that require human smoke verification on a production-grade environment before going live.
- Regulated workloads where audit requires a documented "production candidate verified" step before traffic switch.

**Do not use it for:**

- **Background workers and queue processors** where consumers can simply be drained and restarted — there is no "traffic" to switch.
- **Schema-incompatible database changes.** Blue-green assumes both versions can talk to the same database. If v2 drops a column v1 needs, the swap is a forward-only commitment, not a safe release.
- **Tiny, low-risk patches.** A typo fix in a static HTML banner does not justify provisioning a second slot. Use rolling deploy instead.
- **Cost-constrained environments** that cannot afford double infrastructure during the cutover window (though slot-based blue-green on App Service shares the App Service Plan, mitigating this).
- **Cases requiring granular percentage traffic shifting.** Blue-green is all-or-nothing. For 5% → 25% → 100% gradual rollout, use **canary** instead.

## Why It Is Important

- **Near-zero downtime.** The swap is a routing change measured in seconds.
- **Pre-warmed releases.** App Service slot swap runs warmup probes against the Green slot before flipping; only a healthy Green becomes Production.
- **Instant rollback.** Reverting is one `az webapp deployment slot swap` away — the previous version is still running in the old slot.
- **Production smoke testing.** Run real production traffic-like requests against Green with `Host: orders-staging.azurewebsites.net` before swap — same database, same Key Vault, same Service Bus, different URL.
- **Decoupled deploy and release.** Deployment (code in Green) and release (traffic to Green) are separate events. Deploy at 11pm; release after the on-call team is staffed at 9am.
- **Compliance & audit.** Reviewers approve the swap, not the deploy — the change-control conversation happens against a tested artifact.

## How It's Used in C# / .NET

### Azure App Service deployment slots

Slots are standalone App Service instances that share the App Service Plan (so no extra compute cost) but have their own hostname, configuration, and deployment.

```bash
# 1. Create a staging slot once (idempotent — Bicep usually owns this)
az webapp deployment slot create \
    --resource-group rg-orders-prod \
    --name orders-api \
    --slot staging

# 2. Deploy the new image to the staging slot
az webapp config container set \
    --resource-group rg-orders-prod \
    --name orders-api \
    --slot staging \
    --container-image-name ghcr.io/contoso/orders-api:1.42.0

# 3. Wait for the slot to be warm and healthy
az webapp show \
    --resource-group rg-orders-prod \
    --name orders-api \
    --slot staging \
    --query state

# 4. (Optional) Smoke test against https://orders-api-staging.azurewebsites.net

# 5. Swap — App Service runs auto-warmup against the new Production candidate
#    and only swaps if it returns 200 within the timeout.
az webapp deployment slot swap \
    --resource-group rg-orders-prod \
    --name orders-api \
    --slot staging \
    --target-slot production
```

### Auto-warmup configuration

App Service hits one or more URLs on the candidate slot before swapping. Configure them via app settings:

```bash
az webapp config appsettings set \
    --resource-group rg-orders-prod \
    --name orders-api \
    --slot staging \
    --settings \
        WEBSITE_SWAP_WARMUP_PING_PATH=/healthz/ready \
        WEBSITE_SWAP_WARMUP_PING_STATUSES=200
```

The `/healthz/ready` endpoint must verify SQL connectivity, Service Bus, Key Vault, and Application Insights — anything that, if broken, would 500 the first real request.

### Slot setting stickiness

Mark settings **"deployment slot setting"** so they **stay with the slot** during a swap rather than moving with the code. Typical sticky settings:

| Setting | Sticky? | Reason |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | Yes | Staging stays Staging after swap |
| `ApplicationInsights__ConnectionString` | Yes | Each slot reports under its own role name |
| `KeyVault__Uri` | Yes | Different vaults per environment |
| `WEBSITE_SWAP_WARMUP_PING_PATH` | Yes | Slot-specific behavior |
| `Image_Tag` / `DOCKER_CUSTOM_IMAGE_NAME` | **No** | This is exactly what we want to move |
| `Features:UseNewCheckout` | Depends | If different per env, sticky; if same, not sticky |

### IHostEnvironment-aware code

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("OrdersDb")!, name: "sql")
    .AddAzureServiceBusQueue(
        builder.Configuration["ServiceBus:ConnectionString"]!,
        "orders", name: "servicebus")
    .AddCheck<KeyVaultBootHealthCheck>("keyvault");

var app = builder.Build();

// Liveness — process is alive
app.MapHealthChecks("/healthz/live", new HealthCheckOptions
{
    Predicate = _ => false
});

// Readiness — dependencies are ready; used by swap warmup and load balancer
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready") || check.Name is "sql" or "servicebus" or "keyvault"
});

app.Run();
```

### Kubernetes blue-green via Service selector

Both versions are deployed as separate `Deployment`s with different labels. A single `Service` routes by `selector`:

```yaml
# orders-blue Deployment — currently live (version: blue)
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-blue }
spec:
  replicas: 6
  selector: { matchLabels: { app: orders, version: blue } }
  template:
    metadata: { labels: { app: orders, version: blue } }
    spec:
      containers:
        - name: api
          image: ghcr.io/contoso/orders-api:1.41.0
          readinessProbe: { httpGet: { path: /healthz/ready, port: 8080 } }
---
# orders-green Deployment — new candidate (version: green)
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-green }
spec:
  replicas: 6
  selector: { matchLabels: { app: orders, version: green } }
  template:
    metadata: { labels: { app: orders, version: green } }
    spec:
      containers:
        - name: api
          image: ghcr.io/contoso/orders-api:1.42.0
          readinessProbe: { httpGet: { path: /healthz/ready, port: 8080 } }
---
# Service — single point of routing; flip selector to switch traffic
apiVersion: v1
kind: Service
metadata: { name: orders }
spec:
  selector: { app: orders, version: blue }   # change to: version: green
  ports: [{ port: 80, targetPort: 8080 }]
```

Switch:

```bash
kubectl patch service orders -p '{"spec":{"selector":{"app":"orders","version":"green"}}}'
```

Rollback:

```bash
kubectl patch service orders -p '{"spec":{"selector":{"app":"orders","version":"blue"}}}'
```

### Database considerations — additive-only schema changes

Both Blue and Green run against the **same database** during the swap window. Migrations must be **expand–contract** (a.k.a. forward-and-backward compatible):

| Phase | Migration v3 (additive) | App v1 (Blue) | App v2 (Green) |
|---|---|---|---|
| Step 1 — Migrate | Add column `OrderItems.DiscountCents NULL` | Ignores column | Reads + writes new column |
| Step 2 — Deploy v2 | — | Still running | Deployed to Green slot |
| Step 3 — Swap | — | Drained | Live |
| Step 4 — Verify | — | Still warm in old slot for rollback | Live |
| Step 5 — Cleanup (next release) | Backfill old rows; make column NOT NULL | Retired | Live |

Never drop a column or rename a table in the same release that introduces blue-green. The previous version must remain runnable for at least one full release cycle.

## Advantages

- **Near-zero downtime** — the switch is a routing change in seconds.
- **Fast rollback** — reverse the swap; previous version is still running and warm.
- **Production smoke testing** — verify Green against real datastores before traffic.
- **Decouples deploy from release** — deploy off-hours, release during business hours when on-call is staffed.
- **Mature audit trail** — "swap approved by X at T+0" is a clean change-control record.
- **Pre-warmed JIT and caches** — the user-visible cold-start cost is eliminated.

## Disadvantages

- **Double infrastructure cost during cutover.** On VMs and dedicated K8s nodes this is real. On App Service slots it shares the plan so the cost is minimal.
- **Database schema gymnastics.** Expand-contract migrations require discipline and two release cycles for some changes.
- **All-or-nothing switch.** No way to expose 5% of traffic to Green — for that, use canary.
- **Stateful connections complicate the swap.** WebSockets, SignalR hubs, long-polling — the swap forces clients to reconnect.
- **Singleton jobs misbehave.** A cron-trigger Function or a singleton background service that runs in both slots simultaneously can double-process. Use slot-aware leases.
- **Slot warmup can mislead.** A 200 from `/healthz/ready` does not guarantee real traffic patterns succeed. Synthetic smoke tests should follow the swap.

## Common Mistakes

### 1. Swapping with a destructive migration in the same release

```sql
-- ❌ Drops column the Blue app still reads. Rollback impossible.
ALTER TABLE Orders DROP COLUMN LegacyShippingCode;
```

**Fix:** Split into two releases. Release N adds `NewShippingCode` and dual-writes. Release N+1 (after Blue is retired and traffic is stable on the new code) drops `LegacyShippingCode`.

### 2. Missing warmup configuration — first user is the canary

```bash
# ❌ Swap happens before the new slot has compiled controllers or filled connection pools.
az webapp deployment slot swap --slot staging --target-slot production
```

First user request after swap takes 12 seconds, times out at the load balancer, and surfaces as a P1 alert.

**Fix:**

```bash
az webapp config appsettings set --slot staging --settings \
    WEBSITE_SWAP_WARMUP_PING_PATH=/healthz/ready \
    WEBSITE_SWAP_WARMUP_PING_STATUSES=200
```

### 3. Treating slot swap as a deploy step in the same workflow as the build

```yaml
# ❌ Build, deploy, and swap in one job — no human gate, no smoke window
- run: docker build -t orders:${{ github.sha }} .
- run: az webapp config container set --slot staging --container-image-name ...
- run: az webapp deployment slot swap --slot staging --target-slot production
```

**Fix:** Two workflows. `deploy-staging.yml` deploys to the staging slot on every merge to `main`. `release-prod.yml` performs the swap, requires `environment: production` approval, and runs only when a release tag is pushed.

### 4. Forgetting slot setting stickiness

```bash
# ❌ ASPNETCORE_ENVIRONMENT not marked sticky — after swap, Production reports as "Staging"
az webapp config appsettings set --slot staging --settings ASPNETCORE_ENVIRONMENT=Staging
```

App Insights shows the wrong role; alerts trigger on the wrong environment; on-call investigates a phantom.

**Fix:**

```bash
az webapp config appsettings set --slot staging \
    --slot-settings ASPNETCORE_ENVIRONMENT=Staging
```

### 5. Singleton background services running in both slots

```csharp
// ❌ Hosted service runs in both Blue and Green during swap — double-charges customers
public sealed class NightlyBillingService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            if (DateTime.UtcNow.Hour == 2)
                await _billing.RunNightlyAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(15), stoppingToken);
        }
    }
}
```

**Fix:** Use a distributed lease (Azure Blob lease, SQL `sp_getapplock`, or Azure App Configuration sentinel) so only one slot wins the right to run the job. Or move scheduled work into an Azure Function with a single timer trigger, separate from the web app.

### 6. No smoke test against the staging slot URL

The pipeline deploys to staging and immediately swaps. The first verification is real customer traffic.

**Fix:** Run `dotnet test --filter Category=Smoke` against the staging slot URL before the swap step. The smoke suite should hit the top 10 endpoints with realistic payloads using test accounts.

### 7. Letting slot warmup pass on a misconfigured app

`/healthz` returns 200 because it only checks the process is alive. Swap succeeds; Production fails.

**Fix:** Split into `/healthz/live` and `/healthz/ready`. Point `WEBSITE_SWAP_WARMUP_PING_PATH` at `/healthz/ready`, which actually queries SQL, Service Bus, and Key Vault. See [devops/health-checks.md](health-checks.md).

## Best Practices

- **Always pair slot swap with auto-warmup** pointing at a real readiness probe.
- **Mark environment-defining settings as sticky** in App Service slots.
- **Make every migration backward-compatible for one release cycle.** Expand–contract is the rule.
- **Separate deploy from release.** Deploy to staging on merge; swap on tag with approval.
- **Smoke test the staging slot URL** before swap, using synthetic transactions over the same datastores Production uses.
- **Capture the previous image tag** before swap so rollback is trivial: `az webapp deployment slot swap --slot staging --target-slot production --action reset` reverts.
- **Lease singleton work** — only one slot may run a given background job at a time.
- **Monitor both slots during the swap window** in Application Insights, split by `cloud_RoleInstance`.
- **For Kubernetes blue-green**, use an Argo Rollouts-style controller rather than raw `kubectl patch` for safety and traffic shifting.
- **Document the rollback runbook**: exact command, expected duration, alert silencing, who to notify.

## Related Concepts

- [devops/rollback-strategy.md](rollback-strategy.md) — slot-swap reversal as the fastest rollback
- [devops/ci-cd-pipelines.md](ci-cd-pipelines.md) — deploy-vs-release separation
- [devops/health-checks.md](health-checks.md) — readiness probes used by warmup
- [devops/environment-configuration.md](environment-configuration.md) — sticky settings, per-slot config
- [devops/kubernetes-basics.md](kubernetes-basics.md) — Services, Deployments, selectors
- [azure/app-service.md](../azure/app-service.md) — slot mechanics in depth
- [data-access/migrations.md](../data-access/migrations.md) — expand–contract patterns

## Real-World Usage

### Azure App Service slot swap for a payments API

A payments processor runs `payments-api` on Premium V3 App Service with a `staging` slot. Releases follow this flow: GitHub Actions deploys the tagged image to staging on every Friday morning. The QA team runs a smoke test suite against `payments-api-staging.azurewebsites.net` against the Production database (read-only test accounts). On Monday at 10am, the release manager triggers the `swap-prod.yml` workflow; environment approval requires one signer from SRE and one from Compliance. The swap completes in 6 seconds; auto-warmup proved Green could handle SQL + Key Vault + Service Bus before any customer touched it. The old Production image stays warm in the staging slot for 48 hours for emergency rollback.

### Kubernetes blue-green for an order ingestion API

A retail platform runs orders ingestion on AKS. Each release deploys a `green` `Deployment` with `version: <new-sha>`. Argo Rollouts manages traffic by patching the `Service` selector. The release runs a 5-minute analysis using Prometheus metrics — request rate, error rate, P99 latency — comparing Green against Blue before promoting. On anomaly, the rollout auto-aborts and stays on Blue. The previous Blue stays alive for 24 hours.

### Database expand-contract during a 6-month column rename

A SaaS team needed to rename `CustomerEmail` to `BillingEmail`. Release 1 added `BillingEmail`, dual-wrote both columns, read from `CustomerEmail`. Release 2 read from `BillingEmail` with fallback. Release 3 (one month later, after audit confirmed all rows migrated) removed `CustomerEmail` from the model. Release 4 (one more cycle later) dropped the column. Every release was blue-green-safe.

### Cost trade-off vs canary

For the same workload, the team compared:

- **Blue-green (slot swap):** 1 extra App Service slot (free, shares plan), all-or-nothing switch, 6-second cutover, free instant rollback.
- **Canary on AKS with Argo Rollouts:** Per-percentage traffic shift, ~20-minute progressive rollout, automated metric analysis, more complex setup.

For mature SRE teams with strong observability, canary is preferred for risky changes. For most CRUD APIs with well-tested releases, slot-swap blue-green is the simpler, cheaper choice.

## Code Example — Before and After

### Before — in-place restart deploy, no warmup, schema-destructive migration

```yaml
# .github/workflows/deploy.yml — single-step, downtime-inducing
name: deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: 8.0.x }

      - name: Build & test
        run: |
          dotnet build -c Release
          dotnet test -c Release

      - name: Apply EF migrations (destructive!)
        run: dotnet ef database update --connection "$PROD_CONN"  # ❌ drops columns

      - name: Stop, deploy, start
        run: |
          az webapp stop --name orders-api --resource-group rg-prod
          az webapp deploy --name orders-api --src-path ./publish.zip
          az webapp start --name orders-api --resource-group rg-prod
```

Problems: 60-second downtime, destructive schema change with no rollback path, no smoke test, no approval gate, first user is the canary.

### After — slot-based blue-green with warmup, additive migration, gated swap

```yaml
# .github/workflows/deploy-staging.yml — runs on every merge to main
name: deploy-staging
on:
  push:
    branches: [main]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write, packages: write }
    steps:
      - uses: actions/checkout@v4

      - name: Build & test
        run: |
          dotnet build -c Release
          dotnet test -c Release --filter Category!=Smoke

      - name: Build & push image
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

      - uses: azure/login@v2
        with: { creds: ${{ secrets.AZURE_CREDENTIALS }} }

      - name: Apply additive-only migration
        run: dotnet ef database update --connection "${{ secrets.SQL_PROD_RO_TO_ADMIN }}"
        # Migration policy enforced by ./tools/check-additive-only.ps1 in CI

      - name: Deploy image to staging slot
        run: |
          az webapp config container set \
            --resource-group rg-orders-prod --name orders-api --slot staging \
            --container-image-name ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Smoke test the staging slot
        run: dotnet test ./tests/Smoke -c Release \
             -- TestRunParameters.Parameter\(name=\"BaseUrl\",value=\"https://orders-api-staging.azurewebsites.net\"\)
---
# .github/workflows/release-prod.yml — runs on release tag v* with approval
name: release-prod
on:
  push:
    tags: ['v*']
jobs:
  swap:
    runs-on: ubuntu-latest
    environment: production   # requires reviewer approval + branch protection
    steps:
      - uses: azure/login@v2
        with: { creds: ${{ secrets.AZURE_CREDENTIALS_PROD }} }

      - name: Capture pre-swap image for rollback
        run: |
          OLD_IMAGE=$(az webapp config container show \
            --resource-group rg-orders-prod --name orders-api \
            --query "[?name=='DOCKER_CUSTOM_IMAGE_NAME'].value | [0]" -o tsv)
          echo "OLD_IMAGE=$OLD_IMAGE" >> $GITHUB_ENV

      - name: Slot swap (auto-warmup runs /healthz/ready)
        run: |
          az webapp deployment slot swap \
            --resource-group rg-orders-prod --name orders-api \
            --slot staging --target-slot production

      - name: Post-swap smoke
        run: dotnet test ./tests/Smoke -- TestRunParameters.Parameter\(name=\"BaseUrl\",value=\"https://orders-api.azurewebsites.net\"\)

      - name: Auto-rollback on smoke failure
        if: failure()
        run: |
          az webapp deployment slot swap \
            --resource-group rg-orders-prod --name orders-api \
            --slot staging --target-slot production --action reset
```

```csharp
// Health checks consumed by App Service slot warmup
app.MapHealthChecks("/healthz/live", new HealthCheckOptions { Predicate = _ => false });
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = async (ctx, report) =>
    {
        ctx.Response.ContentType = "application/json";
        await ctx.Response.WriteAsync(JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new { name = e.Key, status = e.Value.Status.ToString() })
        }));
    }
});
```

Result: zero customer-visible downtime, smoke-tested release candidate, one-command instant rollback, additive schema that supports both versions concurrently.

## Interview Questions and Answers

### 1. Walk me through a production blue-green release on Azure App Service.

**Why this matters:** Tests concrete operational knowledge, not just theory.

**Answer:** Pipeline deploys the new image to the `staging` slot. App Service starts the container, pulls Key Vault references, runs migrations (additive-only). A smoke test job hits `https://app-staging.azurewebsites.net` against the Production datastores. On success, a release-tag workflow with environment approval triggers `az webapp deployment slot swap`. App Service runs auto-warmup against `WEBSITE_SWAP_WARMUP_PING_PATH=/healthz/ready` on the staging slot; only if it returns 200 within timeout does it flip the routing. The whole user-visible cutover is 5–10 seconds. The old image now sits in `staging`, ready for `--action reset` rollback.

**Trade-off:** Requires discipline around sticky settings and additive migrations. Pays off with near-zero downtime and instant rollback.

**Real project:** On a payments service we ran this pattern weekly for 18 months with zero customer-visible deploy incidents. The two times we rolled back, the swap-reset took 8 seconds.

### 2. Your release added a new required SQL column. Why is that a blue-green problem and how do you fix it?

**Why this matters:** Tests schema-evolution discipline.

**Answer:** During the swap window both Blue and Green talk to the same database. Blue's INSERT statements don't include the new required column, so they fail. Even worse, if the swap fails and you roll back to Blue, Blue can no longer write. Fix: make the column nullable in release N, deploy code that populates it in release N, backfill old rows over time, and only in release N+1 (after Blue is fully retired) tighten the column to `NOT NULL`. This expand-contract approach takes two releases but guarantees Blue stays runnable throughout.

**Trade-off:** Slower schema evolution. Worth it for safe deploys.

**Real project:** We renamed `User.PhoneNumber` to `User.PrimaryPhoneE164`. It took three releases over six weeks: add column, dual-write, switch reads, drop old column. No incidents.

### 3. What is the difference between blue-green and canary?

**Why this matters:** Tests understanding of progressive delivery.

**Answer:** Blue-green is **all-or-nothing**: 0% traffic on Green, then a switch to 100%. Canary is **gradual**: 5% → 25% → 50% → 100%, with automated metric checks between stages. Blue-green is simpler to operate and reason about; canary catches subtle issues that only appear under partial load (cache thrash, hot key contention). On App Service, blue-green is built-in via slot swap with cheap implementation; canary requires Front Door / Application Gateway weighted routing or AKS with Argo Rollouts. Choose blue-green for confidence in your test suite; canary for risky releases or large traffic.

**Trade-off:** Canary needs strong observability; blue-green needs strong tests.

**Real project:** We used blue-green for the API tier and canary for the front-end CDN release. The front-end's per-region cache behavior made canary essential; the API's deterministic tests made blue-green sufficient.

### 4. After swap, error rate doubles. What do you do?

**Why this matters:** Tests incident response under time pressure.

**Answer:** Roll back first, diagnose after. `az webapp deployment slot swap --slot staging --target-slot production --action reset` reverses the swap. Error rate should return to baseline within a minute as App Service flips routing back to the previous slot, which is still warm. Then investigate: check Application Insights filtered to the post-swap window, look at top 10 failing endpoints, correlate with deployed code changes since the previous release. Common culprits: missing config in the new slot, Key Vault reference failing because the slot's managed identity wasn't granted access, an additive migration that wasn't actually additive.

**Trade-off:** Reverting before diagnosing can hide a real bug — but customer impact comes first.

**Real project:** A swap doubled 500 rate because the new code introduced an `IOptions<>` with `[Required]` that was missing in the slot's app settings. The pod started but failed every request that needed it. Rollback took 7 seconds; the fix (adding the config) took 90 seconds; second swap was clean.

### 5. Why mark slot settings as sticky?

**Why this matters:** Tests understanding of slot-swap mechanics.

**Answer:** During swap, most app settings move with the code from staging to production. Sticky settings ("deployment slot settings") stay with the slot. You want `ASPNETCORE_ENVIRONMENT` sticky so the Production slot is always `Production` regardless of what's deployed. You want the image tag NOT sticky so the new code actually goes live. You want region-specific Key Vault URIs sticky so each slot points at its own vault. Getting this backwards means the swap produces a slot labeled as the wrong environment, sends telemetry under the wrong role, or fails health checks because it can't reach the right secrets.

**Trade-off:** None — this is a one-time configuration that prevents an entire class of bug.

**Real project:** An engineer marked `ApplicationInsights__ConnectionString` non-sticky. After swap, Production telemetry routed to the Staging App Insights instance. We discovered it only when an alert based on Prod traffic never fired. Marking it sticky fixed it permanently.

### 6. How does blue-green work on Kubernetes?

**Why this matters:** Tests cross-platform fluency.

**Answer:** Two `Deployment`s with distinct version labels (`version: blue`, `version: green`) run side by side. A `Service` (or `Ingress` backend) has a `selector` that picks one. Switching is `kubectl patch service` to change the selector — the Service immediately routes to the new pods. Both Deployments must have working readiness probes; pods become endpoints only when ready. For real production, use Argo Rollouts or Flagger which add pre-promotion analysis, automatic rollback on metric anomaly, and gradual traffic shifting if you want canary semantics. Without those tools, raw `kubectl patch` is operationally fine for small teams but lacks safety rails.

**Trade-off:** Double pod count during swap doubles compute cost for a brief window. Argo Rollouts adds operational surface area.

**Real project:** A 30-replica orders Deployment used Argo Rollouts for blue-green. Promotion was gated on a Prometheus query for 5xx rate over the previous 5 minutes. Two failed promotions auto-rolled-back without paging anyone.

### 7. How do background jobs and singletons behave during a slot swap?

**Why this matters:** Tests awareness of non-HTTP workload risks.

**Answer:** Both slots run the full app code, including `BackgroundService` hosted services. During the swap window (a few seconds) both versions execute scheduled work. For idempotent jobs (retry-safe queue processors) this is harmless. For non-idempotent jobs ("send nightly invoice email") this causes duplicate sends. Mitigations: (1) move scheduled work to a separate Azure Function with one timer trigger so the web app has no singletons; (2) use a distributed lease — Azure Blob lease or SQL `sp_getapplock` — so only one slot wins the right to run the job; (3) add an idempotency key to the work so duplicate execution is a no-op.

**Trade-off:** Distributed leases add complexity but are the only correct answer for stateful, non-idempotent work.

**Real project:** A nightly billing job sent duplicate invoices to ~200 customers during a swap. We extracted the job into a Timer-triggered Function and added an idempotency key based on `(customerId, billingPeriod)`. No duplicates since.

### 8. Your boss wants to skip the staging slot to save cost. What do you tell them?

**Why this matters:** Tests ability to justify engineering practices to non-technical stakeholders.

**Answer:** On Azure App Service, slots share the App Service Plan, so there is no extra compute cost for the slot itself — only a small storage cost for the additional content. The slot is the only way to get zero-downtime deploys, pre-warmup verification, and instant rollback on the platform. Removing it means accepting deploy downtime, no smoke window, and rollback measured in minutes (re-deploy the previous artifact) instead of seconds (reverse swap). For a service with SLA commitments, the cost of one downtime incident usually outweighs years of slot overhead. If the concern is genuinely about deployment complexity, simplify the swap workflow but keep the slot.

**Trade-off:** On non-App-Service platforms (raw VMs) the second environment really does cost extra. The conversation is then about SLA value vs infrastructure spend.

**Real project:** A finance team challenged us on slot cost. We showed Azure billing: $0/month for the staging slot itself, and the operational metric — zero deploy-related incidents in 12 months — versus the previous year's three deploy-caused outages averaging 25 minutes each. Conversation ended.

## Summary Checklist

- [ ] I can explain blue-green as a routing change, not a deploy.
- [ ] I can configure an Azure App Service deployment slot with sticky settings and auto-warmup.
- [ ] I can run a slot swap from a GitHub Actions workflow with environment approval.
- [ ] I can roll back a slot swap in seconds with `--action reset`.
- [ ] I write additive-only EF Core migrations that keep Blue runnable.
- [ ] I implement `/healthz/live` and `/healthz/ready` so warmup is meaningful.
- [ ] I can implement Kubernetes blue-green via Service selector patching or Argo Rollouts.
- [ ] I prevent duplicate background-job execution during the swap window.
- [ ] I can articulate when blue-green is the right choice vs canary.
- [ ] I can defend the cost of a staging slot against SLA-driven business value.
