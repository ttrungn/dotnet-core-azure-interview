# Clean Code

## What It Is

Clean code is code that another engineer — including future-you — can read, understand, and change safely without spending an hour first decoding what it does. It's defined by what it lets the reader do: name a thing correctly, isolate one responsibility, fit on a screen, and modify one behavior without breaking three others.

A small before/after captures the spirit:

```csharp
// Before — what does this do?
public bool Process(Customer c, decimal a, int t)
{
    if (t == 1 && c.S == "G" && a > 100) return true;
    if (t == 2 && a > 50) return c.A > 30;
    return false;
}

// After — the same logic, written for a reader
public bool IsEligibleForExpressShipping(Customer customer, decimal orderAmount, ShippingTier tier)
{
    return tier switch
    {
        ShippingTier.Standard when customer.Status == LoyaltyStatus.Gold && orderAmount > 100m => true,
        ShippingTier.Express  when orderAmount > 50m => customer.AccountAgeInDays > 30,
        _ => false
    };
}
```

Same behavior. One version requires a debugger to understand; the other reads like a spec.

## Why It Exists

Most code is read far more often than it is written. A line written in five minutes will be read fifty times over its lifetime — during reviews, bug investigations, onboarding, audits, and feature work. If reading takes ten minutes per visit, the cost of poor clarity dwarfs the cost of writing.

Robert Martin's *Clean Code* (2008) popularized the term, but the ideas predate it: Kernighan and Plauger's *Elements of Programming Style* (1974), Kent Beck's "code smells," and Fowler's *Refactoring* all argue the same point — software economics are dominated by maintenance cost, and maintenance cost is dominated by readability.

In the .NET world specifically, the shift to async/await, dependency injection, and microservices made clean code more critical, not less: an unreadable async method with three layers of `Task.Result` and a manually-constructed `HttpClient` is a production incident waiting to happen, and "we'll clean it up later" almost never arrives.

## When To Use It

**Use clean-code practices when:**

- You're writing code that will live longer than the next sprint.
- You're working in a team — clean code is a team contract.
- You're in a domain where bugs have real cost: payments, auth, billing, healthcare, anything regulated.
- You're onboarding new engineers — clean code is the cheapest onboarding doc.
- You're handing off to another team or open-sourcing the project.
- Tests are hard to write — that's a signal the production code isn't clean.

**Do not over-apply clean-code rules when:**

