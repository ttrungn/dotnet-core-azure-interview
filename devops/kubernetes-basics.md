# Kubernetes Basics

## What It Is

**Kubernetes** (k8s) is a container orchestrator. You hand it a desired state ("I want 5 replicas of `order-api:v1.4.2` listening on port 8080, with 512 MB memory each, behind an HTTPS load balancer") expressed as YAML, and Kubernetes makes the cluster match that state — scheduling pods onto nodes, replacing them when they crash, rolling new versions out gradually, scaling horizontally on demand, and routing traffic.

In Azure the managed offering is **AKS** (Azure Kubernetes Service). Microsoft operates the control plane (API server, scheduler, controller-manager, etcd); you operate the node pools, the workloads, and the application config.

The core objects every .NET engineer should know:

| Object | What it is |
|---|---|
| **Pod** | One or more containers sharing a network and storage. The smallest deployable unit. Pods are usually managed by Deployments — you rarely create them directly. |
| **Deployment** | Declares "I want N replicas of this Pod template" and handles rolling updates. |
| **ReplicaSet** | The intermediate object a Deployment uses to track replicas (auto-managed). |
| **Service** | Stable virtual IP/DNS name (`order-api.production.svc.cluster.local`) that load-balances across the Pods matching a label selector. |
| **Ingress** | Routes external HTTP(S) traffic by host/path to Services. Backed by an ingress controller (NGINX, Application Gateway Ingress Controller). |
| **ConfigMap** | Non-secret key/value config injected as env vars or files. |
| **Secret** | Like ConfigMap but for sensitive values (base64-encoded; treat as protected). |
| **HorizontalPodAutoscaler (HPA)** | Auto-scales the number of Pods in a Deployment based on CPU, memory, or custom metrics. |
| **Namespace** | Logical partition of the cluster (`production`, `staging`, `team-payments`). |
| **Job / CronJob** | Run-to-completion workloads (DB migrations, batch processing). |

All control is via `kubectl` (e.g., `kubectl apply -f deployment.yaml`, `kubectl get pods`, `kubectl rollout status`) talking to the cluster's API server.

## Why It Exists

Containers solved "works on my machine" but introduced new problems at scale: which VM runs which container, how do you replace one that crashes, how does a payment-api pod find a fresh-just-started catalog-api pod, how do you do a zero-downtime release across 20 instances, how do you scale up when Black Friday traffic hits?

Before Kubernetes, teams hand-built this with custom scripts, load balancers, and on-call pages. Kubernetes standardizes the answers:

- **Scheduling** — given a pod's requirements (1 CPU, 512 MB), find a node with capacity.
- **Self-healing** — if a pod's liveness probe fails, kill it and let the Deployment recreate it.
- **Service discovery** — pods get a DNS name; you call `http://catalog-api/products` and the cluster routes to a healthy replica.
- **Rolling updates** — replace pods one at a time, wait for each to be ready, abort if too many fail.
- **Autoscaling** — HPA adds pods when CPU exceeds 70%; cluster autoscaler adds nodes when pods can't be scheduled.
- **Declarative state** — your YAML in git *is* the production config. Drift gets reconciled automatically.

Kubernetes is the de facto standard because every cloud (Azure AKS, AWS EKS, GCP GKE) implements the same API. Workloads move between clouds with minimal change.

## When To Use It

**Use Kubernetes / AKS for:**

- Microservices architectures with 5+ services that need orchestration.
- Workloads that need autoscaling beyond what App Service Plans handle gracefully.
- Multi-team platforms where namespaces + RBAC isolate workloads.
- Services with custom networking needs (network policies, mTLS via service mesh).
- Workloads requiring sidecars (Dapr, OpenTelemetry collectors, log shippers).
- Multi-cloud or hybrid deployments where workloads must be portable.

**Do not use Kubernetes for:**

