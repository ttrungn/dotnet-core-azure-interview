# Deployment to Azure

## What It Is

Deployment to Azure is the end-to-end practice of taking a built .NET artifact and getting it running reliably in a target Azure environment with the right configuration, identity, networking, and observability. Modern deployment is **declarative** (infrastructure as code), **automated** (CI/CD pipelines), **identity-based** (federated credentials, Managed Identity), and **progressive** (slots, revisions, canary, blue/green).

The three main models you'll deploy into:

- **App Service / Functions** — managed PaaS, slot-based deploys.
- **Container Apps** — managed Kubernetes-lite with revisions and traffic splitting.
- **AKS** — full Kubernetes for complex workloads; usually Helm + GitOps (Flux or Argo CD).

```yaml
# A modern pipeline: build once, deploy via OIDC (no stored secret), use a slot, swap on success
- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

- uses: azure/webapps-deploy@v3
  with:
    app-name: contoso-payments-api
    slot-name: staging
    package: ./publish
```

## Why It Exists

Hand-rolled deployments (RDP into a VM, copy a zip, edit a config) failed in five dimensions:

1. **Drift** — production "configurations" lived in operators' heads and on whiteboards. No two environments matched.
2. **Downtime** — restart-to-deploy meant scheduled outages.
3. **Secret leakage** — pipelines stored long-lived service principal secrets that anyone with repo write could exfiltrate.
4. **Rollback was redeployment** — recovery took as long as deploy itself, sometimes longer.
5. **Audit** — "who deployed what when" had no answer.

Modern Azure deployment fixes each:

- **Bicep / Terraform** declares infrastructure; every change is a commit, reviewable and auditable.
- **Slots, revisions, blue/green** enable zero-downtime cutover and instant rollback.
- **OIDC federated credentials** eliminate stored CI/CD secrets entirely.
- **Managed Identity** removes runtime credentials.
- **GitHub Actions / Azure DevOps / Azure Developer CLI (`azd`)** put deploy artifacts, logs, and approvals in one auditable workflow.

## When To Use It

This page covers deployment patterns. Use these patterns when:

- **Use App Service deployment slots** — for web APIs and apps with simple swap-based promotion, where the staging slot can hold a single warmed instance.
- **Use Container Apps revisions** — for containerized HTTP APIs and Dapr-style workloads where you want traffic-split canaries with no Kubernetes overhead.
- **Use AKS + Helm + Flux** — for complex topologies, sidecars, GPUs, or when you've outgrown PaaS limits.
- **Use Azure Functions deployment slots** — for serverless workloads; same swap model as App Service.
- **Use `azd up`** — for full environments (infra + app code) from a single repo, ideal for new projects, demos, and ephemeral test environments.
- **Use Bicep over ARM JSON** — always, for new infrastructure-as-code.
- **Use GitHub Actions with OIDC** — for CI/CD; no client secrets in GitHub.

**Do not use:**

- Manual portal-driven deploys for anything beyond a one-off demo.
- Long-lived service principal client secrets in CI/CD — use OIDC.
- A single production slot with no staging — there's no rollback path.
- Plain `kubectl apply` from a dev laptop into production — use a GitOps controller.

## Why It Is Important

Deployment quality is the single biggest lever on production reliability. A well-engineered deploy pipeline:

- **Achieves zero-downtime releases** via slot swap or traffic-split revisions.
- **Enables instant rollback** — swap back, shift traffic to old revision — in seconds, not minutes.
- **Eliminates secret leakage** — OIDC federated credentials mean no PAT, no client secret, no long-lived key in any pipeline variable.
- **Provides audit trail** — every deploy is a Git commit with an approver, a CI run, and an Azure activity log entry.
- **Makes environments reproducible** — `azd up` or `terraform apply` rebuilds the entire stack from scratch.
- **Catches misconfiguration before users do** — health checks, smoke tests, Live Metrics validation gate the swap.

Interviewers ask "walk me through your deploy pipeline" because the answer reveals whether you've shipped production software or only built features.

