# Docker

## What It Is

**Docker** is a container platform that packages an application together with everything it needs to run — the .NET runtime, OS-level libraries, configuration files, and the published binaries — into a single immutable **image**. That image is then started as a **container**: an isolated process tree with its own file system, network namespace, and resource limits.

The three core concepts you must internalize:

- **Image** — a read-only, layered, content-addressable filesystem snapshot built from a `Dockerfile`. Tagged with a registry, repository, and tag (e.g., `contosoacr.azurecr.io/order-api:abc1234`).
- **Container** — a running instance of an image. Multiple containers can run from the same image. They share the host kernel but are isolated from each other.
- **Registry** — a network-addressable image store. The Microsoft-recommended registry on Azure is **Azure Container Registry (ACR)**. Public registries include Docker Hub and `mcr.microsoft.com` (Microsoft Container Registry).

For .NET 8 / 9 the official base images live on `mcr.microsoft.com/dotnet/*`:

| Image | Purpose | Size (~) |
|---|---|---|
| `mcr.microsoft.com/dotnet/sdk:8.0` | Build stage — has SDK + CLI | ~800 MB |
| `mcr.microsoft.com/dotnet/aspnet:8.0` | Runtime — ASP.NET Core only | ~220 MB |
| `mcr.microsoft.com/dotnet/runtime:8.0` | Runtime — console apps / workers | ~190 MB |
| `mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled` | Minimal Ubuntu Chiseled (no shell, no package manager) | ~115 MB |
| `mcr.microsoft.com/dotnet/aspnet:9.0` | .NET 9 ASP.NET runtime | ~220 MB |

You build images with `docker build`, push to a registry with `docker push`, and run them locally with `docker run` or in production via App Service Containers, Container Apps, or AKS.

## Why It Exists

Before containers, .NET deployment meant publishing to a server with a specific .NET Framework version, a specific IIS configuration, and specific OS patches. The pain pattern:

- A new server in the farm had a slightly different IIS module installed → mysterious 502s.
- The team upgraded to .NET 6 in dev, but ops hadn't installed the runtime on the staging boxes → deployment failed at midnight.
- The build worked on a developer's Windows 10 laptop but not on the Windows Server 2019 build agent because of a missing C++ redist.

Docker solves "works on my machine" by **shipping the machine**. The image *is* the runtime. If `mcr.microsoft.com/dotnet/aspnet:8.0` runs your app in the test environment, the bit-for-bit identical image will run it in production.

It also exists to enable:
- **Horizontal scaling** — start 50 containers from one image in seconds (essential for Kubernetes and Azure Container Apps).
- **Process isolation** — one misbehaving service can't crash a neighbor on the same node.
- **Immutable infrastructure** — you never patch a running container; you build a new image and replace it.
- **Polyglot deployment** — the same orchestrator runs .NET, Node, Python, Go services without per-language tooling on the hosts.

## When To Use It

**Use Docker for:**

- Any service deployed to **AKS** or **Azure Container Apps** — Kubernetes requires container images.
- ASP.NET Core APIs deployed to **App Service for Containers** (the modern alternative to in-process App Service).
- Background workers (`BackgroundService`) where you want consistent runtime across environments.
- Local development with `docker compose` to spin up SQL Server, Redis, Azurite, and your API together.
- Integration tests using **Testcontainers** (`Testcontainers.MsSql`, `Testcontainers.Redis`) so tests get a fresh, real database every run.
- CI/CD — the build artifact promoted across environments is a tagged image.

**Do not use Docker for:**

- A pure **Azure Functions** project on the Consumption plan — the platform manages the runtime; containers add complexity for no benefit (unless you specifically need a custom container).
- A Windows desktop app or a WinForms tool — containers are designed for server workloads.
- A small in-process App Service site with no scale requirements where in-process deployment is simpler.
- Static websites — use **Azure Static Web Apps** or a CDN.

## Why It Is Important

For a senior .NET engineer, Docker is not a "DevOps thing" — it's a runtime contract you own.

