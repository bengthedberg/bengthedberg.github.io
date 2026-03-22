---
layout: post
title: ".NET Observability — Part 3: Distributed Tracing with OpenTelemetry"
date: 2026-05-14 00:00:00 +0000
tags:
  - dotnet
  - observability
  - opentelemetry
  - tracing
  - jaeger
  - distributed-systems
series: ".NET Observability"
series_part: 3
---


## Series Overview

This is a three-part series on building observable .NET applications, from structured logging through request tracing to full distributed tracing with OpenTelemetry.

1. [Part 1: Structured Logging with Serilog](./2026-04-dotnet-observability-part1-serilog.md)
2. [Part 2: Better Request Tracing with User Context](./2026-05-dotnet-observability-part2-request-tracing.md)
3. **Part 3: Distributed Tracing with OpenTelemetry** (this article)


## The Challenge of Distributed Systems

In [Part 1](./2026-04-dotnet-observability-part1-serilog.md), we set up structured logging. In [Part 2](./2026-05-dotnet-observability-part2-request-tracing.md), we enriched our logs with user context. These are powerful tools, but they operate within the boundary of a single application.

Modern systems are rarely a single application. A user's request might hit an API gateway, which calls a backend service, which queries a database, publishes a message to a bus, and triggers processing in two other services. When something goes wrong, or when performance degrades, you need to see the entire picture — not just one service's perspective.