## How It's Used in C# / .NET

Tooling:

- **`azd` (Azure Developer CLI)** — opinionated end-to-end environment provisioning + app deploy.
- **Bicep** — Microsoft's first-party declarative IaC for Azure, compiles to ARM.
- **GitHub Actions** with `azure/login@v2` (OIDC) and `azure/webapps-deploy@v3`.
- **Azure DevOps Pipelines** with workload identity federation.
- **Helm** for AKS deployments.
- **Flux / Argo CD** for GitOps on AKS.

### App Service with deployment slots

```bash
# Build and publish
dotnet publish -c Release -o ./publish

# Deploy to the staging slot
az webapp deploy \
  --resource-group rg-contoso-prod \
  --name contoso-payments-api \
  --slot staging \
  --src-path ./publish.zip

# Run smoke tests against https://contoso-payments-api-staging.azurewebsites.net
./tools/run-smoke-tests.sh staging

# Swap staging → production (atomic; takes ~10 seconds, no downtime)
az webapp deployment slot swap \
  --resource-group rg-contoso-prod \
  --name contoso-payments-api \
  --slot staging \
  --target-slot production
```

Slot settings: configure environment-specific app settings (Key Vault references, connection strings) with **slot stickiness** so swap doesn't move the wrong values.

### Container Apps with revisions and traffic splitting

```bash
# Deploy a new revision in single-revision mode → traffic shifts 100% to it
az containerapp update \
  --name contoso-orders-api \
  --resource-group rg-contoso-prod \
  --image contosoacr.azurecr.io/orders-api:1.2.3

# Multi-revision: keep both, split traffic 90/10 for canary
az containerapp ingress traffic set \
  --name contoso-orders-api \
  --resource-group rg-contoso-prod \
  --revision-weight contoso-orders-api--v122=90 contoso-orders-api--v123=10
```

After the canary proves healthy, shift 100% to the new revision; after stability, deactivate the old.

### GitHub Actions with OIDC federated credentials (no secrets)

```yaml
# .github/workflows/deploy.yml
name: Build and deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet publish src/Payments.Api -c Release -o publish
      - uses: actions/upload-artifact@v4
        with: { name: app, path: publish }

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with: { name: app, path: publish }

      # OIDC — no client secret, no PAT
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: contoso-payments-api
          slot-name: staging
          package: ./publish

      - name: Smoke tests
        run: ./tools/smoke-tests.sh https://contoso-payments-api-staging.azurewebsites.net

  promote:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production    # gated by required reviewer
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Swap slots
        run: |
          az webapp deployment slot swap \
            --resource-group rg-contoso-prod \
            --name contoso-payments-api \
            --slot staging --target-slot production

      - name: Watch Live Metrics for 5 minutes
        run: ./tools/watch-live-metrics.sh contoso-payments-api 300
```

### Bicep — infrastructure as code

```bicep
@description('Environment name (dev/staging/prod)')
param environmentName string

@description('Azure region')
param location string = resourceGroup().location

resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-contoso-payments-${environmentName}'
  location: location
  sku: { name: 'P1v3', tier: 'PremiumV3' }
  kind: 'linux'
  properties: { reserved: true }
}

resource site 'Microsoft.Web/sites@2023-01-01' = {
  name: 'contoso-payments-api-${environmentName}'
  location: location
  identity: { type: 'SystemAssigned' }   // Managed Identity for Key Vault / SQL
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      healthCheckPath: '/health/ready'
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: '@Microsoft.KeyVault(VaultName=${kvName};SecretName=AppInsights--ConnectionString)'
        }
        {
          name: 'ASPNETCORE_ENVIRONMENT'
          value: environmentName == 'prod' ? 'Production' : 'Staging'
        }
      ]
    }
  }
}

resource stagingSlot 'Microsoft.Web/sites/slots@2023-01-01' = if (environmentName == 'prod') {
  parent: site
  name: 'staging'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: site.properties.siteConfig
  }
}
```

