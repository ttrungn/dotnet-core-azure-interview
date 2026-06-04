# Static Analysis

## What It Is

Static analysis is the practice of inspecting source code **without running it** to find bugs, security vulnerabilities, style violations, and maintainability problems. In .NET it's powered primarily by **Roslyn analyzers** — compiler plugins that walk the syntax tree and semantic model during build and surface diagnostics with the same severity machinery as compiler errors.

A tiny before/after of what it catches:

```csharp
// Before — analyzer warning CA2007: ConfigureAwait missing in library code
public async Task<Invoice> GetAsync(Guid id)
{
    return await _db.Invoices.FindAsync(id);
}

// After — analyzer satisfied
public async Task<Invoice> GetAsync(Guid id, CancellationToken ct)
{
    return await _db.Invoices.FindAsync(new object[] { id }, ct).ConfigureAwait(false);
}
```

The bug — a deadlock risk in a library consumed by a UI thread — was caught at build time, not at 2 AM in production.

## Why It Exists

Code review by humans is necessary but not sufficient. Reviewers get tired, miss subtle issues, and can't enforce hundreds of conventions consistently across a 200,000-line codebase. Static analysis exists because **the compiler already understands your code** — the same machinery that finds `error CS0103: name 'x' does not exist` can find `warning CA1062: validate arguments of public methods`, `warning S2589: change condition (it always evaluates to true)`, or `warning SCS0005: weak random number generator`.

Before Roslyn (released with .NET Compiler Platform in 2015), .NET analysis lived in separate tools: FxCop, StyleCop, ReSharper. The Roslyn era unified them into MSBuild-integrated analyzers that run on every build, light up in the IDE in real time, and fail CI when severity is `error`. This shifted "we'll catch it in review" to "the compiler catches it before the PR opens."

## When To Use It

**Use static analysis when:**

- You're starting a new project — enable strict analyzers from day one when the rule count is zero.
- You want consistent code style across a team without bikeshedding in reviews.
- You need security scanning for OWASP-class issues (injection, weak crypto, secrets in source).
- You're working in a regulated environment (PCI, HIPAA, SOC 2) that requires SAST evidence.
- A class of bug keeps slipping through review — pick the analyzer rule that catches it, raise its severity.
- You're adopting nullable reference types and need NRT warnings to be errors.
- You're operating a public API or NuGet package — analyzers enforce API stability rules.

**Do not use static analysis when:**

