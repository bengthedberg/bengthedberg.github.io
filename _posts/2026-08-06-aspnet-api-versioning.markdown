---
layout: post
title: "API Versioning in ASP.NET Core: A Complete Guide"
date: 2026-08-06 00:00:00 +0000
tags:
  - dotnet
  - api-versioning
  - aspnet
  - rest-api
---


# API Versioning in ASP.NET Core: A Complete Guide

When your API serves multiple clients -- mobile apps, third-party integrations, partner systems -- breaking changes become expensive. Renaming a field, changing a response structure, or altering behavior can break every consumer simultaneously.

API versioning solves this by letting your API evolve independently from its clients. Instead of forcing a breaking change, you introduce a new version while keeping the old one running.

In this guide, we will cover:

- Why API versioning matters
- Setting up versioning in ASP.NET Core
- Different versioning strategies (URL, header, query string)
- Versioning controllers and minimal APIs
- Deprecating old versions gracefully

## Why You Need API Versioning

A breaking change is any modification that can cause existing clients to fail. Common examples include:

- Removing or renaming API endpoints or parameters
- Changing the behavior of existing APIs
- Modifying the response contract (adding required fields, removing fields)
- Changing error codes or error response formats

What counts as a breaking change depends on your team's API design guidelines. For instance, adding an optional field to a response may or may not be breaking depending on how your clients deserialize data.

The key insight is this: **plan for versioning from the start.** It is far easier to add versioning to a new API than to retrofit it later when clients are already depending on unversioned endpoints.

## Setting Up API Versioning

You need three NuGet packages, depending on whether you use controllers, minimal APIs, or both:

```powershell
Install-Package Asp.Versioning.Http              # For Minimal APIs
Install-Package Asp.Versioning.Mvc               # For Controllers
Install-Package Asp.Versioning.Mvc.ApiExplorer    # For Swagger integration
```

Configure versioning in your service registration:

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.ReportApiVersions = true;
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
})
.AddMvc() // Required for controllers
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'V";
    options.SubstituteApiVersionInUrl = true;
});
```

Here is what each option does:

| Option | Purpose |
|--------|---------|
| `DefaultApiVersion` | The version used when no version is specified (typically v1) |
| `ReportApiVersions` | Adds `api-supported-versions` to response headers |
| `AssumeDefaultVersionWhenUnspecified` | Falls back to the default version for unversioned requests |
| `ApiVersionReader` | Defines how the version is read from the request |

The `AddApiExplorer` configuration is especially useful if you use Swagger -- it substitutes the version route parameter in the generated documentation.

## Versioning Strategies

The `Asp.Versioning.Http` library supports several strategies through `IApiVersionReader` implementations:

### URL Versioning (Recommended)

```
GET https://api.example.com/api/v1/workouts
GET https://api.example.com/api/v2/workouts
```

This is the most explicit and discoverable approach. The version is right there in the URL -- no ambiguity, no hidden headers. This is what [Microsoft's API guidelines](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#12-versioning) recommend.

### Header Versioning

```
GET https://api.example.com/api/workouts
X-Api-Version: 1
```

Keeps URLs clean but hides the version from casual inspection. Harder to test in a browser.

### Query String Versioning

```
GET https://api.example.com/api/workouts?api-version=1
```

Easy to use but clutters the query string, which should ideally be reserved for filtering and pagination.

You can combine multiple readers using `ApiVersionReader.Combine()` to support more than one strategy simultaneously.

## Versioning Controllers

Decorate your controller with `ApiVersion` attributes to declare supported versions, and use `MapToApiVersion` on individual actions:

```csharp
[ApiVersion(1)]
[ApiVersion(2)]
[ApiController]
[Route("api/v{v:apiVersion}/workouts")]
public class WorkoutsController : ControllerBase
{
    [MapToApiVersion(1)]
    [HttpGet("{workoutId}")]
    public IActionResult GetWorkoutV1(Guid workoutId)
    {
        return Ok(new GetWorkoutByIdQuery(workoutId).Handle());
    }

    [MapToApiVersion(2)]
    [HttpGet("{workoutId}")]
    public IActionResult GetWorkoutV2(Guid workoutId)
    {
        return Ok(new GetWorkoutByIdQuery(workoutId).Handle());
    }
}
```

The route parameter `v{v:apiVersion}` enables URL-based versioning with paths like `/api/v1/workouts/123` and `/api/v2/workouts/123`.

## Versioning Minimal APIs

For minimal APIs, you define an `ApiVersionSet` and attach it to your endpoints:

```csharp
ApiVersionSet apiVersionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1))
    .HasApiVersion(new ApiVersion(2))
    .ReportApiVersions()
    .Build();

app.MapGet("api/v{version:apiVersion}/workouts/{workoutId}", async (
    Guid workoutId,
    ISender sender,
    CancellationToken ct) =>
{
    var query = new GetWorkoutByIdQuery(workoutId);
    Result<WorkoutResponse> result = await sender.Send(query, ct);
    return result.Match(Results.Ok, CustomResults.Problem);
})
.WithApiVersionSet(apiVersionSet)
.MapToApiVersion(1);
```

Specifying the version set on every endpoint gets repetitive. Use **route groups** to set it once:

```csharp
ApiVersionSet apiVersionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1))
    .ReportApiVersions()
    .Build();

RouteGroupBuilder group = app
    .MapGroup("api/v{version:apiVersion}")
    .WithApiVersionSet(apiVersionSet);

group.MapGet("workouts", ...);
group.MapGet("workouts/{workoutId}", ...);
```

## Deprecating API Versions

When you want to phase out an old version, set the `Deprecated` property. The version remains functional but clients receive a `api-deprecated-versions` header warning them to migrate:

```csharp
[ApiVersion(1, Deprecated = true)]
[ApiVersion(2)]
[ApiController]
[Route("api/v{v:apiVersion}/workouts")]
public class WorkoutsController : ControllerBase
{
}
```

This gives consumers a clear signal without breaking their integration immediately.

## Key Takeaways

- Implement API versioning from day one -- retrofitting is painful
- URL versioning is the simplest and most explicit strategy
- Agree as a team on what constitutes a breaking change and document it
- Use deprecation headers to signal upcoming version sunsets
- Route groups in minimal APIs keep versioning configuration DRY

## References

- [Asp.Versioning NuGet Packages](https://www.nuget.org/packages?q=Asp.Versioning)
- [Microsoft REST API Guidelines - Versioning](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#12-versioning)
- [ASP.NET API Versioning Documentation](https://github.com/dotnet/aspnet-api-versioning)
- [API Versioning in ASP.NET Core (Microsoft Learn)](https://learn.microsoft.com/en-us/aspnet/core/web-api/versioning)
