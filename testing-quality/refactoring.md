# Refactoring

## What It Is

Refactoring is the discipline of improving the internal structure of code **without changing its observable behavior**. The API contract, the HTTP responses, the database state, and the side effects all stay identical — only the shape of the code changes.

It's not "rewrite the module" and it's not "add a feature while cleaning up." Both of those mix concerns. Real refactoring is a series of small, safe, behavior-preserving transformations: extract a method, rename a variable, introduce a parameter object, replace a nested `if` chain with polymorphism, pull a class out of a god service.

A tiny before/after captures the idea:

```csharp
// Before — pricing logic inlined in a controller
[HttpPost]
public async Task<IActionResult> Checkout(CheckoutRequest req)
{
    var subtotal = req.Items.Sum(i => i.Price * i.Quantity);
    var discount = req.CouponCode == "WELCOME10" ? subtotal * 0.10m : 0m;
    var tax = (subtotal - discount) * 0.08m;
    var total = subtotal - discount + tax;
    // ... 40 more lines
}

// After — extracted into a tested service
[HttpPost]
public async Task<IActionResult> Checkout(CheckoutRequest req, CancellationToken ct)
{
    var quote = await _pricing.QuoteAsync(req.Items, req.CouponCode, ct);
    return Ok(quote);
}
```

The HTTP response is identical. The code is now testable in isolation.

## Why It Exists

Software entropy is real. Every feature, hotfix, and "just this once" shortcut adds weight. After a year, the `OrderController` has a 400-line action method, the `BillingService` has 17 private helpers nobody dares to touch, and changing the tax calculation means reading three classes and praying.

Refactoring exists because the alternatives are worse: **big-bang rewrites** ship late, lose business logic that was never documented, and rarely make it to production; **leaving the code alone** makes every future change slower and riskier until the team stops shipping. Martin Fowler's *Refactoring* (1999, 2nd ed. 2018) formalized the practice precisely because teams needed a vocabulary and a safe procedure for cleaning code without breaking customers.

In .NET specifically, the rise of unit testing (xUnit, NUnit), powerful IDE tooling (Rider, Visual Studio, ReSharper), and analyzer-driven feedback (Roslyn) made small, frequent refactors safe enough to be a daily habit instead of a quarterly project.

## When To Use It

**Use refactoring when:**

- You're about to add a feature and the surrounding code is hard to change — refactor first, feature second ("make the change easy, then make the easy change").
- A bug is hard to reproduce because logic is scattered.
- A class has grown beyond ~300 lines or a method beyond ~30 lines.
- Duplication has appeared in two or more places and you've just been asked to change all of them.
- You're onboarding to legacy code and need to understand it — extracting and renaming as you read is a learning tool.
- Tests are slow or fragile because the code under test is tangled with infrastructure.

**Do not use refactoring when:**

- You don't have tests or a way to verify behavior — write **characterization tests** first that pin down current behavior, then refactor.
- The code is going to be deleted next sprint.
- You're tempted to mix behavior changes into the same commit (split them).
- The deadline is tomorrow and the code works — log the debt, ship, refactor next sprint.
- You're refactoring for personal style preference, not for measurable change cost.

## Why It Is Important

In production .NET systems, refactoring is what keeps a service **deployable** for years. A codebase that can't be refactored becomes a codebase that can't be changed safely, which becomes a codebase that gets rewritten, which usually means a multi-quarter project with high risk of regressions.

Concretely it drives:

- **Mean Time To Change (MTTC):** how long it takes to add a small feature. Clean code keeps this in hours, not days.
- **Defect rate per change:** smaller, focused methods have fewer edge cases per change.
- **Onboarding speed:** a new engineer should be productive in the first week.
- **Confidence to deploy on Friday:** if you're afraid to touch the payment service, refactoring is overdue.
- **Cost of compliance audits:** PCI-DSS, SOC 2, and HIPAA reviewers ask "show me where the card data is handled." A refactored, single-responsibility service answers that in one file.

In cloud-native systems, refactoring also enables **architectural evolution**: extracting a module into its own class is the first step toward extracting it into its own microservice via the Strangler Fig pattern.

