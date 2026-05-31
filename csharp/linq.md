# LINQ

## Concept Explanation

LINQ provides readable query operations over in-memory collections and external providers such as Entity Framework Core.

For C# backend work, focus on what this concept changes in everyday code: type safety, readability, runtime behavior, testability, and how services express business rules. A good explanation should connect the language feature to concrete API or service code, not stop at syntax.

When discussing it in an interview, describe the problem it solves, where it appears in a typical ASP.NET Core application, and what can go wrong when it is overused or misunderstood. Mention how you would test code that uses it and how it affects maintainability for the next developer.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **LINQ** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Prefer readable, intention-revealing C# over clever syntax.
- Use modern C# features when they reduce bugs: nullable reference types, records for immutable data, pattern matching, and collection expressions where the team supports them.
- Keep business rules in services or domain models, not hidden inside controllers or EF queries.
- For interviews, explain how the language feature helps reliability, testability, or maintainability.

## Interview Questions and Answers

### 1. What is LINQ?

**Answer:** LINQ is a C# query syntax and method API for filtering, projecting, grouping, and ordering data. It can run against in-memory collections or providers such as EF Core.

**Example:** `orders.Where(o => o.Status == OrderStatus.Pending).Select(o => o.Id)` reads clearly and avoids manual loops for simple transformations.

### 2. What is the difference between `IEnumerable` and `IQueryable`?

**Answer:** `IEnumerable` runs LINQ in memory. `IQueryable` represents a query that a provider, such as EF Core, can translate to SQL. The difference matters because calling the wrong method too early can load too much data.

**Example:** Calling `ToList()` before `Where()` loads all rows first. Keeping the query as `IQueryable` until filters and projections are applied lets EF Core generate better SQL.

### 3. What is deferred execution?

**Answer:** Deferred execution means many LINQ queries do not run when they are defined. They run when enumerated, such as by `foreach`, `ToList()`, `FirstOrDefault()`, or `Count()`.

**Example:** `var query = orders.Where(o => o.IsActive);` does not filter yet. `query.ToList()` performs the filtering.

### 4. How do you use LINQ safely with EF Core?

**Answer:** Project only the fields needed, filter before materializing, use `AsNoTracking()` for read-only queries, and check generated SQL for important paths.

**Example:** For an order list page, project to `OrderSummaryDto` instead of loading full orders with all navigation properties.

### 5. What are common LINQ performance mistakes?

**Answer:** Common mistakes include calling `ToList()` too early, using client-side methods that cannot translate to SQL, causing N+1 queries, and loading full entities when a projection is enough.

**Example:** Calling a custom C# method inside an EF query may fail translation or move work to memory. Prefer expressions EF Core can translate.

### 6. When is a loop better than LINQ?

**Answer:** A loop is better when the logic is stateful, complex, or clearer step by step. LINQ is valuable for readable query-like transformations, not for proving that every operation can be written as a chain.

**Example:** Calculating a multi-step pricing rule with validation messages may be clearer as a normal loop than a long chain of `Aggregate`, `SelectMany`, and anonymous objects.

## Coding Example

```csharp
public async Task<IReadOnlyList<OrderSummaryDto>> GetRecentOrdersAsync(
    Guid customerId,
    CancellationToken ct)
{
    return await _dbContext.Orders
        .AsNoTracking()
        .Where(order => order.CustomerId == customerId)
        .OrderByDescending(order => order.CreatedAt)
        .Take(20)
        .Select(order => new OrderSummaryDto(order.Id, order.Status, order.Total))
        .ToListAsync(ct);
}

public sealed record OrderSummaryDto(Guid Id, string Status, decimal Total);
```

## Real-World Scenario

An order history endpoint should not load every order and filter in memory. LINQ should express filtering, sorting, paging, and projection before materialization so EF Core can translate the work into efficient SQL.

## Common Mistakes

- Calling `ToList()` before filtering and paging.
- Forgetting that EF Core LINQ must translate to SQL.
- Writing long LINQ chains that are harder to read than loops.
- Loading entities when a DTO projection would be cheaper.
- Ignoring generated SQL for critical queries.

## Summary Checklist

- [ ] I can explain `IEnumerable`, `IQueryable`, and deferred execution.
- [ ] I can avoid early materialization.
- [ ] I can use projections for API responses.
- [ ] I can explain when a loop is clearer than LINQ.
