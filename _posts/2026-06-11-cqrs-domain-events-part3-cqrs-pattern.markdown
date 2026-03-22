---
layout: post
title: "CQRS & Domain Events in .NET — Part 3: CQRS Pattern — Replacing MediatR"
date: 2026-06-11 00:00:00 +0000
tags:
  - dotnet
  - cqrs
  - domain-events
  - mediatr
  - clean-architecture
  - scrutor
series: "CQRS & Domain Events in .NET"
series_part: 3
---


## Series Overview

This is a four-part series exploring how to implement CQRS and domain events in .NET, from foundational concepts to production-ready validation pipelines.

1. [Part 1: Domain Events for Loosely Coupled Systems](/posts/cqrs-domain-events-part1-domain-events/)
2. [Part 2: Building a Custom Domain Events Dispatcher](/posts/cqrs-domain-events-part2-dispatcher/)
3. **Part 3: CQRS Pattern — Replacing MediatR** (this article)
4. [Part 4: CQRS Validation with Pipeline Behaviors and FluentValidation](/posts/cqrs-domain-events-part4-validation/)


## Introduction

MediatR became almost synonymous with CQRS in the .NET ecosystem. Many developers assume you need MediatR to implement CQRS, but that is not the case. CQRS and MediatR are not the same thing. MediatR is a library that implements the mediator pattern. CQRS is an architectural pattern that separates reads from writes. Most projects use MediatR as a thin dispatching layer for commands and queries — a use case that can be covered with a few straightforward abstractions.

With [MediatR moving to a commercial license](https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/) for larger companies, now is a good time to evaluate whether you actually need it.

By building your own CQRS infrastructure, you gain:

- **Full control** over your CQRS pipeline
- **Predictable, explicit** handler dispatching
- **Simpler debugging** and onboarding for new team members
- **Cleaner DI setup** and better testability

In this article, we will build a minimal CQRS setup with commands, queries, handlers, and decorator support. No hidden DI magic. Just clean, predictable code.

## Defining Commands, Queries, and Handlers

Let us start with the basic contracts. These are marker interfaces that structure your application logic around intention: write operations go through `ICommand`, read operations through `IQuery`.

```csharp
// ICommand.cs
public interface ICommand;
public interface ICommand<TResponse>;

// IQuery.cs
public interface IQuery<TResponse>;
```

The handler interfaces follow the same model:

```csharp
// ICommandHandler.cs
public interface ICommandHandler<in TCommand>
    where TCommand : ICommand
{
    Task<Result> Handle(TCommand command, CancellationToken cancellationToken);
}

public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    Task<Result<TResponse>> Handle(TCommand command, CancellationToken cancellationToken);
}
```

```csharp
// IQueryHandler.cs
public interface IQueryHandler<in TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    Task<Result<TResponse>> Handle(TQuery query, CancellationToken cancellationToken);
}
```

These interfaces are nearly identical to MediatR's `IRequest` and `IRequestHandler` APIs, which makes migration straightforward if you are moving off MediatR.

All return types use a `Result` wrapper. This is optional, but it promotes explicit success/failure handling and encourages consistency across the application boundary. The `Result` type avoids the need to throw exceptions for expected failure cases — a practice that leads to cleaner control flow and better performance.

## Practical Example: A Command Handler

To see these abstractions in action, let us implement a command that marks a todo item as completed:

```csharp
// CompleteTodoCommand.cs
public sealed record CompleteTodoCommand(Guid TodoItemId) : ICommand;

// CompleteTodoCommandHandler.cs
internal sealed class CompleteTodoCommandHandler(
    IApplicationDbContext context,
    IDateTimeProvider dateTimeProvider,
    IUserContext userContext)
    : ICommandHandler<CompleteTodoCommand>
{
    public async Task<Result> Handle(CompleteTodoCommand command, CancellationToken cancellationToken)
    {
        TodoItem? todoItem = await context.TodoItems
            .SingleOrDefaultAsync(
                t => t.Id == command.TodoItemId && t.UserId == userContext.UserId,
                cancellationToken);

        if (todoItem is null)
        {
            return Result.Failure(TodoItemErrors.NotFound(command.TodoItemId));
        }

        if (todoItem.IsCompleted)
        {
            return Result.Failure(TodoItemErrors.AlreadyCompleted(command.TodoItemId));
        }

        todoItem.IsCompleted = true;
        todoItem.CompletedAt = dateTimeProvider.UtcNow;

        todoItem.Raise(new TodoItemCompletedDomainEvent(todoItem.Id));

        await context.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

A few important things to note:

- The **command** is an immutable value object — just data, no behavior.
- The **handler** encapsulates all business logic: validation, state change, raising domain events, and persistence.
- There is **no mediator**, no `ISender`, no hidden dispatching. The handler is invoked directly through our custom interfaces.

This makes intent explicit, avoids magic, and keeps dependencies minimal.

## Adding Decorators for Cross-Cutting Concerns

To support cross-cutting concerns like logging, validation, and transactions without modifying handlers, we apply the **decorator pattern**. Each decorator wraps the handler, injecting behavior before or after the core logic.

### Logging Decorator

```csharp
using Serilog.Context;

internal sealed class LoggingCommandHandler<TCommand, TResponse>(
    ICommandHandler<TCommand, TResponse> innerHandler,
    ILogger<CommandHandler<TCommand, TResponse>> logger)
    : ICommandHandler<TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    public async Task<Result<TResponse>> Handle(TCommand command, CancellationToken cancellationToken)
    {
        string commandName = typeof(TCommand).Name;

        logger.LogInformation("Processing command {Command}", commandName);

        Result<TResponse> result = await innerHandler.Handle(command, cancellationToken);

        if (result.IsSuccess)
        {
            logger.LogInformation("Completed command {Command}", commandName);
        }
        else
        {
            using (LogContext.PushProperty("Error", result.Error, true))
            {
                logger.LogError("Completed command {Command} with error", commandName);
            }
        }

        return result;
    }
}
```

This decorator wraps any `ICommandHandler<TCommand, TResponse>`, adding structured logging around the command execution without touching the core business logic.

### Validation Decorator

```csharp
using FluentValidation;
using FluentValidation.Results;

