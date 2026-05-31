# Azure Key Vault

## Concept Explanation

Azure Key Vault stores secrets, certificates, and keys outside application code and configuration files.

For Azure work, focus on the production responsibility this service or practice takes on: hosting, scaling, configuration, secrets, messaging, storage, telemetry, deployment, security, or recovery. A good explanation should connect the Azure feature to an application requirement and an operational concern.

When discussing it in an interview, describe the workload, configuration, identity model, failure handling, monitoring, and cost or scaling trade-off. The goal is to show that you can run the .NET system reliably in Azure, not just name the service.

## Core Ideas and Examples

Azure Key Vault stores secrets, keys, and certificates outside application code.

- **Secrets:** Connection strings, API keys, and passwords.
- **Keys:** Cryptographic keys used by applications or services.
- **Certificates:** TLS or client certificates.
- **Managed identity:** Lets Azure-hosted apps access Key Vault without storing credentials.
- **Access policy/RBAC:** Controls which app or user can read each secret.

Example: an App Service uses managed identity to read a payment provider API key from Key Vault at startup.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Azure Key Vault** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

Interviewers will listen for:
- Correct use of .NET terminology.
- Practical examples from backend systems.
- Awareness of maintainability, performance, security, and operations.
- Ability to explain trade-offs without overusing buzzwords.

## Practical Notes for .NET Projects

- Separate code, configuration, and secrets.
- Use managed identity where possible instead of storing credentials.
- Add Application Insights telemetry and actionable alerts before production incidents.
- Know the scaling, cost, networking, and security implications of each Azure service.

## Interview Questions and Answers

### 1. What production problem does Azure Key Vault solve in a .NET application?

**Answer:** Azure Key Vault stores secrets, certificates, and keys outside application code and configuration files. The production problem is usually hosting, scaling, messaging, storage, secrets, telemetry, deployment, or reliability. A strong answer ties the service to one workload and one operational risk.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

### 2. Which application settings, secrets, and connection strings should be externalized?

**Answer:** Externalize environment-specific settings, connection strings, feature flags, and secrets. Use Key Vault and managed identity where possible so credentials are not stored in code or committed configuration files.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

### 3. How would you configure this service for dev, test, staging, and production?

**Answer:** Use separate resources or clearly separated settings per environment. Keep configuration outside the artifact, use managed identity, and make deployment repeatable through pipeline or infrastructure automation.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

### 4. How would you monitor failures, latency, and dependency health?

**Answer:** Use Application Insights, structured logs, metrics, dependency tracking, and alerts. Monitor both technical signals such as errors and business signals such as failed order messages.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

### 5. What are the cost, scaling, security, and operational trade-offs?

**Answer:** Evaluate expected traffic, scaling model, network isolation, identity, data sensitivity, and operational ownership. The cheapest service is not always best if it creates reliability or support risk.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

### 6. An order API deployment works locally but fails in Azure after release. What would you inspect first?

**Answer:** Check configuration, identity permissions, connection strings, Key Vault access, networking, logs, deployment slot settings, and dependency health. Local success often hides missing cloud permissions or environment settings.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

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

Use a deployed .NET API or background worker as the reference point. Explain why the Azure service is used, how it is configured per environment, how identity and secrets are handled, what happens during failure, and how telemetry proves it is healthy.

## Common Mistakes

- Naming an Azure service without explaining the production problem it solves.
- Storing secrets in application code or local configuration files.
- Ignoring managed identity, networking, scaling, cost, and failure modes.
- Deploying without Application Insights telemetry and actionable alerts.
- Assuming local behavior is enough to prove cloud readiness.

## Summary Checklist

- [ ] I can explain the service role in a production .NET system.
- [ ] I can discuss configuration, identity, secrets, and networking.
- [ ] I can describe monitoring, scaling, cost, and failure handling.
- [ ] I can connect the Azure choice to a business requirement.