- A single ASP.NET Core API — **App Service** or **Container Apps** is dramatically simpler.
- A pure event-driven workload — **Azure Functions** or **Container Apps Jobs** is a better fit.
- Static websites — **Azure Static Web Apps**.
- Small teams without platform engineering capacity — operating AKS is a real job.
- Workloads where a managed PaaS already exists — don't deploy SQL Server on Kubernetes when Azure SQL is one ARM template away.

The hierarchy for .NET workloads on Azure: **App Service → Container Apps → AKS**. Move down the list only when you have a concrete reason.

## Why It Is Important

A senior .NET engineer is expected to know Kubernetes well enough to:

1. **Write production manifests** — Deployment with proper probes, resource limits, security context.
2. **Debug pods** — `kubectl logs`, `kubectl describe`, `kubectl exec`, reading events.
3. **Map ASP.NET Core concepts** to k8s primitives — `/health/live` to a liveness probe, `appsettings.Production.json` to a ConfigMap, Key Vault to the CSI driver and Workload Identity.
4. **Design for resilience** — multiple replicas, anti-affinity, PodDisruptionBudgets, graceful shutdown via `SIGTERM`.
5. **Operate safely** — rolling updates with maxSurge/maxUnavailable, blue-green via Service selector swap, rollback via `kubectl rollout undo`.

Without this, you ship code that works on `kubectl apply` but crashes under load, drifts from git, or stops the entire team during the next AKS upgrade.

## How It's Used in C# / .NET

### 1. Deployment + Service for an ASP.NET Core API

`k8s/order-api/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  namespace: production
  labels:
    app: order-api
    version: v1.4.2
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: order-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # up to 1 extra pod during rollout
      maxUnavailable: 0       # never drop below desired count
  template:
    metadata:
      labels:
        app: order-api
        version: v1.4.2
        azure.workload.identity/use: "true"   # enable Workload Identity for this pod
    spec:
      serviceAccountName: order-api-sa
      automountServiceAccountToken: true
      terminationGracePeriodSeconds: 30
      containers:
        - name: api
          image: contosoacr.azurecr.io/order-api:abc1234   # immutable SHA, never :latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ASPNETCORE_URLS
              value: http://+:8080
            - name: DOTNET_RUNNING_IN_CONTAINER
              value: "true"
            - name: ConnectionStrings__OrdersDb
              valueFrom:
                secretKeyRef:
                  name: order-api-secrets
                  key: SqlConnectionString
          envFrom:
            - configMapRef:
                name: order-api-config
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet: { path: /health/live, port: http }
            failureThreshold: 30
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]  # give load balancer time to drain
          securityContext:
            runAsNonRoot: true
            runAsUser: 1654
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ "ALL" ]
---
apiVersion: v1
kind: Service
metadata:
  name: order-api
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: order-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

Key elements aligned to ASP.NET Core:

- **`livenessProbe`** → `MapHealthChecks("/health/live", ...)` returning only "process is alive". Failing this triggers a pod restart.
- **`readinessProbe`** → `MapHealthChecks("/health/ready", ...)` checking SQL, Service Bus, Key Vault. Failing this removes the pod from the Service load balancer (but doesn't restart it).
- **`startupProbe`** → grace period for slow startup (cold JIT, DB migrations). Liveness and readiness don't run until startup passes.
- **`preStop` + `terminationGracePeriodSeconds: 30`** → graceful shutdown so in-flight requests finish before SIGKILL. ASP.NET Core's `IHostApplicationLifetime` hooks into SIGTERM automatically.
- **`securityContext`** → non-root, read-only rootfs, no capabilities. Production hardening.

See [health-checks.md](health-checks.md) for the ASP.NET Core side.

### 2. Ingress for external traffic

`k8s/order-api/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [ api.contoso.com ]
      secretName: api-contoso-tls
  rules:
    - host: api.contoso.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-api
                port:
                  name: http
```

For AKS the most common options are the **NGINX Ingress Controller** (Helm-installed) or **Application Gateway Ingress Controller (AGIC)** for native Azure integration.

### 3. ConfigMap and Secret

`k8s/order-api/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-api-config
  namespace: production
