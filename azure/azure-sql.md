# Azure SQL

## What It Is

Azure SQL is the family of **fully managed SQL Server database services** in Azure. It runs the same engine as on-prem SQL Server (a recent build, always patched) without you provisioning a VM, installing SQL, or managing backups. There are three deployment options:

1. **Azure SQL Database (Single Database)** — one logical database, fully isolated. Most common choice for new apps. Two purchasing models: **DTU** (bundled compute+storage+IO) and **vCore** (granular, recommended).
2. **Azure SQL Elastic Pool** — multiple Single Databases sharing one pool of DTUs/vCores. Good for SaaS with many small tenant DBs.
3. **Azure SQL Managed Instance** — almost-100%-compatible with on-prem SQL Server (SQL Agent, cross-DB queries, CLR, Service Broker). Best for lift-and-shift of legacy systems with SSIS/Agent dependencies.

```text
┌────────────── Azure Subscription / Resource Group ───────────────┐
│                                                                   │
│   ┌─── Logical Server (orders-sql.database.windows.net) ─────┐   │
│   │   - Entra ID admin: dba-team@contoso.com                 │   │
│   │   - Firewall: deny-all + private endpoint                │   │
│   │   - TLS 1.2 enforced                                     │   │
│   │                                                          │   │
│   │   ┌─── orders-db (vCore General Purpose, 2 vCore) ──┐    │   │
│   │   │   - Geo-replication → westus secondary           │    │   │
│   │   │   - PITR: 7 days, LTR weekly for 1 year         │    │   │
│   │   └─────────────────────────────────────────────────┘    │   │
│   │   ┌─── audit-db (Serverless, auto-pause after 1 hr) ┐    │   │
│   │   └─────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

In a typical .NET service, Azure SQL is the system of record for transactional data — orders, customers, payments, audit trails — accessed via Entity Framework Core or Dapper.

## Why It Exists

Running production SQL Server on a VM means:

- Provisioning Windows + SQL Server VMs, sizing storage IOPS, configuring tempdb.
- Setting up Always On Availability Groups for HA, a witness server, and quorum.
- Writing backup jobs, testing restores quarterly, paying for backup storage.
- Patching SQL Server cumulative updates monthly, with downtime.
- Capacity planning a year in advance.

Azure SQL exists to remove every one of those operational chores. Microsoft runs the engine, you run the queries. HA (99.99%-99.995% SLA), backups (point-in-time restore), patching, and zone-redundancy come included.

It also unlocks **elastic scale and serverless billing** patterns that aren't possible with SQL on a VM — pause a database during the night and wake it on the first connection.

## When To Use It

**Use Azure SQL for:**

- **Transactional data** for an ASP.NET Core API — orders, customers, payments, inventory.
- **Audit logs and journal tables** where you need ACID transactions and `GETUTCDATE()`.
- **Reporting databases** where T-SQL views and stored procedures are valuable.
- **Multi-tenant SaaS** — Elastic Pool for many small tenant DBs, or sharding via shard map manager for large tenants.
- **Lift-and-shift of on-prem SQL Server** — use Managed Instance to keep SQL Agent, SSIS, cross-DB queries, and CLR.

**Do not use Azure SQL for:**

- **High-write throughput, schema-flexible workloads** — use Cosmos DB or PostgreSQL.
- **Document/graph models** as the primary storage — SQL Server has JSON support but isn't a document DB; use Cosmos or MongoDB.
- **Petabyte-scale analytics** — use Synapse, Fabric, or Databricks.
- **Real-time event streams** — use Event Hubs or Kafka.
- **Caching** — use Redis ([azure-storage.md](azure-storage.md) covers persistence; Azure Cache for Redis covers caching).

## Why It Is Important

Azure SQL is **the** default relational database in Azure. In any interview for a .NET role hitting Azure, you'll be asked to:

1. **Pick the right deployment option and tier** — Single DB vs Elastic Pool vs Managed Instance, DTU vs vCore, General Purpose vs Business Critical, provisioned vs serverless.
2. **Authenticate via Entra ID / Managed Identity** — the modern, secret-less pattern.
3. **Secure the network surface** — private endpoint, firewall, deny-public-network-access.
4. **Plan for HA and DR** — geo-replication, auto-failover groups, PITR, LTR.
5. **Tune queries with Query Store and Automatic Tuning**.

Getting these wrong is the difference between a $200/month database and a $20K/month bill, and between 99.99% uptime and weekly outages.

## How It's Used in C# / .NET

### 1. Connection string — no passwords, Entra ID auth

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "OrdersDb": "Server=tcp:orders-sql.database.windows.net,1433;Database=orders;Authentication=Active Directory Default;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}
```

