---
layout: post
title: ".NET Observability — Part 2: Better Request Tracing with User Context"
date: 2026-05-07 00:00:00 +0000
tags:
  - dotnet
  - observability
  - tracing
  - logging
  - middleware
  - aspnetcore
series: ".NET Observability"
series_part: 2
---


## Series Overview

This is a three-part series on building observable .NET applications, from structured logging through request tracing to full distributed tracing with OpenTelemetry.

1. [Part 1: Structured Logging with Serilog](./2026-04-dotnet-observability-part1-serilog.md)
2. **Part 2: Better Request Tracing with User Context** (this article)
3. [Part 3: Distributed Tracing with OpenTelemetry](./2026-05-dotnet-observability-part3-opentelemetry.md)


## The Missing Piece in Your Logs

In [Part 1](./2026-04-dotnet-observability-part1-serilog.md), we set up structured logging with Serilog. Our logs are now machine-readable, searchable, and enriched with metadata. But there is a gap: when a user reports a problem, how do you find *their* log entries among thousands?

The answer is to attach user identity to every log entry and trace automatically. Instead of manually passing user IDs into every log call, we build a middleware that enriches the entire request pipeline with user context. This is a small investment that pays off enormously during incident response.

## Why User Context Changes Everything

Without user context, investigating a reported issue means correlating timestamps, guessing which requests belong to the affected user, and hoping you find the right log entries. With user context attached to every log entry, you can:

- **Filter logs by user ID** to see exactly what happened during their session
- **Track user journeys** through your application to understand behavior patterns
- **Troubleshoot specific users** without wading through irrelevant log noise
- **Monitor performance per user segment** to detect issues affecting subsets of your user base

## Two Mechanisms, One Middleware

ASP.NET Core gives us two complementary mechanisms for attaching context to requests, and our middleware will use both:

### Logging Scopes

A **logging scope** in ASP.NET Core lets you attach key-value pairs to all log messages created within that scope. If you add a user ID to a logging scope, every log message produced while that scope is active — across controllers, services, repositories, anywhere — will carry that user ID. This works with Serilog, the built-in logger, and most third-party logging frameworks.

### The Activity Class

The `System.Diagnostics.Activity` class is .NET's built-in abstraction for distributed tracing. An `Activity` represents a unit of work and can carry tags (key-value pairs) that flow across service boundaries. When you set a tag on `Activity.Current`, that information becomes part of the distributed trace and shows up in your tracing backend (Jaeger, Zipkin, Application Insights, etc.).

By using both mechanisms together, we ensure that user context appears in both our logs and our traces.

## Building the Middleware

Here is the complete middleware implementation:

```csharp
using System.Diagnostics;
using System.Security.Claims;

namespace MyApp.Middleware;

public sealed class UserContextEnrichmentMiddleware(
    RequestDelegate next,
    ILogger<UserContextEnrichmentMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        string? userId = context.User?.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId is not null)
        {
            Activity.Current?.SetTag("user.id", userId);

            var data = new Dictionary<string, object>
            {
                ["UserId"] = userId
            };

            using (logger.BeginScope(data))
            {
                await next(context);
            }
        }
        else
        {
            await next(context);
        }
    }
}
```

Let's walk through what this does:

1. **Extract the user ID** from the authenticated user's claims using `FindFirstValue(ClaimTypes.NameIdentifier)`. This is the standard claim type for user identifiers in ASP.NET Core authentication.
2. **Tag the current Activity** with `user.id`. This makes the user ID visible in distributed traces.
3. **Create a logging scope** with the user ID. Every log entry produced by downstream middleware, controllers, and services will now include `UserId` as a property.
4. **Handle anonymous requests gracefully** by simply passing through to the next middleware without enrichment.

## Registering the Middleware

The middleware must be placed **after** authentication and authorization in the pipeline, so that the user identity is already established:

```csharp
app.UseAuthentication();
app.UseAuthorization();

// Add the user context enrichment middleware after authentication
app.UseMiddleware<UserContextEnrichmentMiddleware>();

app.MapControllers();

app.Run();
```

Order matters here. If you place the enrichment middleware before authentication, `context.User` will not have the identity claims populated, and the middleware will treat every request as anonymous.

## Connecting with OpenTelemetry

If you are using OpenTelemetry to export your logs (which we will cover in [Part 3](./2026-05-dotnet-observability-part3-opentelemetry.md)), you need to tell the OpenTelemetry logging provider to include scope data in the exported log records:

```csharp
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeScopes = true;
    options.IncludeFormattedMessage = true;
});
```

Without `IncludeScopes = true`, the `UserId` property we carefully added via `BeginScope` would not appear in your exported telemetry data.

## Handling PII Responsibly

Adding user identifiers to logs and traces raises important privacy considerations:

- **Use opaque identifiers**. User IDs should be GUIDs or internal database IDs — not email addresses, names, or other personally identifiable information.
- **Avoid logging sensitive data**. Do not add claims like email, phone number, or address to logging scopes.
- **Audit your sinks**. Make sure user IDs are not being exported to systems where they should not be stored.
- **Plan for data subject requests**. If you need to comply with GDPR or similar regulations, implement log retention policies and the ability to purge user-specific data from your logs.

## Expanding Beyond User IDs

The same middleware pattern works for other contextual information. Here are two practical extensions:

### Feature Flags

If you use feature flags for gradual rollouts or A/B testing, adding flag state to your traces lets you correlate behavior and performance with specific feature configurations:

```csharp
// Inside the middleware
if (featureFlagService.IsEnabled("NewFeature", userId))
{
    Activity.Current?.SetTag("features.newfeature", "enabled");
    // Add to logging scope as well
}
```

### Tenant Context for Multi-Tenant Applications

In multi-tenant applications, knowing which tenant a request belongs to is just as important as knowing the user:

```csharp
string? tenantId = context.User?.FindFirstValue("TenantId");
if (tenantId is not null)
{
    Activity.Current?.SetTag("tenant.id", tenantId);
    // Add to logging scope
}
```

This lets you isolate logs and traces per tenant, which is invaluable for investigating tenant-specific issues.

## The Impact

With user context enrichment in place, your troubleshooting workflow transforms:

1. **User reports a problem** — you immediately filter logs by their user ID
2. **Usage pattern analysis** — you see which features are used and by whom
3. **Performance segmentation** — you identify slow requests for specific user groups
4. **A/B test analysis** — you correlate metrics with feature flag states

This is a small amount of middleware code that delivers an outsized improvement in your ability to understand and debug your application.

## What's Next?

In [Part 3: Distributed Tracing with OpenTelemetry](./2026-05-dotnet-observability-part3-opentelemetry.md), we take the final step: instrumenting your entire distributed system with OpenTelemetry. You will learn how to collect traces across multiple services, visualize request flows in Jaeger, and see exactly how requests travel through your architecture — including database queries, HTTP calls, message bus interactions, and cache lookups.

## References

- [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/) — Official middleware documentation
- [System.Diagnostics.Activity](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.activity) — .NET Activity class reference
- [ILogger.BeginScope](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger.beginscope) — Logging scope documentation
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) — Standard attribute naming for traces
- [GDPR and Logging](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation) — Data protection considerations
