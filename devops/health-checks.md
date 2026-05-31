# Health Checks

## Concept Explanation

Health checks expose whether an application is alive, ready to receive traffic, and connected to critical dependencies.

For DevOps work, focus on how this concept changes delivery safety, environment consistency, observability, deployment speed, rollback options, and runtime reliability. A strong explanation should connect the practice to the path from code commit to production behavior.

When discussing it in an interview, explain the pipeline or runtime step, what can fail, how the team detects failure, and how the system recovers. Include the trade-off between automation, operational complexity, cost, and team maturity.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Health Checks** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Build once, promote the same artifact, and configure per environment.
- Fail fast in CI with tests, linting, security checks, and repeatable builds.
- Use health checks and observability as deployment safety signals.
- Have a rollback or forward-fix strategy before releasing risky changes.

## Interview Questions and Answers

### 1. What is Health Checks, and where have you used it in a .NET backend project?

**Answer:** Health checks expose whether an application is alive, ready to receive traffic, and connected to critical dependencies. The value is safer, repeatable delivery from source code to production with fewer manual steps and clearer recovery when something fails.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 2. What problem does Health Checks solve for an API or business application?

**Answer:** Health checks expose whether an application is alive, ready to receive traffic, and connected to critical dependencies. The value is safer, repeatable delivery from source code to production with fewer manual steps and clearer recovery when something fails.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 3. How would you implement or apply Health Checks in an ASP.NET Core service?

**Answer:** Health checks should distinguish liveness from readiness. Readiness should fail when critical dependencies are unavailable so traffic does not reach an instance that cannot serve requests.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 4. What are common mistakes developers make with Health Checks?

**Answer:** Health checks should distinguish liveness from readiness. Readiness should fail when critical dependencies are unavailable so traffic does not reach an instance that cannot serve requests.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 5. What trade-offs should a senior developer consider before using Health Checks?

**Answer:** Health checks should distinguish liveness from readiness. Readiness should fail when critical dependencies are unavailable so traffic does not reach an instance that cannot serve requests.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 6. An order API is slow, hard to test, and risky to deploy. How could Health Checks help, and what would you check first?

**Answer:** Health checks should distinguish liveness from readiness. Readiness should fail when critical dependencies are unavailable so traffic does not reach an instance that cannot serve requests.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

## Coding Example

```csharp
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("OrdersDb")!)
    .AddAzureServiceBusQueue(
        builder.Configuration["ServiceBus:ConnectionString"]!,
        queueName: "order-events");

app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

## Real-World Scenario

Use the path from pull request to production as the reference point. Explain what is automated, what gates protect release quality, how environments differ, how health is verified, and how the team rolls forward or rolls back after a failed deployment.

## Common Mistakes

- Explaining tools without explaining the release risk they reduce.
- Building artifacts differently per environment instead of promoting the same artifact.
- Ignoring secrets, configuration, health checks, and rollback behavior.
- Treating deployment success as the same thing as production health.
- Adding operational complexity the team cannot support.

## Summary Checklist

- [ ] I can explain the delivery flow from commit to production.
- [ ] I can describe build, test, package, deploy, verify, and rollback steps.
- [ ] I can discuss environment configuration and runtime health.
- [ ] I can explain the trade-off between automation and operational complexity.
