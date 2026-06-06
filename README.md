# .NET Developer Career Path

## Purpose

This wiki helps a .NET developers focusing on C#, ASP.NET Core, REST APIs, CQRS, Domain-Driven Design, clean architecture, Azure, DevOps, Docker, Kubernetes, testing, and code quality.

The content is written for practical interview preparation. Each page explains one concept, breaks down the core ideas with examples, explains why it matters, adds practical notes, provides interview questions with direct answers and examples, includes a realistic backend example when useful, and finishes with a real-world scenario, common mistakes, and a checklist.

## Who This Wiki Is For

- .NET developers with around 3+ years of experience.
- Backend developers preparing for technical interviews in English.
- Candidates who need to explain architecture, API, Azure, DevOps, and testing decisions clearly.
- Developers who want realistic business examples rather than toy examples.

## Recommended Study Path

1. Start with C# fundamentals, dependency injection, async and await, exception handling, and nullable reference types.
2. Move to ASP.NET Core request pipeline, RESTful API design, authentication, validation, error handling, logging, and OpenAPI.
3. Study architecture topics: SOLID, clean architecture, CQRS, DDD, aggregates, domain events, and microservices.
4. Review EF Core, transactions, query performance, migrations, and concurrency.
5. Cover Azure services: App Service, Functions, SQL, Storage, Service Bus, Key Vault, Application Insights, and deployment.
6. Practice DevOps and deployment: CI/CD, Docker, Kubernetes, health checks, blue-green deployment, and rollback.
7. Finish with testing, code review, clean code, refactoring, and communication practice.

## Folder Structure

```text
dotnet-interview-wiki/
  README.md
  CONTRIBUTING.md
  csharp/
    index.md
  aspnet-core/
    index.md
  architecture/
    index.md
  data-access/
    index.md
  azure/
    index.md
  devops/
    index.md
  testing-quality/
    index.md
  communication/
    index.md
```

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before adding or changing topics so new content follows the same structure, tone, and question-answer format.

## High-Priority Topics for This Job Description

- RESTful API design
- Dependency Injection
- Async and await
- SOLID principles
- Clean Architecture
- CQRS
- Domain-Driven Design
- Entity Framework Core and query performance
- Azure App Service and Azure Functions
- Azure Service Bus
- Docker and Kubernetes basics
- CI/CD pipelines
- Unit testing and code review practices
- Explaining technical decisions and trade-offs in English

## Suggested Interview Preparation Plan

| Day | Focus | Outcome |
|---|---|---|
| 1 | C# and ASP.NET Core fundamentals | Explain backend language and web API foundations clearly. |
| 2 | REST APIs, auth, validation, errors, logging | Design and defend production-ready API behavior. |
| 3 | SOLID, clean architecture, CQRS, DDD | Explain architectural decisions and trade-offs. |
| 4 | EF Core, transactions, performance, concurrency | Discuss persistence risks and practical fixes. |
| 5 | Azure services and deployment | Connect application design to cloud operations. |
| 6 | DevOps, Docker, Kubernetes, rollback | Explain delivery, runtime, and recovery strategy. |
| 7 | Testing, code review, communication | Practice strong English answers with examples. |

## Topic Map

