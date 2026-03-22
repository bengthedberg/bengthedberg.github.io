---
layout: post
title: "Building a REST API with .NET — Part 3: Business Logic and Validation"
date: 2025-09-04 00:00:00 +0000
tags:
  - dotnet
  - rest-api
  - csharp
  - validation
  - fluentvalidation
  - middleware
series: "Building a REST API with .NET"
series_part: 3
---


## Series Overview

This is a 6-part series on building a production-ready REST API with .NET:

1. [Project Setup, Contracts, and Controllers](2025-08-dotnet-rest-api-part1-foundations.md) — Solution structure, contracts, repository pattern, controllers, and mapping
2. [Database Integration with Dapper](2025-08-dotnet-rest-api-part2-database.md) — PostgreSQL with Docker, Dapper ORM, migrations, and slugs
3. **Business Logic and Validation** (this article) — Service layer, FluentValidation, middleware, and cancellation tokens
4. [Authentication and Authorization](2025-09-dotnet-rest-api-part4-auth.md) — JWT tokens, claims-based authorization, and user identity
5. [Filtering, Sorting, and Pagination](2025-09-dotnet-rest-api-part5-features.md) — Query parameters, dynamic sorting, paginated responses
6. [Production Readiness](2025-09-dotnet-rest-api-part6-production.md) — Versioning, Swagger/OpenAPI, health checks, caching, and API key auth


## Introduction

In [Part 2](2025-08-dotnet-rest-api-part2-database.md) we added PostgreSQL and Dapper. Our controller currently talks directly to the repository — this works, but as business logic grows, the controller becomes a dumping ground. In this part we extract a service layer, add validation with FluentValidation, build middleware to handle validation errors consistently, and thread cancellation tokens through every layer.

## The Service Layer

### Why a Service Layer?

There are two common anti-patterns:

1. **Fat controllers** — Business logic in controllers. Controllers should only handle HTTP concerns (status codes, headers, routing).
2. **Smart repositories** — Business rules in repositories. Repositories should only handle data persistence.

The service layer sits between them and owns the business logic:

```
Controller → Service → Repository
   (HTTP)    (Business)  (Data)
```

### Interface

```csharp
// Movies.Application/Services/IMovieService.cs
public interface IMovieService
{
    Task<bool> CreateAsync(Movie movie);
    Task<Movie?> GetByIdAsync(Guid id);
    Task<Movie?> GetBySlugAsync(string slug);
    Task<IEnumerable<Movie>> GetAllAsync();
    Task<Movie?> UpdateAsync(Movie movie);
    Task<bool> DeleteAsync(Guid id);
}
```

Notice that `UpdateAsync` returns `Movie?` instead of `bool`. This is a deliberate design choice — the service can return the updated movie (with any computed properties like the slug), or `null` if the movie was not found. The controller gets a richer return type to work with.

### Implementation

```csharp
// Movies.Application/Services/MovieService.cs
public class MovieService : IMovieService
{
    private readonly IMovieRepository _movieRepository;

    public MovieService(IMovieRepository movieRepository)
    {
        _movieRepository = movieRepository;
    }

    public async Task<bool> CreateAsync(Movie movie)
    {
        return await _movieRepository.CreateAsync(movie);
    }

    public async Task<Movie?> GetByIdAsync(Guid id)
    {
        return await _movieRepository.GetByIdAsync(id);
    }

    public async Task<Movie?> GetBySlugAsync(string slug)
    {
        return await _movieRepository.GetBySlugAsync(slug);
    }

    public async Task<IEnumerable<Movie>> GetAllAsync()
    {
        return await _movieRepository.GetAllAsync();
    }

    public async Task<Movie?> UpdateAsync(Movie movie)
    {
        var movieExists = await _movieRepository.ExistsByIdAsync(movie.Id);
        if (!movieExists)
            return null;

        await _movieRepository.UpdateAsync(movie);
        return movie;
    }

    public async Task<bool> DeleteAsync(Guid id)
    {
        return await _movieRepository.DeleteAsync(id);
    }
}
```

Right now the service is thin — it mostly delegates. That is fine. As we add validation, authorization, caching, and other cross-cutting concerns, the service will grow. The important thing is that the structure is in place.

> **Note:** In larger applications, you might use the **MediatR** pattern (Command/Query Responsibility Segregation) to further decouple handlers from the pipeline. For our Movies API, a service layer strikes the right balance between simplicity and structure.

### Update the Controller

The controller now depends on `IMovieService` instead of `IMovieRepository`:

```csharp
[ApiController]
public class MoviesController : ControllerBase
{
    private readonly IMovieService _movieService;

    public MoviesController(IMovieService movieService)
    {
        _movieService = movieService;
    }

    [HttpPost(ApiEndpoints.Movies.Create)]
    public async Task<IActionResult> Create([FromBody] CreateMovieRequest request)
    {
        var movie = request.ToMovie();
        await _movieService.CreateAsync(movie);
        var response = movie.ToMovieResponse();
        return CreatedAtAction(nameof(Get), new { identity = movie.Id }, response);
    }

    [HttpGet(ApiEndpoints.Movies.Get)]
    public async Task<IActionResult> Get([FromRoute] string identity)
    {
        var movie = Guid.TryParse(identity, out var id)
            ? await _movieService.GetByIdAsync(id)
            : await _movieService.GetBySlugAsync(identity);

        if (movie is null)
            return NotFound();

        return Ok(movie.ToMovieResponse());
    }

    [HttpGet(ApiEndpoints.Movies.GetAll)]
    public async Task<IActionResult> GetAll()
    {
        var movies = await _movieService.GetAllAsync();
        return Ok(movies.ToMoviesResponse());
    }

    [HttpPut(ApiEndpoints.Movies.Update)]
    public async Task<IActionResult> Update([FromRoute] Guid id,
        [FromBody] UpdateMovieRequest request)
    {
        var movie = request.ToMovie(id);
        var updatedMovie = await _movieService.UpdateAsync(movie);
        if (updatedMovie is null)
            return NotFound();

        return Ok(updatedMovie.ToMovieResponse());
    }

    [HttpDelete(ApiEndpoints.Movies.Delete)]
    public async Task<IActionResult> Delete([FromRoute] Guid id)
    {
        var deleted = await _movieService.DeleteAsync(id);
        if (!deleted)
            return NotFound();

        return Ok();
    }
}
```

### Register the Service

```csharp
// Movies.Application/ServiceExtensions.cs
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddSingleton<IMovieRepository, MovieRepository>();
    services.AddSingleton<IMovieService, MovieService>();
    return services;
}
```

## FluentValidation

ASP.NET Core has built-in model validation with data annotations (`[Required]`, `[Range]`, etc.), but FluentValidation gives you more power:

- Complex rules expressed in readable C# (not attributes)
- Async validation (e.g., checking the database for uniqueness)
- Testable validator classes
- Conditional validation

Install the package:

```bash
dotnet add Movies.Application package FluentValidation
dotnet add Movies.Application package FluentValidation.DependencyInjectionExtensions
```

### Movie Validator

```csharp
// Movies.Application/Validators/MovieValidator.cs
using FluentValidation;

public class MovieValidator : AbstractValidator<Movie>
{
    private readonly IMovieRepository _movieRepository;

    public MovieValidator(IMovieRepository movieRepository)
    {
        _movieRepository = movieRepository;

        RuleFor(x => x.Id)
            .NotEmpty();

        RuleFor(x => x.Genre)
            .NotEmpty();

        RuleFor(x => x.Title)
            .NotEmpty();

        RuleFor(x => x.Year)
            .LessThanOrEqualTo(DateTime.UtcNow.Year);

        RuleFor(x => x.Slug)
            .MustAsync(ValidateSlug)
            .WithMessage("This movie already exists in the system");
    }

    private async Task<bool> ValidateSlug(Movie movie, string slug,
        CancellationToken cancellationToken)
    {
        var existingMovie = await _movieRepository.GetBySlugAsync(slug);
        if (existingMovie is not null)
        {
            // It's OK if the slug belongs to the movie being updated
            return existingMovie.Id == movie.Id;
        }
        return true;
    }
}
```

The slug validation is **async** because it queries the database. This is one of FluentValidation's strengths — try doing that cleanly with data annotations.

### Assembly Marker for Auto-Registration

FluentValidation can discover validators automatically from an assembly. We need a marker interface:

```csharp
// Movies.Application/IApplicationMarker.cs
public interface IApplicationMarker { }
```

Register all validators from the assembly:

```csharp
// Movies.Application/ServiceExtensions.cs
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddSingleton<IMovieRepository, MovieRepository>();
    services.AddSingleton<IMovieService, MovieService>();
    services.AddValidatorsFromAssemblyContaining<IApplicationMarker>(ServiceLifetime.Singleton);
    return services;
}
```

### Add Validation to the Service

The service calls the validator before delegating to the repository:

```csharp
// Movies.Application/Services/MovieService.cs
public class MovieService : IMovieService
{
    private readonly IMovieRepository _movieRepository;
    private readonly IValidator<Movie> _movieValidator;

    public MovieService(IMovieRepository movieRepository,
        IValidator<Movie> movieValidator)
    {
        _movieRepository = movieRepository;
        _movieValidator = movieValidator;
    }

    public async Task<bool> CreateAsync(Movie movie)
    {
        await _movieValidator.ValidateAndThrowAsync(movie);
        return await _movieRepository.CreateAsync(movie);
    }

    // ... other methods

    public async Task<Movie?> UpdateAsync(Movie movie)
    {
        await _movieValidator.ValidateAndThrowAsync(movie);

        var movieExists = await _movieRepository.ExistsByIdAsync(movie.Id);
        if (!movieExists)
            return null;

        await _movieRepository.UpdateAsync(movie);
        return movie;
    }
}
```