- It produces too many false positives for the value — suppress with documented justification.
- The rule conflicts with framework guidance (e.g., some CA rules don't apply to ASP.NET Core minimal APIs).
- The team hasn't agreed on the rule — analyzer rules are political; enable them through discussion, not edict.
- A one-off script or throwaway prototype doesn't need 50 rule violations to ship.
- The rule is purely stylistic and `dotnet format` would handle it better than a compiler warning.

## Why It Is Important

In production .NET systems, static analysis is the **lowest-cost defect prevention** available — it runs every build, requires no test data, and catches issues that unit tests rarely target (null derefs on uncovered paths, deadlock risks in unexercised code, unsafe deserialization).

It drives:

- **Lower defect escape rate** — analyzer-caught bugs never reach `main`, let alone production.
- **Faster reviews** — humans review intent and architecture, not whitespace or `var` vs explicit types.
- **Security posture** — Microsoft.CodeAnalysis.NetAnalyzers ships rules for many CWE categories; CodeQL adds dataflow analysis for taint-based vulnerabilities.
- **Onboarding** — new engineers learn the codebase's conventions from squiggles, not from a 40-page style guide nobody reads.
- **Audit readiness** — SOC 2 and PCI reviewers ask "show me your SAST." A CodeQL workflow in GitHub Actions with passing runs is the answer.
- **Backwards-compatibility enforcement** — Public API analyzers fail the build when a public signature changes without a corresponding entry in `PublicAPI.Shipped.txt`.

## How It's Used in C# / .NET

**Core analyzer packages** (NuGet):

| Package | What it covers |
|---|---|
| `Microsoft.CodeAnalysis.NetAnalyzers` | Built-in to the .NET SDK. CA rules: design, performance, security, reliability, usage. |
| `StyleCop.Analyzers` | SAxxxx rules for naming, ordering, documentation, layout. |
| `SonarAnalyzer.CSharp` | Sxxxx rules from SonarSource — bug detection, code smells, security hotspots. |
| `Roslynator.Analyzers` | Hundreds of refactor and simplification suggestions. |
| `Meziantou.Analyzer` | Stricter rules for async, exceptions, and common pitfalls. |
| `Microsoft.VisualStudio.Threading.Analyzers` | VSTHRD rules for async and threading correctness. |
| `SecurityCodeScan.VS2019` | OWASP-style security rules (SCSxxxx). |

**.editorconfig — the single source of truth for severity:**

```ini
# .editorconfig at repo root
root = true

[*.cs]
# Treat nullable warnings as errors
dotnet_diagnostic.CS8600.severity = error
dotnet_diagnostic.CS8602.severity = error
dotnet_diagnostic.CS8603.severity = error

# Security
dotnet_diagnostic.CA2100.severity = error   # SQL injection
dotnet_diagnostic.CA3075.severity = error   # XML XXE
dotnet_diagnostic.CA5350.severity = error   # weak crypto
dotnet_diagnostic.CA5394.severity = error   # insecure RNG

# Reliability
dotnet_diagnostic.CA2007.severity = warning # ConfigureAwait in libraries
dotnet_diagnostic.CA1062.severity = warning # validate args on public methods

# Style — kept as suggestion so they don't fail CI on hotfixes
dotnet_diagnostic.IDE0011.severity = suggestion  # add braces

# Suppress noisy rule
dotnet_diagnostic.CA1303.severity = none    # don't pass literal strings to localized methods

# Naming conventions
dotnet_naming_rule.private_fields_with_underscore.symbols  = private_fields
dotnet_naming_rule.private_fields_with_underscore.style    = prefix_underscore
dotnet_naming_rule.private_fields_with_underscore.severity = error
```

Severity levels are `error` (fails build), `warning` (shows but builds), `suggestion` (IDE hint only), `silent` (IntelliSense only), `none` (disabled).

**Build-time enforcement in `Directory.Build.props`:**

```xml
<Project>
  <PropertyGroup>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
    <Nullable>enable</Nullable>
    <WarningsAsErrors>nullable</WarningsAsErrors>
    <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="9.0.0" PrivateAssets="all" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="10.0.0.*" PrivateAssets="all" />
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-*" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

**`dotnet format`** — applies whitespace, `using` ordering, and analyzer fixes automatically:

```bash
dotnet format --verify-no-changes --severity warn   # CI gate
dotnet format                                       # local fix
```

**Suppressions when justified:**

```csharp
[SuppressMessage("Security", "CA5394:Do not use insecure randomness",
    Justification = "Used for jitter in retry backoff, not security-sensitive")]
private static int NextJitter() => Random.Shared.Next(0, 100);
```

Or in `.editorconfig` for a whole folder:

```ini
[tests/**/*.cs]
dotnet_diagnostic.CA1707.severity = none  # tests use underscores in names by convention
```

**CodeQL in GitHub Actions:**

```yaml
# .github/workflows/codeql.yml
name: CodeQL
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [{ cron: '0 6 * * 1' }]   # weekly full scan

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: csharp
          queries: security-and-quality
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: 9.0.x }
      - run: dotnet build --configuration Release
      - uses: github/codeql-action/analyze@v3