### `azd` — full environment from one command

```bash
# Initialize once
azd init --template contoso/payments-api

# Provision infra and deploy app code together
azd up --environment prod
```

`azd` reads `azure.yaml`, runs Bicep for provisioning, builds the .NET app, and deploys to the provisioned services. Ideal for fresh projects and per-PR ephemeral environments.

## Advantages

- **Zero downtime** via slot swap, revision traffic splitting, or rolling pod restart.
- **Instant rollback** — swap back, shift traffic, redeploy previous artifact.
- **No stored secrets** — OIDC federated credentials in CI/CD, Managed Identity at runtime.
- **Declarative reproducibility** — Bicep + pipelines rebuild any environment from Git.
- **Per-environment isolation** — separate resource groups, vaults, App Configurations.
- **Approval gates** in GitHub Environments / Azure DevOps Environments enforce review for prod.
- **Audit trail** — every deploy is a Git commit + CI run + Azure activity log.
- **Progressive delivery** — canary, blue/green, percentage rollout, feature flags all compose.

## Disadvantages

- **Learning curve** — Bicep, IaC, OIDC federation, deployment slots, GitOps each have their own quirks.
- **Slot warm-up cost** — App Service slots count toward the plan; complex apps need staging instances always running.
- **Configuration sprawl** — slot settings, app settings, Key Vault references, App Configuration, environment variables — five layers, each with override rules.
- **Container Apps and AKS upgrade cycles** add platform-level deployments on top of app deployments.
- **GitOps adds latency** — Flux reconciles on a poll; pushed-then-pulled deploys take 1–5 minutes longer.
- **Cost** — Premium App Service plans for slots, Container Apps revision retention, AKS node pools, all add up vs. a single deploy slot.

## Common Mistakes

### 1. Storing service principal client secrets in GitHub Actions

```yaml
# ❌ Long-lived secret in repo; anyone with maintainer can exfiltrate
- uses: azure/login@v2
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}    # contains clientSecret
```

**Fix:** Use OIDC federated credentials. Configure a federated credential on an Entra App Registration scoped to the repo + branch + environment:

```yaml
# ✅ No stored secret; GitHub mints a JWT and exchanges with Entra at job time
permissions: { id-token: write, contents: read }
- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

### 2. Deploying straight to production with no staging slot or canary

```bash
# ❌ Any deploy bug is a customer-facing outage
az webapp deploy --slot production --src-path ./publish.zip
```

**Fix:** Deploy to staging, run smoke tests, swap:

```bash
# ✅
az webapp deploy --slot staging --src-path ./publish.zip
./tools/smoke.sh staging
az webapp deployment slot swap --slot staging --target-slot production
```

### 3. Forgetting slot stickiness on environment-specific settings

When you swap, **app settings move with the slot** unless marked as deployment-slot settings ("slot sticky"). Forget this and your production slot starts pointing at the staging database.

**Fix:** Mark connection strings, environment names, Key Vault URIs as **slot settings** in the portal or Bicep:

```bicep
// Slot settings stay with the slot during swap
resource siteConfig 'Microsoft.Web/sites/config@2023-01-01' = {
  parent: site
  name: 'slotConfigNames'
  properties: {
    appSettingNames: ['ASPNETCORE_ENVIRONMENT', 'KeyVault__Uri']
    connectionStringNames: ['OrdersDb']
  }
}
```

### 4. No health check probe gating the swap

A swap completes when warm-up completes — without a health check path, "warm-up" is a TCP-port check that's true even when the app's database connection is broken.

**Fix:** Set `healthCheckPath: '/health/ready'`. The slot only becomes swap-eligible when `/health/ready` returns 200 from a real readiness probe (DB, Service Bus, Key Vault):

```csharp
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### 5. Treating Bicep files as one-off scripts instead of source-of-truth

Engineers edit settings in the portal, then someone runs `bicep deploy` later and overwrites those edits. The portal-modified version is gone with no audit.