data:
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft_AspNetCore: "Warning"
  Features__BulkOrderImport: "true"
  Cache__DefaultTtlSeconds: "300"
```

ASP.NET Core picks up `Logging__LogLevel__Default` as `Logging:LogLevel:Default` via the env var configuration provider — no code change needed.

### 4. HorizontalPodAutoscaler (HPA)

`k8s/order-api/hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-api
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # don't scale down for 5 min after a spike
```

For event-driven scaling (Service Bus queue depth, Event Hub backlog), use **KEDA** (Kubernetes Event-Driven Autoscaler) — installed on AKS as an add-on, ships with .NET-friendly scalers.

### 5. Secrets via Key Vault + Workload Identity

Don't store production secrets as Kubernetes Secrets in git. Use the **Secrets Store CSI Driver** with the Azure Key Vault provider, authenticated via **Azure Workload Identity** (the modern replacement for AAD Pod Identity):

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: order-api-kv
  namespace: production
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "00000000-0000-0000-0000-000000000000"   # workload identity client ID
    keyvaultName: "contoso-prod-kv"
    tenantId: "00000000-0000-0000-0000-000000000000"
    objects: |
      array:
        - |
          objectName: SqlConnectionString
          objectType: secret
        - |
          objectName: ServiceBusConnectionString
          objectType: secret
  secretObjects:
    - secretName: order-api-secrets
      type: Opaque
      data:
        - objectName: SqlConnectionString
          key: SqlConnectionString
```

In the Deployment, mount the volume and reference the synthesized K8s Secret:

```yaml
volumes:
  - name: kv-secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: order-api-kv
```

The ServiceAccount `order-api-sa` is annotated with `azure.workload.identity/client-id` and federated with the Entra ID app registration — no secrets transit anywhere.

### 6. Jobs for DB migrations

`k8s/order-api/migrate-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: order-api-migrate-abc1234
  namespace: production
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: order-api-sa
      containers:
        - name: migrate
          image: contosoacr.azurecr.io/order-api-migrations:abc1234   # bundled `dotnet ef migrations bundle`
          env:
            - name: ConnectionStrings__OrdersDb
              valueFrom:
                secretKeyRef:
                  name: order-api-secrets
                  key: SqlConnectionString
          command: [ "/app/efbundle", "--connection", "$(ConnectionStrings__OrdersDb)" ]
```

CD applies the Job first, waits for completion, then applies the new Deployment.

### 7. Essential kubectl commands

```bash
# Apply manifests
kubectl apply -f k8s/order-api/ -n production

# Inspect
kubectl get pods -n production -l app=order-api
kubectl describe pod order-api-7d5b9c8f6-xqz4l -n production
kubectl logs -f order-api-7d5b9c8f6-xqz4l -n production --tail=100

# Roll out / roll back
kubectl rollout status deployment/order-api -n production
kubectl rollout history deployment/order-api -n production
kubectl rollout undo deployment/order-api -n production

# Scale manually
kubectl scale deployment/order-api --replicas=5 -n production

# Port forward for debugging
kubectl port-forward svc/order-api 8080:80 -n production

# Run a one-off command in a pod
kubectl exec -it order-api-7d5b9c8f6-xqz4l -n production -- /bin/sh
```

### 8. AKS specifics

- **Managed control plane** — Microsoft runs and patches it.
- **Workload Identity** — federated identity from Entra ID to a Kubernetes ServiceAccount. Replaces AAD Pod Identity (deprecated).
- **Azure CNI / Cilium networking** — pod IPs from the VNet, real Azure networking.
- **AKS-managed add-ons** — Application Gateway Ingress Controller, KEDA, Azure Monitor for containers, Defender for Containers.
- **Cluster autoscaler** — adds/removes nodes based on pod scheduling pressure.
- **Image pull from ACR** — attach ACR to AKS (`az aks update --attach-acr`), no imagePullSecrets needed.

## Advantages