```

Results surface in the **Security** tab of the repo and gate PRs via branch protection rules.

**SonarCloud / SonarQube** adds duplication detection, cognitive complexity, test coverage overlay, and historical trends. Integrates with `dotnet-sonarscanner` and Azure DevOps or GitHub Actions.

## Advantages

- **Catches bugs at build time** — earliest possible feedback, cheapest fix.
- **Zero runtime cost** — analysis happens during compilation, not at request time.
- **Consistent enforcement** — same rules for every developer, no human variance.
- **Security coverage** for classes of bug that tests rarely target (SQL injection, weak crypto, XXE).
- **IDE integration** — fixes are available with `Ctrl+.` / `Alt+Enter`, often automatable.
- **Documents conventions** — `.editorconfig` is executable style guide.
- **Frees code review** for design discussion instead of nitpicking.
- **Audit-ready evidence** for compliance frameworks.

## Disadvantages

- **False positives** — some rules misfire on perfectly correct code, requiring suppressions.
- **Noise on legacy codebases** — turning on `AnalysisMode=All` on a 5-year-old repo yields thousands of warnings.
- **Build time** — heavy analyzer packages add seconds to incremental builds; minutes to full builds.
- **Rule conflicts** — StyleCop and SonarAnalyzer sometimes contradict each other.
- **Suppression fatigue** — teams that suppress without justification end up with disabled-by-default analysis.
- **Doesn't catch logic bugs** — analyzers find patterns, not "the tax rate should be 9% not 8%."
- **Version churn** — analyzer updates can introduce new errors that block urgent hotfixes.

## Common Mistakes

### 1. Disabling all warnings instead of fixing them

```xml
<!-- BAD — drops all signal -->
<NoWarn>CA1062;CA2007;CA1303;CS8602</NoWarn>
```

**Fix** — disable rules individually in `.editorconfig` with severity `none` and a comment explaining why, or suppress with `[SuppressMessage]` + `Justification`.

### 2. Suppressing without a justification

```csharp
// BAD
[SuppressMessage("Security", "CA2100")]
public List<Order> FindByCustomer(string customerId) =>
    _db.Database.SqlQueryRaw<Order>($"SELECT * FROM Orders WHERE CustomerId = '{customerId}'").ToList();
```

**Fix** — actually fix the SQL injection, don't suppress it.

```csharp
public List<Order> FindByCustomer(string customerId) =>
    _db.Orders.Where(o => o.CustomerId == customerId).ToList();
