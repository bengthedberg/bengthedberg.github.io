---
layout: post
title: "CQRS & Domain Events in .NET — Part 2: Building a Custom Domain Events Dispatcher"
date: 2026-06-04 00:00:00 +0000
tags:
  - dotnet
  - cqrs
  - domain-events
  - ddd
  - dependency-injection
  - clean-architecture
series: "CQRS & Domain Events in .NET"
series_part: 2
---


## Series Overview

This is a four-part series exploring how to implement CQRS and domain events in .NET, from foundational concepts to production-ready validation pipelines.

1. [Part 1: Domain Events for Loosely Coupled Systems](2026-05-cqrs-domain-events-part1-domain-events.md)
2. **Part 2: Building a Custom Domain Events Dispatcher** (this article)
3. [Part 3: CQRS Pattern — Replacing MediatR](2026-06-cqrs-domain-events-part3-cqrs-pattern.md)
4. [Part 4: CQRS Validation with Pipeline Behaviors and FluentValidation](2026-06-cqrs-domain-events-part4-validation.md)


## Introduction

In [Part 1](2026-05-cqrs-domain-events-part1-domain-events.md), we used MediatR to dispatch domain events. MediatR is a solid library, but it is a third-party dependency — and for something as fundamental as event dispatching, you may want full control over the implementation.

In this article, we will build a lightweight, custom domain event dispatcher using nothing but .NET's built-in dependency injection. The core dispatching logic will have zero external dependencies. You will understand every line of code, which makes debugging and customization straightforward.

We will cover why you might want a custom dispatcher, how to define the abstractions, how to implement and register handlers, how to build a strongly-typed dispatcher, and the trade-offs involved.

## The Problem: Tight Coupling

Before diving into the implementation, let us look at the problem domain events solve with a concrete example. Consider this tightly coupled service:

```csharp
public class UserService
{
    public async Task RegisterUser(string email, string password)
    {
        var user = new User(email, password);
        await _userRepository.SaveAsync(user);

        // Directly coupled to email service
        await _emailService.SendWelcomeEmail(user.Email);

        // Directly coupled to analytics
        await _analyticsService.TrackUserRegistration(user.Id);

        // What if we need to add more features?
        // This method will keep growing...
    }
}
```

Every new side effect requires modifying `RegisterUser`. With domain events, the method becomes focused:

```csharp
public class UserService
{
    public async Task RegisterUser(string email, string password)
    {
        var user = new User(email, password);
        await _userRepository.SaveAsync(user);

        // Publish event — let other parts of the system react
        await _domainEventsDispatcher.DispatchAsync(
            [new UserRegisteredDomainEvent(user.Id, user.Email)]);
    }
}
```

Now `UserService` handles user registration and nothing else. New side effects are added by creating new handlers, without touching existing code. This is the Open/Closed Principle in action.

## Defining the Abstractions

We need just two interfaces to form the foundation of our event system:

```csharp
// Marker interface for all domain events.
public interface IDomainEvent
{
    // You could add common properties here like:
    // DateTime OccurredAt { get; }
    // Guid EventId { get; }
}

// Generic interface for handling domain events.
public interface IDomainEventHandler<in T> where T : IDomainEvent
{
    Task Handle(T domainEvent, CancellationToken cancellationToken = default);
}
```

This design gives you type safety through generic constraints while keeping publishers and handlers completely decoupled. You can add new events or handlers without touching existing code, and everything remains easily testable in isolation.

Notice that `IDomainEvent` does not inherit from any third-party interface — unlike the MediatR-based approach from Part 1. Your domain layer has zero external dependencies.

## Implementing Handlers

Handlers are simple classes that implement `IDomainEventHandler<T>` for a specific event type. Multiple handlers can react to the same event:

```csharp
// Handles sending welcome emails when users register
internal sealed class SendWelcomeEmailHandler(IEmailService emailService)
    : IDomainEventHandler<UserRegisteredDomainEvent>
{
    public async Task Handle(
        UserRegisteredDomainEvent domainEvent,
        CancellationToken cancellationToken = default)
    {
        var welcomeEmail = new WelcomeEmail(domainEvent.Email, domainEvent.UserId);
        await emailService.SendAsync(welcomeEmail, cancellationToken);
    }
}

// Handles analytics tracking for new user registrations
internal sealed class TrackUserRegistrationHandler(IAnalyticsService analyticsService)
    : IDomainEventHandler<UserRegisteredDomainEvent>
{
    public async Task Handle(
        UserRegisteredDomainEvent domainEvent,
        CancellationToken cancellationToken = default)
    {
        await analyticsService.TrackEvent(
            "user_registered",
            new
            {
                user_id = domainEvent.UserId,
                registration_date = domainEvent.RegisteredAt
            },
            cancellationToken);
    }
}
```

Each handler focuses on a single concern and can be tested independently.

## Registering Handlers with DI

You can register handlers manually:

```csharp
// In your Program.cs or Startup.cs
services.AddScoped<IDomainEventHandler<UserRegisteredDomainEvent>, SendWelcomeEmailHandler>();
services.AddScoped<IDomainEventHandler<UserRegisteredDomainEvent>, TrackUserRegistrationHandler>();
```