- [C# and .NET Study Order](csharp/index.md)
- [ASP.NET Core and REST APIs Study Order](aspnet-core/index.md)
- [Architecture and Design Study Order](architecture/index.md)
- [Data Access Study Order](data-access/index.md)
- [Azure Study Order](azure/index.md)
- [DevOps and Deployment Study Order](devops/index.md)
- [Testing and Quality Study Order](testing-quality/index.md)
- [Communication and Interview Readiness Study Order](communication/index.md)

## C# and .NET

- [C# Fundamentals for Backend Development](csharp/fundamentals.md)
- [Object-Oriented Programming in C#](csharp/object-oriented-programming.md)
- [Interfaces and Abstractions](csharp/interfaces-and-abstractions.md)
- [Dependency Injection](csharp/dependency-injection.md)
- [LINQ](csharp/linq.md)
- [Async and Await](csharp/async-await.md)
- [Exception Handling](csharp/exception-handling.md)
- [Generics](csharp/generics.md)
- [Records and Immutability](csharp/records-and-immutability.md)
- [Nullable Reference Types](csharp/nullable-reference-types.md)

## ASP.NET Core and REST APIs

- [ASP.NET Core Request Pipeline](aspnet-core/request-pipeline.md)
- [Controllers and Minimal APIs](aspnet-core/controllers-and-minimal-apis.md)
- [RESTful API Design](aspnet-core/restful-api-design.md)
- [HTTP Methods and Status Codes](aspnet-core/http-methods-and-status-codes.md)
- [API Versioning](aspnet-core/api-versioning.md)
- [Authentication and Authorization](aspnet-core/authentication-and-authorization.md)
- [JWT Authentication](aspnet-core/jwt-authentication.md)
- [Input Validation](aspnet-core/input-validation.md)
- [Error Handling and Problem Details](aspnet-core/error-handling-and-problem-details.md)
- [Logging and Monitoring](aspnet-core/logging-and-monitoring.md)
- [Swagger and OpenAPI](aspnet-core/swagger-openapi.md)

## Architecture and Design

- [SOLID Principles](architecture/solid-principles.md)
- [Clean Architecture](architecture/clean-architecture.md)
- [CQRS](architecture/cqrs.md)
- [Domain-Driven Design](architecture/domain-driven-design.md)
- [Entities and Value Objects](architecture/entities-and-value-objects.md)
- [Aggregates](architecture/aggregates.md)
- [Repositories](architecture/repositories.md)
- [Unit of Work](architecture/unit-of-work.md)
- [Domain Events](architecture/domain-events.md)
- [Application Services](architecture/application-services.md)
- [Microservices Architecture](architecture/microservices-architecture.md)
- [Monolith vs Microservices](architecture/monolith-vs-microservices.md)
- [Event-Driven Architecture](architecture/event-driven-architecture.md)
- [Saga Pattern](architecture/saga-pattern.md)
- [Outbox Pattern](architecture/outbox-pattern.md)
- [Reliability Design](architecture/reliability-design.md)
- [Scalability Design](architecture/scalability-design.md)

## Data Access

- [Entity Framework Core](data-access/entity-framework-core.md)
- [DbContext Lifetime](data-access/dbcontext-lifetime.md)
- [Migrations](data-access/migrations.md)
- [Query Performance](data-access/query-performance.md)
- [Transactions](data-access/transactions.md)
- [Optimistic Concurrency](data-access/optimistic-concurrency.md)
- [Repository Pattern with EF Core](data-access/repository-pattern-with-ef-core.md)

## Azure

- [Azure App Service](azure/app-service.md)
- [Azure Functions](azure/azure-functions.md)
- [Azure SQL](azure/azure-sql.md)
- [Azure Storage](azure/azure-storage.md)
- [Azure Service Bus](azure/azure-service-bus.md)
- [Azure Key Vault](azure/azure-key-vault.md)
- [Application Insights](azure/application-insights.md)
- [Deployment to Azure](azure/deployment-to-azure.md)
- [Azure Configuration and Secrets Management](azure/configuration-and-secrets-management.md)

## DevOps and Deployment

- [CI/CD Pipelines](devops/ci-cd-pipelines.md)
- [Docker](devops/docker.md)
- [Kubernetes Basics](devops/kubernetes-basics.md)
- [Environment Configuration](devops/environment-configuration.md)
- [Health Checks](devops/health-checks.md)
- [Blue-Green Deployment](devops/blue-green-deployment.md)
- [Rollback Strategy](devops/rollback-strategy.md)

## Testing and Quality

- [Unit Testing](testing-quality/unit-testing.md)
- [Integration Testing](testing-quality/integration-testing.md)
- [Mocking](testing-quality/mocking.md)
- [Testable Code Design](testing-quality/testable-code-design.md)
- [Code Review Practices](testing-quality/code-review-practices.md)
- [Clean Code](testing-quality/clean-code.md)
- [Refactoring](testing-quality/refactoring.md)
- [Static Analysis](testing-quality/static-analysis.md)

## Communication and Interview Readiness

- [Explaining Technical Decisions in English](communication/explaining-technical-decisions-in-english.md)
- [Handling System Design Questions](communication/handling-system-design-questions.md)
- [Explaining Trade-Offs](communication/explaining-trade-offs.md)
- [Behavioral Questions for .NET Developers](communication/behavioral-questions-for-dotnet-developers.md)
- [Code Review Discussion Questions](communication/code-review-discussion-questions.md)
