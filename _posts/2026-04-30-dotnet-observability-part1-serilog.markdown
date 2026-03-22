---
layout: post
title: ".NET Observability — Part 1: Structured Logging with Serilog"
date: 2026-04-30 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - observability
  - logging
  - serilog
  - aspnetcore
series: ".NET Observability"
series_part: 1
---


## Series Overview

This is a three-part series on building observable .NET applications, from structured logging through request tracing to full distributed tracing with OpenTelemetry.

1. **Part 1: Structured Logging with Serilog** (this article)
2. [Part 2: Better Request Tracing with User Context](/posts/dotnet-observability-part2-request-tracing/)
3. [Part 3: Distributed Tracing with OpenTelemetry](/posts/dotnet-observability-part3-opentelemetry/)


## Why Structured Logging Matters

If you have ever tried to debug a production issue by searching through plain-text log files, you know the pain. Unstructured log messages are written for humans to read one at a time, but they are terrible for searching, filtering, and aggregating at scale.

**Structured logging** solves this by treating every log entry as a data record with well-defined fields, rather than an arbitrary string. When your logs are structured — as JSON documents, database rows, or any machine-readable format — you unlock the ability to query them like data. Filter by user ID, correlate by request, aggregate by error type. This is foundational for observability.

In the .NET ecosystem, [Serilog](https://serilog.net/) is the go-to library for structured logging, and it integrates seamlessly with ASP.NET Core.

## Installing Serilog

Getting started is straightforward. Add the `Serilog.AspNetCore` NuGet package to your project:

```powershell
Install-Package Serilog.AspNetCore
```

This package provides a clean API for wiring Serilog into the ASP.NET Core host. In your `Program.cs`, call `UseSerilog` on the host builder and `UseSerilogRequestLogging` on the application pipeline:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((context, configuration) =>
    configuration.ReadFrom.Configuration(context.Configuration));

var app = builder.Build();

app.UseSerilogRequestLogging();

app.Run();
```

Two important things are happening here:

- **`UseSerilog`** replaces the default .NET logging provider with Serilog. The call to `ReadFrom.Configuration()` means Serilog will pull its settings from `appsettings.json`, which keeps configuration flexible and environment-specific.
- **`UseSerilogRequestLogging`** adds automatic HTTP request logging. Every incoming request gets a single, structured log entry with the method, path, status code, and elapsed time — far cleaner than the multiple log lines the default ASP.NET Core logger produces.

## Configuring Serilog via appsettings.json

Serilog's configuration lives in a `Serilog` section of your `appsettings.json`. This is where you define sinks (where logs go), minimum log levels, and enrichers (additional context attached to every log entry).

Here is a practical configuration that writes to both the console and rolling JSON log files:

```json
"Serilog": {
  "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.File" ],
  "MinimumLevel": {
    "Default": "Information",
    "Override": {
      "Microsoft": "Warning",
      "System": "Warning"
    }
  },
  "WriteTo": [
    { "Name": "Console" },
    {
      "Name": "File",
      "Args": {
        "path": "/logs/log-.txt",
        "rollingInterval": "Day",
        "rollOnFileSizeLimit": true,
        "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
      }
    }
  ],
  "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ]
}
```

A few things worth noting:

- **MinimumLevel overrides** let you silence the noisy `Microsoft.*` and `System.*` namespaces while keeping your application logs at `Information` level.
- **Enrichers** like `FromLogContext`, `WithMachineName`, and `WithThreadId` automatically attach metadata to every log entry without you having to pass it explicitly.
- The **CompactJsonFormatter** produces compact, machine-readable JSON — ideal for ingestion by log aggregation tools like Seq, Elasticsearch, or Grafana Loki.

For the full list of configuration options, see the [Serilog.Settings.Configuration documentation](https://github.com/serilog/serilog-settings-configuration).

## Using Serilog in Your Code

One of Serilog's best design decisions is that it integrates with the standard `ILogger` interface from `Microsoft.Extensions.Logging`. You do not need to take a dependency on Serilog types in your application code. Just inject `ILogger<T>` as you normally would:

```csharp
app.MapGet("/serilog-is-cool", (ILogger logger) =>
{
    logger.LogInformation("This is a log inside of the Minimal API endpoint.");

    return Results.Ok(new { Message = "success" });
});
```

At runtime, Serilog provides the implementation behind that interface. If you ever decide to switch logging libraries, your application code stays untouched.

## The Power of Message Templates

Where structured logging truly shines is in Serilog's **message template** syntax. Instead of interpolating values into a string (which destroys their identity), you define named placeholders and pass values separately:

```csharp
var book = new { Author = "Domain-Driven Design", Title = "Eric Evans" };
var orderNumber = 1;

log.LogInformation(
    "Processing book {@Book}, order number = {@OrderNumber}",
    book,
    orderNumber);
```

The key details:

- **`{OrderNumber}`** captures a scalar value as a named property on the log entry.
- **`{@Book}`** uses the `@` destructuring operator, which tells Serilog to serialize the object's properties rather than just calling `ToString()`.

The result is a log entry where `Book` and `OrderNumber` are queryable, filterable fields. You can search for all log entries where `OrderNumber == 42` or where `Book.Author == "Eric Evans"` — something that is impossible with flat string logs.

## Why This Matters for Observability

Structured logging is the first pillar of observability. It gives you:

- **Searchability**: Machine-readable logs can be indexed and queried efficiently.
- **Context**: Named properties provide rich information about what was happening when a log entry was created.
- **Correlation**: When combined with request IDs and user context (which we will explore in Part 2), structured logs let you reconstruct the full story of any request.
- **Root cause analysis**: When errors occur, structured logs provide enough detail to identify the cause without needing to reproduce the issue.

Getting structured logging right is the foundation everything else builds on. Without it, request tracing and distributed tracing have far less value.

## What's Next?

In [Part 2: Better Request Tracing with User Context](/posts/dotnet-observability-part2-request-tracing/), we will build on this foundation by enriching our logs with user identity information. You will learn how to use ASP.NET Core middleware to automatically attach user IDs to every log entry and activity trace, making it trivial to filter logs for a specific user when troubleshooting.

## References

- [Serilog](https://serilog.net/) — Official Serilog website
- [Serilog.AspNetCore](https://github.com/serilog/serilog-aspnetcore) — ASP.NET Core integration package
- [Serilog.Settings.Configuration](https://github.com/serilog/serilog-settings-configuration) — Configuration via appsettings.json
- [Serilog.Formatting.Compact](https://github.com/serilog/serilog-formatting-compact) — Compact JSON formatter
- [Microsoft.Extensions.Logging](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/) — .NET logging fundamentals
