# Async and Await

## What It Is

`async` and `await` are C# keywords that let a method perform an operation (usually I/O — a database query, an HTTP call, a message-bus send) without **blocking** the thread that is running it. The method **suspends** at the `await` point, the thread is returned to the .NET thread pool to do other work, and when the awaited operation completes the method **resumes** on an available thread.

Under the hood the C# compiler rewrites your async method into a state machine. The `Task` (or `Task<T>`, or `ValueTask<T>`) returned by an async method is a handle to that ongoing operation — callers can `await` it, attach continuations to it, or pass it around.

```csharp
// Synchronous — the request thread is blocked for ~50 ms while SQL runs
public Order GetOrder(Guid id)
{
    return _db.Orders.Single(o => o.Id == id); // thread is stuck here
}

// Asynchronous — the thread is released back to the pool during the SQL round-trip
public async Task<Order> GetOrderAsync(Guid id, CancellationToken ct)
{
    return await _db.Orders.SingleAsync(o => o.Id == id, ct);
}
```

The key insight: `async`/`await` is not about **doing work faster**. It is about **not holding a thread hostage** while the work is happening somewhere else (SQL Server, the Stripe API, an Azure Service Bus broker).

## Why It Exists

Before `async`/`await` (C# 5, .NET 4.5, 2012), writing non-blocking code in .NET meant one of three painful options:

1. **Synchronous, thread-per-request**: ASP.NET classic blocked a worker thread for the duration of every request. A 200 ms downstream API call held a thread for 200 ms. Under load, the thread pool was exhausted and new requests were queued — you saw it as cliff-edge latency spikes.
2. **`BeginXxx`/`EndXxx` (APM)**: callbacks-as-pairs. Logic was scattered across two methods, exceptions did not flow naturally, and composition was awful.
3. **`ContinueWith` (TPL)**: continuations passed as lambdas, hand-managed `TaskCompletionSource`, nested closures. Readable for trivial cases, unreadable for real business code.

`async`/`await` was introduced so that **asynchronous code reads like synchronous code** while still releasing the thread during I/O. In cloud-hosted ASP.NET Core, where you typically pay per vCPU and downstream calls dominate latency, this is the difference between an app that scales to 5,000 concurrent connections per instance and one that falls over at 200.

## When To Use It

**Use async for:**

- Any I/O-bound work: EF Core (`ToListAsync`, `SaveChangesAsync`), `HttpClient` (`SendAsync`), Azure SDKs (`BlobClient.UploadAsync`, `ServiceBusSender.SendMessageAsync`), file streams (`Stream.ReadAsync`).
- ASP.NET Core controllers, minimal API handlers, and middleware that touch the network or disk.
- Background workers (`BackgroundService`, Azure Functions) that wait on queues, timers, or HTTP triggers.
- Any code path where you want to honor cancellation (`CancellationToken`).

**Do NOT use async for:**

- Pure CPU work (hashing, image resize, JSON-deserialize a 50 KB payload). Wrapping it in `Task.Run` on an ASP.NET request thread just moves the work to another pool thread — it does not free a thread overall and adds scheduling overhead.
- Trivial in-memory operations that complete in microseconds.
- Event handlers in non-UI .NET code where `async void` would swallow exceptions.

## Why It Is Important

In a typical ASP.NET Core API a request looks like: **validate → query SQL (40 ms) → call payment provider (120 ms) → publish event (20 ms) → return**. Of those ~180 ms, the CPU is doing real work for maybe 2 ms. The other 178 ms is **waiting**.

If those waits are synchronous, every concurrent request pins one thread for 180 ms. The default ASP.NET Core thread pool starts at `Environment.ProcessorCount` worker threads and grows slowly (≈ 1 thread/second) — so a traffic spike causes a backlog before the pool can grow. The symptom: p99 latency explodes from 200 ms to 8 s while CPU sits at 20%.

Async releases the thread during those 178 ms of waiting. The same instance can now interleave thousands of in-flight requests. On Azure App Service, Azure Container Apps, or AKS this directly reduces the number of pods/instances you need — i.e. it reduces your monthly bill.

It also enables **cooperative cancellation**. When a client disconnects, ASP.NET Core triggers `HttpContext.RequestAborted`. If your code respects that token, you stop hammering SQL with work whose result nobody will read.

## How It's Used in C# / .NET

The async surface you will actually touch in a backend codebase:

| API / Type | Purpose |
| --- | --- |
| `Task`, `Task<T>` | Standard return types. One allocation per call. |
| `ValueTask<T>` | Avoids allocation when the operation usually completes synchronously (caches, pooled buffers). |
| `CancellationToken` | Passed through every async call. Hooked up to `HttpContext.RequestAborted` automatically. |
| `IAsyncEnumerable<T>` + `await foreach` | Stream rows from EF Core or Azure Cosmos without buffering all into memory. |
| `Task.WhenAll(...)` | Fan out multiple independent I/O calls in parallel. |
| `ConfigureAwait(false)` | In **library** code, skip the captured context. In ASP.NET Core app code it is a no-op (no sync context). |
| `IAsyncDisposable` + `await using` | Async cleanup for `DbContext`, `ServiceBusClient`, `BlobClient`. |
| `async Task Main(...)` | Top-level async entry point. |
| `Channel<T>` (System.Threading.Channels) | Producer/consumer with async pull semantics. |

A representative ASP.NET Core handler:

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly OrderingDbContext _db;
    private readonly IPaymentGateway _payments;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        OrderingDbContext db,
        IPaymentGateway payments,
        ILogger<OrdersController> logger)
    {
        _db = db;
        _payments = payments;
        _logger = logger;
    }

    [HttpPost]
    public async Task<ActionResult<OrderDto>> PlaceAsync(
        [FromBody] PlaceOrderRequest request,
        CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Items);

        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        var result = await _payments.AuthorizeAsync(order.Id, order.Total, ct);
        if (!result.Approved)
        {
            _logger.LogWarning("Payment declined for {OrderId}: {Reason}", order.Id, result.Reason);
            return BadRequest(new ProblemDetails { Title = "Payment declined", Detail = result.Reason });
        }

        await _db.SaveChangesAsync(ct);
        return CreatedAtAction(nameof(GetAsync), new { id = order.Id }, OrderDto.From(order));
    }
}
```

Note how `ct` flows from the framework, through EF Core, through the payment gateway. That single token unwinds the whole pipeline when the client disconnects.

## Advantages

- **Scalability under I/O wait** — handle thousands of in-flight requests per instance without growing the thread pool.
- **Reads like synchronous code** — no callback pyramids, exceptions flow naturally up the call stack.
- **First-class cancellation** through `CancellationToken`.
- **Composable** with `Task.WhenAll`, `Task.WhenAny`, `Polly` policies, `IAsyncEnumerable`.
- **Universally supported** — EF Core, ADO.NET, `HttpClient`, Azure SDKs, gRPC, SignalR all expose async APIs.

## Disadvantages

- **State-machine allocation** per `Task` (mitigated by `ValueTask` for hot paths).
- **Async contagion** — once one method is async, callers must be too. Mixing sync and async causes deadlocks.
- **Stack traces are messier** — they show the state machine, not your call site.
- **Easy to misuse** for CPU work, producing the illusion of parallelism without actually freeing threads.
- **`async void` traps** — unobserved exceptions crash the process.

## Common Mistakes

### 1. Blocking on async with `.Result` or `.Wait()`

```csharp
// WRONG — risk of thread-pool starvation; in some hosts, a deadlock
public OrderDto Get(Guid id)
{
    return _orderService.GetAsync(id, CancellationToken.None).Result;
}
```

```csharp
// RIGHT — async all the way
public async Task<OrderDto> GetAsync(Guid id, CancellationToken ct)
{
    return await _orderService.GetAsync(id, ct);
}
```

### 2. Using `async void` outside event handlers

```csharp
// WRONG — exception is unobserved; if it throws, the process can crash
public async void HandleMessage(ServiceBusReceivedMessage msg)
{
    await ProcessAsync(msg);
}
```

```csharp
// RIGHT — return Task so the caller (and framework) can observe failures
public async Task HandleMessageAsync(ServiceBusReceivedMessage msg, CancellationToken ct)
{
    await ProcessAsync(msg, ct);
}
```

### 3. Forgetting to pass `CancellationToken`

```csharp
// WRONG — even if the client disconnects, SQL keeps running and the row is saved
public async Task SaveAsync(Order order)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();
}
```

```csharp
// RIGHT — cancellation flows end-to-end
public async Task SaveAsync(Order order, CancellationToken ct)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct);
}
```

### 4. Awaiting inside a `foreach` when calls are independent

```csharp
// WRONG — serial: total latency = sum of all calls
foreach (var id in orderIds)
{
    results.Add(await _payments.GetStatusAsync(id, ct));
}
```

```csharp
// RIGHT — parallel: total latency = slowest call
var tasks = orderIds.Select(id => _payments.GetStatusAsync(id, ct));
var results = await Task.WhenAll(tasks);
```

### 5. Wrapping CPU work in `Task.Run` inside an API handler

```csharp
// WRONG — moves work to another thread-pool thread; no scalability gain
public async Task<string> Compute([FromBody] Payload p, CancellationToken ct)
    => await Task.Run(() => ExpensiveHash(p));
