# Azure Functions

## What It Is

Azure Functions is a **serverless compute service** for running short-lived, event-driven code in C# (and other languages) without managing servers, plans, or load balancers. You write a method, decorate it with a **trigger** attribute (HTTP, Timer, Queue, Service Bus, Blob, Event Grid, Cosmos DB change feed), and Azure invokes it when the trigger fires. The platform handles scaling, retries, and concurrency.

A Function app has three runtime models in .NET:

1. **Isolated worker model (.NET 8+, .NET 9)** — your code runs in its own process, separate from the Functions host. This is the **only supported model going forward**.
2. **In-process model (.NET 6 LTS only, EOL November 2026)** — legacy; do not start new projects on this.
3. **Durable Functions** — an extension that adds long-running, stateful orchestrations (chaining, fan-out/fan-in, monitor, human interaction).

```text
┌─── Trigger (Service Bus message arrives) ───┐
│                                              │
│   ┌──── Function Host (managed) ────────┐    │
│   │   - reads message from queue         │    │
│   │   - invokes your function method    │    │
│   │   - handles retry / DLQ on failure   │    │
│   └──────────────┬──────────────────────┘    │
│                  │                            │
│   ┌──────────────▼─────────────────────────┐ │
│   │  Isolated Worker Process (your code)   │ │
│   │  - ProcessOrder(OrderMessage msg) {…}  │ │
│   └────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

Functions are billed differently depending on the **hosting plan** — pay-per-execution (Consumption), pay-per-instance with pre-warmed workers (Premium), or pay-per-plan like an App Service (Dedicated).

## Why It Exists

Before Functions, every "small piece of code that runs on a trigger" required:

- An always-on VM or App Service plan (even if the code ran for 200 ms once an hour).
- A custom polling loop to read from a queue.
- A separate scheduler (Windows Task Scheduler, cron, Hangfire) for timer jobs.
- Hand-rolled scaling logic to add workers under load.

Teams paid full compute price 24/7 for code that ran <1% of the time. Functions exists to solve this **utilization problem**: you pay only for execution time (Consumption plan), and the platform scales workers from 0 to thousands automatically based on queue depth or HTTP load.

The second problem Functions solves is **integration glue** — moving an event from one Azure service to another. Without Functions, every "when a blob is uploaded, send a Service Bus message and update SQL" workflow needed its own service. With Functions, it is 20 lines of code with two attributes.

## When To Use It

**Use Functions for:**

- **Event-driven processing**: a webhook from Stripe arrives → validate → enqueue → return 200.
- **Scheduled jobs**: nightly cleanup of expired carts, daily report generation, hourly cache warming.
- **Queue/stream consumers**: drain a Service Bus queue of `order-placed` events, write to SQL, fan out notifications.
- **Blob/file triggers**: when an invoice PDF is uploaded to Blob Storage, run OCR, save metadata to Table Storage.
- **Long-running workflows via Durable Functions**: order processing that spans payment authorization → fraud check → inventory reservation → confirmation email, with retries and human-approval steps.
- **API endpoints with unpredictable traffic** — webhooks that go from 0 to 1000 RPS in seconds and back to 0.

**Do not use Functions for:**

- **Always-on APIs with steady traffic** — use [app-service.md](app-service.md). Functions Consumption has cold starts; Premium is more expensive than App Service for steady load.
- **Long-running CPU-bound jobs (>10 min Consumption, >60 min Premium)** — use Container Apps, Batch, or AKS.
- **Real-time bidirectional communication** — use SignalR Service.
- **Stateful single-instance workloads** — Functions are stateless and scale horizontally; if you need a singleton, use Durable Functions singleton pattern or a different service.
- **Heavy startup workloads** — anything that takes >2 seconds to initialize will suffer from cold starts on Consumption.

## Why It Is Important

Functions is the **default Azure compute choice for event-driven .NET workloads**, which is most modern backend integration code. A senior engineer interview will probe:

1. **Plan selection** — Consumption vs Premium vs Dedicated. Picking wrong costs 10x or causes cold-start outages.
2. **Trigger and binding mechanics** — which triggers retry automatically, which dead-letter, which are at-least-once vs at-most-once.
3. **Durable Functions patterns** — the standard answer for "orchestrate this multi-step workflow" in Azure interviews.
4. **Isolated vs in-process** — the in-process model is end-of-life in November 2026; knowing this is a basic competency check.
5. **Cold start mitigation** — pre-warmed instances, dependency injection footprint, AOT compilation, ReadyToRun.

## How It's Used in C# / .NET

### 1. Project setup (isolated worker, .NET 8)

```xml
<!-- Orders.Functions.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.22.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.17.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.ServiceBus" Version="5.20.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http" Version="3.2.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="1.4.0" />
    <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" />
  </ItemGroup>
