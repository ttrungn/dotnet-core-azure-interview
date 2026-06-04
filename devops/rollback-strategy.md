# Rollback Strategy

## What It Is

A **rollback strategy** is the documented, rehearsed, automated mechanism by which a team returns production to the last known-good state when a release causes incidents. It answers four operational questions before the incident, not during:

1. **What signal triggers rollback?** (5xx rate, P99 latency, queue depth, business KPI)
2. **What is the exact command or button?** (`az webapp deployment slot swap --action reset`, flip feature flag, `kubectl rollout undo`)
3. **Who is authorized to execute it?** (on-call engineer with break-glass approval, or fully automated)
4. **What is the maximum tolerable recovery time?** (seconds for flag flip, minutes for slot swap, hours for redeploy)

A strategy is not "we'll figure it out." It is a written runbook with specific commands, expected outcomes, and who to notify.

## Why It Exists

Every production release carries non-zero risk: an edge case the tests missed, a config that worked in Staging but not Production, a database query plan that flips at production scale. The historical pain that drove rollback discipline:

- **Long MTTR (Mean Time To Recovery).** Teams without a strategy spent hours during incidents debugging *while customers were impacted*. The right move — revert first, debug after — was not in muscle memory.
- **Hostile rollbacks.** A team would attempt to revert and discover the previous code couldn't run against the new database schema. The "rollback" caused a worse outage.
- **Heroic fix-forward.** Pressure to ship a hotfix while production was burning often introduced a second incident on top of the first.
- **Lost artifacts.** "Just deploy the previous version" doesn't work if the CI runner is gone, the Docker tag was overwritten, or no one knows which commit was running.

The modern principle: **make rollback safe, fast, and boring**. Treat it as a routine operation, not a heroic event.

## When To Use It

**You always need a rollback plan**, but the *mechanism* depends on the change type:

| Change | Rollback Mechanism | Typical Recovery Time |
|---|---|---|
| Code-only bug behind a flag | Flip feature flag in App Configuration | 10–30 seconds |
| Bad image deployed to App Service slot | Slot swap reversal (`--action reset`) | 5–15 seconds |
| Bad image deployed to AKS | `kubectl rollout undo deployment/orders` | 30–90 seconds |
| Bad release across multiple services | Re-deploy pinned image tag per service | 5–15 minutes |
| Bad additive schema migration | Stop using new column (deploy previous code); leave column for later cleanup | 5–15 minutes |
| Bad destructive schema migration | **There is no clean rollback** — restore from backup, accept data loss | Hours |

**Do not rely solely on rollback for:**

- **Data corruption.** Once the new code wrote wrong data to the DB, rolling code back does not unwrite it. You need data repair scripts or PITR (point-in-time restore).
- **Outbound side effects already fired.** Emails sent, webhooks delivered, payments captured. Rollback doesn't recall them; you need compensating actions.
- **Releases that mixed multiple unrelated changes.** Rolling back to undo bug X also undoes good feature Y. Keep release scope narrow.

## Why It Is Important

- **MTTR is a primary SRE metric.** Faster rollback = fewer customer-impacting minutes = SLA preservation.
- **Reduces release fear.** Teams ship more often when they trust rollback. More frequent, smaller releases reduce the per-release risk further — a virtuous cycle.
- **Decouples decision from debugging.** During an incident, the question "is this caused by today's release?" becomes irrelevant — revert first, prove causation later.
- **Supports progressive delivery.** Canary, blue-green, and feature flags only work when rollback is automated and reliable.
- **Compliance.** SOC 2, PCI DSS, and HIPAA all require documented change-management procedures including rollback.
- **Postmortem quality.** "How did you recover?" is answered with a timestamp and command, not a war story.

## How It's Used in C# / .NET

### 1. Azure App Service slot swap reversal — the gold standard

If the previous release used a deployment slot, rollback is a reversed swap:

