# Azure App Service

## Concept Explanation

Azure App Service hosts web apps and APIs with managed scaling, deployment slots, TLS, configuration, and platform patching.

For Azure work, focus on the production responsibility this service or practice takes on: hosting, scaling, configuration, secrets, messaging, storage, telemetry, deployment, security, or recovery. A good explanation should connect the Azure feature to an application requirement and an operational concern.

When discussing it in an interview, describe the workload, configuration, identity model, failure handling, monitoring, and cost or scaling trade-off. The goal is to show that you can run the .NET system reliably in Azure, not just name the service.

## Why This Matters in a .NET Developer Interview

This role expects a developer who can build maintainable APIs and services using C#, ASP.NET Core, Azure, CI/CD, and modern engineering practices. **Azure App Service** is likely to appear because it shows whether you can move beyond syntax and explain design decisions clearly in English.

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

### 1. What production problem does Azure App Service solve in a .NET application?

**Answer:** Azure App Service hosts web apps and APIs with managed scaling, deployment slots, TLS, configuration, and platform patching. The production problem is usually hosting, scaling, messaging, storage, secrets, telemetry, deployment, or reliability. A strong answer ties the service to one workload and one operational risk.

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

### 7. How would you explain Azure App Service to a product owner without using unnecessary jargon?

**Answer:** I would explain Azure App Service as the Azure capability that helps the team run the system reliably, recover from failures, and support growth without manual server management.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

### 8. How would you migrate an existing production feature toward better use of Azure App Service without stopping delivery?

**Answer:** Move one workload first, keep the existing behavior stable, configure observability before cutover, use staged release or parallel run when possible, and monitor cost and failures after deployment.

**Example:** For example, a checkout API can use managed identity to read secrets from Key Vault, publish order events to Service Bus, and send telemetry to Application Insights.

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