</Project>
```

### 2. `Program.cs` — DI, Managed Identity, Application Insights

```csharp
using Azure.Identity;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Azure;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices((ctx, services) =>
    {
        var credential = new DefaultAzureCredential();

        services.AddAzureClients(c =>
        {
            c.UseCredential(credential);
            c.AddServiceBusClientWithNamespace(ctx.Configuration["ServiceBus:Namespace"]!);
            c.AddBlobServiceClient(new Uri(ctx.Configuration["Storage:BlobUri"]!));
        });

        services.AddScoped<IOrderProcessor, OrderProcessor>();

        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
    })
    .Build();

await host.RunAsync();
```

### 3. HTTP-triggered function (Stripe webhook)

```csharp
public class StripeWebhook
{
    private readonly IPaymentEventHandler _handler;
    private readonly ILogger<StripeWebhook> _logger;

    public StripeWebhook(IPaymentEventHandler handler, ILogger<StripeWebhook> logger)
    {
        _handler = handler;
        _logger = logger;
    }

    [Function("StripeWebhook")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "webhooks/stripe")]
        HttpRequestData req,
        CancellationToken ct)
    {
        var signature = req.Headers.GetValues("Stripe-Signature").FirstOrDefault();
        var body = await new StreamReader(req.Body).ReadToEndAsync(ct);

        try
        {
            await _handler.HandleAsync(body, signature, ct);
            return req.CreateResponse(HttpStatusCode.OK);
        }
        catch (InvalidSignatureException)
        {
            _logger.LogWarning("Invalid Stripe signature");
            return req.CreateResponse(HttpStatusCode.Unauthorized);
        }
    }
}
```

### 4. Service Bus-triggered function (order processor)

```csharp
public class ProcessOrder
{
    private readonly IOrderProcessor _orders;

    public ProcessOrder(IOrderProcessor orders) => _orders = orders;

    [Function("ProcessOrder")]
    public async Task Run(
        [ServiceBusTrigger("order-placed", Connection = "ServiceBus")]
        OrderPlacedMessage message,
        FunctionContext context,
        CancellationToken ct)
    {
        await _orders.HandleAsync(message, ct);
        // No return → message is auto-completed.
        // Throw → message is abandoned; after MaxDeliveryCount it goes to DLQ.
    }
}
```

Note `Connection = "ServiceBus"` — this is the **prefix** of app-setting keys, not a connection string. With Managed Identity, you set:

```text
ServiceBus__fullyQualifiedNamespace = orders-bus.servicebus.windows.net
```

The Functions host uses Managed Identity to authenticate.

### 5. Timer-triggered function (nightly cleanup)

```csharp
public class CleanupExpiredCarts
{
    private readonly ICartRepository _carts;

    public CleanupExpiredCarts(ICartRepository carts) => _carts = carts;

    [Function("CleanupExpiredCarts")]
    public async Task Run(
        [TimerTrigger("0 0 2 * * *")] TimerInfo timer,   // 02:00 daily UTC
        CancellationToken ct)
    {
        var cutoff = DateTimeOffset.UtcNow.AddDays(-30);
        var removed = await _carts.RemoveExpiredAsync(cutoff, ct);
    }
}
```

### 6. Blob-triggered function (invoice OCR)

```csharp
public class IndexInvoice
{
    private readonly IOcrService _ocr;
    private readonly IInvoiceMetadataStore _store;