```bash
# The new (bad) version is in 'production' slot; previous (good) version is still warm in 'staging' slot
az webapp deployment slot swap \
    --resource-group rg-orders-prod \
    --name orders-api \
    --slot staging \
    --target-slot production \
    --action reset    # reverses the previous swap atomically
```

Recovery time: 5–15 seconds. The previous version is still JIT-warm, connection pools full. No image pull, no cold start.

### 2. Feature flags — instant, fine-grained, no redeploy

A feature flag in Azure App Configuration or LaunchDarkly lets you disable just the broken code path without touching the binary:

```csharp
public sealed class CheckoutService
{
    private readonly IFeatureManager _features;
    private readonly ILegacyCheckout _legacy;
    private readonly INewCheckout _newCheckout;
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(
        IFeatureManager features,
        ILegacyCheckout legacy,
        INewCheckout newCheckout,
        ILogger<CheckoutService> logger)
    {
        _features = features;
        _legacy = legacy;
        _newCheckout = newCheckout;
        _logger = logger;
    }

    public async Task<CheckoutResult> CompleteAsync(
        Guid cartId, CancellationToken ct)
    {
        if (await _features.IsEnabledAsync("NewCheckoutFlow"))
        {
            _logger.LogInformation("Using new checkout for cart {CartId}", cartId);
            return await _newCheckout.CompleteAsync(cartId, ct);
        }

        return await _legacy.CompleteAsync(cartId, ct);
    }
}
```

```csharp
builder.Services.AddAzureAppConfiguration();
builder.Services.AddFeatureManagement();

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(builder.Configuration["AppConfig:Endpoint"]!),
                    new DefaultAzureCredential())
        .UseFeatureFlags(featureOptions =>
            featureOptions.SetRefreshInterval(TimeSpan.FromSeconds(30)));
});
```

Flip `NewCheckoutFlow` from `true` to `false` in the Azure portal — every instance picks it up within 30 seconds. Recovery time: 30 seconds, zero deploys.

LaunchDarkly equivalent:

```csharp
var client = LdClient.Get();
var user = Context.Builder(currentUser.Id).Build();
if (client.BoolVariation("new-checkout-flow", user, defaultValue: false))
{
    return await _newCheckout.CompleteAsync(cartId, ct);
}
```

### 3. Expand–contract migrations — never break N-1

The core schema rule for safe rollback: **every migration must leave the previous version (N-1) of the application runnable**. Never drop or rename a column in the release that stops using it.

| Release | Action | App N reads/writes | App N+1 reads/writes |
|---|---|---|---|
| N (expand) | Add column `Email` NULL, dual-write | `LegacyEmail` | `LegacyEmail` + `Email` |
| N+1 (switch reads) | — | `LegacyEmail` (still runnable) | `Email` (with fallback to `LegacyEmail`) |
| N+2 (cleanup, only after N+1 is stable) | Backfill missing `Email` rows, set NOT NULL | (N is retired) | `Email` only |
| N+3 (contract) | Drop `LegacyEmail` | — | `Email` |

At every release, **rollback to the previous version is safe** because the schema supports both. See [data-access/migrations.md](../data-access/migrations.md).

### 4. Kubernetes rollout undo

```bash
# Deploy new version (managed by Deployment controller)
kubectl set image deployment/orders-api api=ghcr.io/contoso/orders-api:1.42.0

# Watch error rate spike in Application Insights — decide to roll back
kubectl rollout undo deployment/orders-api

# Confirm
kubectl rollout status deployment/orders-api
```

Kubernetes keeps the previous `ReplicaSet` (controlled by `spec.revisionHistoryLimit`, default 10). `rollout undo` scales the old one back up and the new one down. Recovery: 30–90 seconds depending on pod count and readiness probe timing.

### 5. Image tag pinning — never use `:latest` in production

