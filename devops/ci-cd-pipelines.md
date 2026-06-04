# CI/CD Pipelines

## What It Is

A **CI/CD pipeline** is the automated path that takes a commit on a feature branch and turns it into running production code. It has two complementary halves:

- **Continuous Integration (CI)** — every push triggers a build, runs the test suite, performs static analysis and security scanning, and produces a single versioned **artifact** (a NuGet package, a zipped `publish` folder, or a Docker image pushed to Azure Container Registry).
- **Continuous Delivery / Deployment (CD)** — that exact same artifact is promoted through environments (`dev` → `test` → `staging` → `production`) with approvals, smoke tests, and health gates between each stage. *Delivery* stops short of production and waits for human approval; *Deployment* goes all the way without manual gates.

In a .NET shop the two dominant pipeline platforms are **GitHub Actions** (YAML workflows in `.github/workflows/`) and **Azure DevOps Pipelines** (YAML in `azure-pipelines.yml` or classic UI). Both run `dotnet restore`, `dotnet test`, and `dotnet publish` against your ASP.NET Core solution, then push a container image to ACR or a zip to App Service.

The cardinal rule: **build once, deploy many times**. The bits that pass CI must be the *exact same bits* that reach production — environment differences come from configuration, not from rebuilding.

## Why It Exists

Before CI/CD, releases were manual. A senior engineer would build on their laptop, RDP into a server, copy a zip file, restart IIS, and pray. The problems were predictable:

- "It works on my machine" — the build that shipped used different SDK versions, NuGet caches, or environment variables than the build that was tested.
- Friday afternoon deployments became all-nighters because nobody could reproduce the production failure locally.
- There was no audit trail — when a customer asked "what changed last Tuesday?", nobody knew.
- Rollback meant copying yesterday's zip out of a network share, assuming someone remembered to keep it.

CI/CD exists to make releases **boring**. A merged PR should produce the same artifact whether it's built on a developer laptop, on a `ubuntu-latest` GitHub runner, or on a self-hosted Azure DevOps agent in your VNet. The platform records who triggered each run, which commit it built, which tests passed, who approved the production stage, and which version is currently live.

It also exists to **shorten the feedback loop**. The faster a developer learns that their commit broke a test or a security scan, the cheaper the fix. A 90-second CI run is worth more than a 30-minute one.

## When To Use It

**Use a full CI/CD pipeline for:**

- Any production service — APIs, background workers, Azure Functions, Static Web Apps.
- Internal tools that more than one person depends on (release deployments, never `xcopy`).
- NuGet libraries published to a feed (artifact-driven promotion is essential).
- Infrastructure-as-Code (Bicep, Terraform, ARM) — `terraform plan` in CI, `terraform apply` in CD with approval.
- Database migrations — run `dotnet ef database update` (or DbUp / Flyway) as a pipeline step against a real connection.

**Do not over-engineer it for:**

- A throwaway proof-of-concept or a one-off script you'll delete next week.
- A personal repo with no collaborators (a single GitHub Action that runs `dotnet test` is enough).
- Code that is not actually deployed (sample apps, exercises).

Even for small projects, the minimum bar is: **PR triggers `dotnet test`, main branch protected, no merge without a green build**.

## Why It Is Important

CI/CD pipelines are the most powerful **risk-reduction** investment a backend team can make. They translate into four operational properties:

1. **Reproducibility** — the artifact in production was built by the same pipeline, with the same SDK version, from the same git SHA, that was tested. There is no manual step that someone could skip.
2. **Speed** — small, frequent deployments (10/day) are dramatically safer than big quarterly releases. CI/CD makes small deployments cheap.
3. **Auditability** — every deployment links a git commit, a pipeline run, an artifact version, and the person who approved it. When auditors or incident responders ask "what changed?", the answer is one click away.
4. **Recoverability** — because every artifact is versioned and stored, rolling back to `1.4.2` is a matter of redeploying the previous image tag. There is no rebuilding, no "where did we put that?".

