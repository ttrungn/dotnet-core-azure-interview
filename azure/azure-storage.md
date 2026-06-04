# Azure Storage

## What It Is

Azure Storage is the **durable, multi-protocol storage platform** under most Azure services. A single **Storage Account** provides four distinct data services that share the account's redundancy, encryption, and identity model:

1. **Blob Storage** — object storage for unstructured data (PDFs, images, videos, backups, logs). Three blob types: **block blobs** (most common — files), **append blobs** (log writes), **page blobs** (VM disks).
2. **Queue Storage** — simple FIFO-ish messaging up to 64 KB per message. Cheap, basic; lacks Service Bus features (sessions, dead-letter, transactions).
3. **Table Storage** — legacy schemaless NoSQL key/value store. Mostly superseded by **Cosmos DB Table API**; Table is still cheap for audit logs.
4. **File Storage** — fully managed SMB / NFS file shares. Used when an app needs a file-system mount instead of an object API.

```text
┌──────────────── Storage Account (orderstore01) ────────────────┐
│   Redundancy: ZRS    Tier: Hot    Encryption: MSFT-managed     │
│   Hierarchical namespace: false   Soft-delete: 30 days         │
│                                                                 │
│   ┌──── Blob Service ────────┐   ┌──── Queue Service ────┐    │
│   │  invoices/              │   │  email-notifications   │    │
│   │    2026/01/inv_123.pdf  │   │  retry-failures        │    │
│   │  uploads/               │   └────────────────────────┘    │
│   │  backups/               │   ┌──── Table Service ─────┐    │
│   │  exports/               │   │  audit-log              │    │
│   └────────────────────────┘   └────────────────────────┘    │
│   ┌──── File Service ────────────────────────────────────┐    │
│   │   shared-config (SMB mount for legacy apps)          │    │
│   └──────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────┘
```

For modern .NET workloads, **Blob is what you'll use 95% of the time**.

## Why It Exists

Before Azure Storage, storing a customer-uploaded invoice meant:

- Provisioning a file server or NAS with enough capacity for years of growth.
- Designing backup/replication so a disk failure doesn't lose data.
- Configuring access control via NTFS ACLs or a custom DB.
- Hosting a CDN separately for downloads.

Azure Storage exists as **commodity, planet-scale storage** with:

- **11 nines of durability** for ZRS (`99.999999999%`).
- **Pay-per-GB** pricing — no over-provisioning.
- **Multiple redundancy tiers** (LRS, ZRS, GRS, GZRS) for geo-DR.
- **HTTP-based access** so any language/client can use it, including from browser via SAS or Entra ID.
- **Lifecycle management** to age data from Hot → Cool → Cold → Archive automatically.
- **Native integration** with Event Grid, Functions, CDN, Front Door.

## When To Use It

**Use Blob Storage for:**

- **User-uploaded files**: invoices (PDF), profile pictures, attachments, exported reports.
- **Backups and archives**: SQL DB exports, log archives, machine images.
- **Static website assets**: HTML/CSS/JS served from `$web` container, fronted by CDN.
- **Data lake** for ETL pipelines (Hierarchical Namespace enabled = **ADLS Gen2**).
- **Cold storage for compliance** (Archive tier for 7-year retention at $0.00099/GB/month).

**Use Queue Storage for:**

- Simple, cheap fire-and-forget messages between two .NET services when you don't need Service Bus features.

**Use Table Storage for:**

- High-volume, write-once audit logs where Cosmos DB cost is prohibitive.
- Simple key-value lookups.

**Use File Storage for:**

- Lifting legacy on-prem apps to Azure that require an SMB mount (shared config, legacy report folders).

**Do not use Azure Storage for:**

- **Transactional data needing relational queries** — use [azure-sql.md](azure-sql.md).
- **Pub/sub or topic-based messaging** — use [azure-service-bus.md](azure-service-bus.md) or Event Grid.
- **Low-latency document database** — use Cosmos DB.
- **Caching** — use Azure Cache for Redis.

## Why It Is Important

Storage is the foundation of most Azure architectures. In interviews you'll be asked:

1. **Tier selection** — Hot vs Cool vs Cold vs Archive. Wrong tier on 50 TB of data = thousands of dollars/month wasted or unreachable.
2. **Redundancy** — LRS vs ZRS vs GRS. Critical to RPO and DR strategy.
3. **Auth model** — SAS tokens (legacy, secret-leaking) vs **Managed Identity with RBAC** (modern, secret-less).
4. **Network** — private endpoint, firewall, disabling key-based auth.
5. **Event-driven pipelines** — Blob Created event → Event Grid → Function → process upload.

A senior .NET engineer should be able to design "user uploads invoice" → "stored in Blob with Managed Identity" → "Event Grid fires" → "Function extracts text" → "metadata indexed in SQL" without reaching for keys or connection strings.

## How It's Used in C# / .NET

### 1. Register `BlobServiceClient` with Managed Identity

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Microsoft.Extensions.Azure;

var builder = WebApplication.CreateBuilder(args);
var credential = new DefaultAzureCredential();

builder.Services.AddAzureClients(clients =>
{
    clients.UseCredential(credential);

    clients.AddBlobServiceClient(new Uri("https://orderstore01.blob.core.windows.net"))
           .ConfigureOptions(o =>
           {
               o.Retry.MaxRetries = 5;
               o.Retry.Mode = Azure.Core.RetryMode.Exponential;
               o.Retry.Delay = TimeSpan.FromMilliseconds(200);
           });

    clients.AddQueueServiceClient(new Uri("https://orderstore01.queue.core.windows.net"));
    clients.AddTableServiceClient(new Uri("https://orderstore01.table.core.windows.net"));
});
```

No connection string, no SAS — just the URI and `DefaultAzureCredential`.

### 2. Upload an invoice PDF

```csharp
public sealed class InvoiceStorage(BlobServiceClient client)
{
    public async Task<Uri> StoreAsync(
        Guid invoiceId,
        Stream content,
        string contentType,
        CancellationToken ct)
    {
        var container = client.GetBlobContainerClient("invoices");
        await container.CreateIfNotExistsAsync(cancellationToken: ct);

        var blob = container.GetBlobClient($"{DateTime.UtcNow:yyyy/MM}/{invoiceId}.pdf");

        var options = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders { ContentType = contentType },
            Metadata = new Dictionary<string, string>
            {
                ["invoiceId"] = invoiceId.ToString(),
                ["uploadedAt"] = DateTime.UtcNow.ToString("O")
            },
            Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All }  // don't overwrite
        };

        await blob.UploadAsync(content, options, ct);
        return blob.Uri;
    }
}
```

### 3. Generate a time-limited download URL via User Delegation SAS

User Delegation SAS is signed with the App Service's Managed Identity, **not** the storage account key. No secret to leak.

```csharp
public async Task<Uri> GetDownloadUrlAsync(
    Guid invoiceId,
    string blobPath,
    CancellationToken ct)
{
    var container = _client.GetBlobContainerClient("invoices");
    var blob = container.GetBlobClient(blobPath);

    // Request a delegation key from Azure AD valid for 1 hour
    var userDelegationKey = await _client.GetUserDelegationKeyAsync(
        startsOn: DateTimeOffset.UtcNow,
        expiresOn: DateTimeOffset.UtcNow.AddHours(1),
        cancellationToken: ct);

    var builder = new BlobSasBuilder
    {
        BlobContainerName = container.Name,
        BlobName = blob.Name,
        Resource = "b",
        ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15)
    };
    builder.SetPermissions(BlobSasPermissions.Read);

    var sas = builder.ToSasQueryParameters(userDelegationKey, _client.AccountName).ToString();
    return new Uri($"{blob.Uri}?{sas}");
}
```

### 4. Blob-triggered Function (audit ingest)

```csharp
public class IndexInvoiceUpload(IInvoiceMetadataStore store)
{
    [Function("IndexInvoiceUpload")]
    public async Task Run(
        [BlobTrigger("invoices/{name}", Source = BlobTriggerSource.EventGrid)]
        Stream blob,
        string name,
        CancellationToken ct)
    {
        await store.IndexAsync(name, blob.Length, ct);
    }
}
```

Event Grid source (vs the legacy polling) is essential in production — polling can take up to 10 minutes to fire for the first blob in an empty container.

### 5. Lifecycle management — auto-archive old invoices

```bicep
resource lifecycle 'Microsoft.Storage/storageAccounts/managementPolicies@2023-05-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    policy: {
      rules: [
        {
          name: 'invoice-tiering'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: { blobTypes: ['blockBlob'], prefixMatch: ['invoices/'] }
            actions: {
              baseBlob: {
                tierToCool:    { daysAfterModificationGreaterThan: 30 }
                tierToCold:    { daysAfterModificationGreaterThan: 90 }
                tierToArchive: { daysAfterModificationGreaterThan: 365 }
                delete:        { daysAfterModificationGreaterThan: 2555 }   // 7 years
              }
            }
          }
        }
      ]
    }
  }
}
```

### 6. Append blob for high-volume audit log

```csharp
var appendBlob = container.GetAppendBlobClient($"audit/{DateOnly.FromDateTime(DateTime.UtcNow):yyyy-MM-dd}.log");
await appendBlob.CreateIfNotExistsAsync(cancellationToken: ct);

