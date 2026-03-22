---
layout: post
title: "Health Checks in ASP.NET Core for Application Monitoring"
date: 2026-08-13 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - health-checks
  - monitoring
  - aspnet
---


# Health Checks in ASP.NET Core for Application Monitoring

In distributed systems and microservices architectures, knowing whether your application is healthy goes beyond "is the process running?" Your app might be up but unable to reach its database. It might respond to requests but with degraded performance because a cache is down. It might be slowly failing because a downstream service is timing out.

Health checks give you proactive visibility into the state of your application and its dependencies -- databases, message queues, caches, and external APIs. They are the foundation of automated monitoring, load balancer routing, and self-healing infrastructure.

ASP.NET Core has built-in support for health checks, and in this article we will explore how to use them effectively.

## Basic Health Check Setup

Getting started requires just two lines of code:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/health");

app.Run();
```

This registers the health check services and maps the `HealthCheckMiddleware` to respond at `/health`. Out of the box, it returns one of three `HealthStatus` values:

| Status | Meaning |
|--------|---------|
| `HealthStatus.Healthy` | Everything is functioning normally |
| `HealthStatus.Degraded` | The application works but with reduced performance or partial functionality |
| `HealthStatus.Unhealthy` | A critical component has failed |

The `Degraded` status is particularly useful -- it signals that the application is still serving requests but something needs attention before it becomes a full outage. For example, if response times spike because a cache is unavailable and the app falls back to direct database queries, returning `Degraded` alerts your monitoring system without triggering an immediate failover.

## Creating Custom Health Checks

You can create custom health checks by implementing the `IHealthCheck` interface. Here is an example that verifies SQL Server connectivity:

```csharp
public class SqlHealthCheck : IHealthCheck
{
    private readonly string _connectionString;

    public SqlHealthCheck(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("Database");
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var sqlConnection = new SqlConnection(_connectionString);

            await sqlConnection.OpenAsync(cancellationToken);

            using var command = sqlConnection.CreateCommand();
            command.CommandText = "SELECT 1";

            await command.ExecuteScalarAsync(cancellationToken);

            return HealthCheckResult.Healthy();
        }
        catch(Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                context.Registration.FailureStatus,
                exception: ex);
        }
    }
}
```

A few design considerations:

- **Keep checks fast.** Use lightweight queries like `SELECT 1` rather than complex operations. Health check endpoints are called frequently by load balancers and monitoring systems.
- **Handle exceptions gracefully.** Return `Unhealthy` with context rather than letting exceptions bubble up.
- **Use cancellation tokens.** Respect the token so checks do not hang indefinitely.

Register the custom health check with a name and a failure status:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<SqlHealthCheck>("custom-sql", HealthStatus.Unhealthy);
```

## Using Existing Health Check Libraries

Before writing a custom check for every dependency, check whether a library already exists. The [`AspNetCore.Diagnostics.HealthChecks`](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks) repository provides packages for dozens of common services:

| Service | Package |
|---------|---------|
| SQL Server | `AspNetCore.HealthChecks.SqlServer` |
| PostgreSQL | `AspNetCore.HealthChecks.Npgsql` |
| Redis | `AspNetCore.HealthChecks.Redis` |
| RabbitMQ | `AspNetCore.HealthChecks.RabbitMQ` |
| AWS S3 | `AspNetCore.HealthChecks.Aws.S3` |
| SignalR | `AspNetCore.HealthChecks.SignalR` |

Adding checks for PostgreSQL and RabbitMQ is a one-liner each:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<SqlHealthCheck>("custom-sql", HealthStatus.Unhealthy)
    .AddNpgSql(pgConnectionString)
    .AddRabbitMQ(rabbitConnectionString);
```

This fluent API makes it easy to compose a comprehensive health picture of your application.

## Formatting the Health Check Response

By default, the health check endpoint returns a plain string like "Healthy" or "Unhealthy". This is fine for a single check, but when you have multiple dependencies, you need to know *which one* failed.

The `AspNetCore.HealthChecks.UI.Client` package provides a structured JSON response writer:

```powershell
Install-Package AspNetCore.HealthChecks.UI.Client
```

Configure the response writer:

```csharp
app.MapHealthChecks(
    "/health",
    new HealthCheckOptions
    {
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
```

Now the response includes individual status for each check:

```json
{
  "status": "Unhealthy",
  "totalDuration": "00:00:00.3285211",
  "entries": {
    "npgsql": {
      "data": {},
      "duration": "00:00:00.1183517",
      "status": "Healthy",
      "tags": []
    },
    "rabbitmq": {
      "data": {},
      "duration": "00:00:00.1189561",
      "status": "Healthy",
      "tags": []
    },
    "custom-sql": {
      "data": {},
      "description": "Unable to connect to the database.",
      "duration": "00:00:00.2431813",
      "exception": "Unable to connect to the database.",
      "status": "Unhealthy",
      "tags": []
    }
  }
}
```

At a glance, you can see that PostgreSQL and RabbitMQ are fine but the SQL Server connection is down. This is invaluable for debugging production issues.

## Practical Use Cases

Health checks are not just for dashboards. Here are some real-world applications:

- **Load balancer routing.** Cloud platforms (AWS ALB, Azure App Gateway, Kubernetes) poll health endpoints to decide whether to route traffic to an instance. An unhealthy instance gets removed from the pool automatically.
- **Auto-scaling and self-healing.** When an instance consistently reports unhealthy, orchestrators like Kubernetes can restart it or spin up a replacement.
- **Deployment validation.** After a deployment, check the health endpoint to verify the new version can reach all its dependencies before routing production traffic to it.
- **Alerting.** Integrate health check results with monitoring platforms (Datadog, Grafana, PagerDuty) for proactive incident response.

## Key Takeaways

- ASP.NET Core has built-in health check support that requires minimal configuration
- Use `Degraded` for partial failures and `Unhealthy` for critical ones
- Leverage existing health check libraries before writing custom implementations
- Always format health check responses as structured JSON in production
- Integrate health checks with your infrastructure for automated failover and alerting

## References

- [ASP.NET Core Health Checks Documentation](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [AspNetCore.Diagnostics.HealthChecks Repository](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)
- [HealthChecks UI Client Package](https://www.nuget.org/packages/AspNetCore.HealthChecks.UI.Client)
- [Kubernetes Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