```yaml
# ❌ Cannot roll back — :latest moves under you
image: ghcr.io/contoso/orders-api:latest

# ✅ Pinned to a specific SHA — reproducible, rollback-able
image: ghcr.io/contoso/orders-api:1.42.0
image: ghcr.io/contoso/orders-api@sha256:a1b2c3d4...
```

Rollback becomes `kubectl set image ... orders-api:1.41.0` — deterministic.

### 6. GitHub Actions revert workflow

A dedicated, fast-path workflow for emergency rollback:

```yaml
# .github/workflows/rollback-prod.yml
name: rollback-prod
on:
  workflow_dispatch:
    inputs:
      target_tag:
        description: 'Image tag to roll back to (e.g., v1.41.0)'
        required: true
        type: string
      reason:
        description: 'Incident ticket / Slack thread URL'
        required: true
        type: string
jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production-rollback  # less-strict gate than normal deploys
    steps:
      - uses: azure/login@v2
        with: { creds: ${{ secrets.AZURE_CREDENTIALS_PROD }} }

      - name: Verify image exists
        run: |
          az acr repository show --name acrprod \
            --image orders-api:${{ inputs.target_tag }}

      - name: Deploy previous image to production slot directly
        run: |
          az webapp config container set \
            --resource-group rg-orders-prod --name orders-api \
            --container-image-name acrprod.azurecr.io/orders-api:${{ inputs.target_tag }}

      - name: Notify Teams
        run: |
          curl -X POST "${{ secrets.TEAMS_INCIDENT_WEBHOOK }}" \
            -H "Content-Type: application/json" \
            -d '{"text":"🔴 Production rolled back to ${{ inputs.target_tag }} — reason: ${{ inputs.reason }}"}'
```

The `production-rollback` environment has a different (looser) approval policy than `production` so on-call can revert without waiting for two-person review.

### 7. Immutable artifacts in ACR / GHCR

Enable **repository policies** that prevent tag overwrite:

```bash
# Azure Container Registry — disallow tag mutation
az acr config retention update --registry acrprod \
    --status enabled --days 90 --type UntaggedManifests
```

For GHCR, treat published tags as immutable by convention and never re-push the same tag.

## Advantages

- **MTTR measured in seconds**, not hours.
- **Reduces release anxiety**, enabling smaller and more frequent releases.
- **Reversible operations** for slot swaps, feature flags, K8s rollouts.
- **Decouples deploy from release** — even after the bits are in Production, the flag controls customer exposure.
- **Auditable** — "who rolled back what, when, why" is captured in pipeline logs and flag change history.
- **Compatible with progressive delivery** — canary aborts and feature-flag percentages are rollback gradations.

## Disadvantages

- **Discipline cost.** Every release must respect N-1 compatibility — expand-contract migrations, sticky settings, backward-compatible API contracts.
- **Flag debt.** Old flags accumulate. Cleanup is unglamorous and gets deferred.
- **No silver bullet for data corruption.** Rolling code back doesn't roll data back.
- **Outbound side effects can't be rolled back.** Emails sent are sent.
- **Image retention costs storage.** Keep older tags around long enough to roll back to them.
- **Distributed system rollbacks are tricky.** Rolling back service A may require rolling back service B simultaneously if they ship together.

## Common Mistakes

### 1. Treating rollback as the last resort instead of the first response

During an incident, debugging in production *before* reverting wastes user-impacting minutes. The on-call playbook should read: **"If this incident correlates with a release in the last 60 minutes, revert first. Investigate after."**

**Fix:** Codify "revert first" in the runbook. Train on it during game days.

### 2. Using `:latest` image tags

```yaml
# ❌ aks-deployment.yaml
image: ghcr.io/contoso/orders-api:latest
```

When you `kubectl rollout undo`, the previous `ReplicaSet` still points to `:latest`, which now resolves to the broken image. Rollback fails.

**Fix:** Always pin by version tag or SHA digest:

```yaml
image: ghcr.io/contoso/orders-api:1.42.0
# or, even better, immutable:
image: ghcr.io/contoso/orders-api@sha256:a1b2c3d4e5f6...
```

