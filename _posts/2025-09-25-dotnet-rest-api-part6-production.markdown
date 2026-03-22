---
layout: post
title: "Building a REST API with .NET — Part 6: Production Readiness"
date: 2025-09-25 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - rest-api
  - csharp
  - versioning
  - swagger
  - health-checks
  - caching
  - api-key
series: "Building a REST API with .NET"
series_part: 6
---


## Series Overview

This is a 6-part series on building a production-ready REST API with .NET:

1. [Project Setup, Contracts, and Controllers](/posts/dotnet-rest-api-part1-foundations/) — Solution structure, contracts, repository pattern, controllers, and mapping
2. [Database Integration with Dapper](/posts/dotnet-rest-api-part2-database/) — PostgreSQL with Docker, Dapper ORM, migrations, and slugs
3. [Business Logic and Validation](/posts/dotnet-rest-api-part3-validation/) — Service layer, FluentValidation, middleware, and cancellation tokens
4. [Authentication and Authorization](/posts/dotnet-rest-api-part4-auth/) — JWT tokens, claims-based authorization, and user identity
5. [Filtering, Sorting, and Pagination](/posts/dotnet-rest-api-part5-features/) — Query parameters, dynamic sorting, paginated responses
6. **Production Readiness** (this article) — Versioning, Swagger/OpenAPI, health checks, caching, and API key auth


## Introduction

Over the previous five parts we built a functional, secure, and queryable REST API. But "it works" and "it's production-ready" are different things. In this final part we add the features that separate a demo from a deployable service: API versioning, interactive documentation, health monitoring, performance caching, and service-to-service authentication.

## API Versioning

APIs evolve. Breaking changes are inevitable. Versioning lets you introduce changes without breaking existing clients.

### Install the Package

```bash
dotnet add Movies.API package Asp.Versioning.Mvc
```

### Configure Versioning

We use **media type versioning** — the version is specified in the `Accept` header rather than the URL. This keeps URLs clean and follows REST principles more closely.

```csharp
// Program.cs
builder.Services.AddApiVersioning(x =>
{
    x.DefaultApiVersion = new ApiVersion(1, 0);
    x.AssumeDefaultVersionWhenUnspecified = true;
    x.ReportApiVersions = true;
    x.ApiVersionReader = new MediaTypeApiVersionReader("api-version");
});
```

| Setting | Purpose |
|---|---|
| `DefaultApiVersion` | Used when no version is specified |
| `AssumeDefaultVersionWhenUnspecified` | Do not reject requests without a version |
| `ReportApiVersions` | Include `api-supported-versions` in response headers |
| `MediaTypeApiVersionReader` | Read version from the Accept header |

### Apply to the Controller

```csharp
[ApiController]
[ApiVersion(1.0)]
public class MoviesController : ControllerBase
{
    // ...
}
```

### Client Usage

Clients specify the version in the `Accept` header:

```http
GET /api/movies
Accept: application/json;api-version=1.0
```

When you need to introduce breaking changes, create a v2 controller and clients can migrate at their own pace.

## Swagger / OpenAPI Documentation

Swagger provides interactive API documentation that developers can use to explore and test your endpoints.

### Configure Swagger with Versioning

```csharp
// Program.cs
builder.Services.AddSwaggerGen(x =>
{
    x.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Movies API",
        Version = "v1.0",
        Description = "A REST API for managing movies, built with .NET"
    });
});
```

### Add JWT Authentication to Swagger

Without this, Swagger cannot send the `Authorization` header when testing protected endpoints:

```csharp
builder.Services.AddSwaggerGen(x =>
{
    x.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Movies API",
        Version = "v1.0"
    });

    x.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        In = ParameterLocation.Header,
        Description = "Enter your JWT token",
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        BearerFormat = "JWT",
        Scheme = "bearer"
    });

    x.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

### Enable the Middleware

```csharp
app.UseSwagger();
app.UseSwaggerUI(x =>
{
    x.SwaggerEndpoint("/swagger/v1/swagger.json", "Movies API v1");
});
```

Now navigate to `https://localhost:5001/swagger` to see the interactive documentation. You can click the "Authorize" button, paste a JWT, and test protected endpoints directly from the browser.

## Health Checks

Health checks let load balancers, orchestrators (Kubernetes), and monitoring tools know if your service is functioning correctly. A healthy service gets traffic; an unhealthy one gets pulled from rotation.

### Basic Health Check Endpoint

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database");

