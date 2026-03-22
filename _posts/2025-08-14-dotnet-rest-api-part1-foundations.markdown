---
layout: post
title: "Building a REST API with .NET — Part 1: Project Setup, Contracts, and Controllers"
date: 2025-08-14 00:00:00 +0000
tags:
  - dotnet
  - rest-api
  - csharp
  - controllers
  - contracts
  - repository-pattern
series: "Building a REST API with .NET"
series_part: 1
---


## Series Overview

This is a 6-part series on building a production-ready REST API with .NET:

1. **Project Setup, Contracts, and Controllers** (this article) — Solution structure, contracts, repository pattern, controllers, and mapping
2. [Database Integration with Dapper](2025-08-dotnet-rest-api-part2-database.md) — PostgreSQL with Docker, Dapper ORM, migrations, and slugs
3. [Business Logic and Validation](2025-09-dotnet-rest-api-part3-validation.md) — Service layer, FluentValidation, middleware, and cancellation tokens
4. [Authentication and Authorization](2025-09-dotnet-rest-api-part4-auth.md) — JWT tokens, claims-based authorization, and user identity
5. [Filtering, Sorting, and Pagination](2025-09-dotnet-rest-api-part5-features.md) — Query parameters, dynamic sorting, paginated responses
6. [Production Readiness](2025-09-dotnet-rest-api-part6-production.md) — Versioning, Swagger/OpenAPI, health checks, caching, and API key auth


## Introduction

In this first part, we lay the groundwork for a Movies REST API. Rather than dumping everything into a single project, we will split the solution into three projects that enforce a clean separation of concerns. By the end of this article you will have a fully working in-memory CRUD API with proper request/response contracts, a repository abstraction, and a controller that ties it all together.

## Solution Structure

A well-structured solution makes it easier to reason about responsibilities. We use three projects:

| Project | Responsibility |
|---|---|
| **Movies.API** | ASP.NET Core host — controllers, middleware, DI wiring |
| **Movies.Application** | Business/domain logic — models, repositories, services |
| **Movies.Contracts** | Request and response DTOs — can be shipped as a NuGet package for clients |

Why separate **Contracts**? If you ever need to publish a client library or share your API models with a Blazor front-end, you can package Contracts independently — it has zero dependency on the rest of your solution.

### Creating the Solution

```bash
dotnet new sln -n Movies
dotnet new webapi -n Movies.API -o Movies.API -controllers
dotnet new classlib -n Movies.Application -o Movies.Application
dotnet new classlib -n Movies.Contracts -o Movies.Contracts
dotnet sln add (ls -r **/*.csproj)
```

Add project references so the API can reach both layers:

```bash
dotnet add ./Movies.API/Movies.API.csproj reference ./Movies.Contracts/Movies.Contracts.csproj
dotnet add ./Movies.API/Movies.API.csproj reference ./Movies.Application/Movies.Application.csproj
```

The dependency graph flows in one direction: **API → Application** and **API → Contracts**. Application knows nothing about ASP.NET Core, and Contracts knows nothing about either.

## Contracts — Request and Response Models

Contracts define the shape of data flowing in and out of the API. They are intentionally simple POCOs with no behaviour.

### Request Models

```csharp
// Movies.Contracts/Requests/CreateMovieRequest.cs
public class CreateMovieRequest
{
    public required string Title { get; init; }
    public required int Year { get; init; }
    public required IEnumerable<string> Genre { get; init; } = Enumerable.Empty<string>();
}
```

```csharp
// Movies.Contracts/Requests/UpdateMovieRequest.cs
public class UpdateMovieRequest
{
    public required string Title { get; init; }
    public required int Year { get; init; }
    public required IEnumerable<string> Genre { get; init; } = Enumerable.Empty<string>();
}
```

### Response Models

```csharp
// Movies.Contracts/Responses/MovieResponse.cs
public class MovieResponse
{
    public required Guid Id { get; init; }
    public required string Title { get; init; }
    public required int Year { get; init; }
    public required IEnumerable<string> Genre { get; init; } = Enumerable.Empty<string>();
}
```

```csharp
// Movies.Contracts/Responses/MoviesResponse.cs
public class MoviesResponse
{
    public required IEnumerable<MovieResponse> Items { get; init; } = Enumerable.Empty<MovieResponse>();
}
```

## Domain Model