await using var stream = new MemoryStream(Encoding.UTF8.GetBytes(jsonLine + "\n"));
await appendBlob.AppendBlockAsync(stream, cancellationToken: ct);
```

### 7. Storage tier cheat sheet

| Tier      | Storage cost / GB / month | Read cost / 10K | Write cost / 10K | Early deletion |
|-----------|---------------------------|------------------|-------------------|----------------|
| Hot       | $0.0184                   | $0.0043          | $0.065            | None           |
| Cool      | $0.01                     | $0.01            | $0.13             | 30 days        |
| Cold      | $0.0036                   | $0.06            | $0.18             | 90 days        |
| Archive   | $0.00099                  | $5 + rehydration | $0.103            | 180 days       |

*(Prices vary by region; this is east US as of 2025; check the Pricing Calculator for current rates.)*

## Advantages

- **11 nines of durability** for ZRS — your files don't get lost.
- **Massive scale** — up to 5 PiB per storage account, 500 TPS per container.
- **Multiple redundancy tiers** — LRS (single zone), ZRS (3 zones), GRS (cross-region async), GZRS (3 zones + cross-region).
- **Multiple tiers** — match cost to access pattern.
- **Managed Identity + RBAC** — no shared keys to rotate.
- **Lifecycle management** — automatic tiering without custom jobs.
- **Event Grid integration** — push notifications on Blob created/deleted in <1 sec.
- **Static website hosting** + CDN/Front Door integration.
- **First-class Azure SDK** with retry, pooling, and streaming support.

## Disadvantages

- **Storage account is a single failure boundary** for keys, SAS, network rules, and DNS.
- **Table Storage is legacy** — slower than Cosmos DB Table API, weaker SLA.
- **Queue Storage lacks Service Bus features** (no sessions, no topics, no transactions, no auto-DLQ).
- **Archive tier has rehydration latency** — up to 15 hours (standard) or 1 hour (high priority).
- **Egress costs** — downloading 1 TB out of Azure costs ~$80; design with CDN.
- **No relational queries** — search by metadata requires an external index (Azure AI Search, Cosmos DB).
- **GRS RPO is ~15 min** — failover can lose recent writes.
- **Some features require Hierarchical Namespace (ADLS Gen2)** but HNS can't be turned on after creation.
- **Vendor lock-in** — SAS, Event Grid integration, lifecycle policies don't transfer.

## Common Mistakes

### 1. Connection string + access key in code

```csharp
// BUG: key in source, no rotation, anyone with the key has full storage account access
var conn = "DefaultEndpointsProtocol=https;AccountName=orderstore01;AccountKey=ABCD...";
var client = new BlobServiceClient(conn);
```

**Fix**: use Managed Identity:

```csharp
var client = new BlobServiceClient(
    new Uri("https://orderstore01.blob.core.windows.net"),
    new DefaultAzureCredential());