1. **Reproducible production** — the image you tested in staging is byte-identical to the image in production. Bug reports become reproducible.
2. **Supply-chain security** — every package, every transitive runtime dependency is pinned in the image. Image scanning (Trivy, Defender for Cloud) detects CVEs before deploy.
3. **Fast horizontal scaling** — Container Apps and AKS can scale from 3 to 300 pods in seconds because the unit of deployment is an immutable image, not a VM.
4. **Simplified rollback** — redeploy the previous tag (`order-api:abc1234`). No re-publish, no slot fiddling, no DLL hunting.
5. **Developer parity** — a new hire runs `docker compose up` and gets the full local stack in 60 seconds.

## How It's Used in C# / .NET

### 1. Multi-stage Dockerfile for ASP.NET Core 8

`src/Order.Api/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7

# ---------- Build stage ----------
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy only project files first so NuGet restore is cached separately from source changes
COPY ["Order.sln", "./"]
COPY ["src/Order.Api/Order.Api.csproj", "src/Order.Api/"]
COPY ["src/Order.Application/Order.Application.csproj", "src/Order.Application/"]
COPY ["src/Order.Domain/Order.Domain.csproj", "src/Order.Domain/"]
COPY ["src/Order.Infrastructure/Order.Infrastructure.csproj", "src/Order.Infrastructure/"]

RUN dotnet restore "src/Order.Api/Order.Api.csproj"

# Now copy the rest of the source and publish
COPY . .
RUN dotnet publish "src/Order.Api/Order.Api.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# ---------- Runtime stage ----------
FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled AS runtime
WORKDIR /app

# Create a non-root user (chiseled images run as non-root by default — uid 1654)
# Copy published output
COPY --from=build /app/publish .

# Environment defaults — overridden per environment at runtime
ENV ASPNETCORE_ENVIRONMENT=Production \
    ASPNETCORE_URLS=http://+:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_USE_POLLING_FILE_WATCHER=false \
    Logging__Console__FormatterName=json

EXPOSE 8080

ENTRYPOINT ["dotnet", "Order.Api.dll"]
```

Key design choices:

- **Multi-stage build** — the heavy SDK image (~800 MB) is discarded; only the slim runtime image is published.
- **Layer caching** — copying `.csproj` files and running `restore` *before* copying source means edits to `.cs` files don't bust the NuGet cache.
- **Chiseled base image** — no shell, no `apt`, no package manager. Smaller attack surface, smaller image (~115 MB).
- **Non-root user** — chiseled images run as UID 1654 by default. Even non-chiseled `aspnet:8.0` images now default to a non-root `app` user in .NET 8+.
- **JSON logging** — production-ready log format for Application Insights / Log Analytics ingestion.
- **Port 8080** — the new default for non-root containers (port 80 requires CAP_NET_BIND_SERVICE).

### 2. `.dockerignore`

Always ship one. Without it, your local `bin/`, `obj/`, `node_modules/`, and `.git/` get copied into the build context (and possibly into the image), inflating size and leaking secrets:

```dockerignore
**/bin
**/obj
**/.vs
**/.vscode
**/.idea
**/node_modules
**/TestResults
**/coverage
.git
.gitignore
.github
.dockerignore
**/*.user
**/*.suo
**/appsettings.*.Development.json
**/secrets.json
README.md
docs/
```

### 3. Building and tagging

```bash
# Build
docker build -t contosoacr.azurecr.io/order-api:abc1234 -f src/Order.Api/Dockerfile .

# Run locally with environment overrides
docker run --rm -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Development \
  -e ConnectionStrings__OrdersDb="Server=host.docker.internal;..." \
  contosoacr.azurecr.io/order-api:abc1234

# Push to ACR
az acr login --name contosoacr
docker push contosoacr.azurecr.io/order-api:abc1234
```

### 4. `docker compose` for local dev

`docker-compose.yml`:

```yaml
services:
  api:
    build:
      context: .
      dockerfile: src/Order.Api/Dockerfile
    ports: [ "8080:8080" ]
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__OrdersDb: "Server=sql,1433;Database=Orders;User Id=sa;Password=Local!Dev123;TrustServerCertificate=true"
      Redis__ConnectionString: "redis:6379"
      ServiceBus__ConnectionString: "Endpoint=sb://localhost..."
    depends_on:
      sql: { condition: service_healthy }
      redis: { condition: service_started }

  sql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "Local!Dev123"
    ports: [ "1433:1433" ]
    volumes: [ "sqldata:/var/opt/mssql" ]
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'Local!Dev123' -C -Q 'SELECT 1' || exit 1"]
      interval: 10s
      retries: 10

  redis:
    image: redis:7-alpine
    ports: [ "6379:6379" ]

volumes:
  sqldata:
```