**Fix:** Portal changes are deltas that must be reflected back into Bicep. Better: lock down portal write access in production; all changes go through PRs.

### 6. AKS deploys via `kubectl apply` from a developer's laptop

```bash
# ❌ Untraceable; bypasses review; uses developer's kube context
kubectl apply -f deployment.yaml
```

**Fix:** GitOps — commit manifests to a Git repo; Flux or Argo CD reconciles them into the cluster. Developer never has prod cluster access:

```yaml
# manifests/payments/release.yaml in a Git repo watched by Flux
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata: { name: payments-api }
spec:
  chart: { spec: { chart: payments-api, version: '1.2.3' } }
  values: { image: { tag: '1.2.3' } }
```

### 7. No automated rollback signal

A deploy passes smoke tests but P95 latency doubles 3 minutes in. The team notices an hour later from customer reports.

**Fix:** A 5-minute post-deploy Live Metrics watch job that auto-swaps back if error rate or P95 exceeds thresholds:

```bash
ERROR_RATE=$(az monitor metrics list --resource $APP --metric "Http5xx" \
  --start-time $(date -u -d '-5 min' +%FT%TZ) --aggregation Total --query "value[0].timeseries[0].data[-1].total" -o tsv)
if (( ERROR_RATE > 50 )); then
  az webapp deployment slot swap --slot production --target-slot staging  # rollback
  exit 1
fi
```

## Best Practices

- **Build once, deploy many** — one artifact promoted dev → staging → prod; never rebuild per environment.
- **Bicep for all infrastructure**, in Git, behind PR review.
- **OIDC federated credentials** for CI/CD — never client secrets.
- **Managed Identity** for all runtime authentication to Azure services.
- **Always a staging slot or canary revision** before production cutover.
- **Health check probes gate slot swap** and rolling restart.
- **Slot-sticky settings** for connection strings and environment-specific config.
- **Approval gate** on the production environment in GitHub / Azure DevOps Environments.
- **Live Metrics watch job** after every prod deploy with auto-rollback thresholds.
- **`azd up` for new projects** to bootstrap infra + app in one workflow.
- **Per-PR ephemeral environments** for integration tests; tear down on PR close.
- **Tag every Azure resource** with `Environment`, `Application`, `CostCenter`, `Owner`.
- **GitOps for AKS** — Flux or Argo CD; never `kubectl apply` from a laptop.

## Related Concepts

- [devops/ci-cd-pipelines.md](devops/ci-cd-pipelines.md) — pipeline patterns, gates, approvals.
- [devops/blue-green-deployment.md](devops/blue-green-deployment.md) — slot-swap and traffic-split strategies.
- [devops/rollback-strategy.md](devops/rollback-strategy.md) — automated and manual rollback patterns.
- [devops/environment-configuration.md](devops/environment-configuration.md) — per-environment configuration layering.
- [azure/app-service.md](azure/app-service.md) — slot mechanics in depth.
- [azure/azure-functions.md](azure/azure-functions.md) — same slot model for serverless.
- [azure/configuration-and-secrets-management.md](azure/configuration-and-secrets-management.md) — Key Vault references in slot settings.

## Real-World Usage

### Payments API on App Service

`contoso-payments-api` (P1v3 plan) has a `staging` slot. GitHub Actions builds on push to `main`, deploys to staging, runs a smoke-test suite, requires a human approval, then swaps. A post-swap Live Metrics watch auto-rolls back on regression. Deploys complete in ~6 minutes; rollback is < 30 seconds via reverse swap.

### Order processing on Container Apps

`orders-api` runs on Container Apps in multi-revision mode. Each deploy creates a new revision; CI splits 10% traffic for 10 minutes, monitors error rate and P95 from Application Insights, then promotes to 100%. Failed canary auto-deactivates the new revision with one CLI call.

### Microservice fleet on AKS with Flux

A platform team runs 40 microservices on AKS. App teams' CI pushes images to ACR and commits a Helm `values.yaml` update to a GitOps repo. Flux reconciles every 60 seconds. Production cluster has read-only RBAC for developers; all writes are Git commits.