### 3. Destructive migrations that trap you on the broken version

```sql
-- ❌ Release N drops a column that the rolled-back app (N-1) still needs
ALTER TABLE Orders DROP COLUMN LegacyCurrencyCode;
```

A rollback to N-1 now crashes on every `SELECT * FROM Orders`.

**Fix:** Expand-contract. Drop the column in a later release, *only after* N has been stable in Production for at least one full release cycle.

### 4. No feature flag on a risky change

A new payment provider integration ships with no flag. When it misbehaves, the only rollback is a full redeploy — and the team discovers the build is broken on `main` because someone merged a separate hotfix.

**Fix:** Wrap every risky behavioral change in a feature flag from day one. The flag costs almost nothing and is the fastest rollback lever you have.

### 5. Forgetting to verify the rollback image exists

```bash
# ❌ Rolls back to a tag that was garbage-collected
az webapp config container set --container-image-name acrprod.azurecr.io/orders-api:1.30.0
```

Deploy fails; recovery becomes panicked image rebuild.

**Fix:** ACR retention policy that keeps the last N tags (or 90 days) explicitly. Verify the tag exists before triggering the rollback workflow.

### 6. Rollback workflow blocked by the same approval gate as the deploy

The `production` environment requires two reviewers and a 30-minute wait. During a P0 incident, the on-call cannot get approvals fast enough.

**Fix:** Separate `production-rollback` environment with a different (faster) policy — e.g., single on-call reviewer, no wait timer. Audit every use after the fact.

### 7. Rolling back without notifying downstream services

Service A rolls back from v2 to v1. Service B was updated to use a new field that v1 doesn't return. Service B now 500s.

**Fix:** Coordinate via release manifests. Tools like Argo CD's ApplicationSet or a service-mesh-aware deployment pipeline can roll back coupled services together. Or — better — enforce API backward compatibility so this scenario can't arise.

### 8. Ignoring the data dimension

Rolling back the API does not roll back the rows the broken API wrote. Customer A was double-charged; reverting code doesn't refund them.

**Fix:** Build idempotency keys and compensating-action procedures into the service from day one. For payment flows, use SQL transactions and outbox patterns so a half-failure can be detected and repaired. See [architecture/outbox-pattern.md](../architecture/outbox-pattern.md).

## Best Practices

- **Revert first, debug after.** Make it the team's reflex.
- **One-command rollback for every service.** If the runbook says "contact the release engineer," the runbook is wrong.
- **Always wrap risky changes in a feature flag.** Flags are the fastest rollback lever.
- **Pin every image by version or digest.** Never `:latest` in any environment past Dev.
- **Expand-contract every schema change.** Two releases minimum for column drops or type changes.
- **Keep deployment slots warm** for at least 48 hours after swap so rollback stays fast.
- **Separate rollback approval policy** from forward-deploy approval — faster gate for emergencies.
- **Practice rollback during game days.** A monthly drill where the team rolls back a non-production deploy keeps the muscle memory alive.
- **Maintain a release manifest** capturing the image tag, SHA, commit, deployer, and timestamp for each environment.
- **Schedule flag cleanup.** Every quarter, retire flags that have been at 100% (or 0%) for over a month.
- **Audit rollback events.** Postmortem every rollback, including successful ones, to learn what triggered it.

## Related Concepts

- [devops/blue-green-deployment.md](blue-green-deployment.md) — slot-swap reversal as primary rollback mechanism
- [devops/ci-cd-pipelines.md](ci-cd-pipelines.md) — where rollback workflows live
- [devops/health-checks.md](health-checks.md) — readiness signals that trigger automated rollback
- [devops/environment-configuration.md](environment-configuration.md) — feature flags as config
- [data-access/migrations.md](../data-access/migrations.md) — expand-contract patterns
- [azure/app-service.md](../azure/app-service.md) — slot mechanics
- [azure/configuration-and-secrets-management.md](../azure/configuration-and-secrets-management.md) — App Configuration feature management
- [architecture/outbox-pattern.md](../architecture/outbox-pattern.md) — idempotency for compensating actions

