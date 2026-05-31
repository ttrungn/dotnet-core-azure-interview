# Blue-Green Deployment

## Concept Explanation

Blue-green deployment keeps two production-like environments so traffic can switch quickly between current and new versions.

For DevOps work, focus on how this concept changes delivery safety, environment consistency, observability, deployment speed, rollback options, and runtime reliability. A strong explanation should connect the practice to the path from code commit to production behavior.

When discussing it in an interview, explain the pipeline or runtime step, what can fail, how the team detects failure, and how the system recovers. Include the trade-off between automation, operational complexity, cost, and team maturity.

## Core Ideas and Examples

Blue-green deployment uses two production-like environments: one active and one idle/new.

- **Blue:** Current production environment.
- **Green:** New version prepared and tested before traffic moves.
- **Traffic switch:** Move users from blue to green quickly.
- **Rollback:** Switch traffic back if the new version fails.
- **Data caution:** Database changes must be backward compatible across both versions.

Example: deploy a new order API version to the green slot, run smoke tests, swap traffic, and swap back if error rate increases.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Blue-Green Deployment** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What role does Blue-Green Deployment play in delivering a .NET API?

**Answer:** Plan rollback before release. Blue-green or slot swaps make rollback faster, but database changes still need backward-compatible migration planning.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 2. What should be built once and configured per environment?

**Answer:** Build once and promote the same artifact. Change environment-specific behavior through configuration and secrets, not different builds.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 3. How do you make builds repeatable and releases auditable?

**Answer:** A good pipeline restores dependencies, builds once, runs tests and checks, packages an artifact, deploys through environments, and records what version is running.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 4. How do health checks, logs, and deployment gates reduce release risk?

**Answer:** Health checks should distinguish liveness from readiness. Readiness should fail when critical dependencies are unavailable so traffic does not reach an instance that cannot serve requests.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 5. What failure modes should a deployment strategy handle?

**Answer:** Blue-green deployment keeps two production-like environments so traffic can switch quickly between current and new versions. In practice, explain how it changes build reliability, release safety, runtime verification, and recovery when a deployment fails.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 6. A new container image passes tests but fails after deployment. How would you diagnose and recover?

**Answer:** Containers package the app and runtime dependencies consistently. The image should be small, reproducible, configured through environment variables, and scanned before deployment.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 7. How would you explain Blue-Green Deployment to a product owner without using unnecessary jargon?

**Answer:** Plan rollback before release. Blue-green or slot swaps make rollback faster, but database changes still need backward-compatible migration planning.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

### 8. How would you migrate an existing production feature toward better use of Blue-Green Deployment without stopping delivery?

**Answer:** Plan rollback before release. Blue-green or slot swaps make rollback faster, but database changes still need backward-compatible migration planning.

**Example:** For example, a pipeline builds one Docker image, runs tests, deploys it to staging, runs smoke checks, promotes it to production, and watches health metrics.

## Coding Example

```csharp
public sealed class PaymentService
{
    private readonly IPaymentGateway _gateway;
    private readonly ILogger<PaymentService> _logger;

    public PaymentService(IPaymentGateway gateway, ILogger<PaymentService> logger)
    {
        _gateway = gateway;
        _logger = logger;
    }

    public async Task<PaymentResult> CaptureAsync(Guid invoiceId, decimal amount, CancellationToken ct)
    {
        _logger.LogInformation("Capturing payment for invoice {InvoiceId}", invoiceId);
        return await _gateway.CaptureAsync(invoiceId, amount, ct);
    }
}
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
