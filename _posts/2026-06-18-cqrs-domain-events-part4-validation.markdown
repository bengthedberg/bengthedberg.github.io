---
layout: post
title: "CQRS & Domain Events in .NET — Part 4: CQRS Validation with Pipeline Behaviors and FluentValidation"
date: 2026-06-18 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - cqrs
  - domain-events
  - validation
  - fluent-validation
  - mediatr
  - middleware
series: "CQRS & Domain Events in .NET"
series_part: 4
---


## Series Overview

This is a four-part series exploring how to implement CQRS and domain events in .NET, from foundational concepts to production-ready validation pipelines.

1. [Part 1: Domain Events for Loosely Coupled Systems](/posts/cqrs-domain-events-part1-domain-events/)
2. [Part 2: Building a Custom Domain Events Dispatcher](/posts/cqrs-domain-events-part2-dispatcher/)
3. [Part 3: CQRS Pattern — Replacing MediatR](/posts/cqrs-domain-events-part3-cqrs-pattern/)
4. **Part 4: CQRS Validation with Pipeline Behaviors and FluentValidation** (this article)


## Introduction

Validation is an essential cross-cutting concern in any application. Before you process a request, you need to be confident it is valid. But validation is not a single concept — there are different types of validation, and each deserves a specific solution.

In the previous parts of this series, we built domain events, a custom dispatcher, and a full CQRS pipeline. In this final article, we focus on validation: how to separate it cleanly from your handlers, how to distinguish between input validation and business validation, and how to build a generic validation pipeline that works with both MediatR and custom CQRS implementations.

## The Problem: Validation Inside Handlers

The most common approach to validation is doing it right before processing the command, directly inside the handler. Here is a typical example:

```csharp
internal sealed class ShipOrderCommandHandler
    : IRequestHandler<ShipOrderCommand>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IShippingService _shippingService;
    private readonly ShipmentSettings _shipmentSettings;

    public async Task Handle(
        ShipOrderCommand command,
        CancellationToken cancellationToken)
    {
        if (!_shipmentSettings
                .SupportedCountries
                .Contains(command.ShippingAddress.Country))
        {
            throw new ArgumentException(nameof(ShipOrderCommand.Address));
        }

        var order = _orderRepository.Get(command.OrderId);

        _shippingService.ShipTo(
            command.ShippingAddress,
            command.ShippingMethod);
    }
}
```

This approach has several problems:

- Validation logic is **tightly coupled** to the handler
- As validation grows more complex, the handler **grows out of control**
- Every change to validation rules also **touches the handler**
- It is hard to **differentiate between input validation and business validation**

What if we could separate validation from handling entirely?

## Input Validation vs. Business Validation

Not all validation is equal. Understanding the distinction between input and business validation leads to cleaner architecture:

**Input validation** verifies that the command is *processable*. These are simple, cheap checks done in memory:

- Is the `OrderId` non-empty?
- Is the `ShippingAddress` not null?
- Is the `ShippingMethod` a non-empty string?

**Business validation** verifies that the command satisfies business rules. These checks are more expensive because they involve reading system state:

- Does the order exist in the database?
- Is the order in a state that allows shipping?
- Is the shipping address in a supported country?

Input validation sits at the entry point of the use case, *before* the handler runs. After it completes, you have a valid command. This is a rule worth following: **an invalid command should never reach the handler**. Business validation, on the other hand, happens inside the handler where you have access to the necessary state.

## Building Validators with FluentValidation