- **Declarative + self-healing** — YAML in git is the source of truth; the cluster reconciles.
- **Horizontal scale** — HPA + cluster autoscaler handle traffic spikes automatically.
- **Zero-downtime deployments** — rolling updates with health gates baked in.
- **Service discovery + load balancing** — built into the cluster, no separate config.
- **Portability** — same manifests run on AKS, EKS, GKE, on-prem (with minor tweaks).
- **Ecosystem** — Helm, Kustomize, Argo CD, Flux, KEDA, Dapr, OpenTelemetry, cert-manager are all first-class.
- **Multi-tenancy** — namespaces + RBAC + network policies let many teams share one cluster.

## Disadvantages

- **High operational burden** — control plane upgrades, node patching, cert rotation, ingress controller updates, CRD upgrades.
- **Steep learning curve** — Pods vs Deployments vs ReplicaSets vs StatefulSets vs DaemonSets; Services vs Ingress; ConfigMap vs Secret vs ExternalSecret.
- **Debuggability** — distributed nature makes incidents harder than App Service ("which pod served that request?").
- **Cost** — minimum 2-3 nodes for HA, plus control plane (often free in AKS but workloads aren't), egress, Log Analytics ingestion.
- **YAML sprawl** — without Helm or Kustomize, every environment duplicates manifests.
- **Networking complexity** — Services, Ingress, NetworkPolicies, mTLS via mesh — every layer adds debug time.
- **Wrong tool for some workloads** — running SQL Server on K8s is almost always worse than Azure SQL.

## Common Mistakes

### 1. No resource requests/limits

```yaml
containers:
  - name: api
    image: order-api:abc1234
    # no resources block
```

Without `requests`, the scheduler doesn't know how much CPU/memory to reserve — pods land on overcommitted nodes and get throttled. Without `limits`, a memory leak can OOM-kill neighbors.

**Fix**: always set both. Start with measured baseline + 30% headroom.

```yaml
resources:
  requests: { cpu: "200m", memory: "256Mi" }
  limits:   { cpu: "1000m", memory: "512Mi" }
```

### 2. Liveness probe checks external dependencies

```yaml
livenessProbe:
  httpGet: { path: /health, port: 8080 }   # returns 503 if SQL is down
```

When SQL has a 2-minute hiccup, every pod fails liveness, gets restarted, the new pod also fails — your service is now in a crash loop and recovery time gets *worse*.

**Fix**: liveness checks only that the process is responsive (e.g., a simple `/health/live` that returns 200 unconditionally). Readiness checks dependencies. See [health-checks.md](health-checks.md).

### 3. Using `:latest` tags

```yaml
image: contosoacr.azurecr.io/order-api:latest
```

You can never tell which version is running. A pod restart after a CI push silently upgrades. Rollback is impossible.

**Fix**: pin to the commit SHA or semver (`order-api:abc1234`).

### 4. Storing secrets in plain Kubernetes Secrets in git

```yaml
apiVersion: v1
kind: Secret
metadata: { name: order-api-secrets }
type: Opaque
data:
  SqlConnectionString: U2VydmVyPS4uLjtVc2VyIElkPXNhO1Bhc3N3b3JkPVByb2QhMjAyNDs=
```

Base64 is *not encryption*. Anyone with repo access has prod credentials.

**Fix**: Use Key Vault + CSI driver + Workload Identity. Or **SOPS** with Azure Key Vault for GitOps. Never commit unencrypted secrets.

### 5. Forgetting `terminationGracePeriodSeconds` and `preStop`

The default 30-second grace works for most ASP.NET Core apps **if** they handle SIGTERM correctly. But the Service load balancer takes a few seconds to remove the pod from rotation after deletion — without a `preStop` sleep, new requests still arrive after the app starts shutting down.

**Fix**: add `preStop: exec: command: ["sh","-c","sleep 10"]` and ensure `terminationGracePeriodSeconds` is larger than your longest in-flight request.

### 6. One replica in production

```yaml
spec:
  replicas: 1
```

A node drain or a single pod crash takes the service down.

**Fix**: minimum 2 replicas, ideally 3 for true HA. Pair with a PodDisruptionBudget:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: order-api-pdb, namespace: production }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: order-api } }
```

### 7. Skipping namespaces

Everything in `default`. RBAC is impossible to scope. A misapplied YAML deletes prod.

**Fix**: one namespace per environment per service, or per team. Apply with `-n production` and reject `default` in CI.

### 8. Ignoring image pull secrets / AKS-ACR attach

Manually managing `imagePullSecrets` per service is error-prone.

**Fix**: `az aks update -n my-aks -g my-rg --attach-acr contosoacr`. AKS injects the right pull credentials automatically using its managed identity.

## Best Practices

- **One Deployment per service**, version with image tags, never with manifest copies.
- **Use Helm or Kustomize** to avoid duplicating YAML across environments.
- **GitOps via Argo CD or Flux** — the cluster pulls desired state from git; no `kubectl apply` from laptops.
- **Liveness checks the process; readiness checks dependencies; startup gives slow apps time.**
- **`terminationGracePeriodSeconds: 30`** + a 10-second `preStop` sleep for graceful drain.
- **Always set resource `requests` and `limits`** — measured, not guessed.
- **3+ replicas in production**, anti-affinity across nodes, PodDisruptionBudget for cluster operations.
- **HPA on CPU + custom metrics (KEDA for queues).**
- **Workload Identity, not Kubernetes Secrets**, for Azure resource access.
- **Tag images with commit SHA**, immutably.
- **Namespaces per environment + RBAC** — block `kubectl` to prod from anyone but CD pipelines.
- **Network policies** to default-deny inter-pod traffic and explicitly allow what's needed.
- **Pod Security Standards** (`restricted` profile) — non-root, read-only rootfs, no privilege escalation.
- **Centralized logging + tracing** — Azure Monitor for containers + OpenTelemetry to Application Insights.

## Related Concepts

- **[docker.md](docker.md)** — the image format Kubernetes orchestrates.
- **[ci-cd-pipelines.md](ci-cd-pipelines.md)** — how `kubectl apply` happens automatically.
- **[health-checks.md](health-checks.md)** — the endpoints liveness/readiness/startup probes call.
- **[blue-green-deployment.md](blue-green-deployment.md)** — implemented in K8s via two Deployments + Service selector swap.
- **[rollback-strategy.md](rollback-strategy.md)** — `kubectl rollout undo` and image-tag pinning.
- **[../azure/deployment-to-azure.md](../azure/deployment-to-azure.md)** — AKS, Container Apps, App Service comparison.
- **[../azure/azure-key-vault.md](../azure/azure-key-vault.md)** — the secrets backend for Workload Identity + CSI driver.
- **[../architecture/microservices-architecture.md](../architecture/microservices-architecture.md)** — the deployment target this typically supports.
- **[environment-configuration.md](environment-configuration.md)** — ConfigMaps and Secrets as the Azure-config equivalent.

## Real-World Usage

**Scenario: payments platform on AKS with 12 microservices**

A fintech runs `payment-api`, `payment-worker`, `ledger-api`, `notification-worker`, and 8 others on a 6-node AKS cluster. Layout:

- **Namespaces**: `production`, `staging`, `monitoring`, `infra`.
- **Manifests**: Helm charts per service in a `platform-charts` repo. **Argo CD** in `infra` watches a `gitops` repo and syncs every commit to the cluster.
- **Workload Identity**: each service has a ServiceAccount federated with an Entra ID app registration; secrets come from Key Vault via the CSI driver.
- **Ingress**: Application Gateway Ingress Controller terminates TLS at `api.fintech.com`, routes by path.
- **HPA + KEDA**: `payment-api` scales on CPU; `payment-worker` scales on Service Bus queue depth via KEDA.
- **Probes**: every Deployment exposes `/health/live` (returns 200 always), `/health/ready` (checks SQL, Service Bus, Key Vault), `/health/startup` (returns 200 once warm).
- **Migrations**: Argo runs a pre-sync Job that executes `dotnet ef migrations bundle` against Azure SQL using Workload Identity. The Deployment only rolls out after the Job completes.
- **Observability**: Azure Monitor for containers + OpenTelemetry exporter to Application Insights for traces.

Deploys are git-push → Argo sync (~2 minutes). A bad release? `git revert` the GitOps repo commit; Argo rolls back automatically.

## Code Example — Before and After

### Before: fragile single-replica deployment with no probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 1
  selector: { matchLabels: { app: order-api } }
  template:
    metadata: { labels: { app: order-api } }
    spec:
      containers:
        - name: api
          image: order-api:latest
          ports: [ { containerPort: 80 } ]
          env:
            - name: ConnectionStrings__OrdersDb
              value: "Server=db.contoso.com;Password=Prod!2024;"
```