## How It's Used in C# / .NET

The .NET ecosystem has first-class refactoring support:

**IDE-driven refactorings** (safe, automated transformations):

| Refactoring | Visual Studio / Rider shortcut | What it does |
|---|---|---|
| Extract Method | `Ctrl+R, M` / `Ctrl+Alt+M` | Pulls selected code into a new method |
| Rename | `F2` / `Shift+F6` | Renames symbol across the whole solution |
| Introduce Parameter | `Ctrl+R, P` / `Ctrl+Alt+P` | Turns a local variable into a parameter |
| Extract Interface | (context menu) | Creates an `I*` interface from a class's public members |
| Inline Method | `Ctrl+R, I` / `Ctrl+Alt+N` | Replaces calls with the method body |
| Change Signature | `Ctrl+R, S` / `Ctrl+F6` | Reorders, adds, or removes parameters everywhere |

**Roslyn analyzers and code fixes** (`Microsoft.CodeAnalysis.NetAnalyzers`, `StyleCop.Analyzers`, `SonarAnalyzer.CSharp`) surface refactoring opportunities as squiggles with one-click fixes — e.g. CA1822 ("member can be marked static"), IDE0290 ("use primary constructor"), S1186 ("methods should not be empty").

**Tests as the safety net:**

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

**`dotnet format`** applies whitespace and `using` ordering refactors uniformly across the solution. Wire it into a pre-commit hook so style discussions never reach a pull request.

**Common .NET refactoring patterns:**

- **Extract Method** — break a 200-line action method into named steps (`ValidateRequest`, `ApplyDiscounts`, `CapturePayment`).
- **Introduce Parameter Object** — when a method takes 6+ parameters, group them: `CreateOrder(customerId, items, couponCode, shippingAddress, billingAddress, notes)` becomes `CreateOrder(CreateOrderRequest req)`.
- **Replace Conditional with Polymorphism** — a `switch` on `PaymentMethod` becomes `IPaymentProcessor` with `StripeProcessor`, `PaypalProcessor`, `BankTransferProcessor`.
- **Replace Magic Number with Constant** — `* 0.08m` becomes `* TaxRates.California`.
- **Extract Class** — `OrderService` doing pricing, persistence, and notifications becomes three classes.
- **Strangler Fig** — wrap a legacy `LegacyBillingService` behind `IBillingService`, route new traffic to a new implementation, delete the old code once traffic is 100% on the new path.

## Advantages

- **Reduces change cost** — the next feature in the same area is faster and safer.
- **Reveals design** — the right abstractions emerge from cleaning, not from upfront speculation.
- **Improves testability** — small, focused units are trivial to unit test.
- **Reduces bug surface** — fewer branches and shorter methods mean fewer places for defects to hide.
- **Enables architectural moves** — Strangler Fig lets you migrate to microservices, swap ORMs, or replace a payment provider without a rewrite.
- **Improves onboarding** — readable code is documentation that can't go stale.
- **Compounds over time** — small refactors weekly produce a healthy codebase in a year.

## Disadvantages

- **Time investment with no visible feature output** — product owners may push back.
- **Risk of behavior change** without tests — a "harmless" rename can break reflection-based code (serializers, DI, `[FromBody]` bindings).
- **Merge conflicts** — large refactors collide with feature branches.
- **Diff noise** in code review — a 2-line behavior change buried in a 500-line refactor is dangerous.
- **Analysis paralysis** — over-refactoring (extracting 3-line methods, wrapping every primitive in a value object) hurts readability.
- **Premature abstraction** — extracting an interface before you have two implementations adds indirection without value.

## Common Mistakes

### 1. Refactoring without tests

```csharp
// BAD — touching a 300-line ProcessOrder method with no tests
public async Task<OrderResult> ProcessOrder(OrderRequest req)
{
    // ... extract methods, rename variables, hope nothing breaks
}
```

**Fix** — write **characterization tests** first. Capture current inputs and outputs (even if behavior is wrong), pin them down with tests, then refactor.