```

If suppression is truly justified, write the reason:

```csharp
[SuppressMessage("Performance", "CA1822", Justification = "Required as instance method by IPaymentGateway interface")]
```

### 3. Not running analyzers in CI

If analyzers only run on the developer's machine, severity drift is inevitable. A warning that fails the build for one dev passes silently for another because they have an older SDK.

**Fix** — `TreatWarningsAsErrors=true` in `Directory.Build.props` so CI fails identically to local builds, plus `dotnet format --verify-no-changes` as a separate gate.

### 4. Enabling `AnalysisMode=All` on day one of a legacy migration

```xml
<AnalysisMode>All</AnalysisMode>
```

You'll get 3,000 warnings, the team will rebel, and someone will set `NoWarn` to everything to ship the hotfix.

**Fix** — start with `AnalysisMode=Recommended` or even `Minimum`, ratchet upward one rule (or rule category) at a time. Make each upgrade its own PR.

### 5. Treating CodeQL findings as informational

CodeQL surfaces real exploits (SQL injection, SSRF, XSS, deserialization). Ignoring them because "it's only a warning" leaves vulnerabilities in production.

**Fix** — branch protection that blocks merge on new CodeQL alerts in the diff. Triage existing alerts on a security debt board.

### 6. Mixing analyzer fixes with feature commits

A PR that adds a feature and fixes 47 analyzer warnings has the same review problem as a refactor mixed with a behavior change.

**Fix** — one PR per analyzer rule for batch cleanups. New code in feature PRs must pass analyzers, but historical cleanup belongs in its own commits.

### 7. Forgetting `PrivateAssets="all"` on analyzer packages

```xml
<!-- BAD — analyzer becomes a runtime dependency of consumers -->
<PackageReference Include="StyleCop.Analyzers" Version="1.2.0-*" />
```

**Fix** — analyzers are build-time only. Always:

```xml
<PackageReference Include="StyleCop.Analyzers" Version="1.2.0-*" PrivateAssets="all" />
```

## Best Practices

- **Enable `Nullable=enable` and `WarningsAsErrors=nullable`** — nullability bugs are real and analyzers catch most of them.
- **One `.editorconfig` at the repo root**, with overrides per folder if needed. Version it; treat it as code.
- **Run `dotnet format --verify-no-changes` in CI** as a separate step from build.
- **Use `PrivateAssets="all"`** on every analyzer package.
- **Ratchet, don't blanket** — when adopting analyzers in a legacy codebase, enable rules one at a time as the warning count for that rule hits zero.
- **Pin analyzer versions** in `Directory.Packages.props` so all projects use the same rules.
- **Run security analyzers in CI** (CodeQL or SonarQube) — they catch what local analyzers miss.
- **Document every suppression** with a `Justification` string. Periodically review and remove stale ones.
- **Keep IDE and CI in sync** — same SDK version, same analyzer versions, same `.editorconfig`.
- **Pair static analysis with tests** — analyzers find pattern bugs, tests find logic bugs. Both are necessary.

## Related Concepts

- [testing-quality/clean-code.md](clean-code.md) — many analyzer rules encode clean-code principles.
- [testing-quality/refactoring.md](refactoring.md) — analyzers surface refactor candidates with one-click fixes.
- [testing-quality/code-review-practices.md](code-review-practices.md) — analyzers handle the mechanical review, humans focus on intent.
- [csharp/nullable-reference-types.md](../csharp/nullable-reference-types.md) — NRT warnings are the most valuable analyzer family in modern C#.
- [csharp/async-await.md](../csharp/async-await.md) — VSTHRD and CA2007 catch async bugs.
- [aspnet-core/input-validation.md](../aspnet-core/input-validation.md) — CA1062 enforces argument validation on public methods.
- [devops/ci-cd-pipelines.md](../devops/ci-cd-pipelines.md) — where analyzer gates live.
- [aspnet-core/authentication-and-authorization.md](../aspnet-core/authentication-and-authorization.md) — security analyzers catch many auth bugs.

## Real-World Usage

**ASP.NET Core APIs** — Microsoft.CodeAnalysis.NetAnalyzers rules CA1062 (validate args), CA2007 (ConfigureAwait), CA2227 (collection properties read-only), and CA5394 (insecure RNG) are the highest-value rules for web APIs. Combined with FluentValidation for runtime checks, they form a defense-in-depth.

**Azure Functions** — analyzers catch missing `CancellationToken` plumbing in function handlers (a common cause of hung executions on host shutdown) and missing `await` on Azure SDK calls.

**Library NuGet packages** — `Microsoft.CodeAnalysis.PublicApiAnalyzers` enforces that every public API change is recorded in `PublicAPI.Shipped.txt` / `PublicAPI.Unshipped.txt`. The build fails if a consumer-visible signature changes silently.

**Multi-project solutions** — `Directory.Build.props` and `Directory.Packages.props` at the solution root apply analyzers and pinned versions to every project automatically. Adding a new project gets the same analysis without per-project setup.

**GitHub Actions code-scanning** — CodeQL workflow runs on every PR. Findings appear inline in the diff as `❌` annotations and block merge under branch protection. Weekly scheduled scans catch issues in code that hasn't been touched recently.

**Azure DevOps Pipelines** — SonarQube task uploads analysis after each build. Quality Gate failures (new bugs, new vulnerabilities, coverage drop) block PR completion via the SonarQube Quality Gate policy.

**Multi-tenant SaaS** — security analyzers catch leaks of `TenantId` across boundaries (e.g., a query missing `.Where(x => x.TenantId == _ctx.TenantId)` won't be caught by analyzers directly, but custom Roslyn analyzers can be written to enforce it — several teams have shipped these to production).

**Pre-commit hook with Husky.Net** runs `dotnet format` and analyzers locally so developers see issues before pushing:

```bash
dotnet husky add pre-commit -c "dotnet format --verify-no-changes && dotnet build --no-restore /p:TreatWarningsAsErrors=true"
```

## Code Example — Before and After

**Before — an unconfigured project:**

```xml
<!-- src/Payments.Api/Payments.Api.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>
</Project>
```

```csharp
// PaymentsController.cs — analyzers would flag many issues
public class PaymentsController : ControllerBase
{
    private static HttpClient http = new HttpClient();   // CA1822, CA2000

