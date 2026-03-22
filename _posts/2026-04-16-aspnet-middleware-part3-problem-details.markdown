---
layout: post
title: "ASP.NET Core Middleware & Error Handling — Part 3: Problem Details"
date: 2026-04-16 00:00:00 +0000
tags:
  - dotnet
  - aspnet
  - error-handling
  - problem-details
  - api-design
series: "ASP.NET Core Middleware & Error Handling"
series_part: 3
---


## Series Overview

- [Part 1: Custom Middleware](2026-04-aspnet-middleware-part1-custom-middleware.md)
- [Part 2: Global Error Handling](2026-04-aspnet-middleware-part2-error-handling.md)
- **Part 3: Problem Details** (this article)


## Introduction

When an API returns an error, the HTTP status code alone is rarely enough. A `400 Bad Request` tells the caller something was wrong with their request, but not what. A `404 Not Found` does not say which resource was missing or why. Without a consistent error format, every API ends up inventing its own, and every consumer has to write custom parsing logic.

**Problem Details** solves this by defining a standardized JSON (and XML) format for error responses, specified in [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (which supersedes RFC 7807). ASP.NET Core has built-in support for Problem Details, and in this final part of the series we will explore how to use it effectively.

## The Problem Details Format

A Problem Details response uses the content type `application/problem+json` and includes these fields:

| Field | Description |
|---|---|
| `type` | A URI reference identifying the problem type |
| `title` | A short, human-readable summary |
| `status` | The HTTP status code |
| `detail` | A human-readable explanation specific to this occurrence |
| `instance` | A URI identifying this specific occurrence of the problem |

Here is an example response:

```json
Content-Type: application/problem+json

{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "detail": "The habit with the specified identifier was not found",
  "instance": "PUT /api/habits/aadcad3f-8dc8-443d-be44-3d99893ba18a"
}
```

This gives API consumers everything they need: a machine-readable type for programmatic handling, a human-readable description for debugging, and context about which request triggered the error.

## Setting Up Problem Details in ASP.NET Core

Getting basic Problem Details support requires just three lines:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Registers services for generating Problem Details responses
builder.Services.AddProblemDetails();

var app = builder.Build();

// Converts unhandled exceptions into Problem Details responses
app.UseExceptionHandler();

// Returns Problem Details for non-successful responses with empty bodies
app.UseStatusCodePages();

app.Run();
```

Let's break down what each piece does:

- **`AddProblemDetails()`** registers the `IProblemDetailsService` and related services. This is what enables the framework to generate Problem Details formatted responses.
- **`UseExceptionHandler()`** adds the built-in exception handling middleware. When an unhandled exception occurs, it catches it and produces a Problem Details response.
- **`UseStatusCodePages()`** handles a subtle case: when your code sets an error status code but writes no response body (for example, returning `NotFound()` from a controller). This middleware detects the empty body and generates a Problem Details response for it.

With this setup, an unhandled exception produces:

```json
Content-Type: application/problem+json

{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.6.1",
  "title": "An error occurred while processing your request.",
  "status": 500
}
```

## Integrating with IExceptionHandler

As we covered in [Part 2](2026-04-aspnet-middleware-part2-error-handling.md), the `IExceptionHandler` interface lets you map specific exception types to appropriate HTTP status codes. Here is a handler that uses pattern matching to determine the status code:

```csharp
internal sealed class CustomExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        int status = exception switch
        {
            ArgumentException => StatusCodes.Status400BadRequest,
            _ => StatusCodes.Status500InternalServerError
        };
        httpContext.Response.StatusCode = status;

        var problemDetails = new ProblemDetails
        {
            Status = status,
            Title = "An error occurred",
            Type = exception.GetType().Name,
            Detail = exception.Message
        };

        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}