```

```csharp
// RIGHT — keep synchronous for CPU work; offload to a queue/worker if needed
public string Compute([FromBody] Payload p) => ExpensiveHash(p);
```

### 6. Returning `async Task` when you only forward a task

```csharp
// SUBOPTIMAL — extra state machine for no reason
public async Task<Order> GetAsync(Guid id, CancellationToken ct)
    => await _repo.GetAsync(id, ct);
```

```csharp
// BETTER — return the Task directly (no try/catch/await needed)
public Task<Order> GetAsync(Guid id, CancellationToken ct)
    => _repo.GetAsync(id, ct);
```

### 7. `Task.WhenAll` swallowing all but one exception in logs

```csharp
try
{
    await Task.WhenAll(taskA, taskB);
}
catch (Exception ex) // only the first exception is in ex
{
    _logger.LogError(ex, "One of the tasks failed");
}
```

```csharp
// RIGHT — inspect all completed tasks for their exceptions
var all = Task.WhenAll(taskA, taskB);
try { await all; }
catch
{
    foreach (var e in all.Exception!.InnerExceptions)
        _logger.LogError(e, "Task failed");
    throw;
}
```

## Best Practices

- **Async all the way** — never mix `.Result`/`.Wait()` into a chain that has `await` in it.
- **Pass `CancellationToken` through every async signature** — make it the last parameter.
- **Name async methods with the `Async` suffix** (`GetOrderAsync`) — it is a convention every .NET dev expects.
- Use `Task.WhenAll` for independent fan-out; resist the urge to fan out dependent calls.
- Return `Task`/`Task<T>` directly (no `async`/`await`) when you are just forwarding and do not need a try/catch.
- Use `IAsyncEnumerable<T>` for unbounded streams (large EF queries, Cosmos pagination, Service Bus batches).
- In **library** code, append `.ConfigureAwait(false)` to every `await` to avoid capturing a caller's sync context.
- Use `ValueTask<T>` only when profiling proves the `Task` allocation matters (e.g. >100k calls/sec, cache hits).
- Never `async void` except for event handlers.
- Hook background work to `IHostApplicationLifetime.ApplicationStopping` so it cancels cleanly on shutdown.

## Related Concepts

- [csharp/exception-handling.md](csharp/exception-handling.md) — how exceptions propagate through async state machines.
- [aspnet-core/request-pipeline.md](aspnet-core/request-pipeline.md) — where `HttpContext.RequestAborted` comes from.
- [data-access/query-performance.md](data-access/query-performance.md) — async EF Core patterns and `AsAsyncEnumerable`.
- [azure/azure-service-bus.md](azure/azure-service-bus.md) — async message handlers and back-pressure.
- [testing-quality/unit-testing.md](testing-quality/unit-testing.md) — testing async code, awaiting in tests.
- [architecture/reliability-design.md](architecture/reliability-design.md) — pairing async with Polly for retries/timeouts.

## Real-World Usage

### ASP.NET Core APIs

Every controller action that touches a database, HTTP service, or message broker should be `async Task<IActionResult>` (or `async Task<ActionResult<T>>`). The framework wires the request `CancellationToken` automatically — accept it as a parameter and ASP.NET Core injects `HttpContext.RequestAborted`.

### Azure Service Bus consumers

`ServiceBusProcessor.ProcessMessageAsync` is invoked with a `ProcessMessageEventArgs` that exposes `CancellationToken`. Async handlers let one consumer pump dozens of in-flight messages concurrently without spinning up threads.

### Azure Functions

The function runtime invokes async functions directly. For timer triggers, queue triggers, and HTTP triggers, return `Task` and accept a `CancellationToken` — the host raises it during scale-in or host shutdown so your work doesn't get cut off mid-write.

### EF Core + SQL Azure

Every EF Core query has an async variant (`FirstOrDefaultAsync`, `ToListAsync`, `SaveChangesAsync`). SQL Azure latency is typically 5–30 ms; releasing the thread during that window is the single biggest scalability win in most ASP.NET Core apps.

### Polly resilience pipelines

`Polly` policies are async-aware (`AsyncRetryPolicy`, `AsyncCircuitBreakerPolicy`). Wrapping an `HttpClient` call in `Polly` keeps async semantics — retries do not block threads.

### Testing

xUnit and NUnit support `async Task` test methods. Use `await` directly in tests; never call `.Result`. For deterministic time-based tests use `Task.Yield()` or `IHostedService` test harnesses rather than `Thread.Sleep`.

## Code Example — Before and After

### Before — synchronous, blocks request threads, no cancellation

```csharp
public sealed class CheckoutService
{
    private readonly OrderingDbContext _db;
    private readonly HttpClient _http;

