---
layout: post
title: "Building a REST API with .NET — Part 2: Database Integration with Dapper"
date: 2025-08-28 00:00:00 +0000
tags:
  - dotnet
  - rest-api
  - csharp
  - dapper
  - postgresql
  - docker
series: "Building a REST API with .NET"
series_part: 2
---


## Series Overview

This is a 6-part series on building a production-ready REST API with .NET:

1. [Project Setup, Contracts, and Controllers](/posts/dotnet-rest-api-part1-foundations/) — Solution structure, contracts, repository pattern, controllers, and mapping
2. **Database Integration with Dapper** (this article) — PostgreSQL with Docker, Dapper ORM, migrations, and slugs
3. [Business Logic and Validation](/posts/dotnet-rest-api-part3-validation/) — Service layer, FluentValidation, middleware, and cancellation tokens
4. [Authentication and Authorization](/posts/dotnet-rest-api-part4-auth/) — JWT tokens, claims-based authorization, and user identity
5. [Filtering, Sorting, and Pagination](/posts/dotnet-rest-api-part5-features/) — Query parameters, dynamic sorting, paginated responses
6. [Production Readiness](/posts/dotnet-rest-api-part6-production/) — Versioning, Swagger/OpenAPI, health checks, caching, and API key auth


## Introduction

In [Part 1](/posts/dotnet-rest-api-part1-foundations/) we built a working CRUD API backed by an in-memory dictionary. That is fine for prototyping, but we need a real database for persistence, concurrency, and querying. In this part we introduce PostgreSQL running in Docker, use Dapper as a lightweight ORM, write database migrations, and add slug-based URLs for a better developer experience.

## Adding a Slug

A slug is a human-readable identifier derived from the resource's data. Instead of asking users to remember `3fa85f64-5717-4562-b3fc-2c963f66afa6`, they can use `the-matrix-1999`. Slugs are great for URLs, logging, and debugging.

### Update the Domain Model

Add a computed `Slug` property to the `Movie` class:

```csharp
// Movies.Application/Models/Movie.cs
public class Movie
{
    public required Guid Id { get; init; }
    public required string Title { get; set; }
    public string Slug => GenerateSlug();
    public required int Year { get; set; }
    public required List<string> Genre { get; init; } = new();

    private string GenerateSlug()
    {
        return Title.ToLower().Replace(" ", "-") + "-" + Year;
    }
}
```

The slug is derived — never stored separately — so it is always consistent with the title and year.

### Update the Response Contract

```csharp
// Movies.Contracts/Responses/MovieResponse.cs
public class MovieResponse
{
    public required Guid Id { get; init; }
    public required string Slug { get; init; }
    public required string Title { get; init; }
    public required int Year { get; init; }
    public required IEnumerable<string> Genre { get; init; } = Enumerable.Empty<string>();
}
```

Update the mapping extension to include the slug:

```csharp
public static MovieResponse ToMovieResponse(this Movie movie)
{
    return new MovieResponse
    {
        Id = movie.Id,
        Slug = movie.Slug,
        Title = movie.Title,
        Year = movie.Year,
        Genre = movie.Genre
    };
}
```

### Accept Both GUID and Slug in Endpoints

Update the route to accept a string, then determine if it is a GUID or a slug:

```csharp
// Movies.API/ApiEndpoints.cs
public static class ApiEndpoints
{
    private const string ApiBase = "api";

    public static class Movies
    {
        private const string Base = $"{ApiBase}/movies";

        public const string Create = Base;
        public const string Get = $"{Base}/{{identity}}";
        public const string GetAll = Base;
        public const string Update = $"{Base}/{{id:guid}}";
        public const string Delete = $"{Base}/{{id:guid}}";
    }
}
```

Update the repository interface with a `GetBySlugAsync` method:

```csharp
// Movies.Application/Repositories/IMovieRepository.cs
public interface IMovieRepository
{
    Task<bool> CreateAsync(Movie movie);
    Task<Movie?> GetByIdAsync(Guid id);
    Task<Movie?> GetBySlugAsync(string slug);
    Task<IEnumerable<Movie>> GetAllAsync();
    Task<bool> UpdateAsync(Movie movie);
    Task<bool> DeleteAsync(Guid id);
}
```

And update the controller's Get action:

```csharp
[HttpGet(ApiEndpoints.Movies.Get)]
public async Task<IActionResult> Get([FromRoute] string identity)
{
    var movie = Guid.TryParse(identity, out var id)
        ? await _movieRepository.GetByIdAsync(id)
        : await _movieRepository.GetBySlugAsync(identity);

    if (movie is null)
        return NotFound();

    var response = movie.ToMovieResponse();
    return Ok(response);
}
```

Now clients can use either `GET /api/movies/3fa85f64-...` or `GET /api/movies/the-matrix-1999`.