```csharp
[Fact]
public async Task ProcessOrder_KnownInput_ProducesKnownOutput()
{
    var req = JsonSerializer.Deserialize<OrderRequest>(File.ReadAllText("fixtures/order-001-request.json"));
    var expected = File.ReadAllText("fixtures/order-001-response.json");
    var actual = JsonSerializer.Serialize(await _sut.ProcessOrder(req));
    Assert.Equal(expected, actual);
}
```

### 2. Mixing refactoring with behavior changes

```csharp
// BAD — one PR that renames, extracts, AND changes the tax rate from 8% to 9%
```

**Fix** — two PRs. PR #1: pure refactor, all tests green. PR #2: behavior change, one focused diff. Reviewers can actually see what changed.

### 3. Big-bang rewrites disguised as refactoring

A "refactor" branch that lives for three months and changes 80 files is a rewrite. It will lose business rules nobody documented and ship late.

**Fix** — Strangler Fig. Keep the old code running, route a slice of traffic to the new implementation, expand the slice, delete the old code last.

### 4. Extracting too eagerly

```csharp
// BAD — extracted because the method had 4 lines, not because it had a name
private decimal AddTax(decimal x) => x * 1.08m;
```

**Fix** — extract when a block has a **name that adds meaning** or when it's called from 2+ places. A one-liner used once is fine inline.

### 5. Renaming public APIs without a deprecation path

```csharp
// BAD — renamed a controller route from /api/orders to /api/v2/orders in a refactor PR
```

**Fix** — public APIs (HTTP routes, NuGet package signatures, message contracts) require versioning or `[Obsolete]` with a deprecation window. See [aspnet-core/api-versioning.md](../aspnet-core/api-versioning.md).

### 6. Refactoring with reflection-fragile changes

```csharp
// BAD — renamed property Email -> EmailAddress, broke JSON deserialization from upstream services
public record CustomerDto(string EmailAddress);
```

**Fix** — preserve the wire contract with `[JsonPropertyName("email")]` or add the alias before renaming.

### 7. No commit hygiene

A single commit titled "refactor billing" with 1,200 changed lines is impossible to bisect when something breaks.

**Fix** — one refactoring per commit. `git log` should read like a tutorial: "Extract OrderPricingService", "Rename CalcTax -> CalculateTaxAmount", "Introduce PricingRequest record".

## Best Practices

- **Refactor before adding a feature**, not after — "make the change easy, then make the easy change."
- **Run tests after every transformation** — not at the end of the day.
- **Keep refactoring commits separate** from behavior-change commits.
- **Use the IDE's automated refactorings** — they're safer than manual edits.
- **Write characterization tests** for legacy code before touching it.
- **Use the Boy Scout Rule** — leave the file cleaner than you found it, in small increments.
- **Time-box exploratory refactors** — if you've been at it 2 hours and it's bigger than you thought, revert and plan it.
- **Use feature flags** for risky restructurings — `if (_features.UseNewPricingEngine)` lets you compare old and new in production.
- **Track refactoring debt explicitly** — a `// TODO: extract` comment with a ticket number is honest; ignoring smells is not.
- **Prefer composition over inheritance** when extracting — interfaces and injected dependencies are easier to test than base classes.

## Related Concepts

- [testing-quality/unit-testing.md](unit-testing.md) — the safety net that makes refactoring safe.
- [testing-quality/clean-code.md](clean-code.md) — the target state refactoring drives toward.
- [testing-quality/code-review-practices.md](code-review-practices.md) — where most refactoring opportunities are spotted.
- [testing-quality/static-analysis.md](static-analysis.md) — surfaces refactor candidates automatically.
- [testing-quality/testable-code-design.md](testable-code-design.md) — how to write code that doesn't need to be refactored to be tested.
- [architecture/solid-principles.md](../architecture/solid-principles.md) — the design principles refactoring tries to satisfy.
- [architecture/clean-architecture.md](../architecture/clean-architecture.md) — a target architecture that emerges from refactoring.
- [csharp/dependency-injection.md](../csharp/dependency-injection.md) — extracted classes typically become injected dependencies.