Or automate registration using assembly scanning with [Scrutor](https://github.com/khellang/Scrutor):

```csharp
services.Scan(scan => scan.FromAssembliesOf(typeof(DependencyInjection))
    .AddClasses(classes => classes.AssignableTo(typeof(IDomainEventHandler<>)), publicOnly: false)
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

The `publicOnly: false` parameter ensures internal handler classes are also discovered. This is important because handlers are typically implementation details that should not be exposed as part of your public API.

## Building the Dispatcher

Now for the core piece: the dispatcher that resolves and invokes the correct handlers for each event. The challenge is that we receive events as `IDomainEvent` at runtime, but our handlers are generic (`IDomainEventHandler<T>`). We need to bridge this gap efficiently.

```csharp
public interface IDomainEventsDispatcher
{
    Task DispatchAsync(
        IEnumerable<IDomainEvent> domainEvents,
        CancellationToken cancellationToken = default);
}

internal sealed class DomainEventsDispatcher(IServiceProvider serviceProvider)
    : IDomainEventsDispatcher
{
    private static readonly ConcurrentDictionary<Type, Type> HandlerTypeDictionary = new();
    private static readonly ConcurrentDictionary<Type, Type> WrapperTypeDictionary = new();

    public async Task DispatchAsync(
        IEnumerable<IDomainEvent> domainEvents,
        CancellationToken cancellationToken = default)
    {
        foreach (IDomainEvent domainEvent in domainEvents)
        {
            using IServiceScope scope = serviceProvider.CreateScope();

            Type domainEventType = domainEvent.GetType();

            Type handlerType = HandlerTypeDictionary.GetOrAdd(
                domainEventType,
                et => typeof(IDomainEventHandler<>).MakeGenericType(et));

            IEnumerable<object?> handlers = scope.ServiceProvider.GetServices(handlerType);

            foreach (object? handler in handlers)
            {
                if (handler is null) continue;

                var handlerWrapper = HandlerWrapper.Create(handler, domainEventType);

                await handlerWrapper.Handle(domainEvent, cancellationToken);
            }
        }
    }

    // Abstract base class for strongly-typed handler wrappers
    private abstract class HandlerWrapper
    {
        public abstract Task Handle(IDomainEvent domainEvent, CancellationToken cancellationToken);

        public static HandlerWrapper Create(object handler, Type domainEventType)
        {
            Type wrapperType = WrapperTypeDictionary.GetOrAdd(
                domainEventType,
                et => typeof(HandlerWrapper<>).MakeGenericType(et));

            return (HandlerWrapper)Activator.CreateInstance(wrapperType, handler)!;
        }
    }

    // Generic wrapper that provides strong typing for handler invocation
    private sealed class HandlerWrapper<T>(object handler) : HandlerWrapper where T : IDomainEvent
    {
        private readonly IDomainEventHandler<T> _handler = (IDomainEventHandler<T>)handler;

        public override async Task Handle(
            IDomainEvent domainEvent,
            CancellationToken cancellationToken)
        {
            await _handler.Handle((T)domainEvent, cancellationToken);
        }
    }
}
```

### How the Wrapper Pattern Works

The key insight is the `HandlerWrapper` abstraction. When the dispatcher encounters a `UserRegisteredDomainEvent`, it creates a `HandlerWrapper<UserRegisteredDomainEvent>` that holds a strongly-typed reference to `IDomainEventHandler<UserRegisteredDomainEvent>`.

Reflection is used only once — during wrapper creation. The actual handler invocation uses compile-time types, giving you the performance benefits of avoiding reflection in the hot path. The `ConcurrentDictionary` caches the constructed generic types so subsequent dispatches of the same event type skip the reflection entirely.

Register the dispatcher with DI:

```csharp
services.AddTransient<IDomainEventsDispatcher, DomainEventsDispatcher>();
```

## Usage Example

Here is how you use the dispatcher in a controller:

```csharp
public class UserController(
    IUserService userService,
    IDomainEventsDispatcher domainEventsDispatcher) : ControllerBase
{
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterUserRequest request)
    {
        try
        {
            var user = await userService.CreateUserAsync(request.Email, request.Password);

            var userRegisteredEvent = new UserRegisteredDomainEvent(user.Id, user.Email);
            await domainEventsDispatcher.DispatchAsync([userRegisteredEvent]);

            return Ok(new { UserId = user.Id, Message = "User registered successfully" });
        }
        catch (Exception ex)
        {
            return BadRequest(new { Error = ex.Message });
        }
    }
}
```

You could also integrate the dispatcher into your EF Core `SaveChangesAsync` override, as we discussed in [Part 1](2026-05-cqrs-domain-events-part1-domain-events.md), to automatically dispatch events from your domain entities.

## Limitations and Trade-offs

This implementation runs entirely in-process, which has important implications:

- **Immediate feedback** — if any handler fails, the exception bubbles up to the caller immediately. No silent failures or eventual consistency surprises.
- **Caller control** — the code that dispatches events decides how to handle failures: rollback transactions, retry operations, or continue despite errors. The dispatcher does not make these decisions for you.
- **Reliability concerns** — if the process crashes after some handlers succeed but before others complete, there is no automatic recovery. Events are not persisted or retried.

For critical side effects that cannot be lost, consider the [Outbox pattern](https://www.milanjovanovic.tech/blog/implementing-the-outbox-pattern). Instead of dispatching events immediately, store them alongside your business data in the same transaction. A background service processes them reliably, decoupling reliability from performance.

## Summary

Domain events do not require a heavyweight framework. The implementation we built here — two interfaces, a set of handlers, and a strongly-typed dispatcher — provides a solid foundation that you can extend as your needs grow.

The beauty of rolling your own solution is that you understand every piece. There is no hidden magic, no implicit behavior, and debugging is straightforward. This pattern fits well in Domain-Driven Design and Clean Architecture systems where decoupling business logic is a priority.

For systems requiring bulletproof reliability or cross-service communication, invest in proper message infrastructure. But for many applications, this simple approach is the right balance between coupling and complexity.

## What's Next?

In [Part 3: CQRS Pattern — Replacing MediatR](2026-06-cqrs-domain-events-part3-cqrs-pattern.md), we will go beyond event dispatching and build a complete CQRS pipeline with commands, queries, and decorators — all without MediatR.

## References

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Scrutor — Assembly Scanning for .NET DI](https://github.com/khellang/Scrutor)
- [Outbox Pattern — Reliable Messaging](https://www.milanjovanovic.tech/blog/implementing-the-outbox-pattern)
- [Microsoft — Dependency Injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [ConcurrentDictionary — Thread-Safe Collections](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2)