```

Grant the identity `Storage Blob Data Contributor` on the container (least privilege).

### 2. Distributing the account key as a SAS to clients

```csharp
// BUG: SAS signed with the account key. If the key rotates, SAS breaks. If the SAS leaks,
// you must rotate the key (breaking every other SAS) to invalidate.
var sas = blob.GenerateSasUri(BlobSasPermissions.Read, DateTime.UtcNow.AddHours(1));
```

**Fix**: use **User Delegation SAS** (signed with Managed Identity / Entra ID):

```csharp
var key = await _client.GetUserDelegationKeyAsync(start, expiry);
var sas = builder.ToSasQueryParameters(key, _client.AccountName);
```

You can revoke a User Delegation SAS by rotating the user delegation key, without affecting other SAS.

### 3. Picking Hot tier for backups

```text
"We're storing 50 TB of weekly backups in Hot. Why is our bill $900/month?"
```

Hot is $0.0184/GB; Cool is $0.01; Cold is $0.0036; Archive is $0.001. **Fix**: lifecycle rules to age data — typically Hot for 30 days, Cool for 90, Cold for a year, Archive thereafter.

### 4. Reusing object names → no soft-delete recovery

```csharp
// BUG: overwrites blob; the old version is gone
await blob.UploadAsync(stream);
```

**Fix**: enable **versioning** on the container, or use unique names (`{guid}-{filename}.pdf`). Soft-delete only catches accidental deletes, not overwrites — versioning catches both.

### 5. Public blob container

```bicep
// BUG: anyone with the URL can read every blob
properties: { publicAccess: 'Container' }
```

**Fix**: set `publicAccess: 'None'`. Issue time-limited SAS for legitimate downloads. For static websites, use CDN with origin authentication.

### 6. Missing retries on transient failures

```csharp
// BUG: a transient 503 throws, request fails
await blob.UploadAsync(stream);
```

The Azure SDK retries by default; if you've disabled it, re-enable:

```csharp
clients.AddBlobServiceClient(...).ConfigureOptions(o =>
{
    o.Retry.MaxRetries = 5;
    o.Retry.Mode = RetryMode.Exponential;
});
```

### 7. Streaming entire files into memory

```csharp
// BUG: 2 GB upload OOMs the App Service instance
var ms = new MemoryStream();
await request.Body.CopyToAsync(ms);
ms.Position = 0;
await blob.UploadAsync(ms);
```

**Fix**: stream directly:

```csharp
await blob.UploadAsync(request.Body, overwrite: false, ct);
```

For files > 100 MB, use parallel block upload (the SDK does this automatically when given a stream).

## Best Practices

- **Use Managed Identity + RBAC for all server-to-storage access** — disable account-key auth at the account level (`allowSharedKeyAccess: false`).
- **Use User Delegation SAS** for time-limited client downloads.
- **Enable soft-delete** for blobs and containers (30 days standard).
- **Enable versioning** for any container holding files that get re-uploaded.
- **Use ZRS or GZRS** for production — LRS loses data if the zone goes down.
- **Apply lifecycle rules** to age data automatically.
- **Set firewall rules + private endpoint** for production; disable public network access.
- **Use Event Grid** (not polling) for Blob-triggered Functions.
- **Front Blob with CDN/Front Door** for high-traffic downloads — reduces egress and latency.
- **Tag blobs with `metadata`** for cheap filtering (`x-ms-meta-invoiceid`) instead of parsing names.
- **Monitor diagnostic logs** (StorageRead, StorageWrite, StorageDelete) sent to Log Analytics.

## Related Concepts

- **Service Bus** ([azure-service-bus.md](azure-service-bus.md)) — proper enterprise messaging vs Queue Storage.
- **Azure Functions** ([azure-functions.md](azure-functions.md)) — Blob/Queue triggers.
- **Key Vault** ([azure-key-vault.md](azure-key-vault.md)) — for storing any remaining account keys you can't eliminate.
- **Application Insights** ([application-insights.md](application-insights.md)) — dependency tracking for blob reads/writes.
- **CDN / Front Door** — global delivery of public blobs.
- **Azure AI Search** — index blob content for search.
- **Outbox pattern** ([../architecture/outbox-pattern.md](../architecture/outbox-pattern.md)) — write metadata to SQL and blob URI atomically.

## Real-World Usage

### Invoice storage for an order API

- **Account**: ZRS, Hot tier, soft-delete 30 days, versioning enabled.
- **Container**: `invoices`, private (no public access).
- **Path scheme**: `{yyyy}/{MM}/{invoiceId}.pdf` for natural lifecycle eligibility.
- **Auth**: App Service Managed Identity with `Storage Blob Data Contributor` on the container.
- **Lifecycle**: Hot 30d → Cool 90d → Cold 1y → Archive 7y → Delete (matches finance retention policy).
- **Downloads**: User Delegation SAS, 15-min expiry, generated on request.
- **Cost (10K invoices, 200 KB each = 2 GB)**: ~$0.04/month storage + reads.

### High-volume audit log

- **Type**: Append Blob, one blob per day (`audit/2026-06-04.log`).
- **Writers**: Multiple App Service instances append concurrently using `AppendBlockAsync` with `IfMaxSizeLessThanOrEqual` precondition to bound size.
- **Lifecycle**: Cool after 30 days, Archive after 1 year.

### Static website + CDN

- **Storage**: `$web` container, public access, hosts a single-page React app.
- **Front Door**: Provides custom domain, TLS, WAF rules.
- **Cost**: $1/month for storage, ~$5/month for Front Door Standard.

### Backup retention

- **Source**: nightly SQL DB COPY_ONLY backup uploaded as `.bacpac` to `backups/sql/`.
- **Tier**: Cold (10x cheaper than Hot for write-once data).
- **Lifecycle**: Archive after 90 days, delete after 7 years for compliance.

## Code Example — Before and After

### Before: account key, no retries, OOM-prone upload

```csharp
// BUG-heavy implementation
public class InvoiceUploader
{
    private const string ConnStr =
        "DefaultEndpointsProtocol=https;AccountName=orderstore01;AccountKey=ABC123==;EndpointSuffix=core.windows.net";