`Authentication=Active Directory Default` tells the Microsoft SqlClient to call `DefaultAzureCredential` under the hood — Managed Identity on App Service/Functions, Visual Studio / `az login` locally. **No password anywhere.**

Grant the Managed Identity SQL access (as a SQL admin):

```sql
CREATE USER [orders-api] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [orders-api];
ALTER ROLE db_datawriter ADD MEMBER [orders-api];
GRANT EXECUTE ON SCHEMA::dbo TO [orders-api];
```

### 2. Register EF Core with Managed Identity

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<OrdersDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("OrdersDb"),
        sql =>
        {
            sql.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);     // SqlAzureExecutionStrategy handles transient errors
            sql.CommandTimeout(30);
        });
});
```

### 3. Health check + outbound retry policy

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(
        builder.Configuration.GetConnectionString("OrdersDb")!,
        name: "orders-sql",
        tags: new[] { "ready" });
```

### 4. Transactions and concurrency

```csharp
public class PlaceOrderHandler(OrdersDbContext db, IServiceBusSender bus)
{
    public async Task<Guid> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);

        var order = new Order(cmd.CustomerId, cmd.Items);
        db.Orders.Add(order);
        db.OutboxMessages.Add(new OutboxMessage(
            type: "order.placed",
            payload: JsonSerializer.Serialize(order.ToEvent())));

        try
        {
            await db.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);
        }
        catch (DbUpdateConcurrencyException)
        {
            await tx.RollbackAsync(ct);
            throw;
        }

        return order.Id;
    }
}
```

### 5. Geo-replication / failover group configuration (Bicep)

```bicep
resource server 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'orders-sql-eastus'
  location: 'eastus'
  properties: {
    administrators: {
      administratorType: 'ActiveDirectory'
      principalType: 'Group'
      login: 'dba-team@contoso.com'
      sid: dbaTeamObjectId
      tenantId: subscription().tenantId
    }
    publicNetworkAccess: 'Disabled'    // private endpoint only
    minimalTlsVersion: '1.2'
  }
}

resource db 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: server
  name: 'orders'
  location: 'eastus'
  sku: { name: 'GP_Gen5_2', tier: 'GeneralPurpose', family: 'Gen5', capacity: 2 }
  properties: {
    zoneRedundant: true                    // 99.995% SLA
  }
}

resource failoverGroup 'Microsoft.Sql/servers/failoverGroups@2023-08-01-preview' = {
  parent: server
  name: 'orders-fog'
  properties: {
    partnerServers: [{ id: secondaryServer.id }]
    readWriteEndpoint: {
      failoverPolicy: 'Automatic'
      failoverWithDataLossGracePeriodMinutes: 60
    }
    databases: [db.id]
  }
}
```

### 6. Serverless tier for a low-traffic audit DB

```bicep
sku: { name: 'GP_S_Gen5_2', tier: 'GeneralPurpose', family: 'Gen5', capacity: 2 }
properties: {
  autoPauseDelay: 60     // pause after 60 min of no activity; $0 compute while paused
  minCapacity: '0.5'
}
```

### 7. Tier comparison