The domain model lives in **Movies.Application** and represents the core business entity. It is not concerned with serialization or HTTP — just the data and any domain logic.

```csharp
// Movies.Application/Models/Movie.cs
public class Movie
{
    public required Guid Id { get; init; }
    public required string Title { get; set; }
    public required int Year { get; set; }
    public required List<string> Genre { get; init; } = new();
}
```

Notice that `Id` uses `init` (set once at creation) while `Title` and `Year` use `set` (can change on update). `Genre` is `init` on the list reference, but the list contents are mutable.

## Repository Pattern

The repository pattern abstracts data access behind an interface. Today we use an in-memory dictionary; in Part 2 we will swap it for PostgreSQL with Dapper — and the controller will not change at all.

### Interface

```csharp
// Movies.Application/Repositories/IMovieRepository.cs
public interface IMovieRepository
{
    Task<bool> CreateAsync(Movie movie);
    Task<Movie?> GetByIdAsync(Guid id);
    Task<IEnumerable<Movie>> GetAllAsync();
    Task<bool> UpdateAsync(Movie movie);
    Task<bool> DeleteAsync(Guid id);
}
```

Every method is async from day one — even the in-memory version. This prevents a breaking change when we introduce real I/O later.

### In-Memory Implementation

```csharp
// Movies.Application/Repositories/MovieRepository.cs
public class MovieRepository : IMovieRepository
{
    private readonly Dictionary<Guid, Movie> _movies = new();

    public Task<bool> CreateAsync(Movie movie)
    {
        _movies[movie.Id] = movie;
        return Task.FromResult(true);
    }

    public Task<Movie?> GetByIdAsync(Guid id)
    {
        _movies.TryGetValue(id, out var movie);
        return Task.FromResult(movie);
    }

    public Task<IEnumerable<Movie>> GetAllAsync()
    {
        return Task.FromResult<IEnumerable<Movie>>(_movies.Values);
    }

    public Task<bool> UpdateAsync(Movie movie)
    {
        if (!_movies.ContainsKey(movie.Id))
            return Task.FromResult(false);

        _movies[movie.Id] = movie;
        return Task.FromResult(true);
    }

    public Task<bool> DeleteAsync(Guid id)
    {
        return Task.FromResult(_movies.Remove(id));
    }
}
```

## Service Extension for DI Registration

Rather than cluttering `Program.cs` with every registration, we provide a clean extension method in the Application project:

```csharp
// Movies.Application/ServiceExtensions.cs
public static class ServiceExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddSingleton<IMovieRepository, MovieRepository>();
        return services;
    }
}
```

In `Program.cs`:

```csharp
builder.Services.AddApplication();
```

This keeps the API project's startup code clean and lets the Application project own its own registrations.

## Centralized API Endpoints

Hard-coding route strings in multiple places is error-prone. A static class keeps them in one place:

```csharp
// Movies.API/ApiEndpoints.cs
public static class ApiEndpoints
{
    private const string ApiBase = "api";

    public static class Movies
    {
        private const string Base = $"{ApiBase}/movies";

        public const string Create = Base;
        public const string Get = $"{Base}/{{id:guid}}";
        public const string GetAll = Base;
        public const string Update = $"{Base}/{{id:guid}}";
        public const string Delete = $"{Base}/{{id:guid}}";
    }
}
```

If a route ever changes, you update it in one place.

## Contract Mapping

Extension methods convert between contracts and domain models. This keeps mapping logic out of the controller and makes it testable in isolation.

```csharp
// Movies.API/Mapping/ContractMapping.cs
public static class ContractMapping
{
    public static Movie ToMovie(this CreateMovieRequest request)
    {
        return new Movie
        {
            Id = Guid.NewGuid(),
            Title = request.Title,
            Year = request.Year,
            Genre = request.Genre.ToList()
        };
    }

    public static Movie ToMovie(this UpdateMovieRequest request, Guid id)
    {
        return new Movie
        {
            Id = id,
            Title = request.Title,
            Year = request.Year,
            Genre = request.Genre.ToList()
        };
    }

    public static MovieResponse ToMovieResponse(this Movie movie)
    {
        return new MovieResponse
        {
            Id = movie.Id,
            Title = movie.Title,
            Year = movie.Year,
            Genre = movie.Genre
        };
    }

    public static MoviesResponse ToMoviesResponse(this IEnumerable<Movie> movies)
    {
        return new MoviesResponse
        {
            Items = movies.Select(m => m.ToMovieResponse())
        };
    }
}
```