In a regulated environment (PCI for payments, HIPAA for healthcare, SOC2 for SaaS) CI/CD with proper approvals is often not optional — it's the only practical way to prove the **four-eyes principle** (no engineer can deploy to production alone).

## How It's Used in C# / .NET

### 1. GitHub Actions — typical .NET 8 build → test → publish → deploy

`.github/workflows/order-api-cicd.yml`:

```yaml
name: order-api-cicd

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  DOTNET_VERSION: '8.0.x'
  REGISTRY: contosoacr.azurecr.io
  IMAGE_NAME: order-api

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}

      - name: Restore
        run: dotnet restore Order.sln

      - name: Build
        run: dotnet build Order.sln --configuration Release --no-restore

      - name: Test
        run: dotnet test Order.sln --configuration Release --no-build --logger trx --collect:"XPlat Code Coverage"

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/TestResults/*.trx'

  publish-image:
    needs: build-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write     # for OIDC federated login to Azure
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC, no secrets)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: ACR login
        run: az acr login --name contosoacr

      - name: Build & push image
        run: |
          IMAGE_TAG=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker build -t $IMAGE_TAG -f src/Order.Api/Dockerfile .
          docker push $IMAGE_TAG

  deploy-staging:
    needs: publish-image
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://order-api-staging.azurewebsites.net
    permissions:
      id-token: write
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service staging slot
        run: |
          az webapp config container set \
            --name order-api \
            --resource-group order-rg \
            --slot staging \
            --container-image-name ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Smoke test
        run: |
          curl --fail --retry 5 --retry-delay 10 \
            https://order-api-staging.azurewebsites.net/health/ready

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production            # requires manual approval (configured in repo settings)
    permissions:
      id-token: write
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Swap staging into production
        run: |
          az webapp deployment slot swap \
            --name order-api \
            --resource-group order-rg \
            --slot staging \
            --target-slot production
```

Key points:
- `actions/setup-dotnet@v4` pins the .NET 8 SDK so every run gets identical bits.
- Caching `~/.nuget/packages` typically halves CI time.
- **OIDC federated identity** (`id-token: write` + `azure/login@v2`) eliminates long-lived service principal secrets. The workflow gets a short-lived token from Entra ID per run.
- The `environment: production` block ties the deploy job to a GitHub Environment that requires a reviewer to approve.
- One image (`:${{ github.sha }}`) is built once and reused for staging and prod — no rebuilding.

### 2. Azure DevOps multi-stage YAML

`azure-pipelines.yml`:

```yaml
trigger:
  branches:
    include: [ main ]

variables:
  buildConfiguration: 'Release'
  dotnetSdk: '8.0.x'
  acrName: 'contosoacr'
  imageRepo: 'order-api'

stages:
- stage: Build
  jobs:
  - job: BuildTest
    pool: { vmImage: 'ubuntu-latest' }
    steps:
    - task: UseDotNet@2
      inputs: { version: $(dotnetSdk) }
    - script: dotnet restore Order.sln
    - script: dotnet build Order.sln -c $(buildConfiguration) --no-restore
    - script: dotnet test Order.sln -c $(buildConfiguration) --no-build --logger trx
    - task: PublishTestResults@2
      condition: succeededOrFailed()
      inputs: { testResultsFormat: 'VSTest', testResultsFiles: '**/*.trx' }
    - task: Docker@2
      inputs:
        containerRegistry: 'contosoacr-sc'
        repository: $(imageRepo)
        command: 'buildAndPush'
        Dockerfile: 'src/Order.Api/Dockerfile'
        tags: '$(Build.SourceVersion)'

- stage: DeployStaging
  dependsOn: Build
  jobs:
  - deployment: Staging
    environment: 'order-api-staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebAppContainer@1
            inputs:
              azureSubscription: 'order-prod-sc'
              appName: 'order-api'
              slotName: 'staging'
              imageName: '$(acrName).azurecr.io/$(imageRepo):$(Build.SourceVersion)'

- stage: DeployProduction
  dependsOn: DeployStaging
  jobs:
  - deployment: Production
    environment: 'order-api-prod'    # has approvals & checks attached
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureAppServiceManage@0
            inputs:
              azureSubscription: 'order-prod-sc'
              Action: 'Swap Slots'
              WebAppName: 'order-api'
              ResourceGroupName: 'order-rg'
              SourceSlot: 'staging'
```