## Real-World Usage

### Azure App Service slot reversal during a checkout incident

A SaaS billing platform deployed v3.18.0 at 14:02. Application Insights flagged a 4× spike in `/api/checkout` 500s at 14:06. The on-call engineer ran `az webapp deployment slot swap --slot staging --target-slot production --action reset`. By 14:06:09 traffic was back on v3.17.4, which was still warm in the staging slot. Total customer impact: 4 minutes. Postmortem identified a null-reference in a new code path; the fix shipped (with a feature flag) the next morning.

### Feature flag as instant rollback for a regional incident

A travel booking platform enabled a new pricing algorithm via Azure App Configuration's `Pricing:UseDynamicV2` flag. After 90 minutes of healthy 5% canary, they ramped to 100%. Within 12 minutes, the European region saw price calculations spiking 3× normal due to an integer overflow on EUR-to-USD conversion. They flipped the flag to `false` in the portal; 28 seconds later every instance picked up the change. The binary stayed running; only the code path changed. The bug fix shipped two days later in a normal release.

### Expand-contract over six weeks for a `User.PhoneNumber` rename

A fintech app needed to migrate from a free-text `PhoneNumber` column to an E.164-validated `PrimaryPhoneE164` column. Release 1 added the new column nullable. Release 2 dual-wrote both columns from a `UserUpdated` handler. Release 3 (two weeks later, after backfill completed) switched reads to the new column with fallback to the legacy. Release 4 (two more weeks later) removed the legacy reads. Release 5 dropped the legacy column. Every release was independently rollback-safe. Zero incidents related to the migration.

### GitHub Actions revert workflow during an outage

A logistics company's `release-prod.yml` runs daily at 9am. One morning the post-deploy smoke test detected elevated 5xx rates from a new validation rule. The on-call clicked "Run workflow" on `rollback-prod.yml`, entered `v2024.11.03-rc2` as the target tag and pasted the incident URL. The workflow pulled the previous image from ACR, updated the App Service, and posted a Teams notification — all in 90 seconds. The forward-fix went out the next morning behind a feature flag.

### Immutable image tags preventing a rollback failure

A platform team had been letting CI overwrite the `staging` tag for development convenience. A senior engineer noticed during a tabletop exercise that this would prevent rollback if the same tag had been promoted to production. They enabled ACR tag-immutability for all `v*` semantic version tags and changed CI to push new SHAs only. Two months later a rollback succeeded that, under the old policy, would have failed because the tag had been overwritten.

## Code Example — Before and After

### Before — no flag, destructive migration, `:latest` tag, no revert workflow

```yaml
# .github/workflows/deploy.yml
name: deploy
on: { push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t ghcr.io/contoso/orders-api:latest .
      - run: docker push ghcr.io/contoso/orders-api:latest
      - run: kubectl set image deployment/orders-api api=ghcr.io/contoso/orders-api:latest
      - run: dotnet ef database update --connection "$PROD_CONN"  # ❌ may drop columns
```

```csharp
// CheckoutService.cs — no flag, all-or-nothing risk
public sealed class CheckoutService
{
    public async Task<CheckoutResult> CompleteAsync(Guid cartId, CancellationToken ct)
    {
        // New algorithm, no escape hatch
        var price = _newPricing.Calculate(cartId);
        return await _gateway.ChargeAsync(cartId, price, ct);
    }
}
```

Incident: bad pricing math hits Production. Rollback options: (a) re-deploy previous code — but `:latest` no longer points at it and the previous image was garbage-collected; (b) revert the destructive migration — impossible without a database restore. Total recovery: 4 hours, partial data loss.

### After — versioned immutable tags, feature flag, expand-contract migration, dedicated rollback workflow