## Real-World Usage

**ASP.NET Core APIs** — controller actions tend to accumulate validation, mapping, business logic, and persistence. The standard refactor is to push validation into FluentValidation, mapping into AutoMapper or hand-written extension methods, business logic into application services, and persistence behind repositories. The controller shrinks to dispatch + return.

**Azure Functions** — function entry points should be thin. A common refactor is extracting the function body into a `*Handler` class registered in DI, so the function class only handles binding and serialization. This also makes the handler unit-testable without the Functions runtime.

**Background workers / Azure Service Bus consumers** — long `ProcessMessage` methods get refactored into pipelines: deserialize → validate → handle → ack. Each step is a method (or a class) that can be tested in isolation. See [azure/azure-service-bus.md](../azure/azure-service-bus.md).

**Legacy monolith → microservices** — Strangler Fig is the standard play. Wrap the legacy module behind an interface, deploy a new service that implements the same interface, route a percentage of traffic to the new service via a feature flag or API gateway rule, scale up, retire the legacy code.

**EF Core data layer** — repositories that have grown to 50+ methods get split by aggregate (`IOrderRepository`, `ICustomerRepository`, `IInvoiceRepository`). N+1 queries discovered in profiling get refactored into `.Include()` chains or projection queries. See [data-access/query-performance.md](../data-access/query-performance.md).

**CI/CD** — `dotnet format --verify-no-changes` in GitHub Actions or Azure DevOps Pipelines blocks PRs with style drift, so style refactoring happens locally before review.

**Multi-tenant SaaS** — extracting a `ITenantContext` from scattered `HttpContext.Items["TenantId"]` reads is one of the most common (and highest value) refactors in a B2B .NET codebase.

## Code Example — Before and After

A real refactor of a payment capture flow.

**Before — everything in the controller:**

```csharp
[ApiController]
[Route("api/payments")]
public class PaymentsController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IConfiguration _config;
    private readonly ILogger<PaymentsController> _logger;

    public PaymentsController(AppDbContext db, IConfiguration config, ILogger<PaymentsController> logger)
    {
        _db = db;
        _config = config;
        _logger = logger;
    }

    [HttpPost("capture")]
    public async Task<IActionResult> Capture([FromBody] CaptureRequest req)
    {
        if (req == null) return BadRequest("Request is null");
        if (req.InvoiceId == Guid.Empty) return BadRequest("InvoiceId required");
        if (req.Amount <= 0) return BadRequest("Amount must be positive");
        if (string.IsNullOrEmpty(req.CardToken)) return BadRequest("CardToken required");

        var invoice = await _db.Invoices.FindAsync(req.InvoiceId);
        if (invoice == null) return NotFound();
        if (invoice.Status == "Paid") return Conflict("Already paid");
        if (invoice.Amount != req.Amount) return BadRequest("Amount mismatch");

        var apiKey = _config["Stripe:ApiKey"];
        var http = new HttpClient();
        http.DefaultRequestHeaders.Authorization = new("Bearer", apiKey);
        var body = new StringContent(
            $"{{\"amount\":{(int)(req.Amount * 100)},\"currency\":\"usd\",\"source\":\"{req.CardToken}\"}}",
            Encoding.UTF8, "application/json");
        var response = await http.PostAsync("https://api.stripe.com/v1/charges", body);
        if (!response.IsSuccessStatusCode)
        {
            _logger.LogError("Stripe failed: {Status}", response.StatusCode);
            return StatusCode(502, "Payment provider error");
        }

        invoice.Status = "Paid";
        invoice.PaidAtUtc = DateTime.UtcNow;
        await _db.SaveChangesAsync();

        return Ok(new { invoice.Id, invoice.Status });
    }
}
```

Problems: validation mixed with orchestration mixed with HTTP client construction mixed with persistence; no `CancellationToken`; `HttpClient` constructed per request (socket exhaustion); secrets read inline; impossible to unit-test without a real Stripe account.

**After — refactored in small, behavior-preserving steps:**