### 3. Branch strategies

| Strategy | When to use | Pipeline shape |
|---|---|---|
| **Trunk-based** | Mature team, fast CI (<10 min), feature flags | Every PR → main → auto-deploy to staging, manual approval to prod |
| **GitFlow** | Regulated industries, infrequent releases | `develop` runs CI, `release/*` deploys to staging, `main` deploys to prod after sign-off |
| **GitHub Flow** | SaaS web apps | `main` is always deployable; short-lived feature branches with PR review |

Most modern .NET teams (and Microsoft internally) favor **trunk-based development with feature flags** — it pairs naturally with CI/CD and avoids long-lived merge nightmares.

### 4. Secret management

Never commit secrets. Use:

- **GitHub Actions** → repository / environment **secrets**, accessed as `${{ secrets.FOO }}`.
- **Azure DevOps** → **variable groups** linked to **Azure Key Vault**.
- **At runtime** → the app reads from Key Vault directly via `Azure.Identity.DefaultAzureCredential` and `Azure.Extensions.AspNetCore.Configuration.Secrets`.

Best practice: prefer **OIDC federated identity** over storing service principal client secrets in CI. GitHub Actions and Azure DevOps both support it. See [../azure/azure-key-vault.md](../azure/azure-key-vault.md).

### 5. Artifact promotion

The same image tag flows through environments:

```
acr.io/order-api:abc1234   ← built once from commit abc1234
  ├─ deployed to dev        (auto)
  ├─ deployed to staging    (auto after dev smoke test)
  └─ deployed to production (after approval)
```

Never tag with `:latest` for promotion. Use the commit SHA, a semantic version (`v1.4.2`), or both.

### 6. Caching for speed

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.nuget/packages
      **/bin
      **/obj
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj', '**/packages.lock.json') }}
```

Pair with a `packages.lock.json` (enable via `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>` in the csproj) for deterministic restores.

## Advantages

- **Repeatable releases** — the same artifact is tested and deployed, eliminating "works on my machine".
- **Faster feedback** — developers know within minutes if a commit breaks the build or tests.
- **Safer production changes** — automated gates (tests, scans, smoke tests, approvals) catch problems before users do.
- **Audit trail** — every deployment is traceable to a commit, a pipeline run, and an approver.
- **Easy rollback** — previous artifact versions are stored and can be redeployed in seconds.
- **Onboarding** — new engineers ship code on day one because the path to production is automated.
- **Forces good engineering** — you can't auto-deploy untested code, so tests become non-negotiable.

## Disadvantages

- **Upfront investment** — building a robust pipeline takes engineering time that doesn't ship features.
- **Pipeline maintenance** — YAML rots; SDK upgrades, deprecated tasks, and runner changes require ongoing care.
- **False sense of safety** — a green pipeline with weak tests still ships bugs.
- **Slow pipelines hurt productivity** — a 45-minute build discourages small commits and encourages people to skip CI.
- **Secret sprawl** — without OIDC and Key Vault, service principal secrets multiply across repos and rotate poorly.
- **Cost** — GitHub Actions and Azure DevOps charge per minute beyond free tiers; chatty pipelines add up.

## Common Mistakes

### 1. Building separately for each environment

```yaml
# BUG: rebuilds for prod with different config — the bits that pass tests aren't the bits that deploy
- run: dotnet publish -c Release --configuration Staging -o staging
- run: dotnet publish -c Release --configuration Production -o prod
```

**Fix**: build once, configure at runtime via `ASPNETCORE_ENVIRONMENT` and config providers. See [environment-configuration.md](environment-configuration.md).

```yaml
- run: dotnet publish -c Release -o publish    # one artifact for all environments
```

### 2. Tagging images as `:latest`

```yaml
docker build -t contosoacr.azurecr.io/order-api:latest .
docker push contosoacr.azurecr.io/order-api:latest
```

`:latest` is mutable — you can never reproduce "what was running yesterday" and rollback becomes ambiguous. **Always tag with the commit SHA or a semver tag.**

### 3. Storing service principal secrets in CI

```yaml
- run: az login --service-principal -u $SP_ID -p $SP_SECRET --tenant $TENANT
```

Long-lived secrets get leaked, forgotten, and never rotated. **Use OIDC federated identity** (`azure/login@v2` with `client-id` + federated credential configured on the app registration).

### 4. No tests gating the deploy

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: az webapp deploy ...    # no `needs: test`, no test step
```

