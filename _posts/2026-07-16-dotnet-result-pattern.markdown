---
layout: post
title: "The Result Pattern: Functional Error Handling in .NET"
date: 2026-07-16 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - result-pattern
  - error-handling
  - csharp
---


# The Result Pattern: Functional Error Handling in .NET

How should you handle errors in your code? If your first instinct is to throw an exception, you are not alone -- but there is a better way for the errors you *expect*.

Exceptions are a powerful mechanism, but using them for flow control is an anti-pattern. It makes code harder to reason about, hides failure modes from callers, and forces every consumer to dig into implementation details to know which exceptions to catch.

In this article, we will explore the **Result pattern** -- a functional approach to error handling that makes your code more explicit, more testable, and easier to maintain.

## The Problem with Exceptions for Flow Control

Consider a `FollowerService` that validates business rules before creating a follow relationship:

```csharp
public sealed class FollowerService
{
    private readonly IFollowerRepository _followerRepository;

    public FollowerService(IFollowerRepository followerRepository)
    {
        _followerRepository = followerRepository;
    }

    public async Task StartFollowingAsync(
        User user,
        User followed,
        DateTime createdOnUtc,
        CancellationToken cancellationToken = default)
    {
        if (user.Id == followed.Id)
        {
            throw new DomainException("Can't follow yourself");
        }

        if (!followed.HasPublicProfile)
        {
            throw new DomainException("Can't follow non-public profile");
        }

        if (await _followerRepository.IsAlreadyFollowingAsync(
                user.Id,
                followed.Id,
                cancellationToken))
        {
            throw new DomainException("Already following");
        }

        var follower = Follower.Create(user.Id, followed.Id, createdOnUtc);

        _followerRepository.Insert(follower);
    }
}
```

What is wrong here? These are not *exceptional* situations -- they are **expected business rule violations**. A user trying to follow themselves is a perfectly normal scenario that should be handled gracefully, not by unwinding the call stack.

The caller has no way of knowing which exceptions this method might throw without reading its source code. The method signature (`Task`) promises nothing about potential failures.

> **Rule of thumb:** Exceptions are for exceptional situations -- things you genuinely do not know how to handle. For everything else, make the possibility of failure explicit.

## Building the Result Pattern

### The Error Record

First, define a simple `Error` type to represent application errors with a machine-readable code and a human-readable description:

```csharp
public sealed record Error(string Code, string Description)
{
    public static readonly Error None = new(string.Empty, string.Empty);
}
```

### The Result Class

Next, build a `Result` class that wraps the outcome of an operation -- either success or failure with a specific error:

```csharp
public class Result
{
    private Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None ||
            !isSuccess && error == Error.None)
        {
            throw new ArgumentException("Invalid error", nameof(error));
        }

        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }

    public bool IsFailure => !IsSuccess;

    public Error Error { get; }

    public static Result Success() => new(true, Error.None);

    public static Result Failure(Error error) => new(false, error);
}
```

Key design decisions:

- The constructor is **private** -- you must use the `Success()` or `Failure()` factory methods, which prevents creating an invalid `Result` (e.g., a successful result with an error, or a failed result without one).
- The guard clause in the constructor enforces this invariant.
- For operations that return a value on success, you would create a generic `Result<T>` variant.