## PostgreSQL with Docker

Docker makes it trivial to run a local PostgreSQL instance. Create a `docker-compose.yml` at the solution root:

```yml
version: "3.8"
services:
  db:
    image: postgres
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: movies
    ports:
      - 5432:5432
```

Start it with:

```bash
docker-compose up -d
```

You now have a PostgreSQL database running on `localhost:5432` with a `movies` database ready to go.

## Database Connection Factory

We need a clean way to create database connections. Define an interface and implementation:

```csharp
// Movies.Application/Database/IDatabaseConnectionFactory.cs
public interface IDatabaseConnectionFactory
{
    Task<IDbConnection> CreateConnectionAsync();
}
```

```csharp
// Movies.Application/Database/NpgSqlConnectionFactory.cs
using Npgsql;

public class NpgSqlConnectionFactory : IDatabaseConnectionFactory
{
    private readonly string _connectionString;

    public NpgSqlConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<IDbConnection> CreateConnectionAsync()
    {
        var connection = new NpgsqlConnection(_connectionString);
        await connection.OpenAsync();
        return connection;
    }
}
```

Install the required NuGet packages in the Application project:

```bash
dotnet add Movies.Application package Npgsql
dotnet add Movies.Application package Dapper
```

## Database Migration

Rather than using a full migration framework, we write a simple migration class that runs on startup. This creates the tables if they do not exist:

```csharp
// Movies.Application/Database/DatabaseMigration.cs
using Dapper;

public class DatabaseMigration
{
    private readonly IDatabaseConnectionFactory _connectionFactory;

    public DatabaseMigration(IDatabaseConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task MigrateAsync()
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();

        await connection.ExecuteAsync("""
            CREATE TABLE IF NOT EXISTS movies (
                id UUID PRIMARY KEY,
                slug TEXT NOT NULL,
                title TEXT NOT NULL,
                year INTEGER NOT NULL
            )
        """);

        await connection.ExecuteAsync("""
            CREATE UNIQUE INDEX CONCURRENTLY IF NOT EXISTS movies_slug_idx
            ON movies
            USING btree(slug)
        """);

        await connection.ExecuteAsync("""
            CREATE TABLE IF NOT EXISTS genres (
                movieId UUID REFERENCES movies(id),
                name TEXT NOT NULL
            )
        """);
    }
}
```

Key design decisions:

- **Movies and genres are separate tables** — this is a one-to-many relationship (a movie has many genres).
- **Slug has a unique index** — ensures no two movies can have the same slug, and makes slug lookups fast.
- `CREATE TABLE IF NOT EXISTS` makes the migration idempotent — safe to run multiple times.

### Connection String Configuration

Add the connection string to `appsettings.json`:

```json
{
  "Database": {
    "ConnectionString": "Server=localhost;Port=5432;Database=movies;User ID=demo;Password=demo;"
  }
}
```

### Running the Migration at Startup

In `Program.cs`:

```csharp
var app = builder.Build();

// Run database migration
var migration = app.Services.GetRequiredService<DatabaseMigration>();
await migration.MigrateAsync();
```

## Dapper-Based Repository

Now we replace the in-memory repository with one that talks to PostgreSQL through Dapper. Dapper is a micro-ORM — it maps SQL results to objects without the overhead of Entity Framework.

