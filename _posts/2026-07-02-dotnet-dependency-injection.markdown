---
layout: post
title: "Mastering Dependency Injection in ASP.NET Core"
date: 2026-07-02 00:00:00 +0000
tags:
  - dotnet
  - dependency-injection
  - aspnet
  - csharp
---


# Mastering Dependency Injection in ASP.NET Core

Dependency Injection (DI) is one of the foundational pillars of ASP.NET Core. If you have worked with the framework at all, you have already encountered it -- every controller, every middleware component, every service you write participates in the DI system. But understanding *why* DI matters and *how* to use it effectively is what separates code that merely works from code that is testable, maintainable, and ready to evolve.

In this article we will walk through what DI is, the different injection styles ASP.NET Core supports, and the service lifetimes that govern how your objects are created and disposed.

## What Is Dependency Injection?

[Dependency Injection](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-7.0) is a design pattern that implements the [Inversion of Control (IoC)](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion) principle. The core idea is simple: instead of a class creating its own dependencies, those dependencies are *provided* to the class from the outside.

Why does this matter?

- **Testability** -- You can substitute real implementations with test doubles (mocks, stubs, fakes) without changing the class under test.
- **Loose coupling** -- Classes depend on abstractions (interfaces) rather than concrete types, so you can swap implementations without ripple effects.
- **Single Responsibility** -- Each class focuses on its own job; wiring things together is the container's responsibility.

ASP.NET Core ships with a built-in DI container that handles object creation, lifetime management, and disposal automatically.

## Injection Styles

ASP.NET Core supports two primary injection styles: **constructor injection** and **method injection**.

### Constructor Injection

This is the most common approach. Dependencies are declared as constructor parameters, and the DI container resolves them when the class is instantiated.

### Method Injection

In controllers and Minimal API endpoints, you can also receive dependencies directly in action method parameters. This is useful when a dependency is only needed for a single endpoint rather than the entire controller.

Here is an example that combines both styles:

```csharp
[ApiController]
[Route("api/activities")]
public class ActivitiesController : ControllerBase
{
    private readonly ILogger<ActivitiesController> _logger;

    // Constructor injection
    public ActivitiesController(ILogger<ActivitiesController> logger)
    {
        _logger = logger;
    }

    [HttpGet]
    public async Task<IActionResult> Get(ISender sender) // Method injection
    {
        var activities = await sender.Send(new GetActivitiesQuery());

        return Ok(activities);
    }
}
```

The `ILogger` is needed throughout the controller, so it is injected via the constructor. The `ISender` (from MediatR) is only needed in this one action, so method injection keeps the constructor lean.

## Service Lifetimes

When you register a service with the DI container, you must choose a [service lifetime](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#service-lifetimes). This controls how and when instances are created and shared.

ASP.NET Core provides three lifetimes:

### Singleton

```csharp
builder.Services.AddSingleton<IMyService, MyService>();
```

A **single instance** is created when first requested and reused for the entire lifetime of the application. Every consumer -- regardless of scope or request -- receives the same object.

**Use for:** stateless services, caches, configuration holders, or anything that is inherently shared and thread-safe.

**Watch out for:** injecting scoped or transient services into a singleton. This is called a *captive dependency* and will cause those shorter-lived services to live far longer than intended, potentially leading to stale data or thread-safety bugs.

### Scoped

```csharp
builder.Services.AddScoped<IMyService, MyService>();
```

A **new instance** is created once per scope. In a web application, each HTTP request creates a new scope, so scoped services are effectively per-request singletons.

**Use for:** database contexts (`DbContext`), unit-of-work implementations, or any service that should be shared within a single request but isolated between requests.

### Transient

```csharp
builder.Services.AddTransient<IMyService, MyService>();
```

A **new instance** is created every time the service is requested from the container. No sharing occurs.

**Use for:** lightweight, stateless services where you want complete isolation between consumers.

**Watch out for:** transient services that hold expensive resources (database connections, file handles). Creating many short-lived instances can be wasteful.

## Choosing the Right Lifetime

A useful mental model:

| Lifetime   | Created          | Shared across         | Typical use case            |
|------------|------------------|-----------------------|-----------------------------|
| Singleton  | Once             | Entire application    | Caches, config, utilities   |
| Scoped     | Once per request | Same HTTP request     | DbContext, UoW              |
| Transient  | Every time       | Never                 | Lightweight stateless logic |

The most common mistake is mismatching lifetimes. The golden rule: **a service should never depend on another service with a shorter lifetime.** A singleton must not depend on a scoped service; a scoped service must not depend on a transient in ways that assume shared state. ASP.NET Core will throw an `InvalidOperationException` at runtime if you enable scope validation (which is on by default in Development).

## Key Takeaways

- Dependency Injection in ASP.NET Core is not optional -- it is baked into the framework. Embracing it fully leads to cleaner, more testable code.
- Use **constructor injection** for dependencies a class needs throughout its lifetime, and **method injection** for one-off endpoint dependencies.
- Choose service lifetimes deliberately. Most application services are scoped, infrastructure services are often singletons, and short-lived helpers are transient.
- Watch for captive dependencies -- the DI container will help catch them in development mode.

## References

- [Dependency injection in ASP.NET Core (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
- [Service lifetimes (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#service-lifetimes)
- [Dependency Inversion Principle (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion)
- [Scope validation in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection#scope-validation)