| Tier                          | Use case                          | HA SLA   | Storage  | Notes                                          |
|-------------------------------|------------------------------------|----------|----------|------------------------------------------------|
| Basic / Standard (DTU)        | Dev/test, tiny apps                | 99.99%   | 2-1 TB   | Legacy; prefer vCore                            |
| GP (General Purpose, vCore)   | Most prod apps                     | 99.99%   | up to 4TB | Compute & storage decoupled                    |
| GP Serverless                 | Sporadic workloads, dev/test       | 99.99%   | 1 TB     | Auto-pauses; $0 while paused                    |
| Business Critical (BC)        | Low-latency, in-memory OLTP        | 99.995%  | 4 TB     | Local SSD; built-in readable replicas; ~3-4x cost of GP |
| Hyperscale                    | Very large DBs (up to 100 TB)      | 99.99%   | 100 TB   | Fast backup/restore regardless of size; can't shrink |
| Managed Instance              | Lift-and-shift on-prem SQL         | 99.99%   | 16 TB    | Most SQL features; VNet only                    |

## Advantages

- **Same engine as SQL Server** — your existing T-SQL, indexes, and EF Core code "just work".
- **Managed Identity / Entra ID auth** — no passwords in connection strings.
- **Point-in-time restore (PITR)** by default for the last 7-35 days.
- **Long-term retention (LTR)** for compliance — weekly/monthly/yearly backups for up to 10 years.
- **Auto-failover groups** for cross-region DR with one DNS endpoint.
- **Query Store and Automatic Tuning** find regressed queries and apply index/plan fixes automatically.
- **Serverless tier** — pay $0 for compute while paused; ideal for dev/test and bursty workloads.
- **Zone-redundant General Purpose** (no extra cost in supported regions) for 99.995% SLA.

## Disadvantages

- **Cost scales fast** — Business Critical 8 vCore is ~$3K/month. Hyperscale 100 TB has high storage charges.
- **DTU model is opaque** — you can't see CPU/IO breakdown; vCore is recommended for clarity.
- **Cross-database queries are limited** in Single Database (allowed in Managed Instance, restricted in SQL DB).
- **No SQL Agent or SSIS** in Single Database — schedule via Azure Functions, Logic Apps, or Elastic Jobs.
- **Hyperscale databases cannot be reverted to other tiers** — pick wisely.
- **Vendor lock-in** — failover groups, automatic tuning, and serverless billing don't transfer to other clouds.
- **Region limitations** — newer tiers (Hyperscale Serverless, Zone Redundant BC) roll out region-by-region.
- **TempDB sizing is fixed by tier** — surprise on workloads with large temp tables.

## Common Mistakes

### 1. SQL admin password in source control

```csharp
// BUG: password rotates require code change, key is in git forever
"Server=tcp:orders-sql.database.windows.net;User Id=sqladmin;Password=P@ssw0rd!;"
```

**Fix**: drop the password entirely with Entra ID auth:

```csharp
"Server=tcp:orders-sql.database.windows.net;Database=orders;Authentication=Active Directory Default;Encrypt=True;"
```

Grant the App Service / Function Managed Identity as a SQL user (see "How It's Used").

### 2. Picking Basic tier for production

```text
"It's only $5/month so let's use Basic."
[a week later] "The API is timing out under load."
```

Basic has 5 DTU — about 1 small CPU and very low IO. **Fix**: start at **GP_Gen5_2** (~$370/month) for any real workload; downgrade only after measuring real utilization in Query Performance Insight.

### 3. Public network access enabled

```bicep
// BUG: anyone with the admin password can connect from anywhere
publicNetworkAccess: 'Enabled'
```

**Fix**: disable public access and use Private Endpoint:

```bicep
publicNetworkAccess: 'Disabled'

resource pe 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'orders-sql-pe'
  // ...
  properties: {
    privateLinkServiceConnections: [{
      name: 'sql'
      properties: {
        privateLinkServiceId: server.id
        groupIds: ['sqlServer']
      }
    }]
  }
}
```

### 4. No retry policy on transient failures

```csharp
// BUG: transient 40197/40501/10928 errors crash requests
options.UseSqlServer(connectionString);
```

**Fix**: enable the Azure execution strategy:

```csharp
options.UseSqlServer(connectionString, sql =>
    sql.EnableRetryOnFailure(maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(30), null));
```

### 5. Ignoring connection pooling under high concurrency