### Multi-region active-active payments

`contoso-payments-api` runs in West Europe (primary) and North Europe (secondary). Bicep modules deploy paired App Services with Front Door routing in front. A deploy promotes through staging → WEU production → NEU production; both regions converge within 10 minutes. A regional outage triggers Front Door failover within seconds.

### Ephemeral PR environments

Each pull request triggers `azd up` against a per-PR resource group with a hashed name. Reviewers get a unique URL to test the branch against real Azure services (small SKUs). On PR close, an action runs `azd down`. Cost is bounded by an Azure Policy on resource SKUs and a cleanup workflow.

### Functions deploy slots

`contoso-billing-functions` uses Functions premium plan with a `staging` slot. The same swap pattern applies; this keeps cold start out of the user request path during deploys.

## Code Example — Before and After

### Before — manual deploy, stored secret, no staging, no rollback

```yaml
# .github/workflows/deploy.yml
on: { push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet publish -c Release -o publish
      - run: zip -r app.zip publish

      # ❌ Long-lived service principal client secret
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # ❌ Direct production deploy, no staging, no smoke test, no swap
      - run: |
          az webapp deploy \
            --resource-group rg-contoso-prod \
            --name contoso-payments-api \
            --src-path app.zip
```

**Problems:**
- Client secret in `secrets.AZURE_CREDENTIALS` — long-lived, hard to rotate, blast radius is the entire subscription.
- Every deploy is a production outage window.
- No rollback path other than redeploying yesterday's commit.
- No approval gate, no smoke tests, no metrics check.

### After — OIDC, staging slot, smoke tests, approval, auto-rollback

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

env:
  APP_NAME: contoso-payments-api
  RG: rg-contoso-prod

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet test -c Release --no-build
      - run: dotnet publish src/Payments.Api -c Release -o publish
      - uses: actions/upload-artifact@v4
        with: { name: app, path: publish }

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://${{ env.APP_NAME }}-staging.azurewebsites.net
    steps:
      - uses: actions/download-artifact@v4
        with: { name: app, path: publish }

      # ✅ OIDC — no stored secret
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      # ✅ Deploy to staging slot, never direct to prod
      - uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.APP_NAME }}
          slot-name: staging
          package: ./publish

      # ✅ Wait for health probe before smoke tests
      - name: Wait for /health/ready
        run: |
          for i in {1..30}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
              https://${{ env.APP_NAME }}-staging.azurewebsites.net/health/ready)
            [ "$STATUS" = "200" ] && exit 0
            sleep 10
          done
          exit 1

      - name: Smoke tests
        run: ./tools/smoke-tests.sh https://${{ env.APP_NAME }}-staging.azurewebsites.net

  promote:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production    # ✅ requires reviewer approval per GitHub Environment protection rules
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      # ✅ Atomic slot swap; zero-downtime cutover
      - name: Swap staging → production
        run: |
          az webapp deployment slot swap \
            --resource-group ${{ env.RG }} \
            --name ${{ env.APP_NAME }} \
            --slot staging --target-slot production

      # ✅ Watch Live Metrics for 5 min; auto-rollback on regression
      - name: Auto-rollback on regression
        run: |
          sleep 60   # let traffic stabilize
          for i in {1..4}; do
            FAILURES=$(az monitor metrics list \
              --resource /subscriptions/${{ vars.AZURE_SUBSCRIPTION_ID }}/resourceGroups/${{ env.RG }}/providers/Microsoft.Web/sites/${{ env.APP_NAME }} \
              --metric "Http5xx" --interval PT1M --aggregation Total \
              --query "value[0].timeseries[0].data[-1].total" -o tsv)
            if (( ${FAILURES:-0} > 50 )); then
              echo "Regression detected (5xx=$FAILURES); rolling back"
              az webapp deployment slot swap \
                --resource-group ${{ env.RG }} \
                --name ${{ env.APP_NAME }} \
                --slot production --target-slot staging
              exit 1
            fi
            sleep 60
          done
          echo "Deploy stable"