```yaml
# .github/workflows/deploy-prod.yml
name: deploy-prod
on:
  push: { tags: ['v*'] }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - id: tag
        run: echo "version=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build & push immutable tag
        run: |
          docker build -t ghcr.io/contoso/orders-api:${{ steps.tag.outputs.version }} .
          docker push ghcr.io/contoso/orders-api:${{ steps.tag.outputs.version }}

      - name: Apply additive-only migration
        run: |
          ./tools/assert-additive-only.ps1   # CI guard
          dotnet ef database update --connection "${{ secrets.SQL_PROD }}"

      - name: Capture previous image for rollback
        run: |
          PREV=$(kubectl get deploy orders-api -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "PREVIOUS_IMAGE=$PREV" >> $GITHUB_ENV

      - name: Deploy new image
        run: |
          kubectl set image deployment/orders-api \
            api=ghcr.io/contoso/orders-api:${{ steps.tag.outputs.version }}
          kubectl rollout status deployment/orders-api --timeout=180s

      - name: Smoke test
        run: dotnet test ./tests/Smoke -c Release

      - name: Auto-rollback on smoke failure
        if: failure()
        run: |
          kubectl set image deployment/orders-api api=$PREVIOUS_IMAGE
          kubectl rollout status deployment/orders-api --timeout=180s
```

```yaml
# .github/workflows/rollback-prod.yml — manual emergency rollback
name: rollback-prod
on:
  workflow_dispatch:
    inputs:
      target_tag: { required: true, type: string }
      reason: { required: true, type: string }
jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production-rollback   # looser approval policy
    steps:
      - run: |
          docker manifest inspect ghcr.io/contoso/orders-api:${{ inputs.target_tag }}
          kubectl set image deployment/orders-api \
            api=ghcr.io/contoso/orders-api:${{ inputs.target_tag }}
          kubectl rollout status deployment/orders-api --timeout=180s
      - run: |
          curl -X POST "${{ secrets.TEAMS_INCIDENT_WEBHOOK }}" \
            -d '{"text":"🔴 Prod rolled back to ${{ inputs.target_tag }} — ${{ inputs.reason }}"}'
```

```csharp
// CheckoutService.cs — risky logic gated by feature flag
public sealed class CheckoutService
{
    private readonly IFeatureManager _features;
    private readonly IPricingV1 _legacyPricing;
    private readonly IPricingV2 _newPricing;
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(
        IFeatureManager features,
        IPricingV1 legacyPricing,
        IPricingV2 newPricing,
        IPaymentGateway gateway,
        ILogger<CheckoutService> logger)
    {
        _features = features;
        _legacyPricing = legacyPricing;
        _newPricing = newPricing;
        _gateway = gateway;
        _logger = logger;
    }

    public async Task<CheckoutResult> CompleteAsync(Guid cartId, CancellationToken ct)
    {
        var useV2 = await _features.IsEnabledAsync("Pricing:UseDynamicV2");
        var price = useV2
            ? await _newPricing.CalculateAsync(cartId, ct)
            : await _legacyPricing.CalculateAsync(cartId, ct);

        _logger.LogInformation("Charging cart {CartId} with {Algorithm}: {Price}",
            cartId, useV2 ? "v2" : "v1", price);

        return await _gateway.ChargeAsync(cartId, price, ct);
    }
}
```

Incident: same bug hits Production. Response: (a) flip `Pricing:UseDynamicV2` to `false` in App Configuration — 28 seconds, every instance back on legacy pricing; OR (b) trigger `rollback-prod.yml` with the previous tag — 90 seconds. Migration was additive, so previous code runs against the new schema without issue. Total recovery: under 2 minutes, zero data loss.

## Interview Questions and Answers

### 1. A release just hit Production and 500 rates are spiking. Walk me through the next five minutes.

**Why this matters:** Tests incident-response reflexes and operational maturity.