    public async Task<string> UploadAsync(Guid id, IFormFile file)
    {
        var client = new BlobServiceClient(ConnStr);
        var container = client.GetBlobContainerClient("invoices");
        await container.CreateIfNotExistsAsync(PublicAccessType.Blob);   // public!

        var blob = container.GetBlobClient($"{id}.pdf");

        var ms = new MemoryStream();
        await file.CopyToAsync(ms);            // loads entire file into memory
        ms.Position = 0;
        await blob.UploadAsync(ms);            // overwrites any existing blob

        return blob.Uri.ToString();            // public URL anyone can crawl
    }
}
```

Problems:
- Account key in code (rotation breaks everything).
- Public container — invoices searchable by URL guessing.
- Full file loaded into memory.
- No retries, no metadata, no content-type set.
- Overwrite without versioning.

### After: Managed Identity, streaming, private, retry, SAS for download

```csharp
public sealed class InvoiceUploader(BlobServiceClient client, ILogger<InvoiceUploader> log)
{
    public async Task<Uri> UploadAsync(
        Guid invoiceId,
        Stream content,
        string contentType,
        long contentLength,
        CancellationToken ct)
    {
        var container = client.GetBlobContainerClient("invoices");
        var blobName = $"{DateTime.UtcNow:yyyy/MM}/{invoiceId}.pdf";
        var blob = container.GetBlobClient(blobName);

        var options = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders { ContentType = contentType },
            Metadata = new Dictionary<string, string>
            {
                ["invoiceId"]   = invoiceId.ToString(),
                ["uploadedAt"]  = DateTime.UtcNow.ToString("O")
            },
            Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All }   // fail if exists
        };

        // SDK uses parallel block upload automatically for streams > 256 KB
        await blob.UploadAsync(content, options, ct);

        log.LogInformation("Invoice {InvoiceId} stored at {BlobName} ({Size} bytes)",
            invoiceId, blobName, contentLength);

        return blob.Uri;
    }

    public async Task<Uri> GetSecureDownloadUrlAsync(
        string blobPath,
        TimeSpan validFor,
        CancellationToken ct)
    {
        var blob = client.GetBlobContainerClient("invoices").GetBlobClient(blobPath);

        var key = await client.GetUserDelegationKeyAsync(
            DateTimeOffset.UtcNow,
            DateTimeOffset.UtcNow.Add(validFor.Add(TimeSpan.FromMinutes(5))),
            ct);

        var sas = new BlobSasBuilder
        {
            BlobContainerName = blob.BlobContainerName,
            BlobName          = blob.Name,
            Resource          = "b",
            ExpiresOn         = DateTimeOffset.UtcNow.Add(validFor)
        };
        sas.SetPermissions(BlobSasPermissions.Read);

        var sasToken = sas.ToSasQueryParameters(key, client.AccountName).ToString();
        return new Uri($"{blob.Uri}?{sasToken}");
    }
}
```

Bicep:

```bicep
resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'orderstore01'
  location: location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    allowSharedKeyAccess: false               // forces Entra ID auth
    publicNetworkAccess: 'Disabled'
    minimumTlsVersion: 'TLS1_2'
    networkAcls: { defaultAction: 'Deny' }
  }
}

