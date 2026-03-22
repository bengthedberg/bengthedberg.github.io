---
layout: post
title: "Building a REST API with .NET — Part 5: Filtering, Sorting, and Pagination"
date: 2025-09-18 00:00:00 +0000
tags:
  - dotnet
  - rest-api
  - csharp
  - filtering
  - sorting
  - pagination
series: "Building a REST API with .NET"
series_part: 5
---


## Series Overview

This is a 6-part series on building a production-ready REST API with .NET:

1. [Project Setup, Contracts, and Controllers](2025-08-dotnet-rest-api-part1-foundations.md) — Solution structure, contracts, repository pattern, controllers, and mapping
2. [Database Integration with Dapper](2025-08-dotnet-rest-api-part2-database.md) — PostgreSQL with Docker, Dapper ORM, migrations, and slugs
3. [Business Logic and Validation](2025-09-dotnet-rest-api-part3-validation.md) — Service layer, FluentValidation, middleware, and cancellation tokens
4. [Authentication and Authorization](2025-09-dotnet-rest-api-part4-auth.md) — JWT tokens, claims-based authorization, and user identity
5. **Filtering, Sorting, and Pagination** (this article) — Query parameters, dynamic sorting, paginated responses
6. [Production Readiness](2025-09-dotnet-rest-api-part6-production.md) — Versioning, Swagger/OpenAPI, health checks, caching, and API key auth


## Introduction

In [Part 4](2025-09-dotnet-rest-api-part4-auth.md) we secured our API with JWT authentication. Now we tackle a problem every API faces as data grows: returning all movies in a single response becomes slow and wasteful. Clients need the ability to filter, sort, and page through results. In this part we add all three.

## Filtering

Filtering lets clients narrow results to what they need. For our Movies API, we support filtering by title (partial match) and year (exact match).

### Request Contract

```csharp
// Movies.Contracts/Requests/GetAllMoviesRequest.cs
public class GetAllMoviesRequest
{
    public string? Title { get; init; }
    public int? Year { get; init; }
    public string? SortBy { get; init; }
    public int? Page { get; init; }
    public int? PageSize { get; init; }
}
```

All fields are nullable — if a client does not provide a filter, we return everything (within pagination limits).

### Options Model

The request contract maps to an options model in the Application layer. This keeps the Application layer independent of HTTP concerns:

```csharp
// Movies.Application/Models/GetAllMoviesOptions.cs
public class GetAllMoviesOptions
{
    public string? Title { get; set; }
    public int? Year { get; set; }
    public string? SortBy { get; set; }
    public SortOrder? SortOrder { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
}

public enum SortOrder
{
    Ascending,
    Descending
}
```

### Mapping

```csharp
// In ContractMapping.cs
public static GetAllMoviesOptions ToOptions(this GetAllMoviesRequest request)
{
    return new GetAllMoviesOptions
    {
        Title = request.Title,
        Year = request.Year,
        SortBy = request.SortBy?.Trim('+', '-'),
        SortOrder = request.SortBy is null ? null
            : request.SortBy.StartsWith('-')
                ? SortOrder.Descending
                : SortOrder.Ascending,
        Page = request.Page.GetValueOrDefault(1),
        PageSize = request.PageSize.GetValueOrDefault(10)
    };
}
```

The sort direction is encoded in the `SortBy` parameter itself: `+title` or `title` for ascending, `-title` for descending. This is a common REST convention.

### Update the Controller

```csharp
[AllowAnonymous]
[HttpGet(ApiEndpoints.Movies.GetAll)]
public async Task<IActionResult> GetAll([FromQuery] GetAllMoviesRequest request,
    CancellationToken token)
{
    var options = request.ToOptions();
    var movies = await _movieService.GetAllAsync(options, token);
    var movieCount = await _movieService.GetCountAsync(options.Title, options.Year, token);

    var response = movies.ToMoviesResponse(
        request.Page.GetValueOrDefault(1),
        request.PageSize.GetValueOrDefault(10),
        movieCount);

    return Ok(response);
}
```

### Parameterized SQL

In the repository, build the WHERE clause dynamically based on which filters are provided:

```csharp
public async Task<IEnumerable<Movie>> GetAllAsync(GetAllMoviesOptions options,
    CancellationToken token = default)
{
    using var connection = await _connectionFactory.CreateConnectionAsync(token);

    var orderClause = string.Empty;
    if (options.SortBy is not null)
    {
        orderClause = $"""
            , m.{options.SortBy}
            ORDER BY m.{options.SortBy} {(options.SortOrder == SortOrder.Ascending ? "asc" : "desc")}
        """;
    }

    var result = await connection.QueryAsync(
        new CommandDefinition($"""
            SELECT m.*,
                   string_agg(DISTINCT g.name, ',') AS genre
                   {orderClause.Split("ORDER BY")[0]}
            FROM movies m
            LEFT JOIN genres g ON m.id = g.movieId
            WHERE (@Title IS NULL OR m.title LIKE ('%' || @Title || '%'))
            AND   (@Year IS NULL  OR m.year = @Year)
            GROUP BY m.id
            {(orderClause.Contains("ORDER BY") ? "ORDER BY" + orderClause.Split("ORDER BY")[1] : "")}
            LIMIT @PageSize
            OFFSET @PageOffset
        """,
        new
        {
            options.Title,
            options.Year,
            options.PageSize,
            PageOffset = (options.Page - 1) * options.PageSize
        }, cancellationToken: token));

    return result.Select(m => new Movie
    {
        Id = m.id,
        Title = m.title,
        Year = (int)m.year,
        Genre = ((string)m.genre)?.Split(',').ToList() ?? new List<string>()
    });
}
```

The `WHERE` clause uses a pattern where `@Title IS NULL OR ...` makes the filter optional. If `@Title` is null, the condition is always true and that filter is skipped. This avoids building SQL strings dynamically with if statements.

> **Security note:** The `SortBy` value is interpolated into SQL, which is an injection risk. Always validate it against a whitelist of allowed columns (see below).

## Sorting

### Validating Sort Fields

Never allow arbitrary column names in `ORDER BY`. Define allowed fields and validate:

```csharp
// Movies.Application/Validators/GetAllMoviesOptionsValidator.cs
using FluentValidation;

public class GetAllMoviesOptionsValidator : AbstractValidator<GetAllMoviesOptions>
{
    private static readonly string[] AllowedSortFields =
    {
        "title", "year"
    };

    public GetAllMoviesOptionsValidator()
    {
        RuleFor(x => x.SortBy)
            .Must(x => x is null || AllowedSortFields.Contains(x, StringComparer.OrdinalIgnoreCase))
            .WithMessage("You can only sort by: " + string.Join(", ", AllowedSortFields));

        RuleFor(x => x.Page)
            .GreaterThanOrEqualTo(1);

        RuleFor(x => x.PageSize)
            .InclusiveBetween(1, 25)
            .WithMessage("Page size must be between 1 and 25");
    }
}
```

Add validation to the service:

```csharp
public async Task<IEnumerable<Movie>> GetAllAsync(GetAllMoviesOptions options,
    CancellationToken token = default)
{
    await _optionsValidator.ValidateAndThrowAsync(options, token);
    return await _movieRepository.GetAllAsync(options, token);
}
```

### Example Requests

```http
### Filter by title
GET https://localhost:5001/api/movies?title=matrix

### Filter by year
GET https://localhost:5001/api/movies?year=1999

### Sort by title ascending
GET https://localhost:5001/api/movies?sortBy=title

### Sort by year descending
GET https://localhost:5001/api/movies?sortBy=-year

### Combine filters and sorting
GET https://localhost:5001/api/movies?title=the&sortBy=-year&page=1&pageSize=5
```

## Pagination

### Why Paginate?

Without pagination, a `GET /api/movies` endpoint returns every movie in the database. With 10,000 movies, that is a large payload, slow query, and high memory usage. Pagination returns a manageable subset and gives clients metadata to navigate through results.

### Response Contract

Update the response to include pagination metadata:

```csharp
// Movies.Contracts/Responses/MoviesResponse.cs
public class MoviesResponse
{
    public required IEnumerable<MovieResponse> Items { get; init; }
        = Enumerable.Empty<MovieResponse>();
    public required int Page { get; init; }
    public required int PageSize { get; init; }
    public required int Total { get; init; }
    public bool HasNextPage => Total > Page * PageSize;
}
```

