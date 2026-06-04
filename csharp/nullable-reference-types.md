# Nullable Reference Types

## What It Is

Nullable Reference Types (NRT) is a C# 8 compiler feature that turns the question "can this reference be `null`?" from an unwritten convention into an **explicit part of the type**. With the feature enabled, `string` means "this should never be null," and `string?` means "this may be null, callers must handle it." The compiler issues warnings (CS86xx series) when code violates these expectations.

NRT is a **static analysis** feature. It does not change runtime behavior — `null` is still `null`, and a deserialized JSON payload can still drop `null` into a `string` property. What changes is that the compiler now treats `null` flow as a first-class type-system concern.

```csharp
#nullable enable

public sealed record Customer(Guid Id, string Name, string? Email);
//                              required ──┘         ──┘ optional

public Customer? FindCustomer(Guid id) // may return null
{
    return _db.Customers.SingleOrDefault(c => c.Id == id);
}

var c = FindCustomer(id);
// var name = c.Name;   // CS8602: Dereference of a possibly null reference
if (c is not null)
    Console.WriteLine(c.Name); // OK — flow analysis narrowed c to Customer
```

Enable it project-wide in the `.csproj`:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

The .NET 6+ templates enable it by default. Most modern NuGet libraries (EF Core, ASP.NET Core, Azure SDKs) are annotated, so the compiler can tell you `_db.Set<T>().FirstOrDefaultAsync(...)` returns `T?`.

## Why It Exists

`NullReferenceException` (NRE) was famously called the "billion-dollar mistake" by Tony Hoare, who introduced null references in ALGOL W in 1965. Before NRT, the C# type system had no way to distinguish "this string might be null" from "this string is always present." That single ambiguity caused a large fraction of production crashes in .NET applications:

- A property defaulted to `null` got dereferenced two layers down.
- A constructor accepted `string name` but didn't validate, so a caller passed `null` and the bug surfaced an hour later in an unrelated logger.
- A method returned `null` to mean "not found"; callers forgot to check.

