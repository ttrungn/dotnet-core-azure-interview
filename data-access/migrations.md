# Migrations

## What It Is

A **migration** in EF Core is a versioned, code-generated description of a schema change — an `Up()` method that applies it and a `Down()` method that reverts it. EF Core compares the current entity model against the last snapshot, generates the difference (`CREATE TABLE`, `ADD COLUMN`, `ALTER COLUMN`, `DROP INDEX`), and writes a C# file plus a model snapshot. You apply migrations by running the generated SQL against the target database; a special `__EFMigrationsHistory` table records which migrations have already run.

Migrations turn the database schema into part of the codebase: a file in `src/Sales.Infrastructure/Migrations/20260601_AddShippingAddressToOrders.cs` lives next to the entity change that requires it, ships in the same commit, and is reviewed in the same PR.

```csharp
// Generated migration file
public partial class AddShippingAddressToOrders : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "ShippingAddress_Street",
            schema: "sales",
            table: "Orders",
            type: "nvarchar(200)",
            nullable: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "ShippingAddress_Street", schema: "sales", table: "Orders");
    }
}
```

## Why It Exists

Before migrations, .NET teams managed schema with hand-written SQL files numbered by date, tracked in a folder, and applied via DBA scripts. Predictable problems:

- The C# entity and the SQL change drifted; nobody knew which version of the schema matched which version of the code.
- Two engineers wrote conflicting `V42__add_column.sql` files and one silently overwrote the other.
- Production updates were a Friday-night ceremony involving the DBA, screenshots, and rollback Word docs.
- Rolling back a deployment required a separate `V42__rollback.sql` that nobody had tested.

EF Core migrations were built to:

- **Co-locate** the schema change and the code change so they ship together and roll back together.
- **Encode the change in C#** so the same migration runs against SQL Server, PostgreSQL, and SQLite via the provider (with provider-specific SQL on the back end).
- **Detect drift** through the model snapshot — if your entity changes don't match a pending migration, EF tells you.
- **Generate idempotent SQL** so the same script can be run multiple times in CI/CD without breaking.
- **Make rollback explicit** through `Down()` methods that ship with every migration.

## When To Use It

**Use EF Core migrations for:**

- Code-first projects where the entity model is the source of truth.
- Schema evolution across environments (dev → test → staging → production).
- Multi-environment deployments with CI/CD pipelines that need a deterministic, reviewable schema change.
- Multi-database support (the same model can produce SQL Server, PostgreSQL, and SQLite migrations via providers).
- Teams that want PR reviews of schema changes alongside code.

**Do not use EF Core migrations for:**

- **Database-first** projects where the schema is owned by a DBA team and you reverse-engineer entities with `dotnet ef dbcontext scaffold`.
- Massive data backfills — migrations can include `migrationBuilder.Sql("UPDATE ...")` but long-running backfills should be **separate background jobs**, not part of a deployment.
- Reorganizations that need downtime — design with the **expand–contract** pattern instead.
- Shared databases owned by multiple services — coordinate via a database-team owned tool (Flyway, Liquibase, SQL Server SSDT).

## Why It Is Important

Migrations are the bridge between code velocity and data safety. Done well, they let a team deploy 10 times a day without a DBA on call. Done poorly, they cause data loss, multi-hour outages, and rollback nightmares.

1. **Deterministic deployments** — every environment goes through the same sequence of migrations in the same order. There is no "production is mysteriously different" scenario.
2. **Auditability** — a migration is a Git commit; you can answer "when did this column appear?" in seconds.
3. **Reversibility** — `Down()` lets you undo a schema change when a deploy goes wrong.
4. **Zero-downtime deploys** — the expand–contract pattern (add nullable column, dual-write, backfill, switch reads, drop old column) lets you ship schema changes without taking the API offline.
5. **Drift prevention** — the model snapshot tells you when the model and the migrations no longer agree, before you ship.

In Azure (App Service, AKS, Container Apps), migrations are usually run **before** the new app version starts. The idempotent SQL script approach lets a release pipeline run the same script against staging and production without manual edits.

## How It's Used in C# / .NET

### 1. Install the tools

```bash
# Once per machine
dotnet tool install --global dotnet-ef

# In the project that has the DbContext
dotnet add package Microsoft.EntityFrameworkCore.Design
```

For Visual Studio Package Manager Console: `Install-Package Microsoft.EntityFrameworkCore.Tools` and use `Add-Migration`, `Update-Database`, `Script-Migration`.