CD without CI is just "push to prod faster". **Always make deploy jobs `needs: build-test`.**

### 5. Hard-coding connection strings in `azure-pipelines.yml`

```yaml
variables:
  SqlConnString: 'Server=...;User Id=admin;Password=P@ssw0rd!'
```

Anyone with read access to the repo now has prod credentials. **Use Key Vault-backed variable groups** and reference the variable by name only.

### 6. No artifact retention policy

CI runs daily for two years produce thousands of images. ACR storage adds up. **Configure ACR retention policies** (`az acr config retention update`) to auto-delete untagged images after 30 days, and keep only the last N versions of each tag.

### 7. Skipping smoke tests after deploy

```yaml
- run: az webapp deployment slot swap ...    # done, ship it
```

The swap succeeded, but the app may be returning 500s on every request. **Always hit `/health/ready` and a small set of business-critical endpoints** before declaring success.

```yaml
- run: |
    for i in {1..5}; do
      curl --fail https://order-api-staging.azurewebsites.net/health/ready && exit 0
      sleep 10
    done
    exit 1
```

## Best Practices

- **One artifact, many deployments.** Build once on `main`, promote the same image/zip through environments.
- **Pin tool versions.** `actions/setup-dotnet@v4` with `dotnet-version: '8.0.x'`. Don't rely on what the runner happens to have.
- **Enable concurrency control.** `concurrency: { group: prod-deploy, cancel-in-progress: false }` prevents two deploys racing into the same slot.
- **Use environments with required reviewers.** Both GitHub Actions and Azure DevOps support per-environment approval gates.
- **Cache aggressively.** NuGet packages, Docker layers, build outputs.
- **Fail fast.** Run unit tests before integration tests, integration before E2E. Static analysis before tests.
- **Scan images.** Integrate `trivy` or Microsoft Defender for Cloud's image scan in CI.
- **Tag with the git SHA** and optionally a semver. Use `dotnet-gitversion` for automatic semver from git history.
- **Keep secrets out of YAML.** Pull from Key Vault at runtime, not from CI variables.
- **Generate SBOMs.** `dotnet CycloneDX` or `sbom-tool` produces a software bill of materials for supply-chain compliance.
- **Test the pipeline itself.** Run it on a sample PR, validate the rollback path, document the on-call runbook.

## Related Concepts

- **[docker.md](docker.md)** — the artifact CI/CD typically produces and promotes.
- **[../azure/deployment-to-azure.md](../azure/deployment-to-azure.md)** — App Service, Container Apps, AKS targets.
- **[blue-green-deployment.md](blue-green-deployment.md)** — the deployment strategy CD uses for zero-downtime releases.
- **[rollback-strategy.md](rollback-strategy.md)** — what the pipeline does when the deploy fails.
- **[health-checks.md](health-checks.md)** — the smoke-test endpoints CD calls before declaring success.
- **[environment-configuration.md](environment-configuration.md)** — how the same artifact behaves differently per environment.
- **[../azure/azure-key-vault.md](../azure/azure-key-vault.md)** — where pipeline secrets live.
- **[../testing-quality/integration-testing.md](../testing-quality/integration-testing.md)** — the tests CI runs against a real database.