[FluentValidation](https://docs.fluentvalidation.net/en/latest/index.html) is an excellent library for defining strongly-typed validation rules using a fluent interface and lambda expressions.

Given this command:

```csharp
public sealed record ShipOrderCommand : IRequest
{
    public Guid OrderId { get; set; }
    public string ShippingMethod { get; set; }
    public Address ShippingAddress { get; set; }
}
```

You create a validator by inheriting from `AbstractValidator<T>` and defining rules in the constructor:

```csharp
public sealed class ShipOrderCommandValidator
    : AbstractValidator<ShipOrderCommand>
{
    public ShipOrderCommandValidator(ShipmentSettings settings)
    {
        RuleFor(command => command.OrderId)
            .NotEmpty()
            .WithMessage("The order identifier can't be empty.");

        RuleFor(command => command.ShippingMethod)
            .NotEmpty()
            .WithMessage("The shipping method can't be empty.");

        RuleFor(command => command.ShippingAddress)
            .NotNull()
            .WithMessage("The shipping address can't be empty.");

        RuleFor(command => command.ShippingAddress.Country)
            .Must(country => settings.SupportedCountries.Contains(country))
            .WithMessage("The shipping country isn't supported.");
    }
}
```

A good naming convention is to append `Validator` to the command name: `ShipOrderCommand` gets `ShipOrderCommandValidator`.

To register all validators from an assembly automatically:

```csharp
services.AddValidatorsFromAssembly(ApplicationAssembly.Assembly);
```

## Running Validation Manually

The simplest approach is to inject `IValidator<T>` directly into your handler:

```csharp
internal sealed class ShipOrderCommandHandler
    : IRequestHandler<ShipOrderCommand>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IShippingService _shippingService;
    private readonly IValidator<ShipOrderCommand> _validator;

    public async Task Handle(
        ShipOrderCommand command,
        CancellationToken cancellationToken)
    {
        _validator.ValidateAndThrow(command);

        var order = _orderRepository.Get(command.OrderId);

        _shippingService.ShipTo(
            command.ShippingAddress,
            command.ShippingMethod);
    }
}
```

The validator's `Validate` method returns a `ValidationResult` with an `IsValid` flag and an `Errors` collection. Alternatively, `ValidateAndThrow` throws a `ValidationException` if validation fails.

This works, but it forces you to add an explicit `IValidator` dependency to *every* handler. We can do better with a generic pipeline behavior.

## Building a Generic Validation Pipeline

The real power comes from implementing validation as a cross-cutting concern that applies to all commands automatically. With MediatR, you use `IPipelineBehavior`. With the custom CQRS approach from [Part 3](/posts/cqrs-domain-events-part3-cqrs-pattern/), you use the decorator pattern.

Here is a complete `ValidationBehavior` implementation using MediatR's pipeline:

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommandBase
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);

        var validationFailures = await Task.WhenAll(
            _validators.Select(validator => validator.ValidateAsync(context)));

        var errors = validationFailures
            .Where(validationResult => !validationResult.IsValid)
            .SelectMany(validationResult => validationResult.Errors)
            .Select(validationFailure => new ValidationError(
                validationFailure.PropertyName,
                validationFailure.ErrorMessage))
            .ToList();

        if (errors.Any())
        {
            throw new Exceptions.ValidationException(errors);
        }

        var response = await next();

        return response;
    }
}
```

There are several design decisions worth highlighting:

- The `where TRequest : ICommandBase` constraint ensures that validation only runs for commands, not queries. Queries are read operations and typically do not need input validation.
- We use `ValidateAsync` instead of `Validate` because some validation rules may be asynchronous (for example, checking uniqueness against a database). If you define async rules but call `Validate`, FluentValidation will throw an exception.
- All registered validators for a command run in parallel via `Task.WhenAll`, which improves performance when there are multiple validators.
- If any validation fails, the behavior short-circuits the pipeline by throwing a `ValidationException` before the handler is ever invoked.

Register the behavior with MediatR:

```csharp
services.AddMediatR(config =>
{
    config.RegisterServicesFromAssemblyContaining<ApplicationAssembly>();
    config.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

## Handling Validation Errors in Your API

When validation fails, you need to return a meaningful error response. A clean approach is a custom middleware that catches `ValidationException` and converts it to a `ProblemDetails` response:

```csharp
public sealed class ValidationExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;

    public ValidationExceptionHandlingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exceptions.ValidationException exception)
        {
            var problemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Type = "ValidationFailure",
                Title = "Validation error",
                Detail = "One or more validation errors has occurred"
            };

            if (exception.Errors is not null)
            {
                problemDetails.Extensions["errors"] = exception.Errors;
            }

            context.Response.StatusCode = StatusCodes.Status400BadRequest;

            await context.Response.WriteAsJsonAsync(problemDetails);
        }
    }
}
```

Register the middleware in your request pipeline:

```csharp
app.UseMiddleware<ValidationExceptionHandlingMiddleware>();
```

This converts validation failures into RFC 7807-compliant `ProblemDetails` responses, which is the standard format for HTTP API error responses in ASP.NET Core.

## Adapting for Custom CQRS (Without MediatR)

If you followed [Part 3](/posts/cqrs-domain-events-part3-cqrs-pattern/) and built your own CQRS pipeline, you do not have `IPipelineBehavior`. Instead, you use the decorator pattern to achieve the same result.

The validation decorator we showed in Part 3 does exactly the same thing as the `ValidationBehavior` above — it wraps the handler, runs all registered validators, and short-circuits if validation fails. The only difference is the wiring mechanism:

- **MediatR**: Register as an open behavior with `AddOpenBehavior`
- **Custom CQRS**: Register as a decorator with Scrutor's `Decorate` method

The validation logic itself is identical. This is one of the strengths of the approach: the core patterns remain the same regardless of the dispatching mechanism.

## Series Recap

Over these four articles, we have built a complete CQRS and domain events infrastructure in .NET:

1. **Part 1** introduced domain events as a DDD tactical pattern for building loosely coupled systems, using EF Core for publishing and MediatR for handling.
2. **Part 2** removed the MediatR dependency from event dispatching by building a custom, strongly-typed dispatcher with zero external dependencies.
3. **Part 3** extended the approach to full CQRS with commands, queries, handlers, and decorators — a complete replacement for MediatR's request pipeline.
4. **Part 4** (this article) tackled validation as a cross-cutting concern, showing how to separate input validation from business validation and implement it generically using FluentValidation.

The common thread throughout is **simplicity and control**. You do not need heavyweight frameworks to implement these patterns effectively. A few well-designed interfaces and thoughtful use of .NET's built-in dependency injection system give you a clean, testable, and extensible architecture.

Start simple, measure what matters, and evolve based on real requirements.

## References

- [FluentValidation Documentation](https://docs.fluentvalidation.net/en/latest/index.html)
- [FluentValidation GitHub Repository](https://github.com/FluentValidation/FluentValidation)
- [MediatR Pipeline Behaviors](https://github.com/jbogard/MediatR/wiki/Behaviors)
- [RFC 7807 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc7807)
- [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware)
- [CQRS Pattern — Microsoft Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