```csharp
// BUG: connections aren't being released; pool exhaustion (default 100)
public async Task<Order?> GetAsync(Guid id)
{
    var conn = new SqlConnection(_connStr);
    await conn.OpenAsync();
    // ... forgot to dispose
}
```

**Fix**: always `await using` connections; rely on EF Core's `DbContext` for pooling (scoped to request via DI). Tune the pool size in the connection string if needed: `Max Pool Size=200`.

### 6. Long-running transactions blocking other work

```csharp
// BUG: tx held open while waiting on external HTTP — blocks every other reader of the row
await using var tx = await db.Database.BeginTransactionAsync();
db.Orders.Add(order);
await db.SaveChangesAsync();
await _stripe.ChargeAsync(...);    // external network call inside the tx
await tx.CommitAsync();
```

**Fix**: keep transactions strictly to local DB work. Use the **outbox pattern** ([../architecture/outbox-pattern.md](../architecture/outbox-pattern.md)) — write the "charge Stripe" intent inside the tx, do the HTTP call outside.

### 7. Wrong purchasing model for the workload

```text
"We're on Hyperscale 8 vCore at $3K/month for a 5 GB database."
```

Hyperscale is for >1 TB databases needing fast restore. **Fix**: for small DBs, use GP_Gen5_2 + serverless if usage is sporadic — could be $50/month for the same workload.

## Best Practices

- **Use vCore over DTU** for new databases — transparent pricing, finer control.
- **Use General Purpose Zone Redundant** for production (99.995% SLA, same price as zonal in supported regions).
- **Use Entra ID auth + Managed Identity** for every connection from Azure-hosted code.
- **Enable Private Endpoint** and set `publicNetworkAccess: 'Disabled'`.
- **Use auto-failover groups** for cross-region DR; configure RW endpoint failover policy based on RPO tolerance.
- **Enable Query Store** (on by default) and review Top Resource Consumers monthly.
- **Enable Automatic Tuning** for `CREATE_INDEX` and `DROP_INDEX` (force-plan is more controversial — review with DBA).
- **Use `EnableRetryOnFailure`** in every EF Core registration.
- **Apply migrations via a startup task or release pipeline**, not on every container start.
- **Set up Azure Monitor alerts** for DTU > 80%, deadlocks, failed logins, long-running queries.
- **Use serverless tier** for dev/test and seasonal workloads — auto-pause is real money saved.
- **Use Elastic Jobs** for cross-DB scheduled tasks instead of SQL Agent (which isn't available in Single Database).

## Related Concepts

- **EF Core** ([../data-access/entity-framework-core.md](../data-access/entity-framework-core.md)) — the ORM you typically use with Azure SQL.
- **Migrations** ([../data-access/migrations.md](../data-access/migrations.md)) — schema changes during deploy.
- **Transactions** ([../data-access/transactions.md](../data-access/transactions.md)) — local tx semantics.
- **Optimistic concurrency** ([../data-access/optimistic-concurrency.md](../data-access/optimistic-concurrency.md)) — `rowversion`/`timestamp` columns.
- **Query performance** ([../data-access/query-performance.md](../data-access/query-performance.md)) — indexes, plan cache, Query Store.
- **Outbox pattern** ([../architecture/outbox-pattern.md](../architecture/outbox-pattern.md)) — coordinate DB + message broker writes.
- **Key Vault** ([azure-key-vault.md](azure-key-vault.md)) — store fallback SQL passwords for legacy systems that can't use Entra ID.
- **Application Insights** ([application-insights.md](application-insights.md)) — dependency tracking shows every SQL call latency.

## Real-World Usage

### Order/checkout database

- **Tier**: GP_Gen5_4 (4 vCore), Zone Redundant, ~$1,200/month.
- **Auth**: Entra ID; the App Service Managed Identity is mapped to `orders-api` SQL user with `db_datareader`, `db_datawriter`, `EXECUTE` on procs.
- **Network**: Private Endpoint in the VNet; `publicNetworkAccess: Disabled`.
- **DR**: Auto-failover group to `westus`; `failoverWithDataLossGracePeriodMinutes: 60` (allow 60 min data loss before failing over).
- **Backup**: 7-day PITR + LTR weekly for 1 year (finance audit requirement).
- **Monitoring**: Diagnostic logs → Log Analytics; alert on `DeadlockEvent > 5/min`, `DTU > 80% for 10 min`.

### Multi-tenant SaaS billing DB

- **Tier**: Elastic Pool (eDTU 100, max 100 DBs), each tenant gets a small DB.
- **Sharding**: Shard map manager points the API at the right tenant DB.
- **Backup**: Per-tenant PITR; one-click restore-of-one-tenant without impacting others.

### Reporting/analytics replica

- **Tier**: Business Critical readable secondary on the primary; reports run on the read-only replica via `ApplicationIntent=ReadOnly` in the connection string.
- **Refresh**: Synchronous replication; reports are <1 second behind.

### Dev/test environments

- **Tier**: GP Serverless, `autoPauseDelay: 60 min`, `minCapacity: 0.5`.
- **Cost**: ~$15/month per dev environment (vs $370 for provisioned).

## Code Example — Before and After

### Before: SQL auth, no retry, no telemetry

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "OrdersDb": "Server=tcp:orders-sql.database.windows.net;User Id=sqladmin;Password=P@ssw0rd!;Database=orders;"
  }
}