**Answer:** First action — revert. If we use App Service slots, `az webapp deployment slot swap --slot staging --target-slot production --action reset` — recovery in 5–15 seconds. If we use Kubernetes, `kubectl rollout undo deployment/orders-api`. If the bad behavior is behind a feature flag, flipping the flag is faster than a redeploy. While the revert is in flight, page the on-call team in Teams, post the incident channel, capture the bad image SHA for forensics, and start the postmortem timer. Diagnosis happens *after* customers are out of pain.

**Trade-off:** Reverting may roll back a partially-good release. Acceptable — we ship again tomorrow with the fix isolated.

**Real project:** We had a 4-minute MTTR on a checkout incident because the on-call reflexively ran the slot reset before investigating. Post-incident review confirmed it was the right call; the fix went out the next morning.

### 2. Why is `:latest` a problem for rollback?

**Why this matters:** Tests understanding of immutable artifacts.

**Answer:** `:latest` is mutable — every push to `:latest` overwrites it. When you try to roll back, the previous `ReplicaSet` or deployment metadata still points to `:latest`, which now resolves to the broken image. `kubectl rollout undo` succeeds but the new pods pull the same bad bits. The fix is to use semver tags or, even better, SHA digests (`@sha256:...`) which are content-addressed and cannot be moved. Pair this with a container registry retention policy so old tags aren't garbage-collected before you can roll back to them.

**Trade-off:** Pinned tags require updating manifests on every release — a one-line change usually automated by the CI workflow.

**Real project:** We enabled ACR tag immutability for all `v*` tags. Two months later we successfully rolled back to a 3-week-old version that would have been silently overwritten under the old policy.

### 3. Walk me through expand-contract for a column rename.

**Why this matters:** Tests schema-evolution discipline that is essential for rollback safety.

**Answer:** To rename `Users.PhoneNumber` to `Users.PrimaryPhoneE164`: Release 1 adds the new column nullable. Release 2 dual-writes both columns — every write touches both. Release 3 (after a backfill job populates old rows) switches reads to the new column with a fallback. Release 4 removes the fallback. Release 5 sets the new column NOT NULL. Release 6 drops the old column. Six releases, four to six weeks. At every release, rolling back to the previous code is safe because both columns exist and the data is consistent.

**Trade-off:** Slow. Worth it. The alternative — a single "big-bang" rename — leaves no rollback path if anything goes wrong.

**Real project:** We migrated a `Customer.Email` to `Customer.BillingEmail` over six weeks. Zero incidents, zero data loss, and at any point we could have rolled back to any previous release without a database restore.

### 4. How do feature flags accelerate rollback?

**Why this matters:** Tests progressive-delivery understanding.

**Answer:** A feature flag separates **deployment** (code is in Production) from **release** (code is exposed to users). When something breaks, flipping a flag in Azure App Configuration or LaunchDarkly stops the broken path in 10–30 seconds — far faster than a deploy. The binary keeps running; only the code path changes. Flags also enable partial rollback — disable the flag for the affected region or customer cohort while leaving it on for others. The cost is **flag debt**: old flags accumulate and must be cleaned up quarterly.

**Trade-off:** Two code paths in the codebase until the flag is retired. Always paired with a cleanup ticket.

**Real project:** A pricing engine bug affected 100% of EU traffic. Flag flip in 28 seconds restored healthy pricing while we shipped the fix two days later. Without the flag, the fix-forward cycle would have been a 90-minute deploy with continuing customer impact.

### 5. What's the difference between rollback and roll-forward, and when do you choose each?

**Why this matters:** Tests judgment.

**Answer:** **Rollback** reverts to a known-good previous version. **Roll-forward** ships a new fix. Rollback is the default when (a) the bad release is recent and the previous version is still warm, (b) the schema supports it, and (c) the fix is non-trivial. Roll-forward is right when (a) you can't go back — destructive migration, irreversible data change — or (b) the fix is trivial and tested. Most incidents want rollback first to stop bleeding, then roll-forward with the proper fix once the team is calm.