## The Movie Controller

With all the pieces in place, the controller is remarkably thin. It receives requests, maps them, delegates to the repository, and maps the result back.

```csharp
// Movies.API/Controllers/MoviesController.cs
[ApiController]
public class MoviesController : ControllerBase
{
    private readonly IMovieRepository _movieRepository;

    public MoviesController(IMovieRepository movieRepository)
    {
        _movieRepository = movieRepository;
    }

    [HttpPost(ApiEndpoints.Movies.Create)]
    public async Task<IActionResult> Create([FromBody] CreateMovieRequest request)
    {
        var movie = request.ToMovie();
        await _movieRepository.CreateAsync(movie);
        var response = movie.ToMovieResponse();
        return CreatedAtAction(nameof(Get), new { id = movie.Id }, response);
    }

    [HttpGet(ApiEndpoints.Movies.Get)]
    public async Task<IActionResult> Get([FromRoute] Guid id)
    {
        var movie = await _movieRepository.GetByIdAsync(id);
        if (movie is null)
            return NotFound();

        var response = movie.ToMovieResponse();
        return Ok(response);
    }

    [HttpGet(ApiEndpoints.Movies.GetAll)]
    public async Task<IActionResult> GetAll()
    {
        var movies = await _movieRepository.GetAllAsync();
        var response = movies.ToMoviesResponse();
        return Ok(response);
    }

    [HttpPut(ApiEndpoints.Movies.Update)]
    public async Task<IActionResult> Update([FromRoute] Guid id,
        [FromBody] UpdateMovieRequest request)
    {
        var movie = request.ToMovie(id);
        var updated = await _movieRepository.UpdateAsync(movie);
        if (!updated)
            return NotFound();

        var response = movie.ToMovieResponse();
        return Ok(response);
    }

    [HttpDelete(ApiEndpoints.Movies.Delete)]
    public async Task<IActionResult> Delete([FromRoute] Guid id)
    {
        var deleted = await _movieRepository.DeleteAsync(id);
        if (!deleted)
            return NotFound();

        return Ok();
    }
}
```

A few things to note:

- **`CreatedAtAction`** returns a 201 with a `Location` header pointing to the new resource. This is the correct REST response for a successful POST.
- **`NotFound()`** returns 404 when a resource does not exist — both for GET, UPDATE, and DELETE.
- The controller has no business logic and no data access details. It is purely an HTTP adapter.

## Testing with an HTTP File

Visual Studio and JetBrains Rider support `.http` files for quick ad-hoc testing. Create a file in the API project:

```http
### Create a movie
POST https://localhost:5001/api/movies
Content-Type: application/json

{
  "title": "The Matrix",
  "year": 1999,
  "genre": ["Action", "Sci-Fi"]
}

### Get all movies
GET https://localhost:5001/api/movies

### Get a movie by ID
GET https://localhost:5001/api/movies/{{movieId}}

### Update a movie
PUT https://localhost:5001/api/movies/{{movieId}}
Content-Type: application/json

{
  "title": "The Matrix Reloaded",
  "year": 2003,
  "genre": ["Action", "Sci-Fi"]
}

### Delete a movie
DELETE https://localhost:5001/api/movies/{{movieId}}
```

Replace `{{movieId}}` with a real GUID from the Create response.

## Summary

We now have a clean, layered API with:

- **Contracts** that can be packaged independently for API consumers
- A **domain model** that represents business data without framework concerns
- A **repository interface** that abstracts data access (currently in-memory)
- **Extension methods** for DI registration and contract mapping
- A **controller** that is thin and focused on HTTP concerns

The in-memory repository is great for getting started, but it loses all data on restart and cannot handle concurrent access well.

## What's Next?

In [Part 2: Database Integration with Dapper](2025-08-dotnet-rest-api-part2-database.md), we will replace the in-memory repository with PostgreSQL using Dapper, add Docker for the database, introduce slugs for human-readable URLs, and write proper database migrations.

## References

- [ASP.NET Core Web API documentation](https://learn.microsoft.com/en-us/aspnet/core/web-api/)
- [Repository pattern](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
- [CreatedAtAction](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.controllerbase.createdataction)