```csharp
// Movies.Application/Repositories/MovieRepository.cs
using Dapper;

public class MovieRepository : IMovieRepository
{
    private readonly IDatabaseConnectionFactory _connectionFactory;

    public MovieRepository(IDatabaseConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task<bool> CreateAsync(Movie movie)
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        using var transaction = connection.BeginTransaction();

        var result = await connection.ExecuteAsync(
            new CommandDefinition("""
                INSERT INTO movies (id, slug, title, year)
                VALUES (@Id, @Slug, @Title, @Year)
            """, movie, transaction));

        if (result > 0)
        {
            foreach (var genre in movie.Genre)
            {
                await connection.ExecuteAsync(
                    new CommandDefinition("""
                        INSERT INTO genres (movieId, name)
                        VALUES (@MovieId, @Name)
                    """, new { MovieId = movie.Id, Name = genre }, transaction));
            }
        }

        transaction.Commit();
        return result > 0;
    }

    public async Task<Movie?> GetByIdAsync(Guid id)
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        var movie = await connection.QuerySingleOrDefaultAsync<Movie>(
            new CommandDefinition("""
                SELECT m.*, string_agg(g.name, ',') AS genre
                FROM movies m
                LEFT JOIN genres g ON m.id = g.movieId
                WHERE id = @Id
                GROUP BY m.id
            """, new { Id = id }));

        return movie;
    }

    public async Task<Movie?> GetBySlugAsync(string slug)
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        var movie = await connection.QuerySingleOrDefaultAsync<Movie>(
            new CommandDefinition("""
                SELECT m.*, string_agg(g.name, ',') AS genre
                FROM movies m
                LEFT JOIN genres g ON m.id = g.movieId
                WHERE slug = @Slug
                GROUP BY m.id
            """, new { Slug = slug }));

        return movie;
    }

    public async Task<IEnumerable<Movie>> GetAllAsync()
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        var movies = await connection.QueryAsync(
            new CommandDefinition("""
                SELECT m.*, string_agg(g.name, ',') AS genre
                FROM movies m
                LEFT JOIN genres g ON m.id = g.movieId
                GROUP BY m.id
            """));

        return movies.Select(m => new Movie
        {
            Id = m.id,
            Title = m.title,
            Year = m.year,
            Genre = ((string)m.genre)?.Split(',').ToList() ?? new List<string>()
        });
    }

    public async Task<bool> UpdateAsync(Movie movie)
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        using var transaction = connection.BeginTransaction();

        // Delete existing genres
        await connection.ExecuteAsync(
            new CommandDefinition("""
                DELETE FROM genres WHERE movieId = @Id
            """, new { Id = movie.Id }, transaction));

        // Update the movie
        var result = await connection.ExecuteAsync(
            new CommandDefinition("""
                UPDATE movies SET slug = @Slug, title = @Title, year = @Year
                WHERE id = @Id
            """, movie, transaction));

        if (result > 0)
        {
            foreach (var genre in movie.Genre)
            {
                await connection.ExecuteAsync(
                    new CommandDefinition("""
                        INSERT INTO genres (movieId, name)
                        VALUES (@MovieId, @Name)
                    """, new { MovieId = movie.Id, Name = genre }, transaction));
            }
        }

        transaction.Commit();
        return result > 0;
    }

    public async Task<bool> DeleteAsync(Guid id)
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        using var transaction = connection.BeginTransaction();

        await connection.ExecuteAsync(
            new CommandDefinition("""
                DELETE FROM genres WHERE movieId = @Id
            """, new { Id = id }, transaction));

        var result = await connection.ExecuteAsync(
            new CommandDefinition("""
                DELETE FROM movies WHERE id = @Id
            """, new { Id = id }, transaction));

        transaction.Commit();
        return result > 0;
    }

    public async Task<bool> ExistsByIdAsync(Guid id)
    {
        using var connection = await _connectionFactory.CreateConnectionAsync();
        return await connection.ExecuteScalarAsync<bool>(
            new CommandDefinition("""
                SELECT COUNT(1) FROM movies WHERE id = @Id
            """, new { Id = id }));
    }
}
```

Important patterns in this repository:

- **Transactions** — Create, Update, and Delete all modify two tables (movies and genres). Wrapping them in a transaction ensures atomicity: either both tables are updated or neither is.
- **`string_agg`** — PostgreSQL function that concatenates genre names into a comma-separated string. We split it back into a list on the C# side.
- **`CommandDefinition`** — Dapper's way of bundling SQL, parameters, and transaction into a single object. This becomes important when we add cancellation tokens in Part 3.
- **`ExistsByIdAsync`** — A lightweight existence check that we will use for validation in Part 3.

### Update the DI Registration

```csharp
// Movies.Application/ServiceExtensions.cs
public static class ServiceExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddSingleton<IMovieRepository, MovieRepository>();
        return services;
    }

    public static IServiceCollection AddDatabase(this IServiceCollection services,
        string connectionString)
    {
        services.AddSingleton<IDatabaseConnectionFactory>(
            _ => new NpgSqlConnectionFactory(connectionString));
        services.AddSingleton<DatabaseMigration>();
        return services;
    }
}
```

And in `Program.cs`:

```csharp
var connectionString = builder.Configuration["Database:ConnectionString"]!;
builder.Services.AddApplication();
builder.Services.AddDatabase(connectionString);
```

## Summary

We have moved from an in-memory prototype to a real database-backed API:

- **Slugs** give users human-readable URLs alongside GUIDs
- **Docker Compose** provides a one-command PostgreSQL setup
- **Dapper** keeps SQL explicit and visible — no magic, no hidden queries
- **Transactions** ensure data consistency across the movies and genres tables
- **Database migrations** run automatically on startup

## What's Next?

In [Part 3: Business Logic and Validation](/posts/dotnet-rest-api-part3-validation/), we will introduce a service layer between the controller and repository, add FluentValidation for request validation (including async slug uniqueness checks), build validation middleware, and thread cancellation tokens through every layer.

## References

- [Dapper documentation](https://github.com/DapperLib/Dapper)
- [PostgreSQL Docker image](https://hub.docker.com/_/postgres)
- [Npgsql — .NET PostgreSQL driver](https://www.npgsql.org/)
- [PostgreSQL string_agg](https://www.postgresql.org/docs/current/functions-aggregate.html)