## Real-World Usage

**Scenario: an e-commerce platform on Azure App Service**

The team has three repos: `OrderApi`, `CatalogApi`, `CheckoutWeb`. Each has a `.github/workflows/cicd.yml` that:

1. On every PR: restore, build, run unit tests (~2,000 tests), run integration tests against a SQL container via Testcontainers, run `dotnet format --verify-no-changes`, post test results to the PR.
2. On merge to `main`: build a Docker image tagged with the commit SHA, push to `contosoacr.azurecr.io`, deploy to the `staging` slot, run a smoke-test script that hits `/health/ready` and `POST /orders` with a synthetic order.
3. **Manual approval** (GitHub Environment "production" requires a reviewer from the platform team) → swap the staging slot into production.
4. After swap: wait 5 minutes, query Application Insights for the error rate. If errors > 1%, automatically swap back.

Engineers ship roughly 15 times a day across the three services. The average time from `git push` to production is 22 minutes (about 8 minutes of build/test + 5 minutes in staging + 5 minutes of approval wait + 4 minutes for the swap + observability gate).

When a payment bug shipped at 3 PM on a Wednesday, the on-call engineer clicked "Re-run" on the previous successful workflow, which redeployed the prior image SHA via slot swap. Recovery took 4 minutes.

## Code Example — Before and After

### Before: manual, untracked, irreproducible

```text
# README.md — "How to deploy"
1. Open Visual Studio.
2. Right-click Order.Api → Publish → "prod-publish.pubxml".
3. Wait for the publish to finish.
4. RDP into order-api-prod-01.
5. Stop the IIS site.
6. Copy the contents of bin\Release\net8.0\publish to D:\sites\OrderApi.
7. Start the IIS site.
8. Open a browser, hit /, make sure it loads.
9. Tell #engineering on Teams.
```

Problems:
- Whoever's laptop builds it determines what ships.
- No record of what was deployed when, by whom, from which commit.
- "Make sure it loads" is not a health check.
- Rollback means re-running the same fragile process with an older branch.

### After: GitHub Actions with OIDC, slot swap, smoke test, auto-rollback