`docker compose up` brings up the API plus SQL Server plus Redis with one command. New hires are productive in minutes.

### 5. Image scanning in CI

Add a scan step to GitHub Actions:

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: contosoacr.azurecr.io/order-api:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'    # fail the build on HIGH/CRITICAL

- name: Upload to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

ACR also offers integrated scanning via **Microsoft Defender for Cloud** — every pushed image is scanned automatically and results show up in Defender.

### 6. Targeted base image selection

| Use case | Recommended base |
|---|---|
| Standard ASP.NET Core API | `mcr.microsoft.com/dotnet/aspnet:8.0` |
| Minimum size + max security | `mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled` |
| Console worker / `BackgroundService` host only | `mcr.microsoft.com/dotnet/runtime:8.0` |
| Self-contained native AOT executable | `mcr.microsoft.com/dotnet/runtime-deps:8.0-jammy-chiseled-extra` |
| Windows-only legacy dependency | `mcr.microsoft.com/dotnet/aspnet:8.0-windowsservercore-ltsc2022` |

**Avoid** `latest` tags in production Dockerfiles — pin to `8.0` or `8.0.11` so a Microsoft base-image update doesn't silently change behavior.

## Advantages

- **Environment parity** — dev, CI, staging, prod all run the same image.
- **Fast startup and horizontal scale** — containers start in seconds, perfect for autoscaling.
- **Process isolation** — one container's resource leak doesn't crash the host or siblings.
- **Reproducible builds** — given the same Dockerfile and base image digest, the output is deterministic.
- **Tooling ecosystem** — Docker Desktop, Compose, BuildKit, Trivy, Defender for Cloud, ACR Tasks all integrate.
- **Polyglot deployment** — same orchestrator runs your .NET service, Node frontend, Python ML worker.
- **Decoupled from host OS** — upgrade the host kernel without touching app images (and vice versa).

## Disadvantages

- **Learning curve** — Dockerfiles, layers, build context, registries, CLI flags add concepts a new dev must master.
- **Image bloat is easy** — a careless Dockerfile produces 2 GB images; chiseled images and multi-stage builds require discipline.
- **Networking complexity** — `host.docker.internal`, bridge networks, port publishing, and DNS quirks confuse beginners.
- **Persistent state needs care** — containers are ephemeral; database state must live in volumes or external services.
- **Build context size matters** — a 5 GB build context (forgotten `.dockerignore`) slows every build dramatically.
- **Licensing** — Docker Desktop requires a paid license for organizations >250 employees or >$10M revenue (consider Podman, Rancher Desktop, or `dotnet publish /t:PublishContainer`).

## Common Mistakes

### 1. Running as root

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
COPY . /app
ENTRYPOINT ["dotnet", "/app/Order.Api.dll"]
# implicitly runs as root
```

A compromised container with root inside has CAP_NET_RAW, CAP_NET_ADMIN, etc., dramatically widening blast radius.

**Fix**: use a chiseled image (already non-root) or create a user explicitly:

```dockerfile
RUN groupadd -r app && useradd -r -g app -u 10001 app
USER 10001
```

### 2. No `.dockerignore`

The build context ships `bin/`, `obj/`, `.git/`, and `appsettings.Development.json` (which often contains real dev secrets) into the build. The image grows, and secrets leak into layers — visible to anyone with image pull access.

**Fix**: always include the `.dockerignore` from §3 above.

### 3. Embedding secrets in the image

```dockerfile
ENV ConnectionStrings__OrdersDb="Server=...;Password=P@ssw0rd!"
```

Secrets baked into image layers are permanent and visible via `docker history`. Anyone with pull access can extract them.

**Fix**: inject secrets at **runtime** via environment variables (App Service Configuration, Container Apps secrets, Kubernetes Secrets, Key Vault CSI driver). The image stays generic.

### 4. Single-stage build with the SDK image

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
COPY . .
RUN dotnet publish -c Release -o /app
ENTRYPOINT ["dotnet", "/app/Order.Api.dll"]
```