internal sealed class ValidationCommandHandler<TCommand, TResponse>(
    ICommandHandler<TCommand, TResponse> innerHandler,
    IEnumerable<IValidator<TCommand>> validators)
    : ICommandHandler<TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    public async Task<Result<TResponse>> Handle(TCommand command, CancellationToken cancellationToken)
    {
        ValidationFailure[] validationFailures = await ValidateAsync(command, validators);

        if (validationFailures.Length == 0)
        {
            return await innerHandler.Handle(command, cancellationToken);
        }

        return Result.Failure<TResponse>(CreateValidationError(validationFailures));
    }

    private static async Task<ValidationFailure[]> ValidateAsync<TCommand>(
        TCommand command,
        IEnumerable<IValidator<TCommand>> validators)
    {
        if (!validators.Any())
        {
            return [];
        }

        var context = new ValidationContext<TCommand>(command);

        ValidationResult[] validationResults = await Task.WhenAll(
            validators.Select(validator => validator.ValidateAsync(context)));

        ValidationFailure[] validationFailures = validationResults
            .Where(validationResult => !validationResult.IsValid)
            .SelectMany(validationResult => validationResult.Errors)
            .ToArray();

        return validationFailures;
    }

    private static ValidationError CreateValidationError(ValidationFailure[] validationFailures) =>
        new(validationFailures.Select(f => Error.Problem(f.ErrorCode, f.ErrorMessage)).ToArray());
}
```

Each decorator handles a single concern and can be layered transparently. Since we are working with generic interfaces, each decorator must explicitly target the same generic contract — you will need separate decorator classes for each handler abstraction (command with result, command without result, query with result).

## Wiring Up with Dependency Injection

With handlers and decorators in place, we register everything using [Scrutor](https://github.com/khellang/Scrutor) for assembly scanning:

```csharp
services.Scan(scan => scan.FromAssembliesOf(typeof(DependencyInjection))
    .AddClasses(classes => classes.AssignableTo(typeof(IQueryHandler<,>)), publicOnly: false)
        .AsImplementedInterfaces()
        .WithScopedLifetime()
    .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<>)), publicOnly: false)
        .AsImplementedInterfaces()
        .WithScopedLifetime()
    .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<,>)), publicOnly: false)
        .AsImplementedInterfaces()
        .WithScopedLifetime());
```

Then apply decorators. **Order matters**, but it may not be intuitive at first:

```csharp
services.Decorate(typeof(ICommandHandler<,>), typeof(ValidationDecorator.CommandHandler<,>));
services.Decorate(typeof(ICommandHandler<>), typeof(ValidationDecorator.CommandBaseHandler<>));

services.Decorate(typeof(IQueryHandler<,>), typeof(LoggingDecorator.QueryHandler<,>));
services.Decorate(typeof(ICommandHandler<,>), typeof(LoggingDecorator.CommandHandler<,>));
services.Decorate(typeof(ICommandHandler<>), typeof(LoggingDecorator.CommandBaseHandler<>));
```

The last decorator applied is the **outermost** one at runtime. So in this example:

1. The base handler is first decorated by **validation**
2. That composite is then decorated by **logging**

This means the logging decorator runs first, captures the full command lifecycle (including validation failures), and then delegates to the validation decorator, which delegates to the core handler.

## Using Handlers from Minimal API Endpoints

Once everything is wired up, using a command handler from a Minimal API endpoint is clean and explicit:

```csharp
internal sealed class Complete : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPut("todos/{id:guid}/complete", async (
            Guid id,
            ICommandHandler<CompleteTodoCommand> handler,
            CancellationToken cancellationToken) =>
        {
            var command = new CompleteTodoCommand(id);

            Result result = await handler.Handle(command, cancellationToken);

            return result.Match(Results.NoContent, CustomResults.Problem);
        })
        .WithTags(Tags.Todos)
        .RequireAuthorization();
    }
}
```

We inject `ICommandHandler<CompleteTodoCommand>` directly into the endpoint. No `ISender`, no mediator layer, no runtime lookup. Everything is resolved explicitly by the container. This makes the code easier to test, reason about, and trace.

## Summary

CQRS does not require a complex framework. With a few small interfaces, some decorator classes, and a clean DI setup, you can build a simple and flexible pipeline for handling commands and queries. It is easy to understand, easy to test, and easy to extend.

The decorator pattern gives you the same cross-cutting concern support that MediatR's `IPipelineBehavior` provides, but with explicit wiring and no hidden indirection.

## What's Next?

In [Part 4: CQRS Validation with Pipeline Behaviors and FluentValidation](/posts/cqrs-domain-events-part4-validation/), we will take a deep dive into validation as a cross-cutting concern — exploring input vs. business validation, building a generic validation pipeline, and handling validation errors gracefully in your API.

## References

- [MediatR Commercial License Announcement](https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/)
- [CQRS Pattern — Microsoft Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Scrutor — Assembly Scanning for .NET DI](https://github.com/khellang/Scrutor)
- [Serilog Structured Logging](https://serilog.net/)
- [FluentValidation Documentation](https://docs.fluentvalidation.net/en/latest/index.html)
- [Result Pattern in .NET](https://www.milanjovanovic.tech/blog/functional-error-handling)