// Program.cs
builder.Services.AddDbContext<OrdersDb>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("OrdersDb")));

// Repository
public async Task<Order?> GetAsync(Guid id)
{
    using var conn = new SqlConnection(_connStr);
    await conn.OpenAsync();
    using var cmd = new SqlCommand($"SELECT * FROM Orders WHERE Id = '{id}'", conn);  // SQL injection!
    using var reader = await cmd.ExecuteReaderAsync();
    // ... no retry on transient error 40197
}
```

Problems:
- Plain-text password in source control.
- SQL injection.
- No retry on transient Azure SQL errors.
- Public network access (default).
- No telemetry, no health check.

### After: Managed Identity, EF Core, retry, private endpoint

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "OrdersDb": "Server=tcp:orders-sql.database.windows.net;Database=orders;Authentication=Active Directory Default;Encrypt=True;Connection Timeout=30;"
  }
}

// Program.cs
builder.Services.AddDbContext<OrdersDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("OrdersDb"),
        sql =>
        {
            sql.EnableRetryOnFailure(5, TimeSpan.FromSeconds(30), null);
            sql.CommandTimeout(30);
        }));

builder.Services.AddHealthChecks()
    .AddDbContextCheck<OrdersDbContext>(tags: new[] { "ready" });

builder.Services.AddApplicationInsightsTelemetry();

// Repository
public sealed class OrderRepository(OrdersDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);
}
```

Plus Bicep:

```bicep
resource db 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: server
  name: 'orders'
  sku: { name: 'GP_Gen5_4', tier: 'GeneralPurpose', family: 'Gen5', capacity: 4 }
  properties: { zoneRedundant: true }
}

resource sqlAdmin 'Microsoft.Sql/servers/administrators@2023-08-01-preview' = {
  parent: server
  name: 'ActiveDirectory'
  properties: {
    administratorType: 'ActiveDirectory'
    login: 'dba-team@contoso.com'
    sid: dbaGroupObjectId
  }
}
```

## Interview Questions and Answers

### 1. When would you choose Managed Instance over Single Database?

**Why this matters**: A migration architect needs to know which SKU fits which workload.

**Answer**: Choose **Managed Instance** when you have an existing on-prem SQL Server workload that depends on features Single Database doesn't have: SQL Agent jobs, cross-database queries, CLR, Service Broker, SSIS via Integration Runtime, linked servers, or full T-SQL compatibility. Choose **Single Database** for greenfield apps where you control the schema and access pattern.

**Trade-off**: Managed Instance requires a VNet and has a 4-hour provisioning time (vs <1 minute for Single DB). Cost is higher (smallest GP MI is ~$700/month vs $370 for GP_Gen5_2 SQL DB).

**Real project**: We migrated a 10-year-old ERP that used 50+ SQL Agent jobs and CLR functions to Managed Instance with zero code changes. A greenfield order API in the same org runs on Single DB GP_Gen5_4.