Various workarounds existed — `Code Contracts`, `[NotNull]` attributes from JetBrains, defensive `if (x == null) throw` at every entry point — but none were universal or enforced by the compiler. NRT (modeled on Kotlin's `String` vs `String?`) finally makes nullness **part of the type signature**, so the compiler enforces it consistently across an entire codebase.

The catch: it's static analysis only. The runtime semantics of `null` are unchanged, so external input (JSON bodies, database rows, reflection) can still inject `null` into a "non-null" field. NRT moves the problem to compile time for code you control, and gives you tools (`[NotNullWhen]`, `[MemberNotNull]`) to express intent precisely.

## When To Use It

**Always enable it for new projects.** It is the default in .NET 6+ templates and there is no good reason to turn it off for greenfield code.

**Enable it gradually in existing projects:**

- Use `<Nullable>enable</Nullable>` at the project level, or `#nullable enable` per-file to migrate incrementally.
- Use `<Nullable>warnings</Nullable>` to get the warnings without changing the type system, as a first migration step.
- Use `<Nullable>annotations</Nullable>` to declare types as nullable without enforcing warnings (lets you ship annotations to consumers ahead of fixing internals).

**Use `T?` when:**

- The field/parameter/return can legitimately be `null` ("not found", "optional"). `Customer? FindCustomer(Guid)`, `string? Email`.
- An API consumer needs to know an empty result is possible.

**Use `T` (non-nullable) when:**

- The value is required by the contract. `string CustomerName`, `Order order` in `Confirm(Order order)`.
- The field is initialized by the constructor and never reassigned to null.

**Do NOT use `!` (null-forgiving operator) when:**

- You don't actually know the value won't be null — `!` only silences the compiler.
- You haven't validated the input — fix the source.

## Why It Is Important

In production .NET services, `NullReferenceException` is consistently in the top three exception types by volume (App Insights' `exceptions` table tells the same story across teams). NRT pushes the detection of those bugs from runtime → compile time, with measurable effect:

- **CI catches them before merge** — no production crash, no rollback.
- **API contracts become self-documenting** — `Customer? FindCustomer(Guid)` is clearer than `Customer FindCustomer(Guid) /* returns null if not found */`.
- **Refactoring is safer** — renaming a property or changing its nullability surfaces every caller as a warning.
- **DTOs and request models** become accurate documentation of which fields are required vs optional, feeding into OpenAPI schema generation.
- **EF Core** uses the annotations to decide `NOT NULL` vs `NULL` columns in migrations — a misannotation creates schema drift.

The combined effect: a measurable drop in NRE-related incident tickets after a team enables and cleans up NRT.

## How It's Used in C# / .NET

| Feature | Use |
| --- | --- |
| `#nullable enable` / `<Nullable>enable</Nullable>` | Turn the feature on. |
| `string` | Non-nullable reference. The compiler warns if you assign `null` or dereference a possibly-null value. |
| `string?` | Nullable reference. Callers must handle null. |
| `!` (null-forgiving) | "Trust me, this isn't null." Suppresses the warning at one site. |
| `??` (null-coalescing) | `value ?? "fallback"`. |
| `??=` | `_cache ??= new();` assign if currently null. |
| `?.` (null-conditional) | `customer?.Address?.City`. |
| `where T : notnull` | Generic constraint — `T` must be non-null reference or non-nullable value type. |
| `T?` on generic | Allowed only when `T : class` or `T : struct`. Use `[MaybeNull]` for unconstrained `T?`. |
| `[NotNull]`, `[MaybeNull]`, `[AllowNull]`, `[DisallowNull]` | Pre/post-condition annotations on params/returns/fields. |
| `[NotNullWhen(true)]` / `[NotNullWhen(false)]` | Conditional post-condition: `bool TryGet<T>(string key, [NotNullWhen(true)] out T? value)`. |
| `[MemberNotNull(nameof(X))]` | "After this method returns, member X is not null" — used by `Initialize` patterns. |
| `[MemberNotNullWhen(true, nameof(X))]` | Conditional member post-condition. |
| `= null!` | Initializer to silence "field must be non-null" warning when a framework sets it later (EF Core navigation properties, dependency injection). |
| `ArgumentNullException.ThrowIfNull(x)` (NET 6+) | Guards a parameter and helps flow analysis. |

A representative request DTO with proper nullability:

```csharp
public sealed record PlaceOrderRequest(
    Guid CustomerId,                            // required
    IReadOnlyList<OrderLine> Items,             // required, may be empty list
    string? CouponCode,                         // optional
    Address ShippingAddress,                    // required
    Address? BillingAddress                     // optional: defaults to shipping
);

public sealed record Address(
    string Line1,        // required
    string? Line2,       // optional
    string City,         // required
    string PostalCode,   // required
    string Country       // required
);
```

EF Core navigation property pattern:

```csharp
public sealed class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }

    // EF Core will fill this; from C#'s POV it's never null after load
    public Customer Customer { get; private set; } = null!;
}
```

`TryGet` pattern with attribute flow:

```csharp
public bool TryGet<T>(string key, [NotNullWhen(true)] out T? value) where T : class
{
    if (_items.TryGetValue(key, out var v)) { value = (T)v; return true; }
    value = null; return false;
}

if (cache.TryGet<Order>("o:42", out var order))
{
    // compiler knows order is not null here
    Console.WriteLine(order.Total);
}
```

## Advantages

- **Static detection of null-bugs** — caught at build time instead of 2 AM.
- **Self-documenting contracts** — signatures show what's required vs optional.
- **Better refactoring** — changing a property to nullable surfaces every consumer as a warning.
- **OpenAPI accuracy** — Swagger/NSwag generate accurate `required` arrays.
- **EF Core schema fidelity** — annotations drive `NOT NULL` constraints.
- **No runtime cost** — pure compile-time feature, no IL change.

## Disadvantages

- **Migration noise** — turning it on in a large legacy codebase produces hundreds of warnings.
- **`null!` and `= null!` workarounds** — needed for EF Core navs, DI, and serializers, which clutter code.
- **External boundaries still allow `null`** — JSON, reflection, dynamic, interop can violate annotations silently.
- **Generic constraints are awkward** (`T?` is allowed only with `class` or `struct` constraint; need `[MaybeNull]` otherwise).
- **Library inconsistency** — older NuGet packages are not annotated, so calls into them produce `object?` or oblivious types and you lose flow analysis through them.

## Common Mistakes

### 1. Using `!` to silence warnings instead of fixing the model

```csharp
// WRONG — hides a real possibility of null; NRE will still happen at runtime
var email = customer.Email!.ToLowerInvariant();
```

```csharp
// RIGHT — handle the null branch explicitly
var email = customer.Email?.ToLowerInvariant() 
            ?? throw new InvalidOperationException("Email required for invoice generation");
```

### 2. Treating NRT as a runtime guarantee

```csharp
// WRONG — assumes the JSON body honors the C# annotations
public IActionResult Place([FromBody] PlaceOrderRequest req)
{
    var len = req.ShippingAddress.City.Length; // can NRE if City was sent as null
    return Ok();
}
```

```csharp
// RIGHT — validate at the boundary; let the validator/ModelState reject invalid payloads
[ApiController] // automatic model validation, returns 400 ProblemDetails on invalid input
public sealed class OrdersController : ControllerBase
{
    public IActionResult Place([FromBody] PlaceOrderRequest req, [FromServices] IValidator<PlaceOrderRequest> v)
    {
        var result = v.Validate(req);
        if (!result.IsValid) return ValidationProblem(result.ToDictionary());
        // ... now safe to dereference
    }
}
```

### 3. Marking everything nullable to silence the compiler

```csharp
// WRONG — abdicates contract design
public sealed record Order(Guid? Id, string? CustomerName, decimal? Total);
```

```csharp
// RIGHT — model the actual rules: Id and Total required, alias optional
public sealed record Order(Guid Id, string CustomerName, decimal Total, string? CustomerAlias = null);
```

### 4. Forgetting EF Core navigation properties need `null!` init

```csharp
// WRONG — compiler warning CS8618 (uninitialized non-nullable property)
public Customer Customer { get; private set; }
```

```csharp
// RIGHT — EF Core fills it on load; tell the compiler we know
public Customer Customer { get; private set; } = null!;
```

### 5. Returning `default` from a generic method that promises non-null

```csharp
// WRONG — default(T) is null for reference types
public T GetOrCreate<T>(string key) where T : class
{
    return _items.TryGetValue(key, out var v) ? (T)v : default; // returns null
}
```

```csharp
// RIGHT — annotate the return as nullable
public T? GetOrCreate<T>(string key) where T : class
    => _items.TryGetValue(key, out var v) ? (T)v : null;
```

### 6. Confusing `string?` with empty string

```csharp
// WRONG — null and "" behave differently
public void Process(string? code)
{
    if (code.Length > 0) { ... } // NRE if code is null
}
```

```csharp
// RIGHT — collapse both to "no value"
public void Process(string? code)
{
    if (string.IsNullOrEmpty(code)) return;
    // compiler now knows code is non-null
    if (code.Length > 0) { ... }
}
```

### 7. Annotating output parameters wrong on TryXxx

```csharp
// WRONG — caller has to keep null-checking even after TryGet returns true
public bool TryGet(string key, out Order? value)
{
    return _orders.TryGetValue(key, out value);
}
```

```csharp
// RIGHT — flow analysis knows value is non-null when TryGet returned true
public bool TryGet(string key, [NotNullWhen(true)] out Order? value)
{
    return _orders.TryGetValue(key, out value);
}
```

## Best Practices

- **Enable `<Nullable>enable</Nullable>`** at the project level for new code.
- **Validate every external input** (JSON body, query string, DB row, message payload) — NRT is compile-time only.
- **Reserve `!`** for unavoidable cases (EF Core navs, framework-initialized properties) and add a comment.
- **Use attributes** (`[NotNullWhen]`, `[MemberNotNull]`) to express precise flow rules — they remove the need for `!`.
- **Use `ArgumentNullException.ThrowIfNull(x)`** at the top of public methods — it helps flow analysis and produces a uniform error.
- **Return `T?` not `T` with sentinel null** when "not found" is a normal outcome.
- **Migrate gradually** — enable per-file (`#nullable enable`) for a directory at a time, fix warnings, then promote project-wide.
- **Use `string.IsNullOrEmpty` / `string.IsNullOrWhiteSpace`** — they have `[NotNullWhen]` and the compiler narrows.
- **Avoid `T?` in unconstrained generics** unless you understand `[MaybeNull]` / `[AllowNull]`.
- **Don't ship a NuGet library without annotations** — consumers lose flow analysis through your API.

## Related Concepts

- [csharp/records-and-immutability.md](csharp/records-and-immutability.md) — required vs optional record positional parameters.
- [csharp/generics.md](csharp/generics.md) — `where T : notnull`, `[MaybeNull]` for generic returns.
- [aspnet-core/input-validation.md](aspnet-core/input-validation.md) — `[ApiController]`, FluentValidation, runtime enforcement.
- [data-access/entity-framework-core.md](data-access/entity-framework-core.md) — navigation properties and `NOT NULL` columns.
- [csharp/exception-handling.md](csharp/exception-handling.md) — guarding against null at boundaries.
- [aspnet-core/swagger-openapi.md](aspnet-core/swagger-openapi.md) — nullable annotations driving `required` in schema.

## Real-World Usage

### ASP.NET Core APIs

Request DTOs declare required vs optional fields via nullability — `[ApiController]` auto-rejects payloads missing required fields with 400 `ValidationProblemDetails`. NSwag/Swashbuckle read the annotations and emit accurate `required: [...]` arrays in the OpenAPI document, which downstream clients (TypeScript, gRPC, Java) honor.

### EF Core

NRT drives the `NOT NULL` / `NULL` column constraint. `public string Name` becomes a `NOT NULL` column; `public string? Email` becomes nullable. Navigation properties use the `= null!` pattern because EF Core fills them after object materialization.

### Azure SDKs

The current generation of Azure SDKs (`Azure.Storage.Blobs`, `Azure.Messaging.ServiceBus`, `Azure.Identity`) are fully nullable-annotated. `BlobClient.DownloadContentAsync` returns `Response<BlobDownloadResult>` — the response is non-null but the body may be empty.

### JSON serialization

`System.Text.Json` respects nullability when deserializing — but if the JSON doesn't include a required (non-nullable) property, you get `null` in a non-null field. Use validators (FluentValidation, DataAnnotations) at the API boundary to enforce.

### Testing

xUnit, FluentAssertions, and Moq are all annotated. `mock.Setup(x => x.GetAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>())).ReturnsAsync((Order?)null)` is the precise way to express "this call returns null."

### Application Insights

Custom telemetry initializers and `ITelemetryProcessor` implementations work cleanly with NRT; older third-party telemetry packages may lack annotations and require occasional `!` at boundaries.

## Code Example — Before and After

### Before — implicit nullness, runtime NREs, fragile contracts

```csharp
public class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; }       // is this required? unclear
    public string Email { get; set; }      // optional? required?
    public Address Address { get; set; }
}

public class CustomerService
{
    private readonly DbContext _db;

    public CustomerService(DbContext db) => _db = db;

    public Customer GetCustomer(Guid id)
    {
        // returns null if not found, but signature doesn't say so
        return _db.Set<Customer>().SingleOrDefault(c => c.Id == id);
    }

    public string GetEmailDomain(Guid id)
    {
        var c = GetCustomer(id);
        return c.Email.Split('@')[1]; // NRE if customer or email is null
    }
}
```

### After — explicit annotations, validated boundaries, clear contracts

```csharp
#nullable enable

public sealed class Customer
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }      // required by contract
    public string? Email { get; private set; }    // optional
    public Address ShippingAddress { get; private set; } = null!; // EF Core nav, always present after load

    // Factory enforces invariants
    public static Customer Create(string name, string? email, Address shipping)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentNullException.ThrowIfNull(shipping);

        return new Customer
        {
            Id = Guid.NewGuid(),
            Name = name,
            Email = email,
            ShippingAddress = shipping
        };
    }

    private Customer() { Name = default!; } // EF Core ctor
}

public sealed class CustomerService
{
    private readonly OrderingDbContext _db;
    private readonly ILogger<CustomerService> _logger;

    public CustomerService(OrderingDbContext db, ILogger<CustomerService> logger)
    {
        _db = db; _logger = logger;
    }

    public Task<Customer?> GetCustomerAsync(Guid id, CancellationToken ct)
        => _db.Customers
              .Include(c => c.ShippingAddress)
              .FirstOrDefaultAsync(c => c.Id == id, ct);

    public async Task<string?> GetEmailDomainAsync(Guid id, CancellationToken ct)
    {
        var customer = await GetCustomerAsync(id, ct);
        if (customer is null)
        {
            _logger.LogInformation("Customer {Id} not found", id);
            return null;
        }
        if (customer.Email is null)
            return null;

        var atIndex = customer.Email.IndexOf('@');
        return atIndex < 0 ? null : customer.Email[(atIndex + 1)..];
    }
}
```

Every nullable possibility is explicit in the signature. The compiler now refuses to let a future change drop a `null` check.

## Interview Questions and Answers

### 1. Does enabling NRT prevent all `NullReferenceException`s at runtime?

**Why this matters:** Critical to understand the boundary between compile-time and runtime guarantees.

**Answer:** No. NRT is **compile-time static analysis** — it warns about null-flow in code the compiler can see. At runtime, `null` still behaves the same. JSON deserialization, EF Core, reflection, `dynamic`, P/Invoke, and any unannotated library can drop `null` into a "non-null" field. You must still validate at boundaries (HTTP requests, message payloads, DB rows from raw SQL, deserialized cache values).

**Trade-off:** The compiler can't help across boundaries it doesn't understand. Use validators (FluentValidation, `[ApiController]`) for HTTP, configure EF Core to match, and use `ArgumentNullException.ThrowIfNull` at public entry points.

**Real project:** A team enabled NRT and removed all `!= null` checks "since the compiler now enforces it." Two days later an external partner sent a JSON payload with `customerName: null` and the API NREd. Adding FluentValidation at the controller fixed it permanently.

### 2. What does the `!` operator do?

**Why this matters:** Most commonly misunderstood NRT operator.

**Answer:** `!` (the null-forgiving operator) tells the compiler "trust me — this expression isn't null." It only **silences the warning**; it generates no runtime check, no exception, no IL change. If the value actually is null, you'll get the same NRE as before. It's the right tool when you know something the compiler doesn't (e.g., EF Core navigation properties are populated by the framework). It's the wrong tool when you're just trying to avoid thinking about the null case.

**Trade-off:** Every `!` is an unmaintained assertion. Pair it with a comment explaining why, or use attributes like `[MemberNotNull]` to express the invariant precisely.

**Real project:** A code review revealed 87 uses of `!` across our service. About 20 were legitimate (EF navs, factory-initialized fields); the other 67 were "make the warning go away" — and 4 of those were actually causing intermittent NREs in production. We replaced legitimate uses with `[MemberNotNull]` and fixed the rest.

### 3. Why do EF Core navigation properties need `= null!`?

**Why this matters:** Tests practical knowledge of working around NRT in real frameworks.

**Answer:** EF Core populates navigation properties after constructing the entity via reflection — at construction time they truly are null. The compiler sees `public Customer Customer { get; }` as "must be initialized in the constructor" and warns. The convention is `public Customer Customer { get; private set; } = null!;` — telling the compiler "I know this is initially null, but the framework will fill it." For required navigations EF Core enforces this at runtime when the relationship is configured as required.

**Trade-off:** This pattern reduces the safety NRT provides for that field. Some teams use nullable navs (`Customer?`) and check at the call site instead; more verbose but more honest.

**Real project:** We switched from `Customer?` everywhere to `= null!` and saved hundreds of unnecessary null checks where the navigation was always loaded via `Include`. We documented the convention in `.editorconfig` so reviewers know to expect it.

### 4. How do `[NotNullWhen(true)]` and `[MemberNotNull]` work?

**Why this matters:** They're the surgical tools for advanced flow analysis.

**Answer:** `[NotNullWhen(true)]` on an out parameter says "if this method returns true, the out value is not null." That lets the compiler narrow the type in the `if` branch — used by `string.IsNullOrEmpty`, `Dictionary.TryGetValue`, etc. `[MemberNotNull("FieldName")]` on a method says "after this method returns, the listed members are not null." Used for init methods: `[MemberNotNull(nameof(_client))] private void EnsureInitialized() { _client ??= new(); }`. Both let the compiler reason about your custom invariants without resorting to `!`.

**Trade-off:** Subtle. The attribute must accurately describe the behavior — incorrect annotations create silent bugs (the compiler will trust you).

**Real project:** A `Configuration` class had `_client` lazily initialized. We sprinkled `!` everywhere it was used. Replacing with one `[MemberNotNull(nameof(_client))]` on `EnsureClient()` removed all `!`s and made the invariant explicit.

### 5. How do you handle nullable in generics?

**Why this matters:** A common stumbling block.

**Answer:** `T?` is allowed only when `T` has a `class` or `struct` constraint, because the compiler needs to know whether to treat the `?` as a reference null or a `Nullable<T>` struct. For unconstrained generics, use the `[MaybeNull]` and `[AllowNull]` attributes: `[return: MaybeNull] public T Find(...)`. Or constrain with `where T : notnull` if your generic should explicitly disallow null. `default(T)` produces null for reference types and `default` for value types — annotate the return as `T?` accordingly.

**Trade-off:** Unconstrained `T` with nullable analysis is awkward; most teams adopt `where T : class` or `where T : notnull` to make the generic less surprising.

**Real project:** Our generic `IRepository<T>.FindAsync` returned `Task<T>` — we kept getting NREs when nothing was found. Adding `where T : class` and changing the return to `Task<T?>` made the not-found case explicit.

### 6. How would you migrate a 100-file legacy project to NRT?

**Why this matters:** Real-world rollout question.

**Answer:** Three-stage migration. (1) Set `<Nullable>warnings</Nullable>` at the project level — get warnings without changing semantics. (2) For each folder (or feature area), set `#nullable enable` at the top of files and fix warnings. Focus on DTOs and public service contracts first — they shape downstream code. (3) After all files have `#nullable enable`, switch the project to `<Nullable>enable</Nullable>` and remove per-file directives. Use `dotnet-format` and `editorconfig` `dotnet_diagnostic.CS86xx.severity = warning|error` to control noise. Resist the urge to mass-sprinkle `!`.

**Trade-off:** A "big bang" enable produces thousands of warnings and stalls. Incremental works.

**Real project:** A 200-project monorepo migrated over six sprints. Each team owned their domain; PR reviews required no new `!` to land. Production NREs in App Insights dropped about 70% over the next quarter.

### 7. Why is `string?` better than `string.Empty` for "no value"?

**Why this matters:** Semantic clarity vs sentinel values.

**Answer:** `string?` distinguishes "the value is absent" from "the value is the empty string." For a coupon code, `null` means "no coupon" while `""` could mean "tried to apply, got nothing." For an email, `null` means "no email on file" while `""` is an invalid email. Sentinel values force every consumer to know the convention; nullable annotations are universal and compiler-checked. The exception: when an empty list and a null list are semantically identical, prefer `IReadOnlyList<T>` initialized to `Array.Empty<T>()` over `IReadOnlyList<T>?` to avoid one more null check.

**Trade-off:** `string?` adds null-checking ceremony at the call site. For string fields with a clear "not set" meaning it's worth it.

**Real project:** A customer record had `string CouponCode = ""` instead of `string?`. The reporting query `WHERE CouponCode <> ''` looked correct but missed an edge case where the column was genuinely empty for legitimate orders. Switching to nullable plus migrating empty-strings to nulls fixed the report.

### 8. What happens if I send a JSON payload with `null` for a non-nullable property?

**Why this matters:** A common production surprise.

**Answer:** With `System.Text.Json` (the default in ASP.NET Core), `null` is happily deserialized into a non-nullable C# property — the compiler annotation has **no runtime effect** on deserialization. The next dereference NREs. To prevent this, enable model validation: `[ApiController]` triggers built-in validation, or add FluentValidation. You can also configure `System.Text.Json` with a custom converter that rejects null for non-nullable properties, or use `JsonSerializerOptions { RespectNullableAnnotations = true }` (NET 8+) to do this automatically.

**Trade-off:** Validation at the boundary adds a bit of code, but it's the only correct place to enforce input shape.

**Real project:** A partner integration started sending null `customerEmail` for guest checkouts. The C# model had `string CustomerEmail` (non-null). NREs spiked in App Insights. We added FluentValidation with `.RuleFor(x => x.CustomerEmail).NotEmpty()` and enabled `RespectNullableAnnotations`. After that, bad payloads got clean 400 responses instead of crashes.

## Summary Checklist

- [ ] I enable `<Nullable>enable</Nullable>` for all new projects.
- [ ] I model required fields as `T` and optional ones as `T?` — accurately, not for convenience.
- [ ] I treat NRT as compile-time only and validate every external boundary (JSON, DB, messages).
- [ ] I use `!` sparingly, only when I know more than the compiler, and document why.
- [ ] I use `[NotNullWhen]`, `[MemberNotNull]`, `[MaybeNull]` to express precise flow rules.
- [ ] I use `ArgumentNullException.ThrowIfNull(x)` at public entry points.
- [ ] I know the `= null!` pattern for EF Core navigation properties.
- [ ] I prefer returning `T?` over throwing or returning a sentinel for "not found."
- [ ] I migrate legacy code incrementally with `#nullable enable` per file or feature.
- [ ] I configure `[ApiController]` + a validator at the API boundary so nullable annotations match runtime behavior.