resource blobSvc 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: sa
  name: 'default'
  properties: {
    deleteRetentionPolicy: { enabled: true, days: 30 }
    isVersioningEnabled: true
    containerDeleteRetentionPolicy: { enabled: true, days: 30 }
  }
}
```

## Interview Questions and Answers

### 1. Compare Blob, Queue, Table, and File storage. When would you use each?

**Why this matters**: Storage account is one feature, four very different services. Knowing the trade-offs avoids reaching for the wrong one.

**Answer**:

- **Blob** — object storage for unstructured data. Use for files: invoices, images, backups, exports. 95% of new code uses Blob.
- **Queue** — simple FIFO-ish messaging, 64 KB max message size. Use only when you need basic queues and don't want Service Bus's cost/complexity. Lacks sessions, topics, DLQ semantics.
- **Table** — legacy schemaless KV store. Use only for cheap, simple lookups; otherwise use Cosmos DB Table API for the same surface with better SLAs.
- **File** — SMB/NFS file shares. Use only when an app needs a literal mount point and can't be changed to use Blob.

**Trade-off**: Putting messages in Blob (via blob events) gives you a permanent log; putting them in Queue gives you fast pop-once semantics. Different patterns.

**Real project**: We use Blob for invoices, Queue Storage for the simple "send this email" worker queue (low volume, basic), Cosmos DB Table API for our high-throughput audit log (replacing Table Storage), and File Storage on one legacy ERP that mounts a network drive.

### 2. Compare Hot, Cool, Cold, and Archive tiers.

**Answer**:

| Tier    | Storage cost | Access cost | Min duration | Use case                                  |
|---------|--------------|-------------|--------------|-------------------------------------------|
| Hot     | Highest      | Lowest      | None         | Active data, read many times per month    |
| Cool    | ~50% of Hot  | 2x Hot      | 30 days      | Read once a month or so                   |
| Cold    | ~80% off Hot | Higher      | 90 days      | Read once a quarter                       |
| Archive | ~95% off Hot | Very high + rehydration latency | 180 days | Compliance archive |

**Trade-off**: Each tier has a **minimum retention** — moving an Archive blob before 180 days incurs an early deletion fee. Lifecycle rules automate the moves.

**Real project**: We pay $4/month total for 7 years of invoice retention by lifecycle-rolling to Archive after a year. Reading an old invoice requires a 1-15 hour rehydration; finance is okay with that.

### 3. Walk me through a secure file-upload flow without storage keys.

**Answer**: 

1. Client `POST /invoices/{id}/upload` to the API with auth (JWT).
2. API verifies user authorization, generates a **User Delegation SAS** (signed by App Service Managed Identity) granting **Create** permission on `invoices/{date}/{id}.pdf` for 10 minutes.
3. API returns the SAS URL to the client.
4. Client uploads directly to Blob using the SAS — does **not** route bytes through the API.
5. Event Grid fires on `Microsoft.Storage.BlobCreated` → a Function indexes metadata into SQL.

Benefits: no account key anywhere, App Service doesn't proxy multi-GB uploads, SAS revocable via user delegation key rotation, finely scoped (one path, one permission, short expiry).

**Trade-off**: Slightly more client complexity than "POST file to API." Trade is worth it for any file > 10 MB.

**Real project**: Our customers upload 100-200 MB lab reports. The direct-to-Blob pattern means our App Service never sees the bytes — saves bandwidth, memory, and SNAT ports.

### 4. Compare LRS, ZRS, GRS, and GZRS redundancy.

**Answer**:

- **LRS** (Locally redundant): 3 copies in one data center. Lowest cost; loses data if the DC burns down. Dev/test only.
- **ZRS** (Zone redundant): 3 copies across 3 availability zones in one region. Survives a zone failure. Production minimum.
- **GRS** (Geo redundant): LRS in primary + async replication to paired region. Cross-region DR with RPO ~15 min.
- **GZRS** (Geo-zone redundant): ZRS in primary + async to paired region. Best of both.

Pair with **read-access** variants (RA-GRS, RA-GZRS) to read from secondary directly (eventually consistent).

**Trade-off**: GZRS costs ~2x ZRS. Don't pay for cross-region if your RPO/RTO doesn't require it.

**Real project**: User-generated content (invoices) on GZRS so a region outage doesn't lose recent uploads. Static website assets on ZRS (rebuildable from source).

### 5. A blob was accidentally overwritten with corrupt data. How do you recover?

**Answer**:

1. If **versioning is enabled** — list versions (`?versionid=...`) and copy the previous version back. Done.
2. If only **soft-delete is enabled** — soft-delete only catches deletions, not overwrites. Not recoverable without versioning.
3. If neither is enabled — recovery from your nightly backup (if you have one) is the only option.

**Lesson**: enable versioning on every container that holds files that can be re-uploaded. The cost is the storage of older versions (mitigated via lifecycle to roll old versions to Cool/Cold).

**Trade-off**: Versioning grows storage cost. Lifecycle rules can age old versions to Archive or delete them after N days.

**Real project**: A bug in our migration script overwrote 12K invoices with empty files at 2 AM. Versioning let us roll all of them back in a single PowerShell loop within 30 minutes.

### 6. How do you secure a storage account network surface?

**Answer**: 

1. `allowSharedKeyAccess: false` — disable account-key auth at the account level. All access must use Entra ID.
2. `publicNetworkAccess: 'Disabled'` — block internet.
3. **Private Endpoint** for each sub-service (blob, queue, table) you use, in your application VNet.
4. **Resource instance rules** for trusted Azure services that aren't in the VNet (e.g., grant Event Grid access).
5. **Diagnostic settings** to Log Analytics for `StorageRead` / `StorageWrite` / `StorageDelete` so you can audit every operation.
6. **Defender for Storage** enabled — flags malware uploads, anonymous access attempts, and credential exposure.

**Trade-off**: Private Endpoint costs ~$8/month per endpoint and requires VNet design. Worth it for any production workload.

**Real project**: An auditor flagged our storage account because anyone with the account key could exfiltrate the entire invoice corpus. Switching to `allowSharedKeyAccess: false` + RBAC took 2 hours and resolved the finding.

### 7. Your Blob-triggered Function fires 10 minutes after the upload. Why?

**Why this matters**: This is a classic gotcha — the default Blob trigger polls.

**Answer**: The default `BlobTrigger` uses **polling** based on Blob Storage's log files, which can lag 5-10 minutes. **Fix**: use Event Grid as the trigger source:

```csharp
[BlobTrigger("invoices/{name}", Source = BlobTriggerSource.EventGrid)]
```

Subscribe Event Grid to `Microsoft.Storage.BlobCreated` events on the container. Latency drops to <1 second.

**Trade-off**: Event Grid is push-based; slightly more setup. Required for any production pipeline.

**Real project**: Our OCR pipeline went from 8-minute average latency to <2 seconds after switching to Event Grid. Customer support tickets about "where's my receipt?" dropped to zero.

### 8. Your storage egress bill is $400/month. How do you cut it?

**Answer**:

1. **Identify the source** — Storage metrics show egress per container; Defender for Storage classifies traffic.
2. **CDN/Front Door for repeated reads** — caches at edge; storage only pays for the cache fill (1 read per object).
3. **Move computation closer to the data** — if a Function in eastus reads blobs from a westus storage account, move the storage or the function.
4. **Compress before storing** — gzip JSON, optimize images.
5. **Tier audit** — Cool/Cold reads cost more per GB, so moving access-heavy data back to Hot can reduce reads cost (counterintuitive — sometimes Cool is more expensive).

**Trade-off**: CDN adds latency for first read; great for static assets, less great for user-specific docs.

**Real project**: Moving public product images behind Front Door cut egress from $1,200/month to $180/month with one configuration change.

## Summary Checklist

- [ ] I can pick Blob vs Queue vs Table vs File for a given workload.
- [ ] I can pick Hot/Cool/Cold/Archive tiers and configure a lifecycle policy.
- [ ] I can choose LRS/ZRS/GRS/GZRS redundancy based on RPO/RTO requirements.
- [ ] I can register `BlobServiceClient` with `DefaultAzureCredential` and `RBAC` instead of account keys.
- [ ] I can implement a secure direct-to-Blob upload via User Delegation SAS.
- [ ] I can enable soft-delete + versioning and recover from accidental overwrites.
- [ ] I can wire a Blob-trigger Function using Event Grid source (not polling).
- [ ] I can disable shared key auth and block public network access via Bicep.
- [ ] I can stream large uploads without OOMing my service.
- [ ] I can audit a storage account using Diagnostic settings + Defender for Storage.