### 2. Project layout that scales

Most teams put the `DbContext` and migrations in an `Infrastructure` project, separate from the API:

```
Sales.Domain/              // entities, value objects
Sales.Application/         // use cases
Sales.Infrastructure/      // SalesDbContext, configurations, Migrations/
Sales.Api/                 // Program.cs, controllers
```

Run migrations from the API's working directory but target the Infrastructure project:

```bash
dotnet ef migrations add AddShippingAddressToOrders `
  --project src/Sales.Infrastructure `
  --startup-project src/Sales.Api `
  --output-dir Migrations
```

### 3. Configure the migrations assembly

```csharp
builder.Services.AddDbContext<SalesDbContext>(o =>
    o.UseSqlServer(
        builder.Configuration.GetConnectionString("Sales"),
        sql => sql.MigrationsAssembly(typeof(SalesDbContext).Assembly.FullName)));
```

### 4. The everyday workflow

```bash
# 1. Change the entity (e.g., add ShippingAddress)
# 2. Generate the migration
dotnet ef migrations add AddShippingAddressToOrders --project Sales.Infrastructure --startup-project Sales.Api

# 3. Apply to your local dev database
dotnet ef database update --project Sales.Infrastructure --startup-project Sales.Api

# 4. Inspect the SQL it would generate (don't run yet)
dotnet ef migrations script --idempotent --output sql/2026.06.01-add-shipping-address.sql

# 5. Commit the migration file + model snapshot in the same PR as the entity change
```

### 6. Idempotent SQL for CI/CD

```bash
# All-time idempotent script — safe to run against any database
dotnet ef migrations script --idempotent --output deploy/all.sql