This is the problem **distributed tracing** solves, and [OpenTelemetry](https://opentelemetry.io/) is the industry-standard framework for implementing it.

## What is OpenTelemetry?

OpenTelemetry (OTel) is a vendor-neutral, open-source observability framework. It provides standardized APIs, SDKs, and tooling for generating, collecting, and exporting telemetry data. The framework covers three pillars of observability:

- **Traces**: The flow of requests through distributed systems, showing timing and relationships between services. A trace is composed of spans, where each span represents a unit of work (an HTTP call, a database query, a message publish).
- **Metrics**: Numerical measurements over time — request counts, error rates, response times, memory usage.
- **Logs**: Textual event records with structured contextual information (which we covered in Part 1).

The key value proposition is that OpenTelemetry is **vendor-neutral**. You instrument your code once, and you can export the data to any compatible backend — Jaeger, Zipkin, Grafana Tempo, Datadog, Azure Monitor, AWS X-Ray, and many others. No vendor lock-in.

## Adding OpenTelemetry to a .NET Application

OpenTelemetry provides dedicated .NET libraries that automatically capture traces from common frameworks and libraries. Here are the packages you will need:

```powershell
# Core hosting integration
Install-Package OpenTelemetry.Extensions.Hosting

# OTLP exporter (sends data to Jaeger, Grafana, etc.)
Install-Package OpenTelemetry.Exporter.OpenTelemetryProtocol

# Instrumentation packages for automatic trace collection
Install-Package OpenTelemetry.Instrumentation.Http
Install-Package OpenTelemetry.Instrumentation.AspNetCore
Install-Package OpenTelemetry.Instrumentation.EntityFrameworkCore
Install-Package OpenTelemetry.Instrumentation.StackExchangeRedis
Install-Package Npgsql.OpenTelemetry
```

Each instrumentation package hooks into a specific library or framework and automatically produces trace spans without requiring you to modify your application code.

## Configuring Tracing

With the packages installed, configure OpenTelemetry in your service registration:

```csharp
services
    .AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService(serviceName))
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddRedisInstrumentation()
            .AddNpgsql();

        tracing.AddOtlpExporter();
    });
```

Here is what each instrumentation gives you:

- **`AddAspNetCoreInstrumentation`** — Creates a span for every incoming HTTP request, capturing the method, route, status code, and duration.
- **`AddHttpClientInstrumentation`** — Creates a span for every outgoing `HttpClient` call, so you can see when your service calls other services or external APIs.
- **`AddEntityFrameworkCoreInstrumentation`** — Creates spans for EF Core database operations, showing you the SQL being executed.
- **`AddRedisInstrumentation`** — Creates spans for Redis operations, so cache hits and misses are visible in your traces.
- **`AddNpgsql`** — Creates spans for raw PostgreSQL queries made through Npgsql.

The `ConfigureResource` call sets the service name that appears in your tracing backend, so you can distinguish between different services in a distributed trace. The `AddOtlpExporter` sends the trace data using the OpenTelemetry Protocol (OTLP), which is the standard export format.

## Setting Up the Exporter

The OTLP exporter needs to know where to send data. Configure the endpoint through an environment variable:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

This points to your tracing backend's OTLP receiver. In development, this will typically be a local Jaeger instance. In production, it might be a collector that fans out to multiple backends.

## Running Jaeger Locally

[Jaeger](https://www.jaegertracing.io/) is an open-source distributed tracing platform that provides a web UI for exploring and analyzing traces. Running it locally takes one Docker command:

```
docker run -d -p 4317:4317 -p 16686:16686 jaegertracing/all-in-one:latest
```

This exposes two ports:

- **4317** — The OTLP gRPC receiver where your application sends trace data
- **16686** — The Jaeger web UI where you explore traces

Open `http://localhost:16686` in your browser and you are ready to start analyzing.

## What Distributed Traces Look Like in Practice

Once your application is instrumented and sending data to Jaeger, you will see traces that tell the full story of each request.

### A Simple Request Across Two Services

Consider a user registration request. The trace shows the request arriving at the API gateway service, being proxied to the backend API service, which makes several HTTP calls before persisting the new user to the database. Every step is a span, and you can see exactly how long each step took.

### Message Bus Interactions

Traces do not stop at HTTP boundaries. When your service publishes a message (for example, a `UserRegisteredIntegrationEvent` via MassTransit), the trace follows the message to its consumers. You can see multiple services picking up the message and performing their work — complete with their own database operations.

### Database Query Visibility

Individual spans can carry detailed contextual information. A PostgreSQL instrumentation span, for example, includes the actual SQL query being executed. This is invaluable for identifying slow queries or understanding what database operations a request triggers.

### Complex Multi-Service Traces

In a real-world system, a single user action might produce a trace spanning three or more .NET applications, a PostgreSQL database, and a Redis cache. For example, a request to fetch a customer's shopping cart might flow through an API gateway, to a ticketing service that owns the data, which then calls back to an authorization service for permission checks. The distributed trace ties all of this together into a single, coherent view.

## How This Connects to Parts 1 and 2

The three parts of this series build on each other:

- **Structured logging** (Part 1) gives you searchable, machine-readable log entries.
- **User context enrichment** (Part 2) attaches identity information to both logs and the `Activity` (which is .NET's representation of a trace span).
- **OpenTelemetry** (Part 3) exports those activities as distributed traces and provides automatic instrumentation for frameworks you are already using.

When you set `Activity.Current?.SetTag("user.id", userId)` in the middleware from Part 2, that tag shows up as an attribute on the trace span in Jaeger. When you log with Serilog as configured in Part 1, those log entries can be correlated with the trace via the trace ID. The pieces fit together.

## Series Recap

Over these three articles, we have built a complete observability stack for .NET applications:

1. **Part 1** established structured logging with Serilog, giving us machine-readable logs with message templates, configurable sinks, and automatic request logging.
2. **Part 2** added user context enrichment through middleware, attaching user IDs to both logging scopes and Activity traces so we can filter and correlate by user.
3. **Part 3** introduced OpenTelemetry for distributed tracing, providing end-to-end visibility across services, databases, caches, and message buses.

Together, these form a practical observability foundation. You get structured logs you can search, user context you can filter by, and distributed traces that show you exactly how requests flow through your system. The upfront investment in setting this up pays dividends every time you need to debug an issue, analyze performance, or understand system behavior.

## References

- [OpenTelemetry](https://opentelemetry.io/) — Official OpenTelemetry website
- [OpenTelemetry .NET SDK](https://github.com/open-telemetry/opentelemetry-dotnet) — .NET implementation
- [Jaeger](https://www.jaegertracing.io/) — Open-source distributed tracing platform
- [OpenTelemetry.Extensions.Hosting](https://www.nuget.org/packages/OpenTelemetry.Extensions.Hosting) — Hosting integration NuGet package
- [OpenTelemetry Instrumentation for ASP.NET Core](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.AspNetCore) — ASP.NET Core auto-instrumentation
- [OTLP Specification](https://opentelemetry.io/docs/specs/otlp/) — OpenTelemetry Protocol specification
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) — Standard naming conventions for telemetry attributes