`ValidateAndThrowAsync` throws a `ValidationException` if validation fails. We handle that exception globally with middleware.

## Validation Middleware

Instead of try/catch blocks in every controller action, we use middleware to catch `ValidationException` globally and return a consistent 400 response.

### Validation Response Contract

```csharp
// Movies.Contracts/Responses/ValidationFailureResponse.cs
public class ValidationFailureResponse
{
    public required IEnumerable<ValidationResponse> Errors { get; init; }
}

public class ValidationResponse
{
    public required string PropertyName { get; init; }
    public required string ErrorMessage { get; init; }
}
```

### Middleware

```csharp
// Movies.API/Middleware/ValidationMappingMiddleware.cs
using FluentValidation;

public class ValidationMappingMiddleware
{
    private readonly RequestDelegate _next;

    public ValidationMappingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = 400;

            var validationFailureResponse = new ValidationFailureResponse
            {
                Errors = ex.Errors.Select(e => new ValidationResponse
                {
                    PropertyName = e.PropertyName,
                    ErrorMessage = e.ErrorMessage
                })
            };

            await context.Response.WriteAsJsonAsync(validationFailureResponse);
        }
    }
}
```

Register it in `Program.cs` — middleware order matters, so add it early in the pipeline:

```csharp
app.UseMiddleware<ValidationMappingMiddleware>();
app.MapControllers();
```

Now any validation failure from any endpoint returns a structured error response:

```json
{
  "errors": [
    {
      "propertyName": "Title",
      "errorMessage": "'Title' must not be empty."
    },
    {
      "propertyName": "Year",
      "errorMessage": "'Year' must be less than or equal to '2025'."
    }
  ]
}
```

## Cancellation Tokens

### Why Cancellation Tokens?

When a client cancels an HTTP request (closes the browser tab, times out, navigates away), the server should stop processing. Without cancellation tokens, the server continues executing the database query and building the response — wasting resources on a result nobody will receive.

ASP.NET Core provides a `CancellationToken` that is triggered when the client disconnects. We need to thread it through every layer.

### Controller

Add `CancellationToken` as a parameter to every action. ASP.NET Core injects it automatically:

```csharp
[HttpGet(ApiEndpoints.Movies.GetAll)]
public async Task<IActionResult> GetAll(CancellationToken token)
{
    var movies = await _movieService.GetAllAsync(token);
    return Ok(movies.ToMoviesResponse());
}
```

### Service

Use `token = default` so callers are not forced to provide a token:

```csharp
// IMovieService.cs
Task<IEnumerable<Movie>> GetAllAsync(CancellationToken token = default);

// MovieService.cs
public async Task<IEnumerable<Movie>> GetAllAsync(CancellationToken token = default)
{
    return await _movieRepository.GetAllAsync(token);
}
```

### Repository

Pass the token to Dapper's `CommandDefinition`:

```csharp
public async Task<IEnumerable<Movie>> GetAllAsync(CancellationToken token = default)
{
    using var connection = await _connectionFactory.CreateConnectionAsync();
    var movies = await connection.QueryAsync(
        new CommandDefinition("""
            SELECT m.*, string_agg(g.name, ',') AS genre
            FROM movies m
            LEFT JOIN genres g ON m.id = g.movieId
            GROUP BY m.id
        """, cancellationToken: token));

    return movies.Select(m => new Movie
    {
        Id = m.id,
        Title = m.title,
        Year = m.year,
        Genre = ((string)m.genre)?.Split(',').ToList() ?? new List<string>()
    });
}
```

### Database Connection

The connection factory should also accept a cancellation token:

```csharp
public async Task<IDbConnection> CreateConnectionAsync(CancellationToken token = default)
{
    var connection = new NpgsqlConnection(_connectionString);
    await connection.OpenAsync(token);
    return connection;
}
```

The cancellation token now flows from the HTTP request all the way down to the database connection. If the client disconnects, the `OperationCanceledException` bubbles up and ASP.NET Core handles it gracefully.

## Summary

This part addressed three critical concerns:

- **Service layer** — Business logic has a proper home, separate from HTTP concerns and data access
- **FluentValidation** — Declarative, testable validation with async support for database checks
- **Validation middleware** — Consistent error responses without repetitive try/catch
- **Cancellation tokens** — Proper resource cleanup when clients disconnect

## What's Next?

In [Part 4: Authentication and Authorization](2025-09-dotnet-rest-api-part4-auth.md), we will add JWT-based authentication, protect endpoints with authorization policies, and extract user identity from tokens for user-specific features like movie ratings.

## References

- [FluentValidation documentation](https://docs.fluentvalidation.net/)
- [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [CancellationToken best practices](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads)