    [HttpPost("capture")]
    public async Task<IActionResult> Capture(CaptureRequest req)
    {
        var sql = $"SELECT * FROM Invoices WHERE Id = '{req.InvoiceId}'";   // CA2100 SQL injection
        var rnd = new Random();                                              // CA5394 weak RNG for security
        var token = rnd.Next().ToString();
        var invoice = _db.Database.SqlQueryRaw<Invoice>(sql).First();        // CS8602 possible null deref
        await _gateway.CaptureAsync(invoice.Id, req.Amount);                 // CA2007 missing ConfigureAwait in library
        return Ok();
    }
}
```

**After — analyzers configured, code cleaned up:**

```xml
<!-- Directory.Build.props at repo root -->
<Project>
  <PropertyGroup>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="9.0.0" PrivateAssets="all" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="10.0.0.*" PrivateAssets="all" />
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-*" PrivateAssets="all" />
    <PackageReference Include="Meziantou.Analyzer" Version="2.0.*" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

```ini
# .editorconfig
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = crlf
insert_final_newline = true

# Security as errors
dotnet_diagnostic.CA2100.severity = error   # SQL injection
dotnet_diagnostic.CA5350.severity = error   # weak crypto
dotnet_diagnostic.CA5394.severity = error   # insecure RNG
dotnet_diagnostic.CA3075.severity = error   # XXE

# Reliability
dotnet_diagnostic.CA2007.severity = warning
dotnet_diagnostic.CA2016.severity = error   # forward CancellationToken

# Style: only fail build on a curated subset
dotnet_diagnostic.IDE0005.severity = warning  # unused usings
dotnet_diagnostic.IDE0055.severity = error    # formatting

# Suppress noise
dotnet_diagnostic.CA1303.severity = none      # don't enforce localized strings
dotnet_diagnostic.SA1633.severity = none      # file header
```

```csharp
// PaymentsController.cs — all analyzer warnings resolved
[ApiController]
[Route("api/payments")]
public sealed class PaymentsController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IPaymentGateway _gateway;

    public PaymentsController(AppDbContext db, IPaymentGateway gateway)
    {
        _db = db ?? throw new ArgumentNullException(nameof(db));
        _gateway = gateway ?? throw new ArgumentNullException(nameof(gateway));
    }

    [HttpPost("capture")]
    public async Task<IActionResult> Capture(
        [FromBody] CaptureRequest req,
        CancellationToken ct)
    {
        ArgumentNullException.ThrowIfNull(req);

        // Parameterized query via EF Core — CA2100 satisfied
        var invoice = await _db.Invoices
            .FirstOrDefaultAsync(i => i.Id == req.InvoiceId, ct)
            .ConfigureAwait(false);

        if (invoice is null) return NotFound();

        var token = RandomNumberGenerator.GetHexString(32);   // CA5394 satisfied
        var result = await _gateway.CaptureAsync(invoice.Id, req.Amount, ct).ConfigureAwait(false);
        return result.IsSuccess ? Ok() : StatusCode(502);
    }
}
```

```yaml
# .github/workflows/build.yml — CI gate
- name: Build with analyzers as errors
  run: dotnet build --configuration Release /p:TreatWarningsAsErrors=true

- name: Format check
  run: dotnet format --verify-no-changes --severity warn

- name: CodeQL Analyze
  uses: github/codeql-action/analyze@v3
```

The build now fails locally and in CI on any analyzer error. Reviewers can focus on whether the payment logic is correct, not whether `ConfigureAwait(false)` was added.

## Interview Questions and Answers

### 1. Why prefer Roslyn analyzers over runtime validation libraries?

**Why this matters:** The interviewer wants to know whether you understand the shift-left principle: catch bugs as early as possible.

**Answer:** Analyzers run at compile time, so the cost of a defect is the developer pressing Save — not the cost of a failed deployment, a customer-reported bug, or a production incident. Runtime validation is necessary for input the analyzer can't see (user data, external responses), but for anything the compiler can prove — null derefs on uncovered paths, missing `await`, parameter validation, SQL injection in `SqlQueryRaw` — the analyzer is strictly cheaper. Tests also catch bugs, but only on the paths the tests exercise; analyzers see every line.