`HasNextPage` is a computed property — clients can use it to decide whether to show a "Load More" button or prefetch the next page.

### Mapping Update

```csharp
public static MoviesResponse ToMoviesResponse(this IEnumerable<Movie> movies,
    int page, int pageSize, int total)
{
    return new MoviesResponse
    {
        Items = movies.Select(m => m.ToMovieResponse()),
        Page = page,
        PageSize = pageSize,
        Total = total
    };
}
```

### Count Query

Add a method to get the total count of movies matching the filters (without pagination):

```csharp
// IMovieRepository
Task<int> GetCountAsync(string? title, int? year, CancellationToken token = default);

// MovieRepository
public async Task<int> GetCountAsync(string? title, int? year,
    CancellationToken token = default)
{
    using var connection = await _connectionFactory.CreateConnectionAsync(token);

    return await connection.QuerySingleAsync<int>(
        new CommandDefinition("""
            SELECT COUNT(id)
            FROM movies
            WHERE (@Title IS NULL OR title LIKE ('%' || @Title || '%'))
            AND   (@Year IS NULL  OR year = @Year)
        """,
        new { Title = title, Year = year },
        cancellationToken: token));
}
```

The count query uses the same WHERE clause as the main query — but without the JOIN, GROUP BY, LIMIT, or OFFSET. This ensures the total count accurately reflects the filtered dataset.

### Service Layer

```csharp
// IMovieService
Task<int> GetCountAsync(string? title, int? year, CancellationToken token = default);

// MovieService
public async Task<int> GetCountAsync(string? title, int? year,
    CancellationToken token = default)
{
    return await _movieRepository.GetCountAsync(title, year, token);
}
```

### Example Response

```json
{
  "items": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "slug": "the-matrix-1999",
      "title": "The Matrix",
      "year": 1999,
      "genre": ["Action", "Sci-Fi"]
    },
    {
      "id": "7bc94a12-3e5f-4890-a1b2-c3d4e5f67890",
      "slug": "inception-2010",
      "title": "Inception",
      "year": 2010,
      "genre": ["Action", "Sci-Fi", "Thriller"]
    }
  ],
  "page": 1,
  "pageSize": 10,
  "total": 47,
  "hasNextPage": true
}
```

The client knows there are 47 total results, they are on page 1 of 5, and there is a next page to fetch.

## SQL Query Summary

The complete GetAll query combines filtering, sorting, and pagination in a single efficient statement:

```sql
SELECT m.*,
       string_agg(DISTINCT g.name, ',') AS genre
FROM movies m
LEFT JOIN genres g ON m.id = g.movieId
WHERE (@Title IS NULL OR m.title LIKE ('%' || @Title || '%'))
AND   (@Year IS NULL  OR m.year = @Year)
GROUP BY m.id
ORDER BY m.title ASC
LIMIT @PageSize
OFFSET @PageOffset
```

- **WHERE** — Filters rows before grouping (efficient)
- **GROUP BY** — Aggregates genres per movie
- **ORDER BY** — Sorts the result set
- **LIMIT/OFFSET** — Returns only the requested page

## Summary

Our API now supports the three pillars of data retrieval:

- **Filtering** — Narrow results by title and year using parameterized SQL
- **Sorting** — Dynamic ORDER BY with validated sort fields and ascending/descending support
- **Pagination** — LIMIT/OFFSET with total count metadata and `hasNextPage` flag

These features work together: a client can request "action movies from 2020, sorted by title, page 2 of 10 results" in a single request.

## What's Next?

In [Part 6: Production Readiness](2025-09-dotnet-rest-api-part6-production.md), we will add API versioning, Swagger documentation, health checks, output caching, and API key authentication — everything you need before deploying to production.

## References

- [Dapper parameterized queries](https://github.com/DapperLib/Dapper#parameterized-queries)
- [REST API filtering best practices](https://www.moesif.com/blog/technical/api-design/REST-API-Design-Filtering-Sorting-and-Pagination/)
- [PostgreSQL LIMIT and OFFSET](https://www.postgresql.org/docs/current/queries-limit.html)