- You're writing a one-off script or migration that will be deleted.
- You're prototyping to validate an idea — clean it up before merging, not while exploring.
- The rule conflicts with framework expectations (e.g., MVC controllers can't always be 20 lines; an Azure Function entry point has fixed shape).
- "Clean" means premature abstraction — three layers of indirection for a one-time operation is not clean.
- Performance-critical hot paths need denser code with measured trade-offs documented in comments.

## Why It Is Important

In production .NET systems, clean code is what turns "this is hard to change" into "this is a normal Tuesday." It drives:

- **Onboarding speed** — a new engineer should ship a real PR in their first week.
- **Defect rate** — small, single-purpose methods have fewer hiding places for bugs.
- **Review throughput** — clear code reviews fast; tangled code burns hours per PR.
- **Refactoring safety** — code with good names and small methods is easier to test, easier to extract, easier to delete.
- **Compliance** — auditors reading payment or PII code need to see clear intent. "Trust me, it's correct" doesn't pass SOC 2.
- **Cost of incidents** — at 2 AM, the on-call engineer reading the code for the first time needs to find the bug in minutes, not hours.
- **Cross-team handoff** — services that change owners (reorgs, acquisitions, contractor handoffs) are only viable when the code is self-explanatory.

## How It's Used in C# / .NET

C# and the .NET conventions make many clean-code rules concrete:

**Naming conventions (microsoft.com/dotnet docs):**

| Element | Convention | Example |
|---|---|---|
| Namespaces, types, public members | PascalCase | `PaymentCaptureService` |
| Parameters and locals | camelCase | `invoiceId`, `customerEmail` |
| Private fields | `_camelCase` | `_logger`, `_db` |
| Constants | PascalCase | `MaxRetryAttempts` |
| Async methods | `*Async` suffix | `CaptureAsync` |
| Interfaces | `I*` prefix | `IPaymentGateway` |
| Abstract base classes | descriptive name, no suffix | `PaymentProcessorBase` |

**Method size guidelines:**

- ≤20 lines typical, hard ceiling around 50 for non-trivial logic.
- One reason to change (SRP applied at the method level).
- One level of abstraction per method — don't mix HTTP parsing with business rules with logging.

**Argument count:**

- 0–2 ideal. 3 acceptable. 4+ → introduce a parameter object (a `record` works great).
- No boolean flag arguments — split into two methods or use an enum.

**Command-Query Separation (CQS):**

```csharp
// BAD — does and returns at the same time
public Invoice CapturePayment(Guid invoiceId)   // returns invoice AND mutates database
{
    var inv = _db.Invoices.Find(invoiceId);
    inv.Pay();
    _db.SaveChanges();
    return inv;
}

// GOOD — split
public Task<Invoice> GetInvoiceAsync(Guid id, CancellationToken ct);   // query: returns data, no mutation
public Task CaptureAsync(Guid invoiceId, CancellationToken ct);        // command: mutates, returns void/result
```

**Modern C# constructs that improve clarity:**

- `record` for DTOs and value objects — equality and immutability by default.
- Pattern matching (`switch` expressions) for clear branching.
- Primary constructors and collection expressions (C# 12+) reduce ceremony.
- `required` modifier prevents partially-initialized objects.
- File-scoped namespaces flatten indentation.
- Top-level statements for small programs and Azure Functions.

**Framework hooks that encourage clean structure:**

- DI containers (`IServiceCollection.AddScoped<T>`) keep classes focused and testable.
- `IOptions<T>` and the Options pattern remove config-fetching from business code.
- `ILogger<T>` with structured logging keeps logging declarative.
- Minimal APIs with route groups and endpoint filters cut controller boilerplate.
- FluentValidation moves validation out of action methods.
- AutoMapper or hand-written mapping extensions move shape conversion to one place.

**Tooling that enforces it:**

- `.editorconfig` for style consistency.
- `dotnet format` to apply rules automatically.
- Analyzers (`Microsoft.CodeAnalysis.NetAnalyzers`, `SonarAnalyzer.CSharp`) for design rules.
- `EnforceCodeStyleInBuild` to fail builds on drift.

## Advantages

- **Faster reading** — the dominant cost in software is reduced.
- **Easier change** — clean code is refactor-friendly and test-friendly.
- **Lower bug rate** — smaller, focused units have fewer interaction bugs.
- **Better reviews** — humans focus on intent, not on deciphering structure.
- **Faster onboarding** — new engineers contribute meaningfully sooner.
- **Better compliance posture** — clear code passes audits with less friction.
- **Predictable performance** — well-structured code is easier to profile and optimize.

## Disadvantages

- **Initial time cost** — writing clean code on the first pass is slower than writing whatever works.
- **Over-engineering risk** — applying every rule mechanically (3-line methods, value object per primitive) hurts more than helps.
- **Subjective edges** — naming, comments, abstraction boundaries are debated; team consensus takes time.
- **Doesn't catch logic bugs** — beautifully named code can still compute the wrong tax.
- **Friction with quick experiments** — strict rules slow down genuine prototyping.
- **Tooling overhead** — analyzer noise on legacy codebases needs careful ratcheting.

## Common Mistakes

### 1. Vague names

```csharp
// BAD
public async Task<bool> Process(object data) { /* ... */ }

public class Manager { /* ... */ }

var x = GetIt();
```

**Fix** — names should answer "what does this return / do / represent?"

```csharp
public async Task<CaptureResult> CapturePaymentAsync(CaptureRequest request, CancellationToken ct);
public sealed class InvoiceLifecycleCoordinator { /* ... */ }
var pendingInvoices = await GetUnpaidInvoicesAsync(ct);
```

### 2. Boolean flag arguments

```csharp
// BAD — what does true mean?
public Task SendNotificationAsync(Guid userId, string message, bool urgent);
```

**Fix** — split or use an enum.

```csharp
public Task SendEmailAsync(Guid userId, string message);
public Task SendUrgentAsync(Guid userId, string message);

// or
public Task SendAsync(Guid userId, string message, NotificationPriority priority);
```

### 3. Bloated controller actions

```csharp
// BAD — 150 lines of validation, business logic, persistence, and HTTP handling
[HttpPost]
public async Task<IActionResult> CheckOut([FromBody] CheckoutRequest req) { /* ... */ }
```

**Fix** — push everything except dispatch out of the controller.

```csharp
[HttpPost]
public async Task<IActionResult> CheckOut(CheckoutRequest req, CancellationToken ct)
{
    var result = await _checkout.HandleAsync(req, ct);
    return result.ToActionResult();
}
```

See the full payment example in [Code Example — Before and After](#code-example--before-and-after).

### 4. Comments that explain what instead of why

```csharp
// BAD
// Loop through items and sum the price * quantity
decimal subtotal = items.Sum(i => i.Price * i.Quantity);

// BAD
// Increment counter
counter++;
```

**Fix** — let the code speak. Use comments only for *why*: unobvious reasons, business rules, or workarounds.

```csharp
decimal subtotal = items.Sum(i => i.Price * i.Quantity);

// Per finance team 2024-10: refund window extends 7 days during Black Friday week.
var refundWindow = IsBlackFridayWeek(now) ? TimeSpan.FromDays(37) : TimeSpan.FromDays(30);
```

### 5. Magic numbers and strings

```csharp
// BAD
if (order.Status == 3) { /* ... */ }
if (response.StatusCode == 422) { /* ... */ }
var commission = total * 0.085m;
```

**Fix** — name them.

```csharp
if (order.Status == OrderStatus.Shipped) { /* ... */ }
if (response.StatusCode == StatusCodes.Status422UnprocessableEntity) { /* ... */ }
const decimal AffiliateCommissionRate = 0.085m;
var commission = total * AffiliateCommissionRate;
```

### 6. Mixing levels of abstraction

```csharp
// BAD — high-level business logic next to low-level byte manipulation
public async Task ProcessOrder(Order order)
{
    var customer = await _customers.GetAsync(order.CustomerId);
    var jsonBytes = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(order));
    using var ms = new MemoryStream(jsonBytes);
    await _blobs.UploadAsync($"orders/{order.Id}.json", ms);
    order.MarkArchived();
}
```

**Fix** — keep one level of abstraction per method; extract the low-level parts.

```csharp
public async Task ProcessAsync(Order order, CancellationToken ct)
{
    var customer = await _customers.GetAsync(order.CustomerId, ct);
    await ArchiveOrderAsync(order, ct);
    order.MarkArchived();
}

private Task ArchiveOrderAsync(Order order, CancellationToken ct) =>
    _archive.SaveAsync($"orders/{order.Id}.json", order, ct);
```

### 7. Doing two things at once (CQS violation)

```csharp
// BAD — query that mutates state
public Customer GetCustomer(Guid id)
{
    var c = _db.Customers.Find(id);
    c.LastAccessedUtc = DateTime.UtcNow;
    _db.SaveChanges();
    return c;
}
```

**Fix** — separate the read from the write.

```csharp
public Task<Customer?> GetCustomerAsync(Guid id, CancellationToken ct) =>
    _db.Customers.FindAsync(new object[] { id }, ct).AsTask();

public Task RecordAccessAsync(Guid customerId, CancellationToken ct) =>
    _customerAccessLog.AppendAsync(customerId, _clock.UtcNow, ct);
```

## Best Practices

- **Name things to reveal intent** — methods are verbs, types and variables are nouns.
- **Keep methods short and single-purpose** — under ~20 lines is a good ambient target.
- **Limit parameters to 3 or fewer** — introduce a `record` parameter object beyond that.
- **No boolean flag arguments** — split methods or use enums.
- **One level of abstraction per method** — don't mix domain logic with infrastructure.
- **Apply Command-Query Separation** — methods either return data or mutate state, not both.
- **Use modern C# features** — `record`, pattern matching, `required`, primary constructors.
- **Let names replace comments** — comments explain *why* (business rules, workarounds), never *what*.
- **Constants for magic numbers and strings** — every literal that has meaning gets a name.
- **Keep controllers and function entry points thin** — dispatch to a service, return the result.
- **Delete dead code immediately** — version control remembers; don't.
- **Apply the Boy Scout Rule** — leave each file cleaner than you found it, in small steps.

## Related Concepts

- [testing-quality/refactoring.md](refactoring.md) — the technique for moving messy code toward clean code.
- [testing-quality/code-review-practices.md](code-review-practices.md) — where clean-code violations are caught and discussed.
- [testing-quality/static-analysis.md](static-analysis.md) — many clean-code rules are encoded as analyzer rules.
- [testing-quality/testable-code-design.md](testable-code-design.md) — testable code and clean code are nearly the same thing.
- [architecture/solid-principles.md](../architecture/solid-principles.md) — clean code at the class and module level.
- [csharp/dependency-injection.md](../csharp/dependency-injection.md) — DI is the structural foundation that enables clean, testable services.
- [csharp/records-and-immutability.md](../csharp/records-and-immutability.md) — `record` types are the modern clean-code default for DTOs.
- [csharp/nullable-reference-types.md](../csharp/nullable-reference-types.md) — explicit nullability removes a whole class of unclear code.

## Real-World Usage

**ASP.NET Core APIs** — clean controllers do four things: receive the request, dispatch to a service, map the result, return an HTTP response. Validation lives in FluentValidation, business logic in application services, persistence behind repositories or `DbContext`. The controller becomes a 5-line method.

**Azure Functions** — function classes should contain only the binding attributes and a call into an injected handler. The handler is a normal C# class with no Functions dependencies, fully unit-testable.

**Background workers / Service Bus consumers** — long `ProcessMessage` methods become pipelines: `Deserialize → Validate → Handle → Acknowledge`. Each step is named, each is small, each is testable.

**EF Core data layer** — query methods are short and named after what they return (`GetUnpaidInvoicesForTenantAsync`), not after the SQL they generate. Projections to DTOs happen close to the query, not deep in the controller.

**Multi-tenant SaaS** — `TenantId` flows through `ITenantContext`, not as a string parameter on every method. Code reads as "fetch invoices for the current tenant" not "fetch invoices for tenant ID passed as the third argument."

**Domain logic** — entities encapsulate behavior, not just data. `invoice.MarkPaid(now)` is clearer than `invoice.Status = "Paid"; invoice.PaidAtUtc = now;` scattered across services.

**CI/CD** — `.editorconfig`, analyzers, and `dotnet format --verify-no-changes` enforce naming and structure rules so reviews don't burn time on style.

**On-call** — clean code shortens incidents. The runbook says "open `PaymentCaptureService.CaptureAsync` and check the result switch" — and that file is 60 lines, not 600.

## Code Example — Before and After

A real refactor of a checkout endpoint.

**Before — a typical 200-line action method:**

```csharp
[ApiController]
[Route("api/orders")]
public class OrderController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IConfiguration _cfg;
    private readonly ILogger<OrderController> _log;
    private readonly HttpClient _http;

    public OrderController(AppDbContext db, IConfiguration cfg, ILogger<OrderController> log)
    {
        _db = db;
        _cfg = cfg;
        _log = log;
        _http = new HttpClient();   // fresh HttpClient per controller — socket exhaustion
    }

    [HttpPost("checkout")]
    public async Task<IActionResult> CheckOut([FromBody] dynamic body, bool sendEmail = true)
    {
        try
        {
            // validation
            if (body == null) return BadRequest("body required");
            Guid customerId = body.customerId;
            if (customerId == Guid.Empty) return BadRequest("customerId required");
            var items = body.items;
            if (items == null || items.Count == 0) return BadRequest("items required");
            string coupon = body.coupon;

            // load customer
            var cust = _db.Customers.FirstOrDefault(c => c.Id == customerId);
            if (cust == null) return NotFound("customer");
            if (cust.Status == 3) return Forbid();   // 3 = banned

            // calculate subtotal
            decimal sub = 0;
            foreach (var item in items)
            {
                Guid pid = item.productId;
                int qty = item.quantity;
                var product = _db.Products.FirstOrDefault(p => p.Id == pid);
                if (product == null) return BadRequest("product " + pid + " not found");
                sub += product.Price * qty;
            }

            // discount
            decimal disc = 0;
            if (coupon == "WELCOME10") disc = sub * 0.10m;
            else if (coupon == "VIP25" && cust.Status == 1) disc = sub * 0.25m;
            else if (coupon == "FLAT5") disc = 5m;

            // tax
            decimal tax = (sub - disc) * 0.08m;
            decimal total = sub - disc + tax;

            // payment
            var apiKey = _cfg["Stripe:ApiKey"];
            _http.DefaultRequestHeaders.Authorization = new("Bearer", apiKey);
            var pay = await _http.PostAsync("https://api.stripe.com/v1/charges",
                new StringContent($"{{\"amount\":{(int)(total*100)}}}"));
            if (!pay.IsSuccessStatusCode) { _log.LogError("pay failed"); return StatusCode(502); }

            // persist
            var order = new Order { CustomerId = customerId, Subtotal = sub, Discount = disc, Tax = tax, Total = total, Status = 1 };
            _db.Orders.Add(order);
            _db.SaveChanges();

            // email
            if (sendEmail)
            {
                await _http.PostAsync(_cfg["Email:Url"],
                    new StringContent($"to={cust.Email}&body=Order {order.Id} confirmed"));
            }

            return Ok(new { id = order.Id, total });
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "checkout failed");
            return StatusCode(500, "Something went wrong");
        }
    }
}
```

Problems with every line: `dynamic` body, magic numbers for status, fresh `HttpClient`, no `CancellationToken`, blocking-style EF calls, business logic in the controller, bool flag arg, catch-all exception, missing structured logging, magic strings for coupon codes, no validation framework.

**After — clean, decomposed, each unit testable:**

```csharp
// CheckoutRequest.cs — strongly typed input
public sealed record CheckoutRequest(
    Guid CustomerId,
    IReadOnlyList<CheckoutLine> Items,
    string? CouponCode);

public sealed record CheckoutLine(Guid ProductId, int Quantity);

// CheckoutRequestValidator.cs — validation via FluentValidation
public sealed class CheckoutRequestValidator : AbstractValidator<CheckoutRequest>
{
    public CheckoutRequestValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty();
        RuleForEach(x => x.Items).ChildRules(line =>
        {
            line.RuleFor(l => l.ProductId).NotEmpty();
            line.RuleFor(l => l.Quantity).GreaterThan(0);
        });
    }
}

// PricingService.cs — single responsibility: compute price
public sealed class PricingService
{
    private const decimal TaxRate = 0.08m;
    private static readonly IReadOnlyDictionary<string, ICouponPolicy> Coupons =
        new Dictionary<string, ICouponPolicy>(StringComparer.OrdinalIgnoreCase)
        {
            ["WELCOME10"] = new PercentageOff(0.10m),
            ["VIP25"]     = new PercentageOffForVip(0.25m),
            ["FLAT5"]     = new FlatDiscount(5m),
        };

    public PriceQuote Quote(Customer customer, IReadOnlyList<Product> products, IReadOnlyList<CheckoutLine> lines, string? couponCode)
    {
        var subtotal = lines.Sum(l => products.First(p => p.Id == l.ProductId).Price * l.Quantity);
        var discount = couponCode is not null && Coupons.TryGetValue(couponCode, out var policy)
            ? policy.CalculateDiscount(customer, subtotal)
            : 0m;
        var tax = (subtotal - discount) * TaxRate;
        return new PriceQuote(subtotal, discount, tax, subtotal - discount + tax);
    }
}

public sealed record PriceQuote(decimal Subtotal, decimal Discount, decimal Tax, decimal Total);

// CheckoutService.cs — orchestrates the use case
public sealed class CheckoutService
{
    private readonly AppDbContext _db;
    private readonly PricingService _pricing;
    private readonly IPaymentGateway _payments;
    private readonly INotificationSender _notifications;
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(AppDbContext db, PricingService pricing, IPaymentGateway payments,
                           INotificationSender notifications, ILogger<CheckoutService> logger)
    {
        _db = db;
        _pricing = pricing;
        _payments = payments;
        _notifications = notifications;
        _logger = logger;
    }

    public async Task<CheckoutResult> HandleAsync(CheckoutRequest req, CancellationToken ct)
    {
        var customer = await _db.Customers.FindAsync(new object[] { req.CustomerId }, ct);
        if (customer is null) return CheckoutResult.CustomerNotFound;
        if (customer.IsBanned)  return CheckoutResult.CustomerBlocked;

        var productIds = req.Items.Select(i => i.ProductId).ToArray();
        var products = await _db.Products.Where(p => productIds.Contains(p.Id)).ToListAsync(ct);
        if (products.Count != productIds.Length) return CheckoutResult.UnknownProduct;

        var quote = _pricing.Quote(customer, products, req.Items, req.CouponCode);
        var payment = await _payments.CaptureAsync(customer.Id, quote.Total, ct);
        if (!payment.IsSuccess) return CheckoutResult.PaymentFailed;

        var order = Order.Create(customer.Id, req.Items, quote);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        await _notifications.SendOrderConfirmationAsync(customer.Email, order.Id, ct);
        return CheckoutResult.Ok(order.Id, quote.Total);
    }
}

// OrderController.cs — thin dispatcher
[ApiController]
[Route("api/orders")]
public sealed class OrderController : ControllerBase
{
    private readonly CheckoutService _checkout;

    public OrderController(CheckoutService checkout) => _checkout = checkout;

    [HttpPost("checkout")]
    public async Task<IActionResult> CheckOut([FromBody] CheckoutRequest req, CancellationToken ct)
    {
        var result = await _checkout.HandleAsync(req, ct);
        return result.Outcome switch
        {
            CheckoutOutcome.Ok                 => Ok(new { result.OrderId, result.Total }),
            CheckoutOutcome.CustomerNotFound   => NotFound(),
            CheckoutOutcome.CustomerBlocked    => Forbid(),
            CheckoutOutcome.UnknownProduct     => BadRequest(new ProblemDetails { Title = "Unknown product" }),
            CheckoutOutcome.PaymentFailed      => StatusCode(502, new ProblemDetails { Title = "Payment failed" }),
            _                                  => StatusCode(500)
        };
    }
}
```

Every name reveals intent. Every method does one thing. No `dynamic`, no magic numbers, no bool flags, no fresh `HttpClient`, every async call carries `CancellationToken`, business logic is fully testable without HTTP or DB infrastructure, and the controller is 12 lines.

## Interview Questions and Answers

### 1. What makes a method name "good"?

**Why this matters:** Naming is the cheapest signal of code quality. A senior engineer can usually judge a codebase from method names alone.

**Answer:** A good method name is a verb phrase that says what the method does and (for queries) what it returns. It should let me predict the body before reading it. `CapturePaymentAsync(invoiceId)` is good — I know it captures, it's async, it operates on an invoice. `Process(data)` is bad — process what, into what? The async suffix, the parameter name, and the verb together form a contract; if the body doesn't match the name, either the name or the body is wrong.

**Trade-off:** Long names can hurt readability when they appear many times in one line. The fix isn't to shorten the name, it's to assign it to a local or extract a method.

**Real project:** Renaming a 30-method service from `Process*` and `Handle*` to specific verbs (`Capture`, `Refund`, `Reconcile`, `Settle`) shortened average code-review time on that service by about half — reviewers stopped needing to re-read each method body to remember what it did.

### 2. How do you keep an ASP.NET Core controller from becoming a god class?

**Why this matters:** Bloated controllers are the most common smell in real .NET codebases. The interviewer wants a concrete strategy, not "use MVC properly."

**Answer:** Controllers should do four things: deserialize the request, dispatch to a service, map the result to an HTTP response, return. Validation goes into FluentValidation. Business logic goes into application services. Persistence goes behind a repository or directly in the service via `DbContext`. Mapping goes into a DTO or AutoMapper profile. If a controller action exceeds 20 lines or has more than one `if` branch, something belongs outside the controller. Result types (a `record`-based discriminated result) make the controller a clean `switch` over outcomes.

**Trade-off:** This adds layers — a small endpoint can feel ceremonious. For genuinely trivial endpoints (a healthcheck, a one-line lookup), inline is fine; layers are for complexity that justifies them.

**Real project:** A `OrdersController` we inherited had 14 actions averaging 80 lines. After applying the pattern, controllers averaged 12 lines; the extracted application services were unit-testable without a TestServer, and PR review velocity in that area doubled.

### 3. When should comments be used in clean code?

**Why this matters:** Junior engineers tend to comment everything; senior engineers tend to comment little but say a lot. The answer reveals which side of that line you're on.

**Answer:** Comments should explain *why*, never *what*. The code already says what — if it doesn't, fix the code (rename, extract, restructure) before adding a comment. The right uses for comments are: business rules with non-obvious sources ("per finance team 2024-10, refund window extends to 37 days during Black Friday"), workarounds for external constraints ("Stripe rejects amounts under 50 cents; route through manual reconciliation"), references to documentation or tickets ("see RFC 7234 §5.2.2"), and warnings about non-obvious consequences ("changing this allocation triggers a full table rebuild — coordinate with DBAs").

**Trade-off:** Some codebases use doc-comments on every public method for IntelliSense and generated docs. That's a different concern (API documentation), and it's worth the cost for libraries and public APIs.

**Real project:** Removed about 800 obvious "loops through items" and "increments counter" comments from a legacy service during a cleanup pass. Kept the 40 that explained business rules; those 40 prevented several incidents over the next year because new engineers actually read them.

### 4. How do you handle a method that needs many parameters?

**Why this matters:** Long parameter lists are a smell. The interviewer is checking whether you know the standard refactors.

**Answer:** Past three parameters, I introduce a parameter object — typically a `record`. It groups related arguments, makes calls self-documenting at the call site, lets me add new fields without breaking call sites (with default values), and works naturally with `[FromBody]` binding in ASP.NET Core. If the parameter object grows unwieldy, that's usually a signal that the method is doing too much — split it. Boolean flag arguments are a special case: I always split them into separate methods or use an enum, because every caller has to remember which `true` is which.

**Trade-off:** Parameter objects add a type to maintain. For methods called from one place, inline parameters are fine. The threshold is "would the call site be clearer with named arguments?" — if yes, make the parameter object.

**Real project:** Refactored a `CreateOrder(customerId, items, coupon, shippingAddress, billingAddress, notes, sendConfirmation, requestSource)` into `CreateOrderRequest`. Callers became `_orders.CreateAsync(new CreateOrderRequest { ... })` — and we caught two bugs where shipping and billing addresses had been swapped.

### 5. How does clean code interact with performance-critical paths?

**Why this matters:** A common pushback on clean code is "but it's slower." The mature answer respects both concerns.

**Answer:** Clean code and fast code are usually compatible — most performance comes from algorithmic choices and avoiding unnecessary I/O, not from packing logic into one method. For hot paths where measurement proves abstraction has real cost (an inner loop running millions of times per request, a serializer in a high-QPS endpoint), I document the trade-off in a comment, keep the dense code in one isolated function with a clear contract, and cover it heavily with tests. The rest of the codebase stays clean. The mistake is denying readability for imagined performance everywhere — usually 99% of the code isn't hot.

**Trade-off:** Premature optimization makes code hard to change and hard to debug. Premature cleanness in a measured hot path can cost real money. Profile first.

**Real project:** A serialization hot path in a high-throughput API showed up in profiling. We replaced the clean `JsonSerializer.Serialize<T>` calls with a hand-tuned `Utf8JsonWriter`-based serializer, documented why in a `// PERF:` comment, and kept the rest of the service clean. The hot-path file was ugly but isolated.

### 6. What's the Boy Scout Rule and how do you apply it in code review?

**Why this matters:** Tests whether you have a sustainable approach to legacy code or just a tendency to either ignore mess or rewrite everything.

**Answer:** "Leave the campground cleaner than you found it." In code, that means: every time you touch a file for a feature or bug, make one small improvement — rename a variable, extract a method, remove a dead `if`, fix an analyzer warning. Don't do a separate cleanup sprint; let the cleanup happen organically with feature work. In review I encourage small adjacent cleanups in PRs, but I push back if the cleanup grows to dominate the diff — that's when it should split into its own PR. The discipline keeps the codebase trending upward without anyone owning a "cleanup project."

**Trade-off:** Tiny diffs of cleanup mixed with feature changes increase diff noise. The compromise is "small enough to obviously not change behavior, large enough to actually improve something."

**Real project:** Over 18 months on a 200k-line codebase, the Boy Scout Rule eliminated about 60% of legacy `var` overuse, 80% of magic numbers in business code, and most missing `CancellationToken` plumbing — without any dedicated cleanup sprint, just consistent small improvements on every feature PR.

### 7. How do you push back on clean-code suggestions in a review that hurt readability?

**Why this matters:** Clean code rules are guidelines, not laws. The answer reveals professional maturity.

**Answer:** I explain the trade-off concretely. "Extracting this 3-line block into a method would mean the reader has to jump to another method to follow one logical step — I think it's clearer inline." Or "Splitting this into two methods would expose a private helper that has no meaning outside this method." I ground it in the reader's experience, not in rules. If the reviewer still disagrees, I usually defer — consistency across the codebase matters and one method shape isn't worth a debate. But I'll raise the pattern in a team discussion if I see it recurring, because we should align on principles, not relitigate every PR.

**Trade-off:** Pushing back too often slows reviews and creates friction. Capitulating too often creates incoherent code shaped by every individual reviewer's preferences.

**Real project:** A reviewer asked me to extract a 4-line guard clause into `EnsureValid(request)` "for clarity." I pointed out the inline version literally was clearer — three lines of explicit checks vs one line that hid them behind a method that did less than its name suggested. The team agreed in the next standup that "clarity through extraction" should require the extracted method to add a real conceptual name, not just be smaller.

## Summary Checklist

- [ ] I name methods, types, and variables to reveal intent — verbs for methods, nouns for types.
- [ ] My methods are short and do one thing at one level of abstraction.
- [ ] I limit parameters to ~3 and use `record` parameter objects beyond that.
- [ ] I avoid boolean flag arguments — I split methods or use enums.
- [ ] I apply Command-Query Separation — methods return or mutate, not both.
- [ ] I replace magic numbers and strings with named constants or enums.
- [ ] I use comments only to explain *why*, never *what*.
- [ ] I keep controllers and function entry points thin — they dispatch and return.
- [ ] I delete dead code immediately and trust version control to remember.
- [ ] I leave each file cleaner than I found it, in small, safe steps.