**Trade-off:** Analyzers can't catch logic bugs (wrong tax rate, off-by-one in a loop) or runtime issues (network timeouts, race conditions). They're a complement to tests, not a replacement.

**Real project:** Enabling `Nullable=enable` with `WarningsAsErrors=nullable` on a 2-year-old codebase surfaced 280 potential null dereferences in a week. About 30 were genuine bugs — including one in a payment refund path that would have crashed on a specific historical data shape.

### 2. How do you adopt strict analyzers on a legacy codebase without blocking delivery?

**Why this matters:** Turning on every rule produces thousands of warnings and a revolt. The pragmatic answer matters.

**Answer:** Ratchet. Start with `AnalysisLevel=latest-recommended` and `AnalysisMode=Default`, then enable individual rules as errors one at a time. For each rule: fix the existing violations in a dedicated PR, then change severity from `warning` to `error` in `.editorconfig` in the next PR so it stays at zero. Pair this with a "no new warnings" policy on PRs — even if a rule is `warning`, new code must not introduce new instances. Within a quarter you can move from default to strict without ever blocking a hotfix.

**Trade-off:** Slower than a big-bang enable. But a big-bang enable usually ends with `NoWarn` set to everything to ship a P1 fix, and the analysis is effectively disabled forever.

**Real project:** On a 5-year-old monolith, we enabled CA2007 (ConfigureAwait) over three sprints — fix violations sprint 1, raise to warning sprint 2, raise to error sprint 3. Same pattern for CA1062 (validate args) and CA2016 (forward CancellationToken). The codebase passed `TreatWarningsAsErrors` six months in.

### 3. When is it appropriate to suppress a Roslyn warning?

**Why this matters:** Suppressions are a power tool. Misuse turns analyzers into noise; correct use keeps the signal high.

**Answer:** Suppress when the rule produces a false positive in a specific context, when the rule conflicts with an intentional design choice, or when the cost of the fix exceeds the value (e.g., a 10-year-old generated file). Every suppression should have a `Justification` string in the attribute or a comment in `.editorconfig` explaining why. Suppressions should be reviewed periodically — if a rule is suppressed in 50 files, the right fix is usually to disable the rule globally and document the team's stance, not to keep suppressing it everywhere.

**Trade-off:** Easy suppression leads to analyzer rot. I treat a suppression as a tiny piece of design documentation that future readers will need.

**Real project:** Suppressing CA1062 inside a private extension method called only from internal code (parameters already validated upstream) — justified, documented, reviewed. Suppressing CA2100 because "fixing it would require rewriting the query layer" — not justified; we wrote a tracking ticket and fixed it properly within the quarter.

### 4. How does CodeQL differ from local Roslyn analyzers?

**Why this matters:** CodeQL is a different class of tool — dataflow and taint analysis vs syntax-pattern analysis. Understanding the difference shows depth.

**Answer:** Roslyn analyzers look at the syntax and semantic model of a single file or compilation unit. They're fast and run on every build. CodeQL builds a queryable graph of the entire codebase and follows data flow across method and class boundaries — it can determine that user input from an HTTP parameter flows through three method calls into a `Process.Start`, catching command injection even when no single file looks suspicious. Roslyn is your everyday linter; CodeQL is your security audit. Both belong in CI: Roslyn on every build, CodeQL on PR and a weekly scheduled scan.

**Trade-off:** CodeQL takes minutes to run and requires a separate workflow. False positives need expert triage. The value is catching whole classes of vulnerabilities (taint-based injection, SSRF, deserialization) that pattern-based tools miss.

**Real project:** A CodeQL alert flagged an SSRF risk where an HTTP request URL was built from a webhook payload that came from a third-party service. The same pattern existed in three places nobody had reviewed; a single CodeQL run found all three.

### 5. How do you handle analyzer disagreements between StyleCop and SonarAnalyzer?

**Why this matters:** Real teams hit this constantly. The answer reveals whether you have a process or just hope.

**Answer:** Pick one as the source of truth per topic. For style and ordering (using directives, braces, naming), StyleCop wins. For bug detection and code smells, SonarAnalyzer wins. When they conflict, I disable the loser in `.editorconfig` with a comment, and put both decisions in a short ADR (architecture decision record) so the next person doesn't reopen the debate. The goal is consistency, not winning a religious argument about whether `var` or explicit types is better.