```csharp
// 1. Validation moved to FluentValidation
public sealed class CaptureRequestValidator : AbstractValidator<CaptureRequest>
{
    public CaptureRequestValidator()
    {
        RuleFor(x => x.InvoiceId).NotEmpty();
        RuleFor(x => x.Amount).GreaterThan(0);
        RuleFor(x => x.CardToken).NotEmpty();
    }
}

// 2. Payment provider behind an interface (testable, swappable)
public interface IPaymentGateway
{
    Task<PaymentResult> CaptureAsync(Guid invoiceId, decimal amount, string cardToken, CancellationToken ct);
}

public sealed class StripePaymentGateway : IPaymentGateway
{
    private readonly HttpClient _http;            // injected via IHttpClientFactory
    private readonly ILogger<StripePaymentGateway> _logger;

    public StripePaymentGateway(HttpClient http, ILogger<StripePaymentGateway> logger)
    {
        _http = http;
        _logger = logger;
    }

    public async Task<PaymentResult> CaptureAsync(Guid invoiceId, decimal amount, string cardToken, CancellationToken ct)
    {
        var response = await _http.PostAsJsonAsync("v1/charges", new
        {
            amount = (int)(amount * 100),
            currency = "usd",
            source = cardToken,
            idempotency_key = invoiceId.ToString()
        }, ct);

        if (!response.IsSuccessStatusCode)
        {
            _logger.LogError("Stripe capture failed for {InvoiceId} with status {Status}", invoiceId, response.StatusCode);
            return PaymentResult.Failure("provider_error");
        }
        return PaymentResult.Success();
    }
}

// 3. Business orchestration in an application service
public sealed class PaymentCaptureService
{
    private readonly AppDbContext _db;
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentCaptureService> _logger;

    public PaymentCaptureService(AppDbContext db, IPaymentGateway gateway, ILogger<PaymentCaptureService> logger)
    {
        _db = db;
        _gateway = gateway;
        _logger = logger;
    }

    public async Task<CaptureResult> CaptureAsync(CaptureRequest req, CancellationToken ct)
    {
        var invoice = await _db.Invoices.FindAsync(new object[] { req.InvoiceId }, ct);
        if (invoice is null) return CaptureResult.NotFound;
        if (invoice.Status == InvoiceStatus.Paid) return CaptureResult.AlreadyPaid;
        if (invoice.Amount != req.Amount) return CaptureResult.AmountMismatch;

        var payment = await _gateway.CaptureAsync(invoice.Id, req.Amount, req.CardToken, ct);
        if (!payment.IsSuccess) return CaptureResult.ProviderError;

        invoice.MarkPaid(DateTime.UtcNow);
        await _db.SaveChangesAsync(ct);
        return CaptureResult.Ok(invoice.Id);
    }
}

// 4. Controller is now a thin dispatcher
[ApiController]
[Route("api/payments")]
public class PaymentsController : ControllerBase
{
    private readonly PaymentCaptureService _service;
    public PaymentsController(PaymentCaptureService service) => _service = service;

    [HttpPost("capture")]
    public async Task<IActionResult> Capture([FromBody] CaptureRequest req, CancellationToken ct)
    {
        var result = await _service.CaptureAsync(req, ct);
        return result.Outcome switch
        {
            CaptureOutcome.Ok            => Ok(new { result.InvoiceId }),
            CaptureOutcome.NotFound      => NotFound(),
            CaptureOutcome.AlreadyPaid   => Conflict(new ProblemDetails { Title = "Invoice already paid" }),
            CaptureOutcome.AmountMismatch=> BadRequest(new ProblemDetails { Title = "Amount mismatch" }),
            _                            => StatusCode(502, new ProblemDetails { Title = "Payment provider error" })
        };
    }
}

// 5. Wire-up in Program.cs
builder.Services.AddHttpClient<IPaymentGateway, StripePaymentGateway>(c =>
{
    c.BaseAddress = new Uri("https://api.stripe.com/");
    c.DefaultRequestHeaders.Authorization = new("Bearer", builder.Configuration["Stripe:ApiKey"]);
});
builder.Services.AddScoped<PaymentCaptureService>();
builder.Services.AddValidatorsFromAssemblyContaining<CaptureRequestValidator>();
```

