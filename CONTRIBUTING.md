# Contributing Guide

Thank you for helping improve this .NET interview wiki. This repository is a practical study guide that aims to feel like a senior engineer mentoring a mid-level developer — not a textbook. Contributions should make explanations deeper, examples more realistic, and interview answers more actionable.

## What You Can Contribute

- Add a missing .NET, ASP.NET Core, architecture, Azure, DevOps, testing, or communication topic.
- Improve an existing explanation, code example, mistake list, best practice, or interview Q&A.
- Fix inaccurate technical details (especially around EF Core, ASP.NET Core, Azure SDKs).
- Replace shallow or toy examples with realistic backend scenarios (payments, orders, JWT auth, Azure Service Bus, EF Core, etc.).
- Improve English clarity without changing the technical meaning.
- Fix broken links, ordering problems, spelling, or formatting.

## Repository Structure

Each umbrella concept has its own folder and study-order index:

- `csharp/`
- `aspnet-core/`
- `architecture/`
- `data-access/`
- `azure/`
- `devops/`
- `testing-quality/`
- `communication/`

Every folder contains an `index.md` that lists topics in the recommended reading order. When you add a new topic file, also add it to:

- the folder-level `index.md`
- the root `README.md` topic map

## Gold-Standard Exemplar

Before writing or editing a concept page, **read [csharp/dependency-injection.md](csharp/dependency-injection.md) in full**. It is the reference for depth, structure, tone, and example quality. Every concept page in this repo should match its standard.

Target length for a concept page is **500–750 lines**. If a page is much shorter, it is almost certainly missing depth. If it is much longer, consider whether some material belongs in a related page.

## Topic Page Format (Required 14 Sections)

Every concept page must follow this section order. Do not skip sections; if a section legitimately doesn't apply to a non-code topic (such as a communication page), adapt it sensibly (e.g., replace "Code Example" with "Example Dialogues").

````markdown
# Topic Name

## What It Is

Concrete definition. Explain the mechanics in plain language. A small before/after snippet here is often useful to anchor the idea.

## Why It Exists

The historical pain or design problem the concept solves. What did engineers do before this existed and why was that painful?

## When To Use It

Two explicit lists:

- **Use for:** ...
- **Do not use for:** ...

Avoid vague "use it when appropriate" advice.

## Why It Is Important

The production properties this concept drives (testability, scalability, reliability, security, maintainability). Connect it to enterprise / cloud / microservice contexts.

## How It's Used in C# / .NET

**Required when the .NET ecosystem supports it.** Show the concrete API, syntax, NuGet packages, and framework hooks. Examples:

- `AddScoped` / `AddTransient` / `AddSingleton` for DI
- `[Authorize]`, `AddAuthentication().AddJwtBearer()` for auth
- `IOptions<T>` for configuration
- `DbContext`, `[Timestamp]`, `DbUpdateConcurrencyException` for EF Core
- `Polly` `ResiliencePipeline`, `AddResilienceHandler` for reliability
- `Azure.Messaging.ServiceBus`, `DefaultAzureCredential` for Azure
- `.editorconfig`, `Microsoft.CodeAnalysis.NetAnalyzers` for static analysis

For language-agnostic architecture concepts (SOLID, CQRS, DDD), show how the concept is typically expressed in .NET projects (folder layout, MediatR, FluentValidation, MassTransit).

A quick-reference API table at the end of this section is encouraged.

## Advantages

Honest bullet list. Each bullet should be a real production benefit, not a marketing claim.

## Disadvantages

Honest bullet list of real costs (indirection, runtime errors, learning curve, latency, complexity). Every concept has trade-offs — don't hide them.

## Common Mistakes

Numbered list. Each mistake needs:

- A short description of the bug
- A `csharp` code block showing the bug
- A one-line fix or a fixed code block

## Best Practices

Actionable bullets. Each bullet should be something a reviewer could call out in a PR.