**Trade-off:** Roll-forward under pressure produces second incidents. Rollback under pressure is boring and effective.

**Real project:** A bad config push (not a code change) — we rolled forward by editing the config in App Configuration, 5 seconds. A bad code path with no flag and a 30-minute fix — we rolled back via slot reset, then shipped the fix two days later in a normal release.

### 6. Your team uses one approval policy for both deploys and rollbacks. Why is that a problem?

**Why this matters:** Tests incident-response process design.

**Answer:** Deploy approvals exist to prevent bad releases. Rollback approvals — when set the same way — *prevent fast recovery* during an incident. If the production environment requires two reviewers and a 30-minute wait, the on-call cannot revert during a P0. The fix is two separate GitHub environments: `production` with strict gates for forward deploys, and `production-rollback` with a looser policy — single on-call approver, no wait — for emergencies. Every rollback is audited after the fact in the postmortem.

**Trade-off:** A less-strict path means a malicious or sleepy on-call could roll back unnecessarily. Mitigation: alerting on any `production-rollback` execution and mandatory postmortem.

**Real project:** We had a 22-minute P1 because the on-call couldn't get a second approver at 3am. We split the environments the next day. The next rollback took 90 seconds.

### 7. What about side effects that already fired — emails, payments, webhooks?

**Why this matters:** Tests understanding that rollback isn't a time machine.

**Answer:** Rolling back code does not unsend emails, refund payments, or recall webhooks. For each outbound side effect, you need a compensating action: send a correction email, issue a refund, send a retraction webhook. The best defense is **idempotency** and the **outbox pattern** — every outbound action has a deduplication key so the system can detect duplicates and the compensation path can target specific records. For payments, use the gateway's idempotency keys (Stripe, Adyen, Braintree all support this). For internal events, the outbox table records every emitted event so post-incident reconciliation can identify what was sent during the bad window.

**Trade-off:** Compensating actions add complexity. Critical for financial systems; optional for telemetry.

**Real project:** A bug double-billed 47 customers over 12 minutes. Rollback stopped the bleeding; a reconciliation script using Stripe idempotency keys identified the duplicates and refunded them automatically. Total customer-visible impact: refund email within 4 hours. Without idempotency keys, that would have been a manual support exercise.

### 8. How do you practice rollback so the team is ready?

**Why this matters:** Tests SRE-style operational rigor.

**Answer:** Game days — monthly chaos engineering exercises where someone deliberately ships a bad version to a non-production environment and the on-call must detect and roll back without notes. Time it. Postmortem the exercise. Cycle the on-call rotation so everyone gets practice. Also: every real rollback gets a postmortem, even successful ones — "what triggered it, how long did it take, what surprised us." Over time the runbook gets sharper, the alerts get better tuned, and the team's reflex gets faster.

**Trade-off:** Game days cost engineering hours. They pay for themselves the first time a 3am incident is recovered in 90 seconds instead of 30 minutes.

**Real project:** Quarterly game days reduced our P1 MTTR from a 25-minute median to 4 minutes over 18 months. The cost was 2 hours per engineer per quarter.

## Summary Checklist

- [ ] Every production service has a one-command rollback documented in a runbook.
- [ ] Risky changes ship behind feature flags I can flip without a deploy.
- [ ] Every schema migration is additive; column drops happen at least one release later.
- [ ] All container images are tagged with semantic versions or SHA digests — never `:latest`.
- [ ] Container registry retention keeps the previous N versions available for rollback.
- [ ] App Service slots stay warm for at least 48 hours after a successful swap.
- [ ] A dedicated `rollback-prod` workflow exists with a faster approval gate than forward deploys.
- [ ] Smoke tests in the deploy pipeline trigger auto-rollback on failure.
- [ ] Outbound side effects use idempotency keys to enable safe compensation after a rollback.
- [ ] The team runs game days quarterly to practice rollback under pressure.
