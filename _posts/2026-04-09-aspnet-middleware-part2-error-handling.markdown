---
layout: post
title: "ASP.NET Core Middleware & Error Handling — Part 2: Global Error Handling"
date: 2026-04-09 00:00:00 +0000
tags:
  - dotnet
  - aspnet
  - middleware
  - error-handling
  - exceptions
series: "ASP.NET Core Middleware & Error Handling"
series_part: 2
---


## Series Overview

- [Part 1: Custom Middleware](/posts/aspnet-middleware-part1-custom-middleware/)
- **Part 2: Global Error Handling** (this article)
- [Part 3: Problem Details](/posts/aspnet-middleware-part3-problem-details/)


## Introduction

Exceptions are for exceptional situations — but they will inevitably happen in your applications. Database connections drop, external services time out, and invalid state slips through. When an unhandled exception escapes your endpoint code, you need a safety net that catches it, logs it, and returns a meaningful response to the caller.

In [Part 1](/posts/aspnet-middleware-part1-custom-middleware/) we learned how to build custom middleware. In this article we will apply that knowledge to implement global exception handling. We will cover two approaches: the traditional custom middleware technique and the newer `IExceptionHandler` interface introduced in ASP.NET Core 8.

## The Traditional Approach: Exception Handling Middleware

The classic pattern wraps the entire downstream pipeline in a `try-catch`. If anything throws, the middleware catches the exception, logs it, and writes an error response.

Here is a convention-based middleware that does exactly that:

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception exception)
        {
            _logger.LogError(
                exception, "Exception occurred: {Message}", exception.Message);

            var problemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "Server Error"
            };

            context.Response.StatusCode =
                StatusCodes.Status500InternalServerError;

            await context.Response.WriteAsJsonAsync(problemDetails);
        }
    }
}
```

Register it early in the pipeline so it wraps as many downstream components as possible:

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

This approach works well and has been the standard for years. The downside is that all your exception-to-response mapping logic lives in a single `catch` block, which can grow unwieldy as you add more exception types.

## The Modern Approach: IExceptionHandler (ASP.NET Core 8+)

ASP.NET Core 8 introduced the `IExceptionHandler` interface, which plugs into the built-in `ExceptionHandlerMiddleware`. Instead of writing your own middleware from scratch, you implement a handler that the framework calls when an exception occurs.

The interface has a single method — `TryHandleAsync` — that returns `true` if the exception was handled or `false` to pass it to the next handler in the chain.

Here is a `GlobalExceptionHandler` that catches everything:

```csharp
internal sealed class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(
            exception, "Exception occurred: {Message}", exception.Message);

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server error"
        };

        httpContext.Response.StatusCode = problemDetails.Status.Value;

        await httpContext.Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

### Registering IExceptionHandler

Two steps are required:

1. Register the handler with dependency injection.
2. Add the built-in exception handler middleware to the pipeline.

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// ...

app.UseExceptionHandler();
```

Note the call to `AddProblemDetails()` — this registers services that generate Problem Details responses for common exceptions. We will explore this in depth in [Part 3](/posts/aspnet-middleware-part3-problem-details/).

One important detail: `IExceptionHandler` implementations are registered as singletons. Be careful about injecting scoped or transient services into them.

## Chaining Multiple Exception Handlers

The real power of `IExceptionHandler` shows when you have multiple handlers. They execute in the order they are registered, and each one can decide whether to handle a given exception or pass it along.

This is useful when you want to map specific exception types to specific HTTP status codes. For example, you might have custom exceptions like `BadRequestException` and `NotFoundException`:

```csharp
internal sealed class BadRequestExceptionHandler : IExceptionHandler
{
    private readonly ILogger<BadRequestExceptionHandler> _logger;

    public BadRequestExceptionHandler(ILogger<BadRequestExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not BadRequestException badRequestException)
        {
            return false;
        }

        _logger.LogError(
            badRequestException,
            "Exception occurred: {Message}",
            badRequestException.Message);

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status400BadRequest,
            Title = "Bad Request",
            Detail = badRequestException.Message
        };

        httpContext.Response.StatusCode = problemDetails.Status.Value;

        await httpContext.Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

```csharp
internal sealed class NotFoundExceptionHandler : IExceptionHandler
{
    private readonly ILogger<NotFoundExceptionHandler> _logger;

    public NotFoundExceptionHandler(ILogger<NotFoundExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not NotFoundException notFoundException)
        {
            return false;
        }

        _logger.LogError(
            notFoundException,
            "Exception occurred: {Message}",
            notFoundException.Message);

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status404NotFound,
            Title = "Not Found",
            Detail = notFoundException.Message
        };

        httpContext.Response.StatusCode = problemDetails.Status.Value;

        await httpContext.Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

Register them in the order you want them evaluated:

```csharp
builder.Services.AddExceptionHandler<BadRequestExceptionHandler>();
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

The `BadRequestExceptionHandler` runs first. If the exception is not a `BadRequestException`, it returns `false` and the `NotFoundExceptionHandler` gets a chance. If that also returns `false`, the `GlobalExceptionHandler` catches everything else.

## Which Approach Should You Choose?

For new projects targeting ASP.NET Core 8 or later, prefer `IExceptionHandler`. It gives you a cleaner separation of concerns, supports chaining, and integrates naturally with the built-in Problem Details infrastructure.

For older projects or situations where you need full control over the response pipeline, the custom middleware approach remains perfectly valid.

### A Word of Caution

While chaining exception handlers by exception type is a powerful pattern, be thoughtful about using exceptions for flow control. Exceptions carry performance overhead and can make control flow harder to reason about. Consider using the Result pattern for expected failure cases and reserve exceptions for truly exceptional situations.

## What's Next?

Both approaches in this article return Problem Details responses, but we have only scratched the surface of what Problem Details can do. In [Part 3: Problem Details](/posts/aspnet-middleware-part3-problem-details/), we will explore the Problem Details specification in depth, learn how to customize responses with `IProblemDetailsService`, and see the new `StatusCodeSelector` feature in .NET 9.

## References

- [Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [IExceptionHandler interface](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.diagnostics.iexceptionhandler)
- [ExceptionHandlerMiddleware](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.diagnostics.exceptionhandlermiddleware)
- [ASP.NET Core 8 — What's New](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-8.0)