## Related Concepts

Bullet list with markdown links to neighboring topics in this wiki, e.g.
`[architecture/cqrs.md](../architecture/cqrs.md)`

## Real-World Usage

Sub-sections for the contexts where the concept actually appears:

- ASP.NET Core Web API (Azure App Service)
- Azure Functions / Container Apps / AKS
- Testing
- Multi-tenant SaaS
- CI/CD (GitHub Actions, Azure DevOps)

Show real code that engineers actually write.

## Code Example — Before and After

A realistic backend example that shows the value of the concept. The **Before** version should look like real legacy code with clear pain. The **After** version should fix it using the concept being explained. Both must be readable end-to-end.

## Interview Questions and Answers

Six to eight scenario-based questions. Each question follows this exact shape:

### 1. The question?

**Why this matters**: What signal the interviewer is trying to read.

**Answer**: Direct, specific answer with concrete code or names.

**Trade-off**: When relevant, name the cost or alternative.

**Real project**: A sentence describing where this shows up in production.

## Summary Checklist

Eight to ten capability checkboxes the reader can self-verify against.

- [ ] I can explain ...
- [ ] I can compare ...
- [ ] I can recognize ...
````

## Example Quality Rules

Examples are the most important part of every page.

### Allowed (good) examples

- Payment processing (Stripe, Adyen, Azure Communication Services)
- Order placement, cancellation, refund workflows
- Invoice generation and delivery
- Inventory reservation and stock updates
- JWT / OAuth / Azure AD (Entra ID) authentication
- Authorization policies (role/claim/policy/resource-based)
- Notification systems (email, SMS, Teams, push)
- Multi-tenant SaaS scenarios
- Azure Service Bus, Key Vault, Application Insights, App Service, Functions, Container Apps, AKS, Cosmos DB, SQL Database, Storage
- EF Core (DbContext, migrations, concurrency)
- MediatR commands/queries, MassTransit sagas, Polly resilience
- GitHub Actions / Azure DevOps pipelines
- Docker, Kubernetes, Helm
- Real REST API endpoints with status codes and Problem Details

### Forbidden (toy) examples

Reject these unconditionally — they appear in every tutorial and teach nothing about real engineering:

- Animals (cats, dogs, birds)
- Vehicles (cars, trucks)
- Shapes (circle, square, triangle)
- `Foo`, `Bar`, `Baz`
- `HelloWorld`-style demos

If your example would not be out of place in a senior engineer's PR, it is good.

## Code Style for Examples

- Use modern C# (records, primary constructors, file-scoped namespaces, `init` properties, collection expressions where the team would).
- Async methods take `CancellationToken ct` as the last parameter.
- Inject `ILogger<T>`, `IOptions<T>`, `TimeProvider` rather than using static state.
- Use `Guid` for IDs, `decimal` for money, `DateTimeOffset` for timestamps.
- Use realistic namespaces (`Contoso.Checkout.Application`) when they help the example.
- Prefer `sealed` classes for services.
- Show DI registration in `Program.cs` snippets when relevant.
- For Azure SDK code, use `DefaultAzureCredential`, not connection strings with secrets in plain text.

## Interview Q&A Rules

- Six to eight questions per concept.
- **Scenario-based**, not definition-based. Bad: *"What is DI?"* — Good: *"A singleton service needs database data. How do you handle this?"*
- Each answer must explain **why** (not just **what**), include a concrete example, name a trade-off when relevant, and tie back to a real project context.
- Avoid generic answers that would apply to any topic.
- Cover a mix: architecture/design, troubleshooting, refactoring, performance, security, and "when not to use it".

Good shape:

```markdown
### 3. A singleton service needs to read from the database. How do you handle this?

**Why this matters**: Tests for the *captive dependency* bug — silently corrupts state under load.

**Answer**: Never inject `Scoped` `DbContext` into a `Singleton`. Inject `IServiceScopeFactory` or `IDbContextFactory<T>` and open a fresh scope per operation. Sample code...

**Trade-off**: One extra scope per call. Negligible vs the cost of a memory leak.

**Real project**: A `BackgroundService` polling a feature-flag table every 60s — must open a scope per poll.
```

Avoid:

```markdown
### 1. What is async?

**Answer:** Async improves maintainability, performance, security, and production diagnostics.

**Example:** Use it in an order API.
```

## Writing Style

- Senior-engineer-teaching-junior tone. Practical mentoring, not lecture notes.
- Direct, simple English. Short paragraphs.
- Always explain **why**, not only **what**.
- Be honest about trade-offs. No pattern, tool, or cloud service is always the right choice.
- Use correct .NET terminology and current package names (e.g., `Microsoft.AspNetCore.OpenApi`, not legacy Swashbuckle when discussing .NET 9 OpenAPI).
- Avoid buzzwords without definition.
- Avoid repeating the same sentence across multiple files.

## Technical Accuracy

Before submitting, verify:

- C# examples compile conceptually and use modern .NET style.
- ASP.NET Core examples use appropriate status codes, request/response DTOs, validation, and Problem Details.
- EF Core examples mention query shape, tracking, transactions, migrations, or concurrency when relevant.
- Azure examples use Managed Identity (`DefaultAzureCredential`), Key Vault, App Configuration, and the current SDK package names. Do not invent service features.
- DevOps examples reference real GitHub Actions / Azure DevOps features (OIDC federated credentials, environments, approvals, slot swap).
- Testing examples test behavior, not private implementation details.
- Links to other concept pages use correct relative paths.

## Adding a New Topic

1. Choose the correct umbrella folder.
2. Create a kebab-case markdown file (e.g., `rate-limiting.md`).
3. Follow the 14-section topic page format above.
4. Match the depth of [csharp/dependency-injection.md](csharp/dependency-injection.md).
5. Add the topic to the folder `index.md` in the correct reading order.
6. Add the topic to the root `README.md` topic map.
7. Check all relative links.

## Updating Existing Topics

When improving an existing topic:

- Keep the 14-section structure intact. Do not delete sections.
- Replace generic answers with specific, scenario-based ones.
- Replace toy examples (animals, cars, foo/bar) with real backend examples — these are never acceptable.
- Add the "How It's Used in C# / .NET" section if missing.
- Do not remove useful technical nuance just to make the answer shorter.
- If you change reading order, update both the folder `index.md` and root `README.md`.

## Markdown Guidelines

- `#` for the page title, `##` for the 14 required sections, `###` for sub-sections (e.g., numbered interview questions).
- Wrap code in fenced code blocks with a language tag (`csharp`, `yaml`, `bicep`, `json`, `text`).
- Use relative links to other wiki pages.
- File names are lowercase kebab-case.
- Do not add a generated table of contents to topic files.

## Pull Request Checklist

Before opening a pull request, confirm:

- [ ] The page uses all 14 required sections in order.
- [ ] Page length is roughly 500–750 lines.
- [ ] **"How It's Used in C# / .NET"** section is present and shows concrete APIs/packages (if the .NET ecosystem supports the concept).
- [ ] Every interview question is scenario-based and includes *Why this matters*, *Answer*, and *Trade-off* / *Real project* when relevant.
- [ ] **Common Mistakes** are numbered and each has a code example and a fix.
- [ ] **Code Example — Before and After** uses a realistic backend scenario (no animals, cars, or `foo`/`bar`).
- [ ] All code uses modern C# (async/await, `CancellationToken`, `ILogger<T>`, `IOptions<T>`).
- [ ] Trade-offs are named honestly.
- [ ] New files are linked from the folder `index.md` and from `README.md`.
- [ ] No broken relative links were introduced.
- [ ] The page reads like a senior engineer mentoring a mid-level developer.