```

**Why this is better:**
- No stored secrets — OIDC federation issues a short-lived token per job.
- Production deploy is gated by smoke tests, then a human approver, then automatic regression detection.
- Slot swap is atomic and reversible.
- Rollback is sub-30-second — a reverse swap, not a redeploy.
- Every step is auditable in the GitHub Action log and Azure activity log.

## Interview Questions and Answers

### 1. Walk me through your zero-downtime deploy flow for an App Service-hosted .NET API.

**Why this matters:** This is the single most common deployment-architecture question.

**Answer:** Build the artifact once in CI. Deploy to a pre-warmed staging slot. Wait on the `/health/ready` health-check endpoint to return 200 (proves DB, Service Bus, Key Vault are reachable from the slot's identity). Run a smoke-test suite against the staging URL. Pause for human approval via GitHub Environment protection rules. Trigger `az webapp deployment slot swap` — the swap is atomic; existing connections drain, new traffic routes to the new instance, total user-visible downtime is zero. Watch Application Insights Live Metrics for 5 minutes; if `Http5xx` or P95 latency exceed thresholds, reverse-swap automatically.

**Trade-off:** Slot-based deploys require a Premium plan (Standard supports slots too but with limits); the slot consumes plan capacity. For high-traffic apps, also pre-warm the new slot with `WEBSITE_WARMUP_PATH` to avoid the first-request latency hit.

### 2. Explain OIDC federated credentials for GitHub Actions → Azure. Why is this better than a client secret?

**Why this matters:** Modern security baseline; client secrets are increasingly disallowed in regulated environments.

**Answer:** With OIDC, GitHub issues a short-lived JWT identifying the workflow run (`repo:contoso/payments-api:ref:refs/heads/main:environment:production`). Azure Entra has a federated credential registered on an App Registration that trusts that subject pattern. At job time, `azure/login@v2` presents the JWT, Entra validates the subject claim, issues a 1-hour access token. No long-lived secret exists in GitHub or Entra. If the repo is compromised, the attacker still needs to push to `main` *and* match the federated subject pattern. Rotation is automatic — there's nothing to rotate.

**Real project:** On a payments platform we eliminated all `AZURE_CREDENTIALS` GitHub secrets across 22 repos in a two-week migration; the auditor's report went from "shared secret risk" to clean.

### 3. App Service slot swap vs. Container Apps revisions vs. AKS rolling update — when do you pick each?

**Why this matters:** Tests deployment-model fit.

**Answer:** **Slot swap** for monolithic web APIs and apps on App Service — the simplest model, atomic, zero downtime, easy rollback. **Container Apps revisions** for containerized HTTP services where you want traffic splitting for canary (10% to new, watch, promote) — single command, no Kubernetes to manage. **AKS rolling update** when you've outgrown PaaS — sidecars, GPUs, complex topologies, service mesh — and pair it with GitOps and Helm. Slot swap is binary (old or new); revisions and AKS allow gradient rollout.

**Trade-off:** Slot swap is the simplest but can't do percentage canary natively (you'd need Front Door in front for that). Container Apps and AKS are more flexible but have more moving parts.

### 4. Your slot swap accidentally moved a production database connection string to the staging slot. What went wrong and how do you prevent it?

**Why this matters:** A real, common production incident.

**Answer:** App settings move with the slot during swap **unless** marked as "deployment-slot settings" (slot-sticky). The connection string wasn't marked sticky, so the swap dragged the production DB into the staging slot — staging started writing to prod and vice versa. The fix is to flag environment-specific settings (connection strings, Key Vault URIs, App Insights connection strings, `ASPNETCORE_ENVIRONMENT`) as slot settings in Bicep or the portal. Prevention is to keep slot setting flags in IaC so a fresh deploy can never miss them.

### 5. How do you implement a canary release on Container Apps?

**Why this matters:** Progressive delivery literacy.

**Answer:** Enable multi-revision mode. Deploy a new revision (don't activate full traffic). Use `az containerapp ingress traffic set` to allocate 10% to the new revision and 90% to the old. Monitor Application Insights for the new revision's tag (it gets the revision name as a property automatically) — error rate, P95, business metrics. After 10–30 minutes of healthy data, shift to 50%, then 100%. If anything regresses, set the new revision to 0% — instant rollback without redeploy. After 24 hours of stable 100%, deactivate the old revision.

**Trade-off:** Two revisions cost double during the canary window. Some workloads can't run two versions safely (e.g., breaking database migrations) — those need expand/contract schema patterns first.

### 6. Bicep vs. ARM JSON vs. Terraform — what do you use and why?

**Why this matters:** IaC tool selection.

**Answer:** For Azure-only stacks, **Bicep** — it's Microsoft's first-party, compiles to ARM, has clean syntax, IntelliSense, modules, and zero state file to manage (state lives in the ARM resource graph). **Terraform** for multi-cloud or when the team already has Terraform muscle memory; the providers are excellent but you carry state file overhead. **ARM JSON** never, for new code — Bicep transpiles to it transparently. The Bicep deploy is idempotent (`what-if` shows the delta before apply), and Pull Requests review nicely because the syntax is human-readable.

**Trade-off:** Terraform's plan/apply with remote state gives stronger drift detection for cross-cloud or mixed teams. Bicep relies on Azure's own state, which can drift if anyone uses the portal.

### 7. A deploy completes successfully but errors spike 10 minutes later. How do you build an automated guard?

**Why this matters:** Production reliability engineering.

**Answer:** A post-deploy watch job runs for 5–15 minutes after the swap, querying Application Insights `Http5xx`, P95 latency on critical endpoints, and any custom KPI metrics (e.g., `checkout.failed` rate) every minute. If any metric crosses a threshold (e.g., 5xx > baseline + 2σ, P95 > 1.5× baseline, business metric drop > 30%), trigger an automatic reverse slot swap or revision traffic shift back to the previous revision. Notify the on-call channel. This catches the regressions that smoke tests miss — typically performance degradation under real production traffic that staging didn't reproduce.

**Real project:** On an orders API we caught two regressions in the first quarter — one a missing index, one a serialization regression in a hot path. Auto-rollback fired in under 5 minutes both times; users saw negligible impact.

### 8. How do you handle a breaking database migration in a zero-downtime deploy?

**Why this matters:** The hardest deployment problem in real apps.

**Answer:** Break the change into **expand → migrate data → contract** across multiple deploys. Deploy 1 (expand): add the new column/table, deploy code that writes to both old and new and reads from old. Deploy 2 (backfill): run a migration job to copy historical data to the new shape. Deploy 3 (cutover): code reads from new, still writes to both. Deploy 4 (contract): code reads and writes only new; drop the old column/table. This way every deploy is reversible because each deploy's running code is compatible with both schema versions. Slot swap or canary works at every step.

**Trade-off:** Four deploys for one logical change is slow. For trivial apps with maintenance windows the single-deploy approach is fine; for 24/7 services the expand/contract discipline is non-negotiable.

## Summary Checklist

- [ ] I deploy via GitHub Actions or Azure DevOps using OIDC federated credentials — no stored client secrets.
- [ ] I always deploy to a staging slot or canary revision before production cutover.
- [ ] My pipelines run smoke tests against the staging environment before promotion.
- [ ] Production promotion requires human approval via Environment protection rules.
- [ ] Slot swaps are gated by a `/health/ready` health-check probe.
- [ ] Environment-specific app settings are marked slot-sticky in Bicep.
- [ ] Infrastructure is declared in Bicep, reviewed in PRs, applied from CI.
- [ ] Runtime services authenticate to Azure via Managed Identity.
- [ ] A post-deploy watch job auto-rolls back on Live Metrics regression.
- [ ] AKS deploys go through GitOps (Flux / Argo CD), never `kubectl apply` from a laptop.