// In Program.cs
builder.Services.AddExceptionHandler<CustomExceptionHandler>();
```

This works, but we are manually writing the response. We can do better by using the `IProblemDetailsService`.

## Using IProblemDetailsService

When you call `AddProblemDetails()`, the framework registers a default `IProblemDetailsService` implementation. Instead of writing the response yourself, you can delegate to this service. The benefit is that all Problem Details responses — whether generated by your exception handlers, by the framework, or by `UseStatusCodePages` — pass through the same pipeline and receive the same customizations.

```csharp
public class CustomExceptionHandler(IProblemDetailsService problemDetailsService) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var problemDetails = new ProblemDetails
        {
            Status = exception switch
            {
                ArgumentException => StatusCodes.Status400BadRequest,
                _ => StatusCodes.Status500InternalServerError
            },
            Title = "An error occurred",
            Type = exception.GetType().Name,
            Detail = exception.Message
        };

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            Exception = exception,
            HttpContext = httpContext,
            ProblemDetails = problemDetails
        });
    }
}
```

At first glance this looks almost identical to the previous version. The key difference is that `TryWriteAsync` routes the response through the customization pipeline, which brings us to the next section.

## Customizing All Problem Details Responses

The `AddProblemDetails` method accepts a delegate where you can set `CustomizeProblemDetails`. This runs for every Problem Details response in your application, making it the perfect place for cross-cutting concerns like adding diagnostic information.

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance =
            $"{context.HttpContext.Request.Method} {context.HttpContext.Request.Path}";

        context.ProblemDetails.Extensions.TryAdd("requestId", context.HttpContext.TraceIdentifier);

        Activity? activity = context.HttpContext.Features.Get<IHttpActivityFeature>()?.Activity;
        context.ProblemDetails.Extensions.TryAdd("traceId", activity?.Id);
    };
});
```

Now every error response automatically includes the request path, a request ID, and a distributed trace ID:

```json
Content-Type: application/problem+json

{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "instance": "PUT /api/habits/aadcad3f-8dc8-443d-be44-3d99893ba18a",
  "traceId": "00-63d4af1807586b0d98901ae47944192d-9a8635facb90bf76-01",
  "requestId": "0HN7C8PRNMGIA:00000001"
}
```

The `traceId` is particularly valuable — you can use it to find the corresponding distributed traces and logs in monitoring systems like Seq, Jaeger, or Application Insights.

## .NET 9: StatusCodeSelector

.NET 9 introduces an even simpler way to map exceptions to status codes. Instead of writing a full `IExceptionHandler` implementation, you can configure a `StatusCodeSelector` directly on the exception handler options:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    StatusCodeSelector = ex => ex switch
    {
        ArgumentException => StatusCodes.Status400BadRequest,
        NotFoundException => StatusCodes.Status404NotFound,
        _ => StatusCodes.Status500InternalServerError
    }
});
```

This is a clean, declarative approach that works well when all you need is exception-to-status-code mapping. The framework takes care of generating the Problem Details response body.

One caveat: if you also register an `IExceptionHandler` that explicitly sets the response status code, the `StatusCodeSelector` is ignored for that exception. The handler takes priority.

## Series Recap

Over this three-part series, we have built up a complete picture of middleware and error handling in ASP.NET Core:

- In **Part 1**, we learned the three ways to create custom middleware: request delegates, convention-based classes, and factory-based classes implementing `IMiddleware`.
- In **Part 2**, we used middleware to implement global exception handling, first with a custom middleware class and then with the `IExceptionHandler` interface introduced in ASP.NET Core 8.
- In **Part 3**, we brought it all together with Problem Details — the standardized error response format that ensures your API communicates errors consistently, with built-in support for customization and diagnostics.

The combination of `IExceptionHandler` and `IProblemDetailsService` gives you a robust, extensible error handling pipeline that is easy to test, easy to customize, and follows industry standards. Start with `AddProblemDetails()` and `UseExceptionHandler()` in your next project and build from there.

## References

- [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [IProblemDetailsService](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.iproblemdetailsservice)
- [ProblemDetails class](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.problemdetails)
- [What's new in ASP.NET Core 9.0](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-9.0)
