---
layout: post
title: "ASP.NET Core Middleware & Error Handling — Part 1: Custom Middleware"
date: 2026-04-02 00:00:00 +0000
tags:
  - dotnet
  - aspnet
  - middleware
  - pipeline
series: "ASP.NET Core Middleware & Error Handling"
series_part: 1
---


## Series Overview

- **Part 1: Custom Middleware** (this article)
- [Part 2: Global Error Handling](2026-04-aspnet-middleware-part2-error-handling.md)
- [Part 3: Problem Details](2026-04-aspnet-middleware-part3-problem-details.md)


## Introduction

Middleware is the backbone of the ASP.NET Core request pipeline. Every HTTP request that enters your application passes through a series of middleware components before reaching your endpoint, and the response travels back through the same chain in reverse. You are already using many built-in middleware components — authentication, routing, CORS — but at some point you will need to write your own.

Custom middleware is the right tool whenever you need cross-cutting logic that applies to every (or nearly every) request: logging, timing, header manipulation, correlation IDs, or global error handling (which we will cover in Part 2).

In this article, we will look at three approaches to creating custom middleware in ASP.NET Core and discuss the trade-offs between them.

## How Middleware Works

Before diving into code, it helps to understand the core concept. Each middleware component:

1. Receives the incoming `HttpContext`.
2. Optionally does work **before** the next component runs.
3. Calls the next component in the pipeline.
4. Optionally does work **after** the next component has finished.

If a middleware does not call the next component, it **short-circuits** the pipeline — the request never reaches downstream middleware or the endpoint.

## Approach 1: Request Delegates (Inline Middleware)

The simplest way to add middleware is with a **request delegate** — a lambda you register directly on the `WebApplication` instance using the `Use` method.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Use(async (context, next) =>
{
    // Add code before request.

    await next(context);

    // Add code after request.
});
```

By awaiting the `next` delegate you continue the pipeline. If you omit the call to `next`, you short-circuit it.

**When to use this:** Quick, one-off logic during prototyping or for very small applications. For anything more substantial, a dedicated class is easier to test and maintain.

## Approach 2: Convention-Based Middleware

The second approach moves the middleware into its own class. ASP.NET Core discovers it by convention rather than through an interface, so there are a few rules you must follow:

1. Accept a `RequestDelegate` in the constructor.
2. Define an `InvokeAsync` method that takes an `HttpContext` parameter.
3. Call the `RequestDelegate` inside `InvokeAsync`, passing the `HttpContext`.

```csharp
public class ConventionMiddleware(
    RequestDelegate next,
    ILogger<ConventionMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        logger.LogInformation("Before request");

        await next(context);

        logger.LogInformation("After request");
    }
}
```

Register it in the pipeline with:

```csharp
app.UseMiddleware<ConventionMiddleware>();
```

**When to use this:** This is the most common pattern you will see in documentation and open-source projects. It works well for middleware that only needs constructor-injected singleton services (such as `ILogger`). Note that convention-based middleware is instantiated once at startup, so injecting scoped or transient services through the constructor will not behave as expected — inject them as parameters on `InvokeAsync` instead.

## Approach 3: Factory-Based Middleware (IMiddleware)

The third approach implements the `IMiddleware` interface, which defines a single `InvokeAsync` method:

```csharp
public class FactoryMiddleware(ILogger<FactoryMiddleware> logger) : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        logger.LogInformation("Before request");

        await next(context);

        logger.LogInformation("After request");
    }
}
```

Because the framework resolves `FactoryMiddleware` from the DI container at runtime, you must register it as a service:

```csharp
builder.Services.AddTransient<FactoryMiddleware>();
```

And add it to the pipeline the same way:

```csharp
app.UseMiddleware<FactoryMiddleware>();
```

**When to use this:** When you want the middleware itself to participate in dependency injection with a specific lifetime, or when you prefer the compile-time safety of implementing an interface rather than relying on naming conventions.

## Which Approach Should You Choose?

All three approaches produce functionally identical middleware. The differences are ergonomic:

| Approach | Typing | DI Lifetime | Best For |
|---|---|---|---|
| Request Delegate | None (lambda) | N/A | Quick prototyping |
| Convention-Based | Implicit (naming rules) | Singleton by default | Most production middleware |
| Factory-Based (`IMiddleware`) | Explicit (interface) | Configurable | Strong typing, scoped services |

The factory-based approach with `IMiddleware` has a practical advantage worth highlighting: because it is an interface, you can use reflection to scan your assembly for all classes implementing `IMiddleware`, register them in DI automatically, and wire them into the pipeline — eliminating the risk of forgetting to register a middleware component.

## What's Next?

Now that you know how to build custom middleware, the next logical step is to use it for one of the most important cross-cutting concerns in any API: error handling. In [Part 2: Global Error Handling](2026-04-aspnet-middleware-part2-error-handling.md), we will build an exception-handling middleware and explore the `IExceptionHandler` interface introduced in ASP.NET Core 8.

## References

- [ASP.NET Core Middleware documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Write custom ASP.NET Core middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write)
- [IMiddleware interface](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.imiddleware)
