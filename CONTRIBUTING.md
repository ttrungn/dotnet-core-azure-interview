# Contributing Guide

Thank you for helping improve this .NET interview wiki. This repository is a practical study guide, so contributions should make interview preparation clearer, more accurate, and easier to follow.

## What You Can Contribute

- Add a missing .NET, ASP.NET Core, architecture, Azure, DevOps, testing, or communication topic.
- Improve an existing explanation, question, answer, example, or scenario.
- Fix inaccurate technical details.
- Improve English clarity without changing the technical meaning.
- Add realistic backend examples from APIs, databases, messaging, cloud deployment, testing, or production support.
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

Every folder should contain an `index.md` file that lists topics in the recommended reading order. When you add a new topic file, also add it to:

- the folder-level `index.md`
- the root `README.md` topic map

## Topic Page Format

Use this structure for concept pages:

````markdown
# Topic Name

## Concept Explanation

Explain the concept directly and practically.

## Core Ideas and Examples

Teach the concept before asking interview questions. Break down the important parts, terms, rules, or sub-concepts with concrete examples.

## Why This Matters in a .NET Developer Interview

Explain why interviewers ask about it and what signals they are looking for.

## Practical Notes for .NET Projects

- Add focused, practical notes.
- Keep them relevant to real backend work.

## Interview Questions and Answers

### 1. Question?

**Answer:** Give one direct, useful answer.

**Example:** Give one concrete backend example.

## Coding Example

```csharp
// Add a short realistic example when useful.
```

## Real-World Scenario

Describe how the concept appears in a production-style workflow.

## Common Mistakes

- List common mistakes contributors should help readers avoid.

## Summary Checklist

- [ ] Add practical checklist items.
````

## Question and Answer Rules

- Do not separate all questions from all answers.
- Do not jump into interview practice before explaining the concept.
- The `Core Ideas and Examples` section must teach the parts of the concept clearly.
- Each question must be followed immediately by its answer and example.
- Avoid generic answers that could apply to any topic.
- Make answers specific to the concept.
- Prefer realistic examples: orders, payments, invoices, inventory, authentication, logging, deployment, queues, databases, and tests.
- Explain trade-offs where relevant.

Good:

```markdown
### 1. What problem does async solve in web APIs?

**Answer:** Async helps with I/O-bound work such as database calls, HTTP APIs, storage, and queues. It improves scalability by avoiding blocked request threads. It does not automatically make CPU-heavy code faster.

**Example:** `await _dbContext.Orders.ToListAsync(ct)` is useful. Wrapping CPU-heavy price calculation in `Task.Run()` inside an API is usually not the right fix.
```

Avoid:

```markdown
### 1. What is async?

**Answer:** Async improves maintainability, performance, security, and production diagnostics.

**Example:** Use it in an order API.
```

## Writing Style

- Write in clear, simple English.
- Be direct and practical.
- Prefer production-style backend examples over toy examples.
- Avoid buzzwords unless you explain them.
- Avoid repeating the same sentence across many files.
- Keep paragraphs short.
- Use correct .NET terminology.
- Mention trade-offs honestly. No pattern, tool, or cloud service is always the right choice.

## Technical Accuracy

Before submitting, check that:

- C# examples compile conceptually and use modern .NET style.
- The concept is explained before the interview questions begin.
- ASP.NET Core examples use appropriate status codes, request/response DTOs, validation, and error handling.
- EF Core examples mention query shape, tracking, transactions, migrations, or concurrency when relevant.
- Azure examples mention configuration, managed identity, Key Vault, monitoring, scaling, cost, or failure handling when relevant.
- DevOps examples mention build, test, artifact, deployment, health checks, rollback, and environment configuration when relevant.
- Testing examples focus on behavior, not private implementation details.

## Adding a New Topic

1. Choose the correct umbrella folder.
2. Create a kebab-case markdown file, for example `rate-limiting.md`.
3. Follow the topic page format above.
4. Add the topic to the folder `index.md` in the correct reading order.
5. Add the topic to the root `README.md` topic map.
6. Check all relative links.

## Updating Existing Topics

When improving an existing topic:

- Keep the current heading structure unless there is a strong reason to change it.
- Replace generic answers with specific ones.
- Keep examples short enough to study quickly.
- Do not remove useful technical nuance just to make the answer shorter.
- If you change reading order, update both the folder `index.md` and root `README.md`.

## Markdown Guidelines

- Use `#` for the page title and `##` for major sections.
- Use numbered `###` headings for interview questions.
- Wrap code in fenced code blocks with a language, such as `csharp`, `yaml`, `text`, or `json`.
- Use relative links.
- Keep filenames lowercase and kebab-case.
- Do not add generated tables of contents inside each topic file.

## Pull Request Checklist

Before opening a pull request, confirm:

- [ ] The content follows the topic page format.
- [ ] The `Core Ideas and Examples` section explains the concept enough for a beginner to start practicing.
- [ ] Every interview question has one answer and one example directly below it.
- [ ] The answer is specific to the topic.
- [ ] Code examples are short, realistic, and readable.
- [ ] New files are linked from the folder `index.md`.
- [ ] New files are linked from `README.md`.
- [ ] No broken relative links were introduced.
- [ ] The writing is clear for a .NET developer preparing for interviews in English.