Problems:
- One replica → no HA, node drain takes the service down.
- `:latest` → no traceability, no rollback.
- No probes → crashed pods stay in service, slow startups serve 502s.
- No resource requests/limits → unpredictable scheduling.
- Secrets in plaintext in YAML.
- Runs as root (no securityContext).
- No graceful shutdown — in-flight requests die on SIGKILL.

### After: production-grade Deployment + Service + HPA

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  selector: { matchLabels: { app: order-api } }
  template:
    metadata:
      labels: { app: order-api, version: v1.4.2 }
    spec:
      serviceAccountName: order-api-sa
      terminationGracePeriodSeconds: 30
      containers:
        - name: api
          image: contosoacr.azurecr.io/order-api:abc1234
          ports: [ { name: http, containerPort: 8080 } ]
          env:
            - { name: ASPNETCORE_ENVIRONMENT, value: Production }
            - name: ConnectionStrings__OrdersDb
              valueFrom: { secretKeyRef: { name: order-api-secrets, key: SqlConnectionString } }
          resources:
            requests: { cpu: "200m", memory: "256Mi" }
            limits:   { cpu: "1000m", memory: "512Mi" }
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            periodSeconds: 30
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            periodSeconds: 10
          startupProbe:
            httpGet: { path: /health/live, port: http }
            failureThreshold: 30
            periodSeconds: 5
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 10"] }
          securityContext:
            runAsNonRoot: true
            runAsUser: 1654
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: [ "ALL" ] }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: order-api, namespace: production }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: order-api }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Now:
- 3 replicas, rolling update with zero unavailability.
- Pinned, traceable image tag.
- Probes correctly separate liveness from readiness and startup.
- Resource bounds prevent noisy-neighbor problems.
- Secret comes from Key Vault via the CSI driver.
- Non-root, read-only filesystem, no Linux capabilities.
- Graceful shutdown drains traffic before SIGKILL.
- HPA scales horizontally to handle bursts.