    public CheckoutService(OrderingDbContext db, HttpClient http)
    {
        _db = db;
        _http = http;
    }

    public CheckoutResult Checkout(Guid orderId)
    {
        var order = _db.Orders.Single(o => o.Id == orderId);

        var response = _http.PostAsJsonAsync(
            "https://api.stripe.com/v1/charges",
            new { amount = order.Total }).Result; // BLOCKS the thread

        response.EnsureSuccessStatusCode();

        order.MarkPaid();
        _db.SaveChanges();

        return new CheckoutResult(order.Id, "Paid");
    }
}
```

Under load this exhausts the thread pool. There is no way to cancel if the user closes the tab.

### After — async, cancellation propagated, parallel fan-out where independent

```csharp
public sealed class CheckoutService
{
    private readonly OrderingDbContext _db;
    private readonly IPaymentGateway _payments;
    private readonly IInventoryClient _inventory;
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(
        OrderingDbContext db,
        IPaymentGateway payments,
        IInventoryClient inventory,
        ILogger<CheckoutService> logger)
    {
        _db = db;
        _payments = payments;
        _inventory = inventory;
        _logger = logger;
    }

    public async Task<CheckoutResult> CheckoutAsync(Guid orderId, CancellationToken ct)
    {
        var order = await _db.Orders
            .Include(o => o.Lines)
            .SingleAsync(o => o.Id == orderId, ct);

        // Independent calls run in parallel — total latency ≈ max(payment, inventory)
        var paymentTask  = _payments.AuthorizeAsync(order.Id, order.Total, ct);
        var inventoryTask = _inventory.ReserveAsync(order.Lines, ct);

        await Task.WhenAll(paymentTask, inventoryTask);

        var payment = await paymentTask;
        if (!payment.Approved)
        {
            _logger.LogWarning("Payment declined for {OrderId}: {Reason}", order.Id, payment.Reason);
            return CheckoutResult.Declined(payment.Reason);
        }

        order.MarkPaid(payment.TransactionId);
        await _db.SaveChangesAsync(ct);

        return CheckoutResult.Success(order.Id);
    }
}
```

The async version is shorter, honors cancellation, runs the two downstream calls in parallel, and never blocks a thread.

## Interview Questions and Answers

### 1. A teammate says "make this endpoint async to make it faster." Is that correct?

**Why this matters:** Confuses async with parallelism — a common misconception that leads to over-engineering and `Task.Run` abuse.

**Answer:** Async does not make a single request faster. It frees the thread during I/O waits so the **server as a whole** scales better under concurrent load. A 200 ms endpoint stays 200 ms whether it is sync or async; the difference is that the async version uses ~0 thread-time during that 200 ms, while the sync version pins a thread for the full duration.

**Trade-off:** Async has a small per-call overhead (state-machine allocation, context-switch). For an endpoint serving 10 req/s on an in-memory cache, sync is fine. For an endpoint at 1,000 req/s waiting on SQL, async is essential.

**Real project:** On a checkout API that called Stripe + SQL + Service Bus synchronously, we saw p99 latency spike from 300 ms to 12 s during traffic peaks because the thread pool couldn't grow fast enough. Converting the whole chain to async (one PR) dropped p99 to 450 ms with no infrastructure change.

### 2. Why is `.Result` dangerous in ASP.NET Core?

**Why this matters:** Tests whether the candidate understands the threading model under web hosts.

**Answer:** `.Result` synchronously blocks the calling thread until the task completes. In ASP.NET Core (which has no synchronization context), this primarily causes **thread-pool starvation**: every blocked request consumes a thread that cannot serve other requests. The pool grows slowly (one thread/second by default), so a burst of blocked calls produces queueing latency. In ASP.NET classic or WPF, a captured sync context could even deadlock.

**Trade-off:** The fix is always to make the caller async too. If you absolutely must call async code from sync, use `Task.Run(() => asyncMethod()).GetAwaiter().GetResult()` from a non-request thread — but treat this as a smell.

**Real project:** An older sync controller wrapped `payments.AuthorizeAsync(...).Result`. Under a Black Friday spike, that one line caused a cascade: thread pool exhausted → new requests queued → health-check timeouts → load balancer marked the instance unhealthy → traffic shifted → same on the next instance. Converting to async resolved it.

### 3. When would you use `ValueTask<T>` instead of `Task<T>`?

**Why this matters:** Probes understanding of allocation cost and when to optimize.

**Answer:** `ValueTask<T>` avoids the `Task` heap allocation when the operation often completes **synchronously**. Use it on hot paths where you have a cache, a pooled buffer, or precomputed result — e.g., `IMemoryCache.GetOrCreateAsync`, `Stream.ReadAsync` in `System.IO.Pipelines`. For typical EF Core / HTTP calls (which always go async), the savings are negligible and `Task<T>` is simpler.

**Trade-off:** `ValueTask<T>` has rules — you can only `await` it once, you cannot `Task.WhenAll` over it without conversion, and misuse can cause subtle bugs. Prefer `Task<T>` unless profiling shows allocations matter.

**Real project:** Our `IFeatureFlagProvider.IsEnabledAsync` returned cached results 99% of the time. Switching to `ValueTask<bool>` removed ~2 million allocations per minute on a service handling 30 k req/s.

### 4. How do you handle cancellation in a background worker that processes Service Bus messages?

**Why this matters:** Tests end-to-end cancellation hygiene under graceful shutdown.

**Answer:** The hosting `BackgroundService` receives a `stoppingToken` in `ExecuteAsync`. Pass it to `ServiceBusProcessor.StartProcessingAsync`. Inside each `ProcessMessageAsync` handler, use `args.CancellationToken` (or link with `stoppingToken`) for every downstream call — EF Core, HTTP, etc. On shutdown, the host triggers the token; in-flight work either completes or is abandoned cleanly so Service Bus redelivers.

**Trade-off:** Be careful what happens **after** an exception or cancellation — you must call `args.CompleteMessageAsync` or `AbandonMessageAsync` deliberately, otherwise the lock expires and the broker redelivers anyway.

**Real project:** During a rolling deployment we noticed Service Bus messages were being "lost." Root cause: handlers ignored `stoppingToken`, kept running after `SIGTERM`, were killed mid-write by Kubernetes. Threading the token through fixed redelivery and idempotency held the rest.

### 5. What is the difference between `Task.WhenAll` and `Parallel.ForEachAsync`?

**Why this matters:** Distinguishes async (I/O concurrency) from parallel (CPU concurrency).

**Answer:** `Task.WhenAll(tasks)` awaits an existing collection of tasks — typically I/O-bound, unbounded concurrency. `Parallel.ForEachAsync(source, options, body)` runs `body` for each item with **bounded** concurrency (`MaxDegreeOfParallelism`) — useful when you need to throttle (e.g., 10 concurrent calls to a rate-limited API). Use `Task.WhenAll` for a known, small fan-out (≤ 100 independent calls); use `Parallel.ForEachAsync` when you have thousands of items and need backpressure.

**Trade-off:** Unbounded `Task.WhenAll` over 10,000 HTTP calls will hit socket limits, exhaust the connection pool, and get rate-limited. Always cap concurrency for large fan-outs.

**Real project:** A nightly job reconciled 50 k orders against Stripe. The first version did `Task.WhenAll` over all of them and immediately hit Stripe's per-second rate limit. Switching to `Parallel.ForEachAsync` with `MaxDegreeOfParallelism = 8` made it predictable and respectful.

### 6. Why do library authors sprinkle `.ConfigureAwait(false)` everywhere?

**Why this matters:** Tests understanding of synchronization contexts.

**Answer:** When you `await` a task, the continuation by default resumes on the captured `SynchronizationContext` (UI thread in WPF, request context in ASP.NET classic). If your library is consumed by a UI app, capturing the context can deadlock when the caller does `.Result`. `.ConfigureAwait(false)` says "I don't care which thread I resume on" and avoids the capture. In **ASP.NET Core** there is no sync context, so it is effectively a no-op for app code — but library code should still use it because it might be consumed elsewhere.

**Trade-off:** Adds noise to every `await`. In application code (not libraries) it is unnecessary; in shared NuGet packages it is a correctness requirement.

**Real project:** A shared `OrderingClient` NuGet was used by both an ASP.NET Core API (worked) and a WPF admin tool (deadlocked). Adding `.ConfigureAwait(false)` to every `await` in the client fixed WPF without affecting the API.

### 7. How would you implement a 2-second timeout on a downstream HTTP call?

**Why this matters:** Tests practical cancellation composition.

**Answer:** Use `CancellationTokenSource.CreateLinkedTokenSource(originalToken)` plus `cts.CancelAfter(TimeSpan.FromSeconds(2))`. Pass `cts.Token` to the call. On timeout an `OperationCanceledException` is thrown; you can map it to a 504 or fall back. In production, prefer Polly's `AsyncTimeoutPolicy` which composes cleanly with retries and circuit breakers.

**Trade-off:** `HttpClient` has its own `Timeout` property, but it applies to the whole client and throws `TaskCanceledException` (which is misleading). A linked token is per-call and explicit.

**Real project:** A pricing service degraded our checkout p99 from 200 ms to 4 s when it was slow. Wrapping the call in Polly with `TimeoutAsync(2s).WrapAsync(retry).WrapAsync(circuitBreaker)` gave us a fast fail and isolation.

### 8. A new dev wrote `await Task.Run(() => SomeAsyncMethod())`. What is wrong?

**Why this matters:** Common anti-pattern.

**Answer:** `Task.Run` schedules work on the thread pool. If `SomeAsyncMethod` is already async, you have just hopped to another pool thread for no benefit — and you've added overhead. The correct form is `await SomeAsyncMethod()`. `Task.Run` is only justified to offload **synchronous CPU-bound** work off the calling thread (e.g., in a desktop UI handler to keep the UI responsive); in ASP.NET Core it almost never helps and usually hurts.

**Trade-off:** None — there is essentially no legitimate use of `Task.Run` around an already-async call in a server app.

**Real project:** A code review caught `await Task.Run(() => _db.SaveChangesAsync())` across a service. Removing the `Task.Run` wrappers reduced thread-pool pressure measurably and removed a class of subtle bugs around `ExecutionContext`.

## Summary Checklist

- [ ] I can explain that async releases the thread during I/O — it does not make work faster.
- [ ] I know which APIs are async (`SaveChangesAsync`, `SendAsync`, `UploadAsync`) and use them throughout the call chain.
- [ ] I pass `CancellationToken` through every async method and accept it from `HttpContext.RequestAborted`.
- [ ] I never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` in request paths.
- [ ] I never write `async void` outside event handlers.
- [ ] I use `Task.WhenAll` for independent fan-out and `Parallel.ForEachAsync` for bounded concurrency.
- [ ] I know when `ValueTask<T>` is appropriate and when it is over-engineering.
- [ ] I add `.ConfigureAwait(false)` in library code but not in app code.
- [ ] I compose timeouts and retries with Polly rather than rolling my own.
- [ ] I honor `stoppingToken` in `BackgroundService` so deployments are graceful.