If you prefer not to build your own, the [FluentResults](https://github.com/altmann/FluentResults) library provides a battle-tested implementation.

## Applying the Result Pattern

Here is the `FollowerService` refactored to use the Result pattern:

```csharp
public sealed class FollowerService
{
    private readonly IFollowerRepository _followerRepository;

    public FollowerService(IFollowerRepository followerRepository)
    {
        _followerRepository = followerRepository;
    }

    public async Task<Result> StartFollowingAsync(
        User user,
        User followed,
        DateTime utcNow,
        CancellationToken cancellationToken = default)
    {
        if (user.Id == followed.Id)
        {
            return Result.Failure(FollowerErrors.SameUser);
        }

        if (!followed.HasPublicProfile)
        {
            return Result.Failure(FollowerErrors.NonPublicProfile);
        }

        if (await _followerRepository.IsAlreadyFollowingAsync(
                user.Id,
                followed.Id,
                cancellationToken))
        {
            return Result.Failure(FollowerErrors.AlreadyFollowing);
        }

        var follower = Follower.Create(user.Id, followed.Id, utcNow);

        _followerRepository.Insert(follower);

        return Result.Success();
    }
}
```

Notice the improvements:

- **No exceptions** for expected failures.
- The return type `Task<Result>` **explicitly communicates** that this method can fail.
- Each failure case returns a **specific, documented error**.
- The code is **easier to test** -- you can assert on `IsSuccess`, `IsFailure`, and `Error` without catching exceptions.

## Documenting Application Errors

With the `Error` record, you can centralize and document all possible errors for a given domain concept:

```csharp
public static class FollowerErrors
{
    public static readonly Error SameUser = new Error(
        "Followers.SameUser", "Can't follow yourself");

    public static readonly Error NonPublicProfile = new Error(
        "Followers.NonPublicProfile", "Can't follow non-public profiles");

    public static readonly Error AlreadyFollowing = new Error(
        "Followers.AlreadyFollowing", "Already following");
}
```

For errors that need runtime context, use static methods instead of fields:

```csharp
public static class FollowerErrors
{
    public static Error NotFound(Guid id) => new Error(
        "Followers.NotFound", $"The follower with Id '{id}' was not found");
}
```

This approach serves as living documentation. You can even write a tool to scan your codebase for all `Error` definitions and generate an error catalog automatically.

## Converting Results to API Responses

Eventually, the `Result` reaches your API layer. The simplest approach is a conditional check:

```csharp
app.MapPost(
    "users/{userId}/follow/{followedId}",
    (Guid userId, Guid followedId, FollowerService followerService) =>
    {
        var result = await followerService.StartFollowingAsync(
            userId,
            followedId,
            DateTime.UtcNow);

        if (result.IsFailure)
        {
            return Results.BadRequest(result.Error);
        }

        return Results.NoContent();
    });
```

For a more functional style, you can implement a `Match` extension method that takes callbacks for each outcome:

```csharp
public static class ResultExtensions
{
    public static T Match<T>(
        this Result result,
        Func<T> onSuccess,
        Func<Error, T> onFailure)
    {
        return result.IsSuccess ? onSuccess() : onFailure(result.Error);
    }
}
```

This lets you express the same logic more concisely:

```csharp
app.MapPost(
    "users/{userId}/follow/{followedId}",
    (Guid userId, Guid followedId, FollowerService followerService) =>
    {
        var result = await followerService.StartFollowingAsync(
            userId,
            followedId,
            DateTime.UtcNow);

        return result.Match(
            onSuccess: () => Results.NoContent(),
            onFailure: error => Results.BadRequest(error));
    });
```

## When to Use Exceptions vs. Results

The two approaches are complementary, not mutually exclusive:

| Situation | Approach |
|-----------|----------|
| Expected business rule violations | Result pattern |
| Validation errors | Result pattern |
| Resource not found | Result pattern |
| Database connection failure | Exception |
| Null reference / programming bug | Exception |
| Out of memory | Exception |

Use the Result pattern for errors you **know how to handle** and exceptions for errors you **don't**. Catch unhandled exceptions at the lowest level possible (global exception middleware, for example).

## Key Takeaways

- Exceptions are for exceptional situations. Expected failures should be represented explicitly using the Result pattern.
- The `Result` class makes failure a **first-class citizen** in your method signatures, improving readability and testability.
- Centralize error definitions in dedicated error classes to serve as living documentation for your domain.
- Use the `Match` extension method for a clean, functional way to convert results into API responses.

## References

- [FluentResults library](https://github.com/altmann/FluentResults)
- [Error handling in ASP.NET Core (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [Result pattern in Domain-Driven Design](https://enterprisecraftsmanship.com/posts/functional-c-handling-failures-input-errors/)
- [Minimal APIs in ASP.NET Core (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