(See the [GitHub Actions workflow](#1-github-actions--typical-net-8-build--test--publish--deploy) above.)

Now:
- The exact image `contosoacr.azurecr.io/order-api:abc1234` is built once.
- The PR's CI status proves tests passed before the merge.
- Slot swap is atomic — production traffic is never served by a half-deployed instance.
- The smoke test verifies `/health/ready` before declaring success.
- An Application Insights query gates the post-swap monitoring window.
- Rollback is one click: re-run the previous successful workflow run.

## Interview Questions and Answers

### 1. What's the difference between Continuous Delivery and Continuous Deployment?

**Why this matters**: Tests whether the candidate understands the human-approval boundary and can pick the right model for a regulated environment.

**Answer**: Continuous **Delivery** means every commit is automatically built, tested, and made *deployable* to production — but a human still clicks "approve" before it actually goes live. Continuous **Deployment** removes that human gate: every green build flows all the way to production automatically.

**Trade-off**: Deployment ships faster (great for SaaS web apps), but Delivery is mandatory when you need four-eyes approval (PCI, HIPAA, SOC2) or when business stakeholders gate releases on marketing or training.

**Real project**: A B2B invoicing SaaS uses Continuous Delivery — the GitHub Action builds and deploys to staging automatically, but a manual approval on the GitHub `production` environment is required before slot swap. A consumer-facing marketing site on the same team uses Continuous Deployment because the blast radius of a regression is much smaller.

### 2. The deploy step uses `az login --service-principal -u ... -p $SECRET`. What would you change?

**Why this matters**: Long-lived secrets in CI are one of the most common supply-chain attack vectors.

**Answer**: Switch to **OIDC federated identity**. In Entra ID, configure a federated credential on the app registration that trusts the GitHub Actions OIDC issuer for that specific repo + branch + environment. Then in the workflow:

```yaml
permissions:
  id-token: write
  contents: read

steps:
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

No secret leaves Azure. GitHub mints a short-lived token, the federated trust accepts it, and `az login` succeeds. Nothing to rotate, nothing to leak.

**Real project**: After a security audit flagged 14 service principal secrets across pipelines, the platform team migrated everything to OIDC over two sprints. Secret rotation tickets went from monthly to zero.

### 3. Your pipeline takes 35 minutes. How do you speed it up?

**Answer**: A practical attack sequence:

1. **Measure first.** GitHub Actions and Azure DevOps both show per-step timing. Find the long poles before optimizing.
2. **Parallelize independent jobs.** Unit tests, integration tests, and image build can run as parallel jobs once restore is done.
3. **Cache NuGet packages.** `actions/cache@v4` keyed on `hashFiles('**/*.csproj')` typically saves 2-5 minutes.
4. **Cache Docker layers** via BuildKit (`docker buildx build --cache-from type=registry,ref=acr/cache --cache-to ...`).
5. **Skip unchanged projects.** For monorepos, use path filters (`paths:` in the workflow trigger) or tools like Nx.
6. **Run lighter tests on PR, heavy tests on main.** PR feedback should be under 10 minutes; full E2E suite can run on `main`.
7. **Use larger runners.** GitHub-hosted 4-core or self-hosted runners for heavy builds.
8. **Pre-warm Docker base images** on self-hosted runners.

**Trade-off**: Parallelism uses more concurrent minutes (cost); aggressive caching can hide stale-dependency bugs.

### 4. How do you handle EF Core migrations in CI/CD?

**Answer**: Treat migrations as a separate, idempotent pipeline step that runs **before** the new app version goes live, and that is **backward-compatible** with the previous version.

Two common approaches:

1. **CLI in pipeline**: `dotnet ef database update --connection $SqlConn` as a CD step. Simple, but the build agent needs network access to the DB.
2. **Bundle**: `dotnet ef migrations bundle -o efbundle` produces a self-contained executable that takes a connection string. Ship it as an artifact and run it from a deployment job inside the VNet.

The harder question is **schema compatibility**. Use the **expand-contract** pattern:

- *Expand*: deploy migrations that add new columns/tables but don't break old code.
- Deploy the new app version.
- Verify.
- *Contract*: deploy a follow-up migration that removes old columns.

This way, rollback (redeploy old image) still works because the schema is compatible with both versions.

**Real project**: A payment service split a `Customer.Name` column into `FirstName` and `LastName` over three releases — add columns, dual-write, deploy reader, backfill, swap reader, drop old column. Each release was independently rollback-safe.

### 5. The team wants to deploy on Friday afternoon. What guardrails make this safe?

**Why this matters**: "No Friday deploys" is a culture smell — it usually means the team doesn't trust their pipeline. The real answer is to make deploys boring.

**Answer**:

- **Small, frequent deploys** — the smaller the change, the easier the diagnosis.
- **Automated health gate** post-deploy — Application Insights query for error rate; if >1% over 5 minutes, auto-rollback (slot swap back).
- **Feature flags** for risky changes — ship the code dark, enable it Monday morning. See [../architecture/event-driven-architecture.md](../architecture/event-driven-architecture.md) for related patterns.
- **Observable monitoring on call** — Application Insights alerts route to the on-call engineer's phone.
- **Documented rollback runbook** — three commands, not a 30-step procedure.
- **Pipeline gates** — production environment requires approval from someone other than the author (four-eyes).

**Real project**: A team that previously banned Friday deploys moved to a model where any deploy was acceptable provided the post-deploy health gate was green. Mean time to recovery dropped from 45 minutes to 6 minutes.

### 6. Your `dotnet test` step passes locally but fails in CI. How do you debug?

**Answer**: Common causes, in order of likelihood:

1. **Time zone or culture differences** — CI runners often default to UTC and `en-US`; local dev might be Vietnam time and `vi-VN`. Tests that compare formatted dates or numbers break.
2. **Case-sensitive file systems** — `Customer.cs` vs `customer.cs` on Linux CI vs Windows dev. Use case-correct paths.
3. **Missing environment variables** — the test reads `OPENAI_API_KEY` from env which exists locally but not in CI.
4. **Concurrency / ordering bugs** — local runs serial, CI runs parallel.
5. **External dependencies** — test hits a local SQL Server. Fix by using Testcontainers or in-memory providers.
6. **Different SDK version** — local `8.0.404`, CI uses `8.0.x` resolving to `8.0.405` with a behavior change.

Tactics: rerun with `--logger "console;verbosity=detailed"`, upload `TestResults/*.trx` as an artifact, run a single failing test with `dotnet test --filter "FullyQualifiedName~MyTest"`, SSH into a self-hosted runner if available.

### 7. Why is it bad to use `:latest` for Docker image tags in production?

**Answer**: `:latest` is a mutable pointer — `acr/order-api:latest` means "whatever was pushed most recently". This creates three concrete problems:

1. **No reproducibility.** "What was running last Tuesday at 3 PM?" There's no answer.
2. **Ambiguous rollback.** Redeploying `:latest` redeploys the broken version that caused the incident.
3. **Race conditions.** A CI job pushing `:latest` at the same time as another deploy can produce a node running version X while the next pod scheduled runs version Y.

**Fix**: tag with the immutable commit SHA (`acr/order-api:abc1234`) and optionally a semver (`v1.4.2`). Promote by *tag*, never by floating reference. Keep `:latest` only as a convenience alias for local dev (`docker pull acr/order-api:latest`).

### 8. How do you secure secrets used in a CI pipeline?

**Answer**: Layered defenses:

1. **Don't store them in CI at all when possible.** Use **OIDC federated identity** to authenticate to Azure, then have the app read secrets from **Key Vault** at runtime via `Azure.Identity.DefaultAzureCredential`.
2. **For secrets that must live in CI** (third-party API keys, signing certificates): store as **environment secrets** (GitHub) or **variable groups linked to Key Vault** (Azure DevOps). Never in plain YAML.
3. **Scope tightly.** GitHub Environment secrets are gated by environment approval — `production` secrets are only readable in jobs targeting that environment.
4. **Rotate regularly.** Even with vaults, treat any secret older than 90 days as overdue.
5. **Scan for leaks.** Enable GitHub secret scanning, run `trufflehog` or `gitleaks` in CI on every PR.
6. **Audit access.** Periodically review who can read pipeline secrets and remove orphaned principals.

**Real project**: A team enabled GitHub Advanced Security secret scanning and discovered three pushed PRs (over two years) had committed an API key. They rotated the keys and added a `pre-commit` hook with `gitleaks` to prevent recurrence.

## Summary Checklist

- [ ] I can explain the difference between CI and CD and pick the right model for a regulated vs. a SaaS context.
- [ ] I can write a GitHub Actions workflow that builds, tests, publishes a Docker image, and deploys to App Service.
- [ ] I can write a multi-stage Azure DevOps YAML pipeline with environments and approvals.
- [ ] I can explain why "build once, deploy many times" matters and what breaks when you don't follow it.
- [ ] I can configure OIDC federated identity instead of long-lived service principal secrets.
- [ ] I can pin tool versions, cache NuGet packages, and parallelize jobs to keep pipelines fast.
- [ ] I can describe artifact promotion using immutable tags (SHA, semver) and explain why `:latest` is dangerous.
- [ ] I can integrate EF Core migrations with the expand-contract pattern so rollback stays safe.
- [ ] I can configure post-deploy smoke tests and auto-rollback gates.
- [ ] I can handle pipeline secrets via Key Vault and environment-scoped GitHub secrets, and audit them regularly.