### 2. Explain DTU vs vCore.

**Answer**:

- **DTU** (Database Transaction Unit) is a Microsoft-defined bundle of CPU + memory + IO. You buy a bucket (e.g., S3 = 100 DTU); you don't see the breakdown. Simple, but opaque — under load you can't tell whether you're CPU-bound or IO-bound.
- **vCore** is the modern model: you buy a number of vCores (CPU) and storage independently. Same hardware as on-prem SQL Server (Gen5 = Intel/AMD with hyperthreading). You can pick General Purpose (network storage), Business Critical (local SSD + replicas), or Hyperscale (very large DBs). Supports the **Azure Hybrid Benefit** so you reuse on-prem SQL licenses for ~40% discount.

**Trade-off**: vCore is always recommended for production; DTU survives mostly for legacy migration ease.

**Real project**: Migrated from S3 (100 DTU, ~$150/mo) to GP_Gen5_2 (~$370/mo). The vCore SKU gave us 4x more IO under the same CPU and resolved a year-old throttling complaint.

### 3. How does an Azure SQL auto-failover group work?

**Answer**: Two logical servers (`primary` in eastus, `secondary` in westus) each host a copy of the database. Replication is asynchronous. You group them into a **failover group**, which gives you a single read-write endpoint (`fog-orders.database.windows.net`) and a separate read-only endpoint. The app connects to the read-write endpoint.

If the primary region goes down, the failover group **automatically** promotes the secondary after a grace period (`failoverWithDataLossGracePeriodMinutes`, e.g., 60). DNS for the RW endpoint flips to the new primary. Apps reconnect to the same DNS name; they see a brief connection failure and recover via retry.

**Trade-off**: Async replication = potential data loss (RPO seconds). Apps must tolerate `~5 sec` of failures during failover. Cost is roughly 2x (you pay for both DBs).

**Real project**: Black Friday region outage — failover triggered automatically after 60 minutes of no health response from eastus. We had ~30 sec of failed requests, then full recovery in westus. Total data loss: 2 messages (replayed via outbox).

### 4. Your app sees `Login failed for user '<token-identified principal>'`. How do you debug?

**Answer**: 

1. The connection string uses `Authentication=Active Directory Default` (good).
2. Check **the user's existence in the database**: `SELECT * FROM sys.database_principals WHERE name = 'orders-api'`.
3. Verify the Managed Identity is enabled: in App Service → Identity → System assigned = On.
4. Confirm the SQL server admin has run `CREATE USER [orders-api] FROM EXTERNAL PROVIDER` against the **target database**, not master.
5. Confirm the firewall / private endpoint allows the App Service's IP/VNet.
6. Check the audit log: SQL Auditing logs the principal name and rejection reason.

**Common cause**: SQL user created in `master`, not in the application database. Each database needs its own `CREATE USER ... FROM EXTERNAL PROVIDER`.

**Real project**: We hit this in week 2 of a migration — the DBA had created users in master expecting them to inherit. Fix took 30 seconds; finding the cause took 90 minutes.

### 5. How do you handle Azure SQL transient errors gracefully?

**Answer**: 

- **Use `EnableRetryOnFailure`** in EF Core — implements `SqlAzureExecutionStrategy`, retries 40197, 40501, 49918, 49919, 49920, 11001, 10928, 10929, 64, 20, 40540 with exponential backoff.
- **Idempotent operations only** — retry will replay your work; non-idempotent commands (e.g., `INSERT` without `IF NOT EXISTS`) duplicate.
- **Manual transactions don't auto-retry** — use `db.Database.CreateExecutionStrategy().ExecuteAsync()` to wrap explicit transactions.
- **Set `Connection Timeout=30`** and `CommandTimeout(30)` for slow recovery scenarios.
- **Use the new Polly v8 `ResilienceStrategy`** for app-level retry policies on the surrounding HTTP request.

**Trade-off**: Retries hide problems. Alert on retry counts so you know when transient becomes chronic.

**Real project**: Our App Insights dashboard tracks the count of "execution strategy retried" events per hour. A spike usually means SQL is hitting CPU cap; we either scale or throttle inbound traffic.