app.MapHealthChecks("_health");
```

We use an underscore prefix (`_health`) as a convention to indicate this is an infrastructure endpoint, not a business endpoint.

### Custom Database Health Check

```csharp
// Movies.API/Health/DatabaseHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDatabaseConnectionFactory _connectionFactory;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(IDatabaseConnectionFactory connectionFactory,
        ILogger<DatabaseHealthCheck> logger)
    {
        _connectionFactory = connectionFactory;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var connection = await _connectionFactory
                .CreateConnectionAsync(cancellationToken);

            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            const string errorMessage = "Database health check failed";
            _logger.LogError(ex, errorMessage);
            return HealthCheckResult.Unhealthy(errorMessage, ex);
        }
    }
}
```

The three possible states:

| State | Meaning |
|---|---|
| **Healthy** | Everything is working |
| **Degraded** | Working, but with reduced functionality or performance |
| **Unhealthy** | Not functioning — should be removed from load balancer |

### Health Check Response

```http
GET https://localhost:5001/_health

HTTP/1.1 200 OK
Content-Type: text/plain

Healthy
```

For more detailed output (useful in staging/development), configure the response writer:

```csharp
app.MapHealthChecks("_health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description
            })
        };
        await context.Response.WriteAsJsonAsync(result);
    }
});
```

## Caching

Caching reduces database load and improves response times. .NET offers two caching mechanisms:

### Output Caching vs Response Caching

| Feature | Output Caching | Response Caching |
|---|---|---|
| Where | Server-side | Client-side (browser, CDN) |
| Control | Full server control | Depends on client honoring headers |
| Invalidation | Tag-based, programmatic | Time-based only |
| Storage | Server memory | Client storage |

For an API, **output caching** is usually the better choice because you have full control over invalidation.

### Configure Output Caching

```csharp
// Program.cs
builder.Services.AddOutputCache(x =>
{
    x.AddBasePolicy(c => c.Cache());

    x.AddPolicy("MovieCache", c =>
        c.Cache()
         .Expire(TimeSpan.FromMinutes(1))
         .SetVaryByQuery(new[] { "title", "year", "sortBy", "page", "pageSize" })
         .Tag("movies"));
});

app.UseOutputCache();
```

The policy caches responses for 1 minute, varies the cache by query parameters (so different filters get different cache entries), and tags entries with "movies" for targeted invalidation.

### Apply to Endpoints

```csharp
[AllowAnonymous]
[OutputCache(PolicyName = "MovieCache")]
[HttpGet(ApiEndpoints.Movies.GetAll)]
public async Task<IActionResult> GetAll([FromQuery] GetAllMoviesRequest request,
    CancellationToken token)
{
    // ...
}
```

### Cache Invalidation

When a movie is created, updated, or deleted, we need to invalidate the cache so stale data is not returned:

```csharp
[ApiController]
[Authorize]
public class MoviesController : ControllerBase
{
    private readonly IMovieService _movieService;
    private readonly IOutputCacheStore _outputCacheStore;

    public MoviesController(IMovieService movieService,
        IOutputCacheStore outputCacheStore)
    {
        _movieService = movieService;
        _outputCacheStore = outputCacheStore;
    }

    [Authorize("Admin")]
    [HttpPost(ApiEndpoints.Movies.Create)]
    public async Task<IActionResult> Create([FromBody] CreateMovieRequest request,
        CancellationToken token)
    {
        var movie = request.ToMovie();
        await _movieService.CreateAsync(movie, token);

        // Invalidate the movies cache
        await _outputCacheStore.EvictByTagAsync("movies", token);

        var response = movie.ToMovieResponse();
        return CreatedAtAction(nameof(Get), new { identity = movie.Id }, response);
    }

    [Authorize("Admin")]
    [HttpPut(ApiEndpoints.Movies.Update)]
    public async Task<IActionResult> Update([FromRoute] Guid id,
        [FromBody] UpdateMovieRequest request, CancellationToken token)
    {
        var movie = request.ToMovie(id);
        var updatedMovie = await _movieService.UpdateAsync(movie, token);
        if (updatedMovie is null)
            return NotFound();

        await _outputCacheStore.EvictByTagAsync("movies", token);

        return Ok(updatedMovie.ToMovieResponse());
    }

    [Authorize("Admin")]
    [HttpDelete(ApiEndpoints.Movies.Delete)]
    public async Task<IActionResult> Delete([FromRoute] Guid id,
        CancellationToken token)
    {
        var deleted = await _movieService.DeleteAsync(id, token);
        if (!deleted)
            return NotFound();

        await _outputCacheStore.EvictByTagAsync("movies", token);

        return Ok();
    }
}
```

The tag-based eviction (`EvictByTagAsync("movies", ...)`) invalidates all cache entries tagged with "movies", regardless of which query parameters they were cached with.

## API Key Authentication

JWT works well for user-facing authentication. But for service-to-service communication — where another backend calls your API — API keys are simpler and more appropriate.

### API Key Auth Filter

```csharp
// Movies.API/Auth/ApiKeyAuthFilter.cs
using Microsoft.AspNetCore.Mvc.Filters;