Each step (extract validator, extract gateway interface, extract service, slim controller, wire DI) was a separate commit with the full test suite green. No behavior change — same HTTP responses, same database state — but now every piece is unit-testable and the gateway can be swapped to Adyen without touching the controller.

## Interview Questions and Answers

### 1. How do you refactor a legacy method that has no tests?

**Why this matters:** Most production refactoring happens in code that was never tested. The interviewer wants to know if you'll just dive in (dangerous) or build a safety net first.

**Answer:** I write **characterization tests** before touching anything. I capture real inputs and outputs from logs or production traces, save them as fixtures, and assert the current behavior — even if it's wrong. With that net in place, I refactor in small steps, running tests after each one. Once the code is restructured and testable, I write proper unit tests targeting the new seams and delete the characterization tests if they become redundant.

**Trade-off:** Characterization tests pin down current behavior, including bugs. You have to be deliberate about distinguishing "this is the intended contract" from "this is a bug we'll fix separately."

**Real project:** Inheriting a legacy invoice generator with no tests, I captured 50 historical invoice generations, ran them through a snapshot test, then extracted the tax calculation. One snapshot diff revealed a rounding bug that had been in production for two years — we fixed it as a separate, visible commit.

### 2. Why should refactoring and behavior changes be in separate commits or PRs?

**Why this matters:** Mixing them is one of the most common ways to ship bugs. Reviewers can't see the real change inside a sea of cosmetic diffs.

**Answer:** Reviewers focus on the diff. If a PR has 600 changed lines, 595 of which are formatting and renames, the 5 lines that change tax behavior will get rubber-stamped. Splitting refactor and behavior into separate PRs makes the behavior change reviewable in isolation, makes `git bisect` useful when something breaks in production, and lets you revert the behavior change without losing the cleanup.

**Trade-off:** It's more PR overhead — two reviews instead of one. The discipline pays for itself the first time a production incident traces back to a "small fix" hidden in a big refactor.

**Real project:** On a billing service, splitting a refactor PR from the actual VAT-rate change PR meant we could ship the refactor on Tuesday, hold the rate change for the legal team's Thursday sign-off, and never mix the two reviews.

### 3. When would you choose the Strangler Fig pattern over a rewrite?

**Why this matters:** Big-bang rewrites have a notorious failure rate. The interviewer is checking whether you know the production-safe alternative.

**Answer:** Almost always. Strangler Fig lets the old system keep running while the new one grows next to it, with traffic gradually migrated via an interface, feature flag, or routing layer. You can roll back at any percentage, you keep shipping features during the migration, and you don't lose undocumented business rules because the old code is still there as a reference until the cutover is complete. A full rewrite makes sense only for small, well-understood modules with no production traffic.

**Trade-off:** Strangler Fig means running two implementations in parallel for weeks or months, which costs more in infrastructure and cognitive overhead. But it almost always finishes; rewrites often don't.

**Real project:** Migrating a monolith's user notification module to a new microservice, I introduced `INotificationSender` with the legacy implementation, deployed a new Azure Function as a second implementation, routed 5% of traffic via a feature flag, watched error rates and latency in Application Insights, expanded to 100% over three weeks, and deleted the legacy code in the final PR.

### 4. How do you decide when extracting a method is worth it?

**Why this matters:** Over-extraction makes code harder to read, not easier. Junior engineers often extract for line count; seniors extract for meaning.

**Answer:** I extract when the new method has a **name that adds meaning** the inline code doesn't have, or when the same logic appears in two or more places. A 4-line block called once with an obvious name like `total = subtotal + tax;` doesn't need a method. A 4-line block that becomes `ApplyLoyaltyDiscount(customer, cart)` is worth extracting because the name documents intent.

**Trade-off:** Excessive extraction creates "function lasagna" — readers must jump between many small methods to follow one logical flow. Two pages of code split into 30 methods can be harder to read than one well-organized method.