You ship the entire .NET SDK (~800 MB) plus your app to every node. Pull time, registry storage, and attack surface all explode.

**Fix**: multi-stage build (SDK to build, ASP.NET runtime to run).

### 5. Putting `COPY . .` before `dotnet restore`

```dockerfile
COPY . /src
WORKDIR /src
RUN dotnet restore
RUN dotnet publish -c Release -o /app
```

Every source-only edit busts the restore cache, adding minutes to each build.

**Fix**: copy `.csproj`/`.sln` first, restore, *then* copy the rest (as in §1).

### 6. Hard-coding the image tag as `:latest`

```bash
docker build -t contosoacr.azurecr.io/order-api:latest .
docker push contosoacr.azurecr.io/order-api:latest
```

You lose the ability to reproduce or roll back. See [ci-cd-pipelines.md](ci-cd-pipelines.md) for details.

**Fix**: tag with the commit SHA and optionally a semver: `order-api:abc1234`, `order-api:v1.4.2`.

### 7. Forgetting `HEALTHCHECK` or not exposing health endpoints

A container that returns 500 on every request keeps running because Docker doesn't know it's sick. Orchestrators (Kubernetes, Container Apps) can detect this via HTTP probes — but only if your app exposes `/health/live` and `/health/ready`. See [health-checks.md](health-checks.md).

### 8. Ignoring `dotnet publish /t:PublishContainer`

Modern .NET 8+ can build images **without a Dockerfile** using SDK container support:

```bash
dotnet publish -c Release /t:PublishContainer \
  /p:ContainerRegistry=contosoacr.azurecr.io \
  /p:ContainerRepository=order-api \
  /p:ContainerImageTag=abc1234
```

For straightforward services this is simpler than maintaining a Dockerfile. The Dockerfile is still preferable for complex builds, custom base images, or non-.NET steps.

## Best Practices