    [Function("IndexInvoice")]
    public async Task Run(
        [BlobTrigger("invoices/{name}", Source = BlobTriggerSource.EventGrid)]
        Stream blob,
        string name,
        CancellationToken ct)
    {
        var text = await _ocr.ExtractTextAsync(blob, ct);
        await _store.SaveAsync(name, text, ct);
    }
}
```

`Source = BlobTriggerSource.EventGrid` is required in production — the legacy polling source has up to 10 minute delay and misses events under load.

### 7. Durable Functions: fan-out/fan-in for batch invoice generation

```csharp
public class GenerateMonthlyInvoices
{
    // 1. HTTP starter
    [Function("StartMonthlyInvoiceGeneration")]
    public async Task<HttpResponseData> Start(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
        [DurableClient] DurableTaskClient client)
    {
        var instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(GenerateInvoicesOrchestrator),
            input: DateOnly.FromDateTime(DateTime.UtcNow));

        return client.CreateCheckStatusResponse(req, instanceId);
    }

    // 2. Orchestrator (deterministic; no I/O, no DateTime.Now)
    [Function(nameof(GenerateInvoicesOrchestrator))]
    public async Task<int> GenerateInvoicesOrchestrator(
        [OrchestrationTrigger] TaskOrchestrationContext ctx)
    {
        var month = ctx.GetInput<DateOnly>();

        // Fan-out: get all customers, queue invoice generation in parallel
        var customers = await ctx.CallActivityAsync<List<Guid>>(nameof(GetActiveCustomers), month);

        var tasks = customers
            .Select(id => ctx.CallActivityAsync<bool>(nameof(GenerateInvoiceForCustomer),
                                                     new GenerateRequest(id, month)))
            .ToArray();

        // Fan-in: wait for all
        var results = await Task.WhenAll(tasks);

        return results.Count(r => r);
    }

    [Function(nameof(GetActiveCustomers))]
    public Task<List<Guid>> GetActiveCustomers([ActivityTrigger] DateOnly month) =>
        _customerRepo.GetActiveAsync(month);

    [Function(nameof(GenerateInvoiceForCustomer))]
    public Task<bool> GenerateInvoiceForCustomer([ActivityTrigger] GenerateRequest req) =>
        _invoiceService.GenerateAsync(req.CustomerId, req.Month);
}
```

Common Durable patterns:

| Pattern              | Use case                                                                    |
|----------------------|-----------------------------------------------------------------------------|
| **Function chaining** | Step 1 → Step 2 → Step 3 with checkpoints between each.                    |
| **Fan-out/fan-in**    | Generate 50K invoices in parallel, aggregate results.                       |
| **Async HTTP API**    | Long workflow returns a status URL the client polls.                        |
| **Monitor**           | Poll an external system every minute until a condition is met.              |
| **Human interaction** | Wait for an approval event; time out after 7 days.                          |
| **Sub-orchestration** | Compose orchestrations into reusable workflows.                             |

### 8. Hosting plans cheat sheet

| Plan                    | Billing                  | Scale to 0 | Max execution | Cold start | VNet | When to use                          |
|-------------------------|---------------------------|-----------|---------------|------------|------|--------------------------------------|
| **Consumption (Y1)**    | Per execution + GB-sec    | Yes       | 10 min        | Yes (~3s)  | Limited | Webhooks, low-volume scheduled jobs |
| **Flex Consumption**    | Per execution, pre-warmed | Yes       | 60 min        | Lower      | Yes    | Modern Consumption replacement      |
| **Premium (EP1/2/3)**   | Per instance hour         | No (min 1)| 60 min (default), unbounded | Eliminated with always-ready | Yes | Steady event-driven, VNet, large memory |
| **Dedicated (App Service)** | Per plan instance     | No        | Unbounded     | None       | Yes    | Sharing a plan with App Service apps |

## Advantages

- **True pay-per-execution** on Consumption — idle Functions cost $0.
- **Auto-scaling** from 0 to thousands of instances based on queue depth, HTTP load, or custom metrics.
- **Built-in triggers and bindings** for every major Azure service — no polling loops to write.
- **Managed Identity** for all bindings — no connection strings.
- **Durable Functions** provide stateful, long-running orchestrations without managing a state store.
- **First-class Application Insights integration** — automatic correlation across function invocations.
- **Slots** are supported on Premium and Dedicated plans for blue-green deploys.

## Disadvantages

- **Cold starts on Consumption** — 2-5 seconds for .NET, longer with heavy DI graphs. Bad for low-latency HTTP APIs.
- **Execution time limits** — Consumption hard-caps at 10 minutes; Premium at 60 by default.
- **Cost can balloon on steady load** — if your function runs continuously, App Service or Container Apps is cheaper.
- **Local debugging complexity** — the Functions host (`func` CLI) is a separate runtime; debugging triggers requires `azurite` or live Azure resources.
- **In-process model EOL** — November 2026. Existing in-process projects must migrate to isolated worker before then.
- **Vendor lock-in** — bindings and Durable Functions don't transfer to AWS Lambda or GCP Cloud Functions without rewrites.
- **Regional and SKU limits** — Premium plans not in every region; Flex Consumption still rolling out.
- **Service Bus session triggers + isolated worker** — sessions are supported only via specific extension versions; check before designing FIFO workflows.

## Common Mistakes

### 1. Using the deprecated in-process model for new projects

```xml
<!-- BUG: In-process model is EOL November 2026 -->
<PackageReference Include="Microsoft.NET.Sdk.Functions" Version="4.4.1" />
```

**Fix**: start every new Function app on isolated worker:

```xml
<OutputType>Exe</OutputType>
<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.22.0" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.17.0" />
```

### 2. Storing connection strings in `local.settings.json` and shipping them to Azure

```json
// BUG: secret in source control, no rotation
{
  "Values": {
    "ServiceBusConnection": "Endpoint=sb://...;SharedAccessKey=ABC123..."
  }
}
```

**Fix**: use Managed Identity with the `fullyQualifiedNamespace` setting:

```text
ServiceBus__fullyQualifiedNamespace = orders-bus.servicebus.windows.net
```

Grant the Function app's Managed Identity the `Azure Service Bus Data Receiver` role on the namespace.

### 3. Putting business logic in the orchestrator function

```csharp
// BUG: orchestrator must be deterministic — no DateTime.UtcNow, Guid.NewGuid, await Task.Delay
[Function(nameof(BadOrchestrator))]
public async Task BadOrchestrator([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var orderId = Guid.NewGuid();              // BUG: non-deterministic
    var now = DateTime.UtcNow;                  // BUG: non-deterministic
    await Task.Delay(TimeSpan.FromSeconds(30)); // BUG: blocks history replay
}
```

**Fix**: do non-deterministic work in **activity functions**; use `ctx.NewGuid()`, `ctx.CurrentUtcDateTime`, and `ctx.CreateTimer()`:

```csharp
var orderId = ctx.NewGuid();
var now = ctx.CurrentUtcDateTime;
await ctx.CreateTimer(TimeSpan.FromSeconds(30), CancellationToken.None);
```

### 4. Long-running synchronous work in a Service Bus trigger

```csharp
// BUG: blocking on slow downstream call holds the lock; eventually it expires and the message
// is redelivered, processed again — duplicate side effects.
[Function("Process")]
public void Run([ServiceBusTrigger("queue")] Message msg)
{
    Thread.Sleep(TimeSpan.FromMinutes(10));    // lock expires after 5 min default
    CallStripe(msg);
}
```

**Fix**: increase `maxAutoLockRenewalDuration` in `host.json`, make calls async with `CancellationToken`, or split the work into a Durable orchestration with checkpoints.

### 5. Ignoring cold start

```csharp
// BUG: massive DI graph + EF Core migration check on first request = 8 sec cold start
builder.Services.AddDbContext<HugeContext>(...);
builder.Services.AddHostedService<MigrationRunner>();
```

**Fix**: 

- Use **Premium plan** with `alwaysReady` instances for latency-sensitive functions.
- Avoid `AddHostedService` in Functions; the worker process is not a long-lived host.
- Keep the DI graph small; use `Lazy<T>` for heavy dependencies that aren't always needed.
- Compile with `<PublishReadyToRun>true</PublishReadyToRun>` for faster JIT.

### 6. No retry/poison-message handling

```csharp
// BUG: a single bad message is retried 10 times, then silently lost
[Function("Process")]
public void Run([ServiceBusTrigger("orders")] OrderMessage msg) { throw new Exception(); }
```

**Fix**: configure `maxDeliveryCount` and verify the dead-letter queue is monitored. Add explicit handling for poison messages:

```csharp
public async Task Run(
    [ServiceBusTrigger("orders")] ServiceBusReceivedMessage msg,
    ServiceBusMessageActions actions,
    CancellationToken ct)
{
    try
    {
        await _processor.HandleAsync(msg.Body.ToObjectFromJson<OrderMessage>(), ct);
        await actions.CompleteMessageAsync(msg, ct);
    }
    catch (NonRetryableException ex)
    {
        await actions.DeadLetterMessageAsync(msg, deadLetterReason: ex.GetType().Name);
    }
}
```

### 7. Wrong plan for the workload

```text
"We picked Consumption because it was the cheapest tier."
[3 months later] "Cold starts are killing our user experience."
```

**Fix**: for any user-facing HTTP function with strict latency, use **Premium with always-ready instances** or fold the workload into [app-service.md](app-service.md).

## Best Practices

- **Use isolated worker model exclusively** for new projects.
- **Use Managed Identity for every trigger and binding** — Service Bus, Storage, Cosmos, Key Vault all support `fullyQualifiedNamespace` / URI-based auth.
- **One function per file**, one responsibility per function.
- **Keep functions thin**: parse input, call a service, return. All business logic lives in injected services so it's testable without the Functions runtime.
- **Use Durable Functions for orchestrations**, not chained HTTP calls or queue-of-queues.
- **Configure `maxConcurrentCalls` and `maxDeliveryCount`** in `host.json` based on dependency capacity (e.g., SQL DTU).
- **Pre-warm Premium** with `alwaysReady` to eliminate cold starts on critical paths.
- **Compile with ReadyToRun** and enable trimming/AOT where supported to cut startup time.
- **Run integration tests against `azurite`** (Azure Storage emulator) and a local Service Bus emulator or test namespace.
- **Always set `WEBSITE_RUN_FROM_PACKAGE=1`** for atomic deploys.
- **Monitor the DLQ** — alert on any message landing there.

## Related Concepts

- **App Service** ([app-service.md](app-service.md)) — alternative when you have steady traffic.
- **Service Bus** ([azure-service-bus.md](azure-service-bus.md)) — most common trigger source for `.NET` Functions.
- **Storage** ([azure-storage.md](azure-storage.md)) — Blob and Queue triggers; Functions also uses Storage internally for its own state.
- **Event Grid** — push notifications for blob created/deleted, Resource Manager events.
- **Application Insights** ([application-insights.md](application-insights.md)) — telemetry and distributed traces for Functions.
- **Key Vault** ([azure-key-vault.md](azure-key-vault.md)) — `@Microsoft.KeyVault(...)` references work in Function app settings.
- **Outbox pattern** ([../architecture/outbox-pattern.md](../architecture/outbox-pattern.md)) — Functions are the typical Outbox publisher worker.
- **Saga pattern** ([../architecture/saga-pattern.md](../architecture/saga-pattern.md)) — Durable Functions are a natural implementation.

## Real-World Usage

### Stripe webhook receiver

- **Plan**: Premium EP1 (always-ready 1, max 20) for low-latency response.
- **Trigger**: HTTP `POST /webhooks/stripe`.
- **Logic**: Validate signature, enqueue the event to a Service Bus topic, return `200` within 200ms (Stripe retries on slow responses).
- **Downstream**: A separate Service Bus-triggered Function processes the event (fraud check, inventory release, customer notification).

### Order processor

- **Plan**: Premium EP1, scales to 10 on queue depth.
- **Trigger**: `ServiceBusTrigger("order-placed")`.
- **Logic**: Reserve inventory, capture payment, write to SQL, enqueue confirmation email.
- **DLQ monitoring**: A second Function processes DLQ messages, alerts the on-call channel.

### Nightly invoice generation (Durable Functions)

- **Plan**: Premium EP2 (more memory for batch work).
- **Trigger**: Timer `0 0 2 1 * *` (2 AM on the 1st of each month).
- **Orchestrator**: Fan-out to generate ~50,000 invoices in parallel, fan-in to summarize results, send email to finance team.
- **State**: Durable Functions automatically persists progress to Azure Storage; survives Function restarts.

### Blob-triggered OCR pipeline

- **Plan**: Consumption (sporadic uploads).
- **Trigger**: `BlobTrigger("invoices/{name}", Source = BlobTriggerSource.EventGrid)`.
- **Logic**: Send PDF to Azure AI Document Intelligence, write extracted fields to Cosmos DB.

## Code Example — Before and After

### Before: VM-based polling consumer

```csharp
// Console app on a VM, runs forever, polls Service Bus every 5 sec
public class OrderProcessorWorker
{
    private readonly ServiceBusClient _client;
    private readonly string _connString =
        "Endpoint=sb://orders-bus.servicebus.windows.net/;SharedAccessKeyName=Listen;SharedAccessKey=AbC...";

    public async Task RunAsync()
    {
        var receiver = new ServiceBusClient(_connString).CreateReceiver("order-placed");

        while (true)
        {
            try
            {
                var msg = await receiver.ReceiveMessageAsync(TimeSpan.FromSeconds(5));
                if (msg is null) continue;

                var order = JsonSerializer.Deserialize<Order>(msg.Body);
                CaptureStripePayment(order);                       // sync, blocking
                WriteToSqlServer(order);
                await receiver.CompleteMessageAsync(msg);
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
                Thread.Sleep(1000);
            }
        }
    }
}
```

Problems:
- VM costs ~$70/month for code that runs 5 minutes per day.
- Manual scaling: one VM = one consumer; doubling throughput means doubling VMs.
- Hard-coded secret.
- No telemetry, no DLQ handling, no graceful shutdown.

### After: Isolated worker Function with Managed Identity

```csharp
// Program.cs
var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices((ctx, s) =>
    {
        var cred = new DefaultAzureCredential();
        s.AddAzureClients(c =>
        {
            c.UseCredential(cred);
            c.AddServiceBusClientWithNamespace(ctx.Configuration["ServiceBus:Namespace"]!);
        });
        s.AddDbContext<OrdersDb>(o =>
            o.UseSqlServer(ctx.Configuration.GetConnectionString("OrdersDb")));
        s.AddHttpClient<IPaymentGateway, StripePaymentGateway>();
        s.AddScoped<IOrderProcessor, OrderProcessor>();
        s.AddApplicationInsightsTelemetryWorkerService();
        s.ConfigureFunctionsApplicationInsights();
    })
    .Build();
await host.RunAsync();

// ProcessOrder.cs
public class ProcessOrder(IOrderProcessor processor)
{
    [Function("ProcessOrder")]
    public async Task Run(
        [ServiceBusTrigger("order-placed", Connection = "ServiceBus")]
        ServiceBusReceivedMessage msg,
        ServiceBusMessageActions actions,
        CancellationToken ct)
    {
        try
        {
            var order = msg.Body.ToObjectFromJson<OrderMessage>();
            await processor.HandleAsync(order, ct);
            await actions.CompleteMessageAsync(msg, ct);
        }
        catch (PaymentDeclinedException ex)
        {
            // Don't retry — go straight to DLQ
            await actions.DeadLetterMessageAsync(msg,
                deadLetterReason: "PaymentDeclined",
                deadLetterErrorDescription: ex.Message,
                cancellationToken: ct);
        }
    }
}
```

Benefits:
- **Cost**: Consumption plan, ~$0.20/month for the same workload.
- **Scale**: 0 → 200 instances automatically based on queue depth.
- **No secrets**: Managed Identity authenticates to Service Bus and SQL.
- **Telemetry**: Each invocation auto-tracks duration, dependency calls, exceptions in Application Insights.
- **Proper DLQ**: poison messages don't loop forever.

## Interview Questions and Answers

### 1. When would you pick Consumption over Premium over Dedicated?

**Why this matters**: Plan selection is the single biggest cost lever in Functions. Wrong choice doubles the bill or causes outages.

**Answer**: **Consumption** when traffic is sporadic and cold start (~2-5 sec) is acceptable — webhooks, nightly cleanups, dev/test. **Premium (or Flex Consumption)** when you need VNet integration, longer than 10-min executions, no cold start (always-ready instances), or large memory. **Dedicated (App Service plan)** only when you want Functions to share existing App Service compute or you need full VM control.

**Trade-off**: Premium EP1 costs ~$150/month even idle; Consumption costs cents for the same idle. But Consumption's cold start is unacceptable for user-facing latency.

**Real project**: Our Stripe webhook is Premium EP1 (always-ready 1) because Stripe retries on >5s response. Our nightly invoice generator is Consumption because nobody cares if it starts in 3s vs 30s.

### 2. Explain the isolated worker model and why in-process is going away.

**Answer**: In the **in-process model**, your function code runs inside the same .NET process as the Functions host. The host dictates the runtime version — you were stuck on .NET 6 for years because the host hadn't moved. In the **isolated worker model**, your code runs in a separate process; gRPC carries trigger data between the host and your worker. You control your runtime, your DI, your middleware, and you can target any .NET version Azure Functions supports independently.

Microsoft has announced **end-of-support for in-process model in November 2026**. .NET 8 is the last in-process LTS; .NET 9+ is isolated-only. New projects must start on isolated; existing in-process projects must plan a migration.

**Trade-off**: Isolated has a tiny IPC overhead per invocation (~1-2ms). For 99% of workloads this is irrelevant; for extreme high-throughput functions, measure first.

**Real project**: We migrated a large in-process Function app to isolated last quarter. Required rewriting custom bindings, but unlocked .NET 8 + AOT and shaved 800ms off cold start.

### 3. Walk me through Durable Functions fan-out/fan-in.

**Why this matters**: Tests understanding of Azure's standard pattern for parallel orchestration.

**Answer**: An **orchestrator function** receives an input (e.g., a month), fetches a list of items to process (a Get-Customers activity), then schedules N **activity functions** in parallel by collecting their `Task`s and awaiting `Task.WhenAll`. Each activity runs as a separate Function invocation, potentially on different instances. When all complete, the orchestrator resumes, aggregates results, and returns.

The orchestrator's code is **replayed** from history every time it resumes — that's why it must be deterministic (no `DateTime.UtcNow`, no `Guid.NewGuid()`, no `Task.Delay`). Use `ctx.CurrentUtcDateTime`, `ctx.NewGuid()`, `ctx.CreateTimer()` instead.

**Trade-off**: Fan-out is bounded by Service Bus / Storage throughput on the Durable framework. For 100,000+ items, batch into groups of 1000 to keep history small.

**Real project**: Monthly invoice generation for 50K customers — fan-out completes in ~3 minutes on Premium EP2, vs 4 hours sequential.

### 4. Your Service Bus-triggered Function is duplicating side effects in production. What happened?

**Answer**: Service Bus delivery is **at-least-once** — the message can be redelivered if:

- The function exceeds the message lock duration (default 5 min) without renewing.
- The function crashes between completing the work and acknowledging the message.
- The function throws, message is abandoned, redelivered to a different instance.

**Diagnosis**:
1. Check Application Insights for invocations of the same `messageId`.
2. Check whether your function exceeds `maxAutoLockRenewalDuration` (default 5 min).
3. Check whether downstream side effects are **idempotent**.

**Fix**: 
- Make side effects idempotent: write with `INSERT IF NOT EXISTS` keyed on `messageId`, or upsert with `ETag`.
- Use **transactional outbox** so the DB write and the next message-send commit atomically.
- Increase lock duration if processing is genuinely long.

**Trade-off**: Idempotency adds latency (extra read), but at-least-once + non-idempotent processing **always** causes duplicates eventually.

**Real project**: A payment-capture Function double-charged customers when the SQL write took 6 minutes during a DB outage. Fix: capture-then-record via outbox; the outbox row's unique key on `paymentIntentId` blocked the duplicate.

### 5. How do you minimize Functions cold start for a .NET 8 app?

**Answer**:

- **Use Premium plan with always-ready instances** for any HTTP function on the critical path.
- **Enable `ReadyToRun`** in the csproj: `<PublishReadyToRun>true</PublishReadyToRun>`.
- **Trim the DI graph**: register only what's needed for the specific function set in this Function app. Don't load `AddDbContext` for an HTTP-only Function app.
- **Avoid heavy startup work**: no migration runners, no schema validation, no Application Insights ingestion warm-up loops.
- **Compile with native AOT** where supported (preview in some Functions versions).
- **Split unrelated functions into separate Function apps**, so HTTP and Service Bus apps don't share startup cost.

**Trade-off**: Always-ready instances cost money even when idle. For latency-insensitive workloads, accept the cold start and save the spend.

**Real project**: Our webhook receiver went from 4.2s cold start (Consumption, in-process .NET 6, huge DI graph) to 280ms (Premium always-ready, isolated .NET 8, minimal DI).

### 6. How do you secure an HTTP-triggered function?

**Answer**: Layered:

1. **Authorization level** in the attribute: `Anonymous`, `Function` (function key in header), `Admin` (master key). Function keys are fine for trusted webhooks (Stripe sends a signature you also verify); never for user traffic.
2. **App-level auth via Easy Auth** with Microsoft Entra ID for internal APIs.
3. **API Management** in front of Functions for rate limiting, JWT validation, IP allow-listing, and to hide the `*.azurewebsites.net` hostname.
4. **Private endpoint + VNet integration** (Premium only) so the Function isn't internet-reachable at all.
5. **Always validate the trigger payload** — Stripe signatures, GitHub HMACs, Azure AD JWT bearer tokens.

**Trade-off**: APIM adds latency (~30ms) and cost; for low-traffic webhooks, function keys + signature validation is enough.

**Real project**: All public Functions sit behind APIM. Internal-only Functions use Private Endpoint and the corp network ACL.

### 7. The Function says "Status: Healthy" but messages are piling up in the queue. What do you check?

**Answer**:

1. **Scale controller decisions** — Application Insights' `FunctionScaleController` table shows whether the runtime decided to add instances.
2. **Max instances cap** — Premium has a `functionAppScaleLimit`; Consumption defaults to 200. If you've capped low, you bottleneck.
3. **`host.json` concurrency settings** — `serviceBus.messageHandlerOptions.maxConcurrentCalls` (per instance) and `prefetchCount`.
4. **Downstream throttling** — if SQL is at 100% DTU, the function processes slowly. The queue grows even though the function is "healthy".
5. **Long-running invocations** — one slow message blocks a worker for minutes; the queue grows.

**Fix**: identify the bottleneck (CPU? Network? SQL?) and either scale out, increase concurrency, or improve downstream capacity.

**Trade-off**: Cranking concurrency without checking downstream just moves the failure (SQL deadlocks instead of queue backlog).

**Real project**: Black Friday spike caused a 200K message backlog. The Function was healthy, scaled to 100 instances, but SQL was at 95% DTU. We added a Redis cache in front of the lookup query and the backlog drained in 8 minutes.

### 8. Compare Durable Functions vs an external workflow engine like Logic Apps or Temporal.

**Answer**:

- **Durable Functions**: Code-first orchestrations in C#. Best when the team is already on .NET, the workflow logic involves complex branching, and you want to unit-test orchestrators.
- **Logic Apps**: Visual workflow designer. Best for integration-heavy "when X happens in Outlook, call Y in SharePoint" flows that non-developers can edit.
- **Temporal / Cadence**: External workflow engine, polyglot. Best for polyglot orgs or workflows that span multiple clouds.

**Trade-off**: Durable Functions is locked to Azure and is replay-sensitive (every code change to an orchestrator can break in-flight instances). Logic Apps has a steeper bill on high-volume runs. Temporal requires running your own clusters (or paying Temporal Cloud).

**Real project**: We use Durable Functions for our order fulfillment saga (5-step workflow, 50K/day) and Logic Apps for the legacy "approve PO" workflow that the procurement team maintains visually.

## Summary Checklist

- [ ] I can pick between Consumption, Flex Consumption, Premium, and Dedicated based on workload.
- [ ] I can explain isolated worker vs in-process and know in-process is EOL in November 2026.
- [ ] I can wire Managed Identity for Service Bus, Storage, and Cosmos triggers (no connection strings).
- [ ] I can implement HTTP, Timer, Service Bus, and Blob triggers in isolated worker .NET 8.
- [ ] I can describe the five Durable Functions patterns and use fan-out/fan-in correctly.
- [ ] I can keep orchestrators deterministic and put I/O in activity functions.
- [ ] I can recognize and fix the duplicate-side-effect failure mode with idempotent writes or outbox.
- [ ] I can configure `maxConcurrentCalls`, `maxDeliveryCount`, and lock renewal in `host.json`.
- [ ] I can mitigate cold start with Premium always-ready, ReadyToRun, and a small DI graph.
- [ ] I can monitor a Function with Application Insights, KQL queries, and DLQ alerts.