**Real project:** Reviewing a PR that extracted every `if` branch into its own method, I pushed back on the ones whose names were just paraphrases of the condition. We kept the extractions that named business concepts (`IsEligibleForFreeShipping`) and inlined the rest.

### 5. How do you refactor a controller that's grown to 800 lines?

**Why this matters:** Fat controllers are the most common smell in ASP.NET Core codebases. The interviewer wants a concrete plan.

**Answer:** First, characterization tests around the existing endpoints — at minimum, integration tests that hit the routes with realistic payloads and assert the JSON responses. Then I refactor in passes: pass 1 extract validation into FluentValidation; pass 2 extract orchestration into application services (one per use case); pass 3 push persistence behind repositories or directly behind `DbContext` usage in the service; pass 4 introduce DTOs and mappers so the controller doesn't expose domain types. After each pass, every test still passes. The end state is a controller with `[HttpPost]` methods that dispatch to a service and return the result.

**Trade-off:** Done in isolation, this can take a week and produces no visible feature. I bundle it with an actual feature request in the same area so the business value is concrete.

**Real project:** A 900-line `OrdersController` in a B2B SaaS was refactored over three sprints alongside actual order-feature work. End state: 180 lines of controller, 12 application services, 95% unit test coverage of the services, zero behavior change observed by customers.

### 6. What's your workflow when an analyzer flags a refactoring opportunity in CI?

**Why this matters:** Analyzers fire constantly. The team's policy on them shows engineering discipline.

**Answer:** Treat each rule as a deliberate choice: enable as error, enable as warning, or suppress with a justification. New code should never introduce warnings. For existing warnings, I either fix them in the next PR that touches the file (Boy Scout Rule), or for big-impact rules (CA2007 missing `ConfigureAwait`, CA1062 null checks), I batch a dedicated refactor PR. Suppressions go in `.editorconfig` or `[SuppressMessage]` attributes with a `Justification` string — never blanket `#pragma warning disable` without explanation.

**Trade-off:** Treating all warnings as errors can block urgent hotfixes if a legacy file has 50 warnings. The pragmatic answer is `TreatWarningsAsErrors=true` for new projects, opt-in per rule for legacy projects.

**Real project:** Inheriting a 5-year-old codebase with 400 analyzer warnings, we ratcheted: enabled `TreatWarningsAsErrors` only for the rules where the warning count was already zero, then fixed one rule's worth of warnings per sprint until we could enable it project-wide.

### 7. How do you avoid breaking consumers when refactoring a published API?

**Why this matters:** Internal refactoring is safe; refactoring a public contract is a breaking change disguised as cleanup.

**Answer:** Public APIs — HTTP routes, NuGet types, message contracts, gRPC services — are immutable from the consumer's perspective. I either keep the old shape and add a new one beside it (API versioning, `[Obsolete]` with a removal date), or use serialization attributes (`[JsonPropertyName]`, `[DataMember]`) to keep the wire format stable while the C# code changes. I never rename a property on a DTO that's serialized to clients without keeping a backward-compatible alias for at least one release.

**Trade-off:** Maintaining old and new shapes adds code and tests. The alternative — breaking consumers — costs more in customer trust and support load.

**Real project:** Renaming `CustomerEmail` to `EmailAddress` on a webhook payload, I added `[JsonPropertyName("CustomerEmail")]` so existing consumers kept working, documented the new field in release notes, and removed the old name 90 days later after monitoring showed no consumers still depending on it.

## Summary Checklist

- [ ] I can define refactoring as behavior-preserving structural change.
- [ ] I write characterization tests before refactoring legacy code without tests.
- [ ] I keep refactoring commits separate from behavior-change commits.
- [ ] I use the IDE's automated refactorings instead of manual edits.
- [ ] I extract methods only when the name adds meaning or removes duplication.
- [ ] I apply the Strangler Fig pattern for migrating legacy modules.
- [ ] I refactor before adding a feature in messy code, not after.
- [ ] I configure analyzers and `dotnet format` in CI to surface refactor opportunities.
- [ ] I preserve public API contracts with versioning or serialization attributes.
- [ ] I can explain the trade-off between refactoring time and feature delivery.