- **Multi-stage builds always.** Build with `sdk`, run with `aspnet`.
- **Pin base image versions** (`aspnet:8.0` minimum, ideally `aspnet:8.0.11-jammy-chiseled`).
- **Use chiseled images in production** when possible — smaller, no shell, fewer CVEs.
- **Tag with the commit SHA** (`order-api:abc1234`), never `:latest` for promotion.
- **Run as a non-root user.** UID >= 10000 conventionally avoids host UID conflicts.
- **Ship a `.dockerignore`** — first thing in every new repo.
- **Order Dockerfile layers from least to most volatile** — copy `csproj`, restore, then copy source.
- **Set `DOTNET_RUNNING_IN_CONTAINER=true`** so the runtime applies container-aware tuning (GC, thread pool).
- **Configure JSON logging** (`Logging__Console__FormatterName=json`) so log aggregators parse structured fields.
- **Set resource limits** at runtime (`--memory=512m --cpus=1`) and matching `Resources.Requests/Limits` in Kubernetes.
- **Scan every image** with Trivy or Defender for Cloud as a CI gate.
- **Generate an SBOM** (`dotnet CycloneDX` or BuildKit's `--provenance`) for supply-chain compliance.
- **Use BuildKit caching** (`docker buildx build --cache-to type=registry,ref=acr/cache --cache-from type=registry,ref=acr/cache`) in CI.

## Related Concepts

- **[kubernetes-basics.md](kubernetes-basics.md)** — the orchestrator that runs Docker images in production at scale.
- **[ci-cd-pipelines.md](ci-cd-pipelines.md)** — where images are built and tagged.
- **[health-checks.md](health-checks.md)** — the endpoints orchestrators probe to know if a container is healthy.
- **[environment-configuration.md](environment-configuration.md)** — how the same image behaves differently per environment.
- **[blue-green-deployment.md](blue-green-deployment.md)** — swapping container images via App Service slots or K8s services.
- **[../azure/deployment-to-azure.md](../azure/deployment-to-azure.md)** — App Service for Containers, Container Apps, AKS as image hosts.
- **[../azure/azure-key-vault.md](../azure/azure-key-vault.md)** — where runtime secrets come from (not from the image).
- **[../testing-quality/integration-testing.md](../testing-quality/integration-testing.md)** — Testcontainers for spinning real dependencies in tests.

## Real-World Usage

**Scenario: payment processing service on Azure Container Apps**

A payments team operates `payment-api` and `payment-worker` as two services. Both are containerized:

1. CI builds `contosoacr.azurecr.io/payment-api:${commit_sha}` from a multi-stage Dockerfile using the chiseled base image (~115 MB).
2. Trivy scans the image; any HIGH or CRITICAL CVE blocks the merge.
3. Defender for Cloud rescans every 24h and posts findings to the security team's Teams channel.
4. CD deploys to `payment-api-staging` (a separate Container Apps revision), runs synthetic transactions for 10 minutes.
5. After approval, the new revision becomes the active one (100% traffic). Container Apps keeps the old revision warm for 1 hour for instant rollback.
6. Locally, developers run `docker compose up` to get SQL Server, Azurite (Azure Storage emulator), and the API in parallel.
7. Integration tests in CI use Testcontainers to spin a fresh SQL Server per test class, eliminating shared-DB flakiness.

The image rebuild + scan + publish step takes under 4 minutes. Cold-start on Container Apps is ~2.5 seconds thanks to the chiseled image size and `DOTNET_RUNNING_IN_CONTAINER` GC tuning.

## Code Example — Before and After

### Before: bloated, insecure, slow

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0
WORKDIR /app
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o out
ENV ConnectionStrings__OrdersDb="Server=db.contoso.com;User Id=sa;Password=Prod!2024;"
EXPOSE 80
ENTRYPOINT ["dotnet", "out/Order.Api.dll"]
```

Problems:
- Single-stage build — ships the 800 MB SDK to every node.
- `COPY . .` before restore — every code change re-restores NuGet.
- Runs as **root** — full container capabilities, wide blast radius.
- Production **password baked into the image** — visible in `docker history`, leaks to anyone with pull access.
- No `.dockerignore` — `bin/`, `obj/`, `.git/`, `appsettings.Development.json` ship with the image.
- Listens on port 80 — requires root or `CAP_NET_BIND_SERVICE`.
- Base image pinned only to `6.0` (already out of support after Nov 2024).

### After: small, secure, fast, deterministic

```dockerfile
# syntax=docker/dockerfile:1.7

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["Order.sln", "./"]
COPY ["src/Order.Api/Order.Api.csproj", "src/Order.Api/"]
COPY ["src/Order.Application/Order.Application.csproj", "src/Order.Application/"]
COPY ["src/Order.Domain/Order.Domain.csproj", "src/Order.Domain/"]
COPY ["src/Order.Infrastructure/Order.Infrastructure.csproj", "src/Order.Infrastructure/"]
RUN dotnet restore "src/Order.Api/Order.Api.csproj"

COPY . .
RUN dotnet publish "src/Order.Api/Order.Api.csproj" \
    -c Release -o /app/publish --no-restore /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled AS runtime
WORKDIR /app
COPY --from=build /app/publish .

ENV ASPNETCORE_ENVIRONMENT=Production \
    ASPNETCORE_URLS=http://+:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true \
    Logging__Console__FormatterName=json

EXPOSE 8080
ENTRYPOINT ["dotnet", "Order.Api.dll"]
```

With `.dockerignore`:

```dockerignore
**/bin
**/obj
**/.vs
**/.vscode
**/node_modules
**/TestResults
.git
.github
**/appsettings.Development.json
**/secrets.json
```

Now:
- Final image is ~140 MB (versus ~900 MB before).
- Runs as non-root UID 1654 (chiseled default).
- No secrets in the image — injected at runtime by App Service Configuration / Key Vault.
- Layer caching saves 60-80 seconds per CI build when only source changes.
- Listens on port 8080 — no elevated capabilities.
- Pinned to a supported LTS base.

## Interview Questions and Answers

### 1. Walk me through a multi-stage Dockerfile for an ASP.NET Core service.

**Why this matters**: Reveals whether the candidate understands layer caching, image size, and the SDK-vs-runtime split.

**Answer**: There are two stages. The **build stage** uses `mcr.microsoft.com/dotnet/sdk:8.0`, copies `.csproj`/`.sln` first, runs `dotnet restore`, then copies the rest of the source and runs `dotnet publish -c Release -o /app/publish`. The **runtime stage** starts from `mcr.microsoft.com/dotnet/aspnet:8.0` (or the chiseled variant) and uses `COPY --from=build /app/publish .` to bring only the published output. The final image contains the slim runtime, the published bits, and nothing else — typically 140-220 MB.

**Trade-off**: The build is slightly more complex than a single stage, but the resulting image is 4-5x smaller, has a much smaller attack surface, and pulls/starts faster.

**Real project**: A team migrating from single-stage Dockerfiles to multi-stage chiseled images reduced their AKS image pull time from 18 seconds to under 3 seconds, materially improving HPA scale-up responsiveness during traffic spikes.

### 2. The order API container starts but crashes immediately. How do you debug?

**Answer**: My step-by-step:

1. `docker logs <container-id>` (or `kubectl logs <pod>`) — read the actual exception. .NET startup failures are usually `Microsoft.Data.SqlClient.SqlException` (DB unreachable), `Azure.RequestFailedException` (Key Vault auth), or `OptionsValidationException` (missing config).
2. `docker run --rm -it --entrypoint /bin/bash <image>` — only works for non-chiseled images. For chiseled, build a debug variant or `docker run --rm <image> dotnet --info`.
3. Check environment variables — `docker inspect <container> | jq .[0].Config.Env`. Most "works locally, fails in container" issues are missing or wrong env vars.
4. Verify the image was built correctly: `docker history <image>` and `docker run --rm <image> ls /app` to confirm files exist.
5. Check the connection from inside the container to dependencies: `docker run --rm --network=app_net <image> dotnet -e 'TcpClient...'`.
6. Look at container exit code — `137` is OOM (raise memory limit), `139` is segfault (rare in .NET), non-zero from app is usually unhandled exception.

**Real project**: A `payment-api` pod crashed in AKS because the Workload Identity federated token wasn't being picked up — the fix was adding the `azure.workload.identity/use: "true"` pod label.

### 3. Why are chiseled images more secure than the standard Ubuntu base?

**Answer**: Chiseled images (`mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled`) contain **only** the files the .NET runtime needs to execute. There is:

- **No shell** (`bash`, `sh`) — attackers can't `exec` into the container and explore.
- **No package manager** (`apt`, `dnf`) — no way to install tools post-compromise.
- **No setuid binaries** — no `sudo`, no privilege escalation paths.
- **No CA certs beyond what .NET needs.**
- **Default non-root user** (UID 1654).

The image is also dramatically smaller (~115 MB vs ~220 MB), which means fewer transitive CVEs to track and faster pull/start times. The trade-off is debugging: you can't `kubectl exec -it pod -- bash` because there's no bash. You debug via logs, structured telemetry, and rebuilding with a debug variant if needed.

### 4. How do you keep image sizes small?

**Answer**:

1. **Multi-stage build** — discard the SDK.
2. **Chiseled or alpine base** — `aspnet:8.0-jammy-chiseled` over `aspnet:8.0`.
3. **`.dockerignore`** — exclude `bin/`, `obj/`, tests, docs.
4. **`/p:UseAppHost=false`** — skip generating the native executable wrapper if you launch with `dotnet`.
5. **AOT compilation** for small services that don't need reflection.
6. **One layer per logical step** — combine `RUN` commands with `&&` to avoid intermediate layers when modifying files.
7. **Trim unused assemblies** — `dotnet publish -p:PublishTrimmed=true` for non-trim-warning code.
8. **Don't install dev tools in the runtime image** — `curl`, `git`, debuggers belong in build, not runtime.

**Real project**: A microservice went from 1.1 GB to 95 MB by switching to multi-stage + chiseled + `.dockerignore`. ACR storage costs dropped, and pod cold-start time fell from ~12 s to ~2.5 s.

### 5. When would you use `docker compose` vs Kubernetes?

**Why this matters**: Tests whether the candidate picks the right tool for the scale of problem.

**Answer**:

- **`docker compose`**: local development, integration tests, demo environments, single-VM deployments of small internal tools. Limit: one host, no autoscaling, no rolling updates.
- **Kubernetes (AKS)**: production multi-replica services with autoscaling (HPA), rolling updates, service discovery, multi-tenant clusters, complex network policies.

There's a middle ground: **Azure Container Apps**. It runs containers with autoscaling, revisions, and Dapr support without exposing raw Kubernetes — perfect for teams that need scale and rolling updates but don't want to operate AKS.

**Trade-off**: AKS gives full control and is industry-standard, but the operational burden is real (upgrades, certificates, network policies, RBAC). Container Apps is faster to adopt but less flexible.

### 6. The team wants production secrets in the image so deployment is "simpler". What do you say?

**Answer**: Hard no. Once a secret is in an image layer it's there forever — `docker history` reveals it, anyone with pull access to ACR can extract it, and rotating means rebuilding and redeploying every dependent service. We also can't share the image across environments because the dev secret would ship to prod.

The right answer is to keep the image **generic** and inject secrets at runtime:

- **App Service**: App Settings + Key Vault references (`@Microsoft.KeyVault(SecretUri=...)`).
- **Container Apps**: `Secrets` block in the revision config + Key Vault.
- **AKS**: Kubernetes Secrets (preferably from Key Vault via the Secrets Store CSI Driver) + Workload Identity.
- **Local dev**: `dotnet user-secrets` and environment variables in `docker-compose.override.yml`.

The app reads via `IConfiguration` — same code path everywhere. See [environment-configuration.md](environment-configuration.md) and [../azure/azure-key-vault.md](../azure/azure-key-vault.md).

### 7. What does `EXPOSE 8080` in a Dockerfile actually do?

**Answer**: It's metadata — a declaration that the container listens on port 8080. It does **not** publish the port to the host or open a firewall hole. Publishing happens at `docker run -p 80:8080` or via the orchestrator's service definition (Kubernetes `Service`, App Service container settings).

`EXPOSE` matters for documentation and for tools like `docker run -P` (uppercase P) which auto-publishes all `EXPOSE`d ports to random host ports. Orchestrators ignore it.

The change from port 80 → 8080 in .NET 8+ container images happened because non-root containers (the new default) can't bind to ports < 1024 without `CAP_NET_BIND_SERVICE`.

### 8. How do you handle EF Core migrations in a containerized deployment?

**Answer**: Three viable patterns:

1. **Migration bundle as a separate container/job.** Generate `dotnet ef migrations bundle` in CI, ship the bundle as its own tiny image, and run it as a Kubernetes Job or Container Apps Job before the new app version deploys. Idempotent, observable, cancellable.
2. **App applies migrations on startup.** Convenient but dangerous in multi-replica deployments — all replicas race to migrate, lock the schema, and one wins. Only safe with `MigrateAsync` + advisory locks or `if (HostEnvironment.IsDevelopment())`.
3. **Run migrations from the CD pipeline.** A pipeline step executes `dotnet ef database update` against the target environment's SQL connection (using a managed identity). Simple and centrally controlled, but requires network access from the build agent to the DB.

In all cases, use the **expand-contract** pattern so migrations are backward-compatible with the previous app version — this keeps rollback safe. See [rollback-strategy.md](rollback-strategy.md) and [../data-access/migrations.md](../data-access/migrations.md).

## Summary Checklist

- [ ] I can write a multi-stage Dockerfile for ASP.NET Core 8/9 with proper layer caching.
- [ ] I can explain why we copy `.csproj` files before `dotnet restore`.
- [ ] I can pick the right base image (`sdk`, `aspnet`, `runtime`, `chiseled`) for the use case.
- [ ] I can write a `.dockerignore` that excludes build outputs and developer secrets.
- [ ] I can configure non-root execution and explain why it matters.
- [ ] I can use `docker compose` to bring up an API plus SQL Server plus Redis locally.
- [ ] I can integrate Trivy or Defender for Cloud image scanning in CI.
- [ ] I can debug a crashing container using `docker logs`, `docker inspect`, and exit codes.
- [ ] I can explain why secrets must never be baked into images and where they should live instead.
- [ ] I can pick between `docker compose`, App Service for Containers, Container Apps, and AKS based on requirements.