public class ApiKeyAuthFilter : IAuthorizationFilter
{
    private readonly IConfiguration _configuration;

    public ApiKeyAuthFilter(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void OnAuthorization(AuthorizationFilterContext context)
    {
        if (!context.HttpContext.Request.Headers
            .TryGetValue("x-api-key", out var extractedApiKey))
        {
            context.Result = new UnauthorizedObjectResult("API key is missing");
            return;
        }

        var apiKey = _configuration["ApiKey"]!;
        if (!apiKey.Equals(extractedApiKey))
        {
            context.Result = new UnauthorizedObjectResult("Invalid API key");
            return;
        }
    }
}
```

Add the API key to `appsettings.json`:

```json
{
  "ApiKey": "your-secret-api-key-here"
}
```

### Using the Filter

Apply it to specific endpoints or controllers:

```csharp
[ServiceFilter(typeof(ApiKeyAuthFilter))]
[HttpPost("api/admin/seed")]
public async Task<IActionResult> SeedDatabase()
{
    // Only callable with a valid API key
}
```

Register the filter in DI:

```csharp
builder.Services.AddScoped<ApiKeyAuthFilter>();
```

### Combining JWT and API Key Authentication

For endpoints that should accept either JWT or API key, create a custom authorization requirement:

```csharp
// Movies.API/Auth/AdminAuthRequirement.cs
using Microsoft.AspNetCore.Authorization;

public class AdminAuthRequirement : IAuthorizationHandler, IAuthorizationRequirement
{
    private readonly IConfiguration _configuration;

    public AdminAuthRequirement(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public Task HandleAsync(AuthorizationHandlerContext context)
    {
        // Check for API key first
        if (context.Resource is HttpContext httpContext)
        {
            if (httpContext.Request.Headers.TryGetValue("x-api-key", out var extractedApiKey))
            {
                var apiKey = _configuration["ApiKey"]!;
                if (apiKey.Equals(extractedApiKey))
                {
                    context.Succeed(this);
                    return Task.CompletedTask;
                }
            }
        }

        // Fall back to JWT admin claim
        if (context.User.HasClaim("admin", "true"))
        {
            context.Succeed(this);
            return Task.CompletedTask;
        }

        context.Fail();
        return Task.CompletedTask;
    }
}
```

Register and use as a policy:

```csharp
// Program.cs
builder.Services.AddSingleton<IAuthorizationHandler, AdminAuthRequirement>();

builder.Services.AddAuthorization(x =>
{
    x.AddPolicy("Admin", policy =>
        policy.AddRequirements(new AdminAuthRequirement(builder.Configuration)));
});
```

Now the `[Authorize("Admin")]` policy accepts either a JWT with an admin claim or a valid API key in the `x-api-key` header.

## Series Recap

Over six parts, we built a complete, production-ready REST API:

**Part 1 — Foundations**
We established a clean three-project solution structure (API, Application, Contracts), defined request/response contracts, created a domain model, implemented the repository pattern with an in-memory store, and built a CRUD controller with proper HTTP semantics.

**Part 2 — Database**
We replaced the in-memory store with PostgreSQL via Docker and Dapper. We added human-readable slugs, wrote database migrations, and used transactions to maintain data consistency across multiple tables.

**Part 3 — Validation**
We extracted a service layer to separate business logic from HTTP and persistence concerns. FluentValidation gave us declarative, testable validation including async database checks. Validation middleware ensured consistent error responses, and cancellation tokens enabled proper resource cleanup.

**Part 4 — Authentication**
JWT-based authentication proved user identity without database lookups. Claims-based authorization policies controlled access at the endpoint level, distinguishing between public, authenticated, and admin operations.

**Part 5 — Query Features**
Filtering, sorting, and pagination made the API practical for real-world use. Parameterized SQL kept queries safe, validated sort fields prevented injection, and pagination metadata helped clients navigate large datasets.

**Part 6 — Production**
API versioning enabled backward-compatible evolution. Swagger provided interactive documentation. Health checks enabled infrastructure monitoring. Output caching improved performance with tag-based invalidation. API key authentication supported service-to-service communication.

### Architecture Summary

```
Client Request
    ↓
[Middleware] — Validation, Error Handling
    ↓
[Controller] — HTTP concerns, routing, status codes
    ↓
[Service] — Business logic, validation, authorization
    ↓
[Repository] — Data access (Dapper + PostgreSQL)
    ↓
Database
```

Each layer has a single responsibility. Swapping PostgreSQL for SQL Server means changing only the repository. Adding a gRPC endpoint means adding a new host project that reuses the same services. The architecture is flexible because the boundaries are clean.

## References

- [Asp.Versioning.Mvc](https://github.com/dotnet/aspnet-api-versioning)
- [Swashbuckle — Swagger for ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle)
- [Health checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [Output caching in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output)
- [ASP.NET Core Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/)