# Or only the migrations between two named versions
dotnet ef migrations script 20260501_PreviousMigration AddShippingAddressToOrders --idempotent --output deploy/delta.sql
```

What "idempotent" means: each migration is wrapped in `IF NOT EXISTS (SELECT 1 FROM __EFMigrationsHistory WHERE MigrationId = N'20260601...')` so re-running the script is a no-op for already-applied migrations.

```sql
IF NOT EXISTS(SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20260601_AddShippingAddressToOrders')
BEGIN
    ALTER TABLE [sales].[Orders] ADD [ShippingAddress_Street] nvarchar(200) NULL;
END;
GO

IF NOT EXISTS(SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20260601_AddShippingAddressToOrders')
BEGIN
    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20260601_AddShippingAddressToOrders', N'9.0.0');
END;
GO
```

### 7. `IDesignTimeDbContextFactory<T>` — for tooling

The `dotnet ef` tools need to instantiate the context. If your context construction depends on Azure App Configuration, Key Vault, or other runtime sources, give the tooling a deterministic path:

```csharp
public sealed class SalesDbContextDesignTimeFactory : IDesignTimeDbContextFactory<SalesDbContext>
{
    public SalesDbContext CreateDbContext(string[] args)
    {
        var config = new ConfigurationBuilder()
            .AddJsonFile("appsettings.Design.json", optional: false)
            .Build();

        var options = new DbContextOptionsBuilder<SalesDbContext>()
            .UseSqlServer(config.GetConnectionString("Sales"))
            .Options;

        return new SalesDbContext(options);
    }
}
```

### 8. Seeding reference data

Two options:

- **Model-level seeding** (built-in): runs at migration time, idempotent on PK.

  ```csharp
  modelBuilder.Entity<OrderStatus>().HasData(
      new { Id = 1, Name = "Pending" },
      new { Id = 2, Name = "Paid" },
      new { Id = 3, Name = "Shipped" });
  ```

  Best for tiny, fixed reference tables. Inserts/updates/deletes generated automatically.

- **Application-level seeding** (recommended for anything bigger): a `DbSeeder.SeedAsync(SalesDbContext db)` method called once at startup or as a separate CLI job. Easier to test, easier to update, doesn't pollute migration history.

### 9. Bulk SQL inside a migration

For column transformations that can't be expressed in the migration DSL:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(name: "CountryCode", table: "Customers", nullable: true);

    // Backfill in batches to avoid lock escalation
    migrationBuilder.Sql(@"
        DECLARE @batch INT = 5000;
        WHILE 1 = 1
        BEGIN
            UPDATE TOP (@batch) Customers
            SET CountryCode = UPPER(LEFT(Country, 2))
            WHERE CountryCode IS NULL AND Country IS NOT NULL;
            IF @@ROWCOUNT = 0 BREAK;
        END");

    migrationBuilder.AlterColumn<string>(name: "CountryCode", table: "Customers",
        type: "nvarchar(2)", nullable: false, defaultValue: "ZZ");
}
```

**Beware** long-running data migrations inside a deployment — they hold locks and can extend a deploy from seconds to hours. For big tables, prefer a separate backfill job.

### 10. API quick reference

| Command                                                  | Purpose                                          |
|----------------------------------------------------------|--------------------------------------------------|
| `dotnet ef migrations add <Name>`                        | Create a new migration                           |
| `dotnet ef migrations remove`                            | Delete the latest (unapplied) migration          |
| `dotnet ef migrations list`                              | Show all migrations and their applied state      |
| `dotnet ef migrations script [from] [to] --idempotent`   | Generate idempotent SQL                          |
| `dotnet ef migrations script --output bundle.sql`        | Plain SQL (non-idempotent)                       |
| `dotnet ef migrations bundle --self-contained`           | Single-file executable that applies migrations   |
| `dotnet ef database update [target]`                     | Apply migrations against the configured DB       |
| `dotnet ef database update 0`                            | Roll back to the empty database                  |
| `dotnet ef database drop`                                | Drop the database (dev only)                     |
| `dotnet ef dbcontext info` / `optimize` / `scaffold`     | Diagnostics, model compilation, db-first scaffolding |
| `IDesignTimeDbContextFactory<T>`                         | Provide a context to the EF tooling              |
| `MigrationBuilder.Sql("...")`                            | Run raw SQL inside a migration                   |
| `modelBuilder.Entity<T>().HasData(...)`                  | Built-in reference-data seeding                  |

## Advantages

- **Schema lives with code** — same PR, same Git history, same blame.
- **Reproducible across environments** — every environment runs the same script in the same order.
- **Reversible** — `Down()` enables a clean rollback when paired with a versioned deploy.
- **Provider-aware** — the same model produces SQL Server, PostgreSQL, or SQLite migrations.
- **Idempotent SQL** — safe to run multiple times in CI/CD without conditional logic.
- **Tooling integration** — Azure DevOps, GitHub Actions, EF migration bundles, `dotnet ef database update` all work from the same artifact.
- **PR-reviewable** — schema changes get the same review process as code changes.

## Disadvantages

- **Merge conflicts** — two migrations targeting the same model in parallel branches conflict in the model snapshot file.
- **Auto-generated SQL is not always optimal** — `ALTER COLUMN` rebuilds tables on some providers; indexes may need manual hints.
- **Backfills inside migrations are dangerous** — long locks during deployment.
- **`Down()` is rarely tested** — most teams never run it; rollback fantasies often don't survive contact with production.
- **Big-team coordination** — every PR that touches the model touches `SalesDbContextModelSnapshot.cs`, which always conflicts.
- **Limited to the EF model** — schema objects EF doesn't know about (views, stored procedures, sequences) need raw SQL.
- **No native zero-downtime support** — you must design migrations with expand–contract yourself.

## Common Mistakes

### 1. Auto-Apply at App Startup in Production

```csharp
// BUG: every replica races to migrate; one wins, others may fail or block on locks
var app = builder.Build();
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<SalesDbContext>();
    await db.Database.MigrateAsync(); // dangerous in production
}
```

**Fix**: generate an idempotent SQL script in CI, run it once from the pipeline before the new app version starts.

```bash
dotnet ef migrations script --idempotent --output deploy/migrate.sql
sqlcmd -S $sqlServer -d $sqlDatabase -i deploy/migrate.sql
```

### 2. Parallel Migrations on Different Branches

Two engineers add a migration on different branches; both create files numbered around the same timestamp; merging both produces a model snapshot conflict and (worse) two migrations that try to ALTER the same table.

**Fix**:
- Rebase before merging so only one migration sits at the tip of `main` at any time.
- Make migrations a small step at the end of a feature, not the first commit.
- Adopt a "migration PR" convention so the team can serialize schema work.

### 3. Editing an Applied Migration

Editing a migration after it has been deployed silently breaks every other environment — the migrations history table records that "AddShippingAddress" already ran, so the changed SQL is never re-applied.

**Fix**: never edit an applied migration. Add a new follow-up migration that adjusts the schema.

### 4. Destructive Migration Without Expand–Contract

```csharp
// BUG: renames a column in production while v1 of the app (still running on the other slot) writes to the old name
migrationBuilder.RenameColumn(name: "Total", newName: "GrandTotal", table: "Orders");
```

A `RENAME` is instant in the schema, but every old app instance trying to write to `Total` immediately errors.

**Fix**: expand–contract pattern.
1. Add `GrandTotal`, allow `NULL`. Deploy.
2. App dual-writes to both `Total` and `GrandTotal`. Deploy.
3. Backfill `GrandTotal` from `Total` in a background job.
4. App reads from `GrandTotal`. Deploy.
5. Remove `Total` in a follow-up migration.

### 5. Migrations With Massive Data Updates

A `migrationBuilder.Sql("UPDATE Orders SET ... ")` that touches 200 million rows holds locks for the entire transaction, blocks the app, and may time out the deploy script.

**Fix**: separate the schema change (small, instant) from the data update (long-running, batched, restartable, monitored).

### 6. Missing `IDesignTimeDbContextFactory<T>` for Custom DbContext

If your `Program.cs` builds the context with custom DI (e.g., reads from Azure App Configuration with Managed Identity), `dotnet ef` will fail with "Unable to create an object of type 'SalesDbContext'".

**Fix**: provide a design-time factory pointing at a local connection string.

### 7. Forgetting to Commit the Model Snapshot

The `SalesDbContextModelSnapshot.cs` is the source of truth for what EF thinks the schema looks like. Not committing it means the next engineer's `Add-Migration` will generate diffs for *every* model change since the snapshot was last updated — usually a huge migration full of unrelated changes.

**Fix**: always commit `*Snapshot.cs` together with the migration file.

### 8. Trying to Manage Views, Stored Procedures, or Triggers Through Migrations

EF Core's `MigrationBuilder` can run raw SQL but doesn't model views or stored procedures natively. Mixing them with auto-generated DDL leads to confusion.

**Fix**: keep DB-only objects in raw SQL files under source control, applied via the same pipeline step but in a separate folder (e.g., `deploy/views/`).

## Best Practices

- **Generate idempotent SQL scripts and run them from CI/CD** — never auto-apply migrations at app startup in production.
- **One pending migration per PR.** Rebase to keep `main` linear.
- **Design every migration to be backward-compatible** with the *previous* app version so blue/green deploys are safe. See [../devops/blue-green-deployment.md](../devops/blue-green-deployment.md).
- **Use expand–contract for destructive changes** (rename, drop column, change nullability).
- **Separate schema changes from data backfills.** Backfills should be background jobs.
- **Always commit the model snapshot** alongside the migration file.
- **Test the `Down()` method** at least once in a staging environment per release.
- **Review the generated SQL** before committing. Generated DDL can do surprising things on `ALTER COLUMN`.
- **Pin the EF Core tool version** in the repo (`dotnet-tools.json`) so local and CI generate identical SQL.
- **Track applied migrations** in your environment dashboards (`SELECT MigrationId FROM __EFMigrationsHistory`).
- **Use migration bundles** (`dotnet ef migrations bundle`) for environments where you can't install the SDK.

## Related Concepts

- **Entity Framework Core** — see [entity-framework-core.md](entity-framework-core.md).
- **DbContext Lifetime** — see [dbcontext-lifetime.md](dbcontext-lifetime.md). The design-time factory matters here.
- **Transactions** — see [transactions.md](transactions.md). Migrations themselves run inside transactions where the provider supports it.
- **CI/CD Pipelines** — see [../devops/ci-cd-pipelines.md](../devops/ci-cd-pipelines.md). Migration scripts are a deploy artifact.
- **Blue/Green Deployment** — see [../devops/blue-green-deployment.md](../devops/blue-green-deployment.md). The reason expand–contract exists.
- **Rollback Strategy** — see [../devops/rollback-strategy.md](../devops/rollback-strategy.md). Includes database rollback options.
- **Environment Configuration** — see [../devops/environment-configuration.md](../devops/environment-configuration.md).
- **Azure SQL** — see [../azure/azure-sql.md](../azure/azure-sql.md). Some commands need elevated permissions.
- **Outbox Pattern** — see [../architecture/outbox-pattern.md](../architecture/outbox-pattern.md). Often introduced via a single migration.

## Real-World Usage

### Azure DevOps Release Pipeline

A typical deploy stage:

```yaml
# .azure-pipelines/release.yml
- stage: Migrate
  jobs:
  - job: ApplyMigrations
    steps:
    - task: AzureCLI@2
      displayName: 'Run idempotent migration script against Azure SQL'
      inputs:
        scriptType: pscore
        scriptLocation: inlineScript
        inlineScript: |
          $token = az account get-access-token --resource https://database.windows.net/ --query accessToken -o tsv
          Invoke-SqlCmd `
            -ServerInstance "$(SqlServer).database.windows.net" `
            -Database "$(SqlDatabase)" `
            -AccessToken $token `
            -InputFile "$(Pipeline.Workspace)/drop/migrate.sql" `
            -ConnectionTimeout 30 `
            -QueryTimeout 600

- stage: Deploy
  dependsOn: Migrate
  jobs:
  - deployment: DeployApi
    ...
```

The Build stage generated `migrate.sql` via `dotnet ef migrations script --idempotent`. Release stages apply it before the new app version goes live.

### Expand–Contract Rename in Production

The team renamed `Order.Total` to `Order.GrandTotal`:

1. **Migration 1 (sprint 1):** add `GrandTotal` column, nullable. Application keeps writing only `Total`.
2. **Migration 2 (sprint 1, same deploy as code change):** application now dual-writes both `Total` and `GrandTotal`.
3. **Background job (sprint 2):** backfill `GrandTotal = Total` in batches of 10k overnight.
4. **Migration 3 (sprint 3):** make `GrandTotal` non-nullable; application now reads from `GrandTotal`.
5. **Migration 4 (sprint 4):** drop `Total`.

Five small, reversible deploys instead of one risky one. Zero downtime; no on-call incidents.

### Database-First Legacy Schema

For a legacy reporting database owned by another team, the project uses `dotnet ef dbcontext scaffold` to reverse-engineer entities and configurations from the live schema. Migrations are intentionally disabled (`OnConfiguring` throws if migrations are attempted) so EF Core never tries to manage the schema.

### Multi-Tenant SaaS

Each tenant has its own SQL database. The deploy pipeline runs the same migration script against every tenant database via a parallel job. New tenants are provisioned by spinning up a fresh database and running the all-time idempotent script.

## Code Example — Before and After

### Before: Auto-Apply at Startup, Single Big Migration

```csharp
// Program.cs
var app = builder.Build();
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<SalesDbContext>();
    db.Database.Migrate();   // every replica races; one wins, others may time out
}
app.Run();

// Migration file — rename + backfill + drop in one shot, blocking deploy
public partial class RenameTotalToGrandTotal : Migration
{
    protected override void Up(MigrationBuilder mb)
    {
        mb.RenameColumn("Total", "Orders", "GrandTotal");
        mb.Sql("UPDATE Orders SET GrandTotal = ISNULL(GrandTotal, 0)");
        // 200M rows; this UPDATE held locks for 47 minutes during the deploy
    }
    protected override void Down(MigrationBuilder mb) =>
        mb.RenameColumn("GrandTotal", "Orders", "Total");
}
```

Problems: 47-minute migration blocked the deploy. The other still-running app instance was hitting "Invalid column name 'Total'" errors the entire time. Rollback would have required another 47 minutes.

### After: Expand–Contract + CI/CD Script

```csharp
// Migration 1
public partial class AddGrandTotalColumn : Migration
{
    protected override void Up(MigrationBuilder mb) =>
        mb.AddColumn<decimal>("GrandTotal", "Orders", type: "decimal(18,2)", nullable: true);
    protected override void Down(MigrationBuilder mb) =>
        mb.DropColumn("GrandTotal", "Orders");
}

// In application code, dual-write phase
public void RecalculateTotals(Order order)
{
    order.Total = order.Lines.Sum(l => l.Quantity * l.UnitPrice);
    order.GrandTotal = order.Total;     // dual write
}

// Background job (sprint 2) backfills in 10k-row batches over 4 nights.

// Migration 3 (sprint 3)
public partial class MakeGrandTotalRequired : Migration
{
    protected override void Up(MigrationBuilder mb) =>
        mb.AlterColumn<decimal>("GrandTotal", "Orders", type: "decimal(18,2)", nullable: false);
    protected override void Down(MigrationBuilder mb) =>
        mb.AlterColumn<decimal>("GrandTotal", "Orders", type: "decimal(18,2)", nullable: true);
}

// Migration 4 (sprint 4)
public partial class DropOldTotal : Migration
{
    protected override void Up(MigrationBuilder mb) => mb.DropColumn("Total", "Orders");
    protected override void Down(MigrationBuilder mb) =>
        mb.AddColumn<decimal>("Total", "Orders", type: "decimal(18,2)", nullable: true);
}
```

Each migration is small, reversible, and deployable without taking the API offline. The CI pipeline runs:

```bash
dotnet ef migrations script --idempotent --output deploy/migrate.sql
```

…and the release pipeline runs `migrate.sql` once, before the new app instances start.

## Interview Questions and Answers

### 1. Walk me through how you ship a schema change to production.

**Why this matters**: Tests whether the candidate has actually shipped EF Core migrations in a real environment.

**Answer**:
1. Change the entity in the Infrastructure project.
2. Run `dotnet ef migrations add MeaningfulName` to generate the migration file and updated snapshot.
3. Inspect the generated SQL (`dotnet ef migrations script --idempotent`) and the auto-generated `Up()`/`Down()` — make sure the change is what I expected.
4. Apply locally (`dotnet ef database update`) and run the test suite.
5. Open a PR with the entity change, the migration file, and the snapshot in one commit.
6. CI generates `migrate.sql` from the new migrations and stores it as a build artifact.
7. The release pipeline runs `sqlcmd -i migrate.sql` against staging, runs the test suite there, then promotes the artifact to production where it runs the same script before swapping app slots.
8. Never auto-apply migrations from app startup in production.

**Trade-off**: The script-based approach adds a CI step but removes the "racing replicas" failure mode of `Database.Migrate()` at startup.

**Real project**: A team that previously used `Database.Migrate()` at startup had three replicas race to apply the same migration and produce three different error messages in App Insights. Moving to idempotent scripts in the release pipeline eliminated the class of bug.

### 2. Two engineers each added a migration on separate branches. How do you resolve the conflict?

**Answer**: Pick a winner. Have the second engineer:
1. Pull the latest `main` (with the other migration applied locally).
2. Run `dotnet ef migrations remove` to drop their unapplied migration.
3. Re-run `dotnet ef migrations add <Name>` so the new migration is generated against the updated snapshot.
4. Push the rebased PR.

This serializes schema changes through the snapshot file. If the migrations touch unrelated tables, you can manually keep both files — but you must always reconcile `SalesDbContextModelSnapshot.cs`.

**Trade-off**: Forces serialization on schema work, which is annoying but ultimately safer than two migrations fighting over the same model snapshot.

**Real project**: We adopted a "one open schema PR at a time" rule on a team of 12 engineers. Conflicts dropped to zero.

### 3. Explain expand–contract for a column rename and why you'd use it.

**Answer**: Expand–contract splits a destructive change into multiple backward-compatible steps so the old and new app versions can coexist during deployment.

For renaming `Total` → `GrandTotal`:
- **Expand**: add `GrandTotal` (nullable). The new app version dual-writes both columns; the old version still reads/writes `Total`. Deploy is safe.
- **Backfill**: a background job populates `GrandTotal` from `Total` in batches.
- **Switch reads**: app now reads from `GrandTotal`. Deploy.
- **Contract**: drop `Total`. Deploy.

Why: a blue/green or rolling deploy means two app versions run at the same time. A single-step rename breaks the version that hasn't been updated yet. Expand–contract gives every running version a schema it understands.

**Real project**: A column rename done in one migration took the API down for 8 minutes during the rolling deploy as the old pods threw "Invalid column" errors. The team adopted expand–contract afterward.

### 4. When would you write raw SQL inside a migration?

**Answer**: When the EF DSL doesn't cover what you need — typically:
- Filtered indexes (`CREATE INDEX ... WHERE ...`)
- Stored procedures, functions, triggers, views.
- Schema objects EF doesn't model (synonyms, partition functions).
- Small reference-data updates that aren't worth a separate seeder.

For anything **long-running** (backfilling 50M rows, rebuilding an index), prefer a **separate background job** that's monitored, restartable, and tunable for throughput — keep the migration small and instant.

**Trade-off**: Raw SQL is provider-specific. If you ever swap SQL Server for PostgreSQL, you'll have to rewrite. Hide SQL behind a `migrationBuilder.Sql(...)` call so it's at least centralized in the migration file.

### 5. What's the difference between `Update-Database` and the idempotent SQL script?

**Answer**:
- **`Update-Database` / `dotnet ef database update`** connects EF directly to the configured database and applies any pending migrations. Convenient for local dev. **Bad for production**, because it requires the SDK and the connection string on the deploy host, and it can't be reviewed before it runs.
- **Idempotent SQL script** (`dotnet ef migrations script --idempotent`) generates a plain SQL file that:
  - Can be reviewed and stored as a release artifact.
  - Is safe to run multiple times (idempotency comes from `IF NOT EXISTS` checks against `__EFMigrationsHistory`).
  - Runs from any environment that has `sqlcmd` or psql — no SDK needed.
  - Can be applied by a DBA, audited, signed off.

The CI/CD friendly choice is always the idempotent script.

### 6. How do you handle reference-data seeding?

**Answer**: Two approaches, picked by data shape:

1. **Built-in `HasData(...)`** for tiny, fixed lookup tables (order statuses, currencies, country ISO codes). EF Core generates `INSERT` / `UPDATE` / `DELETE` statements as migrations, so the data is versioned.
2. **Application-level seeder** for larger or environment-specific data (demo products, integration test data). A `DbSeeder.SeedAsync()` method called from a one-off CLI command, idempotent on natural keys.

Avoid mixing the two for the same table — they fight each other. And do not use migration-time seeding for anything that changes more than a few times a year; otherwise every change becomes a migration churn.

**Real project**: A team used `HasData` to seed 800 product categories. Every minor change spawned a migration with thousands of generated UPDATE lines. They moved categories to an application seeder and reduced the migration noise by 90%.

### 7. How do you roll back a bad migration?

**Answer**: It depends on whether you've already shipped data changes that depend on the new schema:

- **Schema-only change, no data dependency**: roll the app back to the previous version (your CD pipeline supports this), then run `dotnet ef database update <PreviousMigrationName>` (locally generates the rollback script).
- **Data already written to a new column**: you usually can't truly "undo" — you have to write a **forward fix** migration that restores backward compatibility (re-add the column, copy data back) and roll the app back.
- **Destructive deletes (DROP COLUMN, DROP TABLE)**: there's no undo. Restore from a point-in-time backup or replay the data from an event store / outbox.

This is why expand–contract matters: the "expand" steps are easy to roll back; the "contract" steps are scheduled at the end of a multi-deploy plan, well after the change has proven stable.

**Trade-off**: A rollback that requires restoring from backup may cost an SLA breach. Plan migrations to be reversible by design and never test rollback for the first time in production.

### 8. How would you migrate a legacy database with no migrations history into EF Core?

**Answer**:
1. Generate the entity model from the existing schema:
   ```bash
   dotnet ef dbcontext scaffold "Server=...;Database=Legacy;Trusted_Connection=True" Microsoft.EntityFrameworkCore.SqlServer --output-dir Entities --context LegacyDbContext
   ```
2. Review and clean up the generated `DbContext` — rename ugly columns via `[Column]` or `IEntityTypeConfiguration<T>`, drop properties you don't need.
3. Create an **initial migration** that represents the *current* state:
   ```bash
   dotnet ef migrations add InitialBaseline --project Legacy.Infrastructure
   ```
4. Apply this migration in every environment by inserting the row into `__EFMigrationsHistory` manually (or by running the idempotent script against a *non-empty* DB — the script will skip the actual DDL because the tables already exist, but record the migration in the history table).
5. From this point onward, every schema change goes through a normal migration.

**Trade-off**: The initial baseline migration won't represent every quirk of the legacy schema (custom collations, complex defaults) — keep those in raw SQL files alongside the code-first migrations.

## Summary Checklist

- [ ] I can generate, apply, and roll back a migration via `dotnet ef`.
- [ ] I can produce an idempotent SQL script and explain why it's preferred over auto-applying at startup.
- [ ] I can describe expand–contract and apply it to a column rename.
- [ ] I can resolve parallel migration conflicts on a team.
- [ ] I can provide an `IDesignTimeDbContextFactory<T>` when the runtime DI configuration prevents tooling from running.
- [ ] I can choose between `HasData` and an application seeder for reference data.
- [ ] I can integrate migrations into an Azure DevOps / GitHub Actions release pipeline.
- [ ] I can identify migrations that hold long locks and split them into schema + background backfill.
- [ ] I can never edit an already-applied migration; I add a follow-up migration instead.
- [ ] I can recover from a bad migration with a forward-fix migration plus a versioned app rollback.