## Interview Questions and Answers

### 1. Walk me through what happens when you run `kubectl apply -f deployment.yaml`.

**Why this matters**: Tests whether the candidate understands the control loop, not just the CLI.

**Answer**: `kubectl` sends the YAML to the **API server**, which validates it and writes to **etcd**. The **Deployment controller** sees the new spec, creates or updates a **ReplicaSet** with the matching pod template. The ReplicaSet controller creates **Pod** objects in etcd. The **scheduler** picks a node for each pod with sufficient resources. The kubelet on that node tells the container runtime (containerd) to pull the image and start the container. **Readiness probes** decide when the pod joins the **Service's Endpoints** list and starts receiving traffic. The whole thing is asynchronous and self-healing — if a pod dies, the controllers reconcile back to the declared state.

### 2. A pod is `CrashLoopBackOff`. How do you debug?

**Answer**: Methodical walk:

1. `kubectl describe pod <name> -n <ns>` → look at Events and the container's `Last State` / `Reason`.
2. `kubectl logs <name> -n <ns> --previous` → logs from the most recent crash (current container hasn't started).
3. Common causes:
   - **Wrong image / pull failure** → `ImagePullBackOff` first, then `CrashLoopBackOff`. Check image name/tag, ACR attach, secret.
   - **Failed liveness probe** → kubectl describe shows "Liveness probe failed". Check probe path/port matches the app.
   - **OOMKilled** (exit code 137) → raise memory limit or fix leak.
   - **Missing config/secret** → exception in startup: `OptionsValidationException`, `KeyNotFoundException`.
   - **DB unreachable on startup** with `MigrateAsync()` → use a startup job instead.
4. `kubectl exec` is useless if the pod won't stay up — use a temporary debug container: `kubectl debug -it <pod> --image=mcr.microsoft.com/dotnet/sdk:8.0 --target=api`.

### 3. Difference between liveness, readiness, and startup probes?

**Answer**:

| Probe | Purpose | Failure action | Use for |
|---|---|---|---|
| **liveness** | Is the process responsive? | Restart the container | Detect deadlocks. Should be cheap, no external deps. |
| **readiness** | Can the app handle traffic now? | Remove from Service endpoints (don't restart) | Detect DB outages, warm-up, draining. |
| **startup** | Has the app finished starting? | Restart after `failureThreshold * periodSeconds` | Apps with slow startup (cold JIT, large caches) — disables liveness/readiness until passed. |

**Critical rule**: never check SQL/Key Vault/Service Bus in liveness. When that dependency hiccups, every pod will fail liveness, every pod will restart, and you'll be in a cluster-wide crash loop instead of riding out a 2-minute blip.

**Real project**: A team had readiness mis-configured to also check an external pricing API. When the vendor had a 30-minute outage, K8s removed all pods from the Service. The fix was making readiness check only *critical* dependencies (DB, Service Bus) and treating pricing as an optional dependency with circuit breaker + fallback.

### 4. How do you do zero-downtime deployments in Kubernetes?

**Answer**: Several layers:

1. **Multiple replicas** (≥ 2) so removing one doesn't take the service down.
2. **Rolling update strategy** with `maxSurge: 1, maxUnavailable: 0` — always have at least the desired number of healthy pods.
3. **Readiness probe** that returns 200 only when the pod can actually serve requests. New pods don't get traffic until ready.
4. **`terminationGracePeriodSeconds: 30`** + **`preStop` sleep** so the Service load balancer stops sending new requests before the app starts shutting down.
5. **ASP.NET Core's `IHostApplicationLifetime.ApplicationStopping`** handler that returns 503 from `/health/ready` immediately, then finishes in-flight requests.
6. **`PodDisruptionBudget`** so cluster operations (node drain, AKS upgrade) can't take all replicas down at once.

For **blue-green or canary** patterns, see [blue-green-deployment.md](blue-green-deployment.md). With pure K8s, the simplest blue-green is two Deployments (`order-api-blue`, `order-api-green`) and switching the Service's `selector`.

### 5. The HPA isn't scaling up under load. What do you check?

**Answer**:

1. **Are metrics flowing?** `kubectl top pods` — if it errors, the metrics-server isn't installed/working.
2. **Resource `requests` set?** HPA percentage is `usage / request`. Without requests, utilization is undefined.
3. **HPA configured correctly?** `kubectl describe hpa order-api` shows current vs target metrics, conditions, last scale time.
4. **Scale ceiling reached?** `maxReplicas` may be too low.
5. **Node capacity?** HPA may want 10 pods, but the cluster can only schedule 6. Cluster autoscaler should add nodes — check its logs.
6. **Stabilization window** — by default HPA waits 5 minutes before scaling *down* to avoid flapping; scale-up is faster but configurable.
7. **The wrong signal** — CPU utilization is fine for compute-bound APIs; for queue workers use **KEDA** with the Service Bus scaler instead.

### 6. How do you handle secrets in AKS?

**Answer**: Layered, in order of recommendation:

1. **Workload Identity + direct Key Vault access in code** — best. Pod's ServiceAccount is federated with an Entra ID app registration; `DefaultAzureCredential` picks it up; app calls `SecretClient.GetSecretAsync` at startup. No secrets in K8s at all.
2. **Secrets Store CSI Driver + Workload Identity** — when the app code can only read from env vars or files. CSI driver pulls from Key Vault and synthesizes a K8s Secret which is mounted into the pod.
3. **External Secrets Operator** — similar idea, more cloud-agnostic.
4. **Sealed Secrets / SOPS** — encrypted at rest in git for GitOps workflows.
5. **Plain K8s Secrets** — only acceptable for non-production or non-sensitive values (and even then, base64 is not encryption).

Never, ever check unencrypted production secrets into git.

### 7. When would you prefer Azure Container Apps over AKS?

**Answer**: For most modern .NET workloads, **Container Apps wins** unless you specifically need raw Kubernetes APIs.

Container Apps gives you:
- KEDA-based autoscaling (including scale-to-zero) without managing it.
- Dapr for service-to-service communication.
- Revisions for blue-green / canary releases.
- Ingress termination with managed certificates.
- Workload Identity, Key Vault integration.
- No control plane to upgrade, no node pools to manage.

You move to AKS when:
- You need **kubectl** for in-cluster operators or CRDs (Argo, Istio, custom controllers).
- You need **DaemonSets** or **StatefulSets** (Container Apps doesn't support them).
- You have **strict networking** requirements that need NetworkPolicies or a service mesh.
- You're consolidating **many teams** onto a shared platform with custom tooling.

**Trade-off**: Container Apps is faster to adopt and cheaper to operate for small/medium fleets. AKS is the ceiling — full Kubernetes if you need it.

### 8. Your AKS upgrade is about to happen. How do you make sure it's safe?

**Answer**:

1. **Read the release notes** for the K8s version and AKS add-ons — breaking API removals (e.g., `policy/v1beta1` PDB removal in 1.25) bite.
2. **Test in a non-prod cluster first** — apply the upgrade to staging, validate.
3. **Confirm Pod Disruption Budgets** — every production Deployment needs a PDB so node drains can't kill all replicas at once.
4. **Confirm multiple replicas + anti-affinity** — pods spread across nodes/zones.
5. **Confirm readiness probes** behave correctly during drain (`preStop` sleeps, graceful shutdown).
6. **Schedule during low-traffic window** with on-call coverage.
7. **Upgrade control plane first**, then node pools surge-style (`az aks nodepool upgrade --max-surge 33%` adds one extra node, drains the oldest, repeats).
8. **Monitor** — Application Insights / Azure Monitor for error rate, p95 latency, pod restarts during the rolling upgrade.
9. **Have rollback plan** — AKS doesn't support downgrade; rollback means restoring from a parallel cluster or reapplying old deployments. Run on N-1 if you must roll back.

**Real project**: A team migrated to PDBs cluster-wide before their first AKS upgrade and reduced upgrade-related incident count from "one per upgrade" to zero.

## Summary Checklist

- [ ] I can write a Deployment + Service + HPA YAML for an ASP.NET Core API with proper probes and resource limits.
- [ ] I can map ASP.NET Core `/health/live`, `/health/ready`, `/health/startup` to K8s liveness/readiness/startup probes.
- [ ] I can explain why liveness probes must NOT check external dependencies.
- [ ] I can run rolling updates with `maxSurge`/`maxUnavailable` and trigger `kubectl rollout undo`.
- [ ] I can configure Workload Identity + Key Vault CSI driver instead of plain Kubernetes Secrets.
- [ ] I can debug `CrashLoopBackOff`, `ImagePullBackOff`, and `OOMKilled` pods using `describe`, `logs`, and exit codes.
- [ ] I can configure HPA on CPU and KEDA on Service Bus queue depth.
- [ ] I can pick between AKS, Container Apps, and App Service based on requirements.
- [ ] I can run a safe AKS upgrade with PDBs, multiple replicas, and observability gates.
- [ ] I can structure manifests with Helm/Kustomize and apply via GitOps (Argo CD / Flux).