**Trade-off:** Some teams remove one of the analyzers entirely to avoid conflict. That works but loses real signal — SonarAnalyzer has bug rules StyleCop doesn't, and vice versa.

**Real project:** SA1101 (prefix `this.`) conflicted with our team preference for unprefixed members. We disabled SA1101 in `.editorconfig` and kept the rest of StyleCop. ADR-007 documented the decision.

### 6. How would you enforce a custom team rule that no analyzer ships out of the box?

**Why this matters:** Tests the boundary of off-the-shelf tooling and whether you know analyzers are extensible.

**Answer:** Write a custom Roslyn analyzer. The `Microsoft.CodeAnalysis.CSharp` SDK lets you register `SyntaxNodeAction` or `SymbolAction` callbacks and emit diagnostics. Common custom rules I've shipped: "any LINQ query against `Orders` must include `.Where(x => x.TenantId == ...)` in multi-tenant code", "all public DTOs must be `record`s", "no direct use of `DateTime.Now` — use `IClock`". The analyzer ships as a NuGet package referenced by every project. For smaller teams, BannedApiAnalyzers from `Microsoft.CodeAnalysis.BannedApiAnalyzers` can cover many cases without writing a full analyzer — just list banned APIs in `BannedSymbols.txt`.

**Trade-off:** Writing a Roslyn analyzer takes a few hours to a few days depending on complexity. For one-off rules, code review enforcement is cheaper. For rules that the team will care about for years, the investment pays off.

**Real project:** Shipped a custom analyzer that flagged direct use of `HttpClient` constructors (must use `IHttpClientFactory`) and `DateTime.UtcNow` (must use `IDateTimeProvider`). Caught 12 violations in the first month across multiple PRs that would otherwise have passed review.

### 7. What's your strategy for `TreatWarningsAsErrors` in production projects?

**Why this matters:** Done wrong, this blocks hotfixes. Done right, it eliminates a whole class of "this would have been caught" bugs.

**Answer:** Enable `TreatWarningsAsErrors=true` in `Directory.Build.props` for application projects from day one. For library projects, also enable `WarningsAsErrors=nullable` at minimum. The trick is configuring `.editorconfig` so that only rules the team has agreed are errors fire at severity `error` — everything else stays at `warning` or `suggestion`. This way "warnings as errors" doesn't mean "every analyzer rule blocks the build", it means "the curated set of rules we treat as errors block the build." For emergency hotfixes, the option `/p:TreatWarningsAsErrors=false` exists, but it should be a documented escape hatch, not a habit.

**Trade-off:** New analyzer versions can introduce new errors. I pin analyzer versions in `Directory.Packages.props` and upgrade them deliberately, not via floating versions.

**Real project:** On a high-throughput payment service, `TreatWarningsAsErrors=true` with a curated `.editorconfig` blocked merges on real bugs about twice a month. Most were nullable warnings that would have crashed on edge data; one was a CA2016 (missing CancellationToken forwarding) that would have caused hung requests on host shutdown.

## Summary Checklist

- [ ] I can name the main analyzer packages (`NetAnalyzers`, `StyleCop`, `SonarAnalyzer`, security-focused ones).
- [ ] I configure severity per rule in `.editorconfig`, not via `NoWarn`.
- [ ] I enable `TreatWarningsAsErrors`, `EnforceCodeStyleInBuild`, and `Nullable` in `Directory.Build.props`.
- [ ] I add analyzer packages with `PrivateAssets="all"`.
- [ ] I run `dotnet format --verify-no-changes` in CI as a separate gate.
- [ ] I wire CodeQL (GitHub) or SonarQube into PR checks for security analysis.
- [ ] I document every suppression with a `Justification` string.
- [ ] I adopt strict analyzers in legacy code via ratchet, not big-bang.
- [ ] I can write a custom Roslyn analyzer for team-specific rules when needed.
- [ ] I keep IDE, CI, and team SDK versions aligned so warnings are identical everywhere.