### 6. Compare Provisioned vs Serverless GP tier.

**Answer**:

- **Provisioned**: You pay for the vCore allocation 24/7, even if idle. Predictable bill, no cold start, fixed performance.
- **Serverless**: You pay per-second for CPU, with auto-pause after configurable inactivity (`autoPauseDelay`, min 60 min). Storage is billed separately. **First query after pause has a 5-30 sec resume latency.**

Use serverless for **dev/test, internal tools, low-volume audit DBs**. Use provisioned for **production workloads with steady traffic** — serverless's per-second pricing exceeds provisioned at ~25% sustained utilization.

**Trade-off**: Serverless saves money on idle but is bad UX for user-facing workloads (5-30 sec wakeup on first request).

**Real project**: Our 15 developer environments each run on serverless 0.5-2 vCore, auto-pause at 60 min. Idle weekend cost: $0 each. Production runs on provisioned GP_Gen5_4.

### 7. How do you migrate a 200 GB on-prem SQL DB to Azure SQL with minimal downtime?

**Answer**: 

1. **Assess** with **Data Migration Assistant (DMA)** — flags features not supported in your target (e.g., Single DB doesn't support SQL Agent).
2. **Pick target**: Managed Instance if you have SQL Agent / SSIS / linked servers; Single DB otherwise.
3. **Use Azure Database Migration Service (DMS) in online mode** — full backup + continuous transaction log replication.
4. **Cutover**: stop writes on source for ~5 min, wait for DMS to drain pending logs, update app connection string, resume.
5. **Roll back plan**: keep on-prem read-write capable for 24h; can re-point if something is wrong.

**Trade-off**: Online DMS has setup overhead; offline (one-shot backup-restore) is simpler but has hours of downtime depending on DB size.

**Real project**: Migrated a 400 GB ERP to Managed Instance via DMS online. Total cutover downtime: 8 minutes. We kept the on-prem read replica for two weeks as fallback.

### 8. How do you optimize a slow query in Azure SQL?

**Answer**: 

1. **Query Performance Insight** (portal) shows top queries by CPU/duration/IO with execution counts.
2. **Query Store** (`sys.query_store_*` DMVs) shows query plans, regressions, and forced plans.
3. **Get the actual execution plan** in SSMS or Azure Data Studio (`SET STATISTICS XML ON`).
4. **Look for**: missing indexes (DMV `sys.dm_db_missing_index_*`), implicit conversions (parameter-type mismatch), key lookups (covering index opportunity), parameter sniffing (use `OPTIMIZE FOR UNKNOWN` or `RECOMPILE`).
5. **Enable Automatic Tuning** for `CREATE_INDEX` if you trust Azure to pick indexes.
6. **For chronic regressions**, force the good plan via `sp_query_store_force_plan`.

**Trade-off**: Adding indexes speeds reads but slows writes and grows storage. Index recommendations should be reviewed, not blindly applied.

**Real project**: Our biggest win was a covering index on `Orders(CustomerId, CreatedAt) INCLUDE(Status, Total)` — cut a `/customers/{id}/orders` endpoint from 2.1s to 25ms by eliminating 200K key lookups per request.

## Summary Checklist

- [ ] I can pick between Single DB, Elastic Pool, and Managed Instance based on workload.
- [ ] I can pick a vCore tier (GP / Business Critical / Hyperscale / Serverless) with cost in mind.
- [ ] I can configure connection strings with `Authentication=Active Directory Default` and grant Managed Identity SQL access.
- [ ] I can enable Private Endpoint and disable public network access.
- [ ] I can configure EF Core with `EnableRetryOnFailure` and wrap explicit transactions in `ExecutionStrategy`.
- [ ] I can design an auto-failover group with appropriate RPO grace period.
- [ ] I can read Query Store and add a covering index to fix a slow query.
- [ ] I can run an online migration via Database Migration Service with minimal downtime.
- [ ] I can describe PITR and LTR backup policies and meet a compliance retention requirement.
- [ ] I can recognize the "user in master not in target DB" misconfiguration when Managed Identity login fails.
