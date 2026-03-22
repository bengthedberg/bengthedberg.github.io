---
layout: post
title: "Building a REST API with .NET — Part 4: Authentication and Authorization"
date: 2025-09-11 00:00:00 +0000
tags:
  - dotnet
  - rest-api
  - csharp
  - jwt
  - authentication
  - authorization
series: "Building a REST API with .NET"
series_part: 4
---


## Series Overview

This is a 6-part series on building a production-ready REST API with .NET:

1. [Project Setup, Contracts, and Controllers](2025-08-dotnet-rest-api-part1-foundations.md) — Solution structure, contracts, repository pattern, controllers, and mapping
2. [Database Integration with Dapper](2025-08-dotnet-rest-api-part2-database.md) — PostgreSQL with Docker, Dapper ORM, migrations, and slugs
3. [Business Logic and Validation](2025-09-dotnet-rest-api-part3-validation.md) — Service layer, FluentValidation, middleware, and cancellation tokens
4. **Authentication and Authorization** (this article) — JWT tokens, claims-based authorization, and user identity
5. [Filtering, Sorting, and Pagination](2025-09-dotnet-rest-api-part5-features.md) — Query parameters, dynamic sorting, paginated responses
6. [Production Readiness](2025-09-dotnet-rest-api-part6-production.md) — Versioning, Swagger/OpenAPI, health checks, caching, and API key auth


## Introduction

In [Part 3](2025-09-dotnet-rest-api-part3-validation.md) we added validation and a service layer. But right now, anyone can create, update, or delete movies. We need to control who can access what. In this part we add JWT-based authentication (proving who you are) and claims-based authorization (controlling what you can do).

## JWT Concepts

A JSON Web Token (JWT) is a compact, URL-safe way to represent claims between two parties. It consists of three parts, separated by dots:

```
header.payload.signature
```

### Header

Describes the token type and signing algorithm:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

Contains **claims** — statements about the user:

```json
{
  "sub": "user-id-123",
  "name": "Bengt",
  "admin": "true",
  "exp": 1694476800
}
```

Claims can be standard (like `sub` for subject, `exp` for expiration) or custom (like `admin`).

### Signature

Created by combining the encoded header and payload with a secret key. This proves the token has not been tampered with:

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

### Bearer Scheme

JWTs are sent in the `Authorization` header using the Bearer scheme:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

The server validates the signature, checks expiration, and extracts claims — all without hitting a database.

## Adding JWT Authentication

### Install the Package

```bash
dotnet add Movies.API package Microsoft.AspNetCore.Authentication.JwtBearer
```

### Configuration

Add JWT settings to `appsettings.json`:

```json
{
  "Jwt": {
    "Secret": "YourSuperSecretKeyThatIsAtLeast32CharactersLong!",
    "Issuer": "https://movies.api",
    "Audience": "https://movies.api"
  }
}
```

> **Important:** In production, store the secret in a vault (Azure Key Vault, AWS Secrets Manager) or environment variables — never in source control.

### Configure Authentication in Program.cs

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Add authentication
builder.Services.AddAuthentication(x =>
{
    x.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    x.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    x.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(x =>
{
    x.TokenValidationParameters = new TokenValidationParameters
    {
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)),
        ValidateIssuerSigningKey = true,
        ValidateLifetime = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidAudience = builder.Configuration["Jwt:Audience"],
        ValidateIssuer = true,
        ValidateAudience = true
    };
});
```

Each validation parameter serves a purpose:

| Parameter | Purpose |
|---|---|
| `ValidateIssuerSigningKey` | Verifies the token was signed with our secret |
| `ValidateLifetime` | Rejects expired tokens |
| `ValidateIssuer` | Ensures the token was issued by our API |
| `ValidateAudience` | Ensures the token is intended for our API |

### Add Middleware

Order matters — authentication must come before authorization:

```csharp
var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

### Protect the Controller

Add `[Authorize]` at the controller level to require authentication for all endpoints, and `[AllowAnonymous]` on endpoints that should be publicly accessible:

```csharp
[ApiController]
[Authorize]
public class MoviesController : ControllerBase
{
    private readonly IMovieService _movieService;

    public MoviesController(IMovieService movieService)
    {
        _movieService = movieService;
    }

    [AllowAnonymous]
    [HttpGet(ApiEndpoints.Movies.Get)]
    public async Task<IActionResult> Get([FromRoute] string identity,
        CancellationToken token)
    {
        var movie = Guid.TryParse(identity, out var id)
            ? await _movieService.GetByIdAsync(id, token)
            : await _movieService.GetBySlugAsync(identity, token);

        if (movie is null)
            return NotFound();

        return Ok(movie.ToMovieResponse());
    }

    [AllowAnonymous]
    [HttpGet(ApiEndpoints.Movies.GetAll)]
    public async Task<IActionResult> GetAll(CancellationToken token)
    {
        var movies = await _movieService.GetAllAsync(token);
        return Ok(movies.ToMoviesResponse());
    }

    // Create, Update, Delete remain protected by [Authorize]
    // ...
}
```

With this setup:
- **GET** endpoints are public — anyone can browse movies
- **POST, PUT, DELETE** require a valid JWT — only authenticated users can modify data

If an unauthenticated request hits a protected endpoint, ASP.NET Core returns **401 Unauthorized**.

## Claims-Based Authorization

Authentication tells us *who* the user is. Authorization tells us *what* they can do. We use claims-based policies for fine-grained access control.

### Define Authorization Policies

In `Program.cs`, configure policies after authentication:

```csharp
builder.Services.AddAuthorization(x =>
{
    x.AddPolicy("Admin", policy =>
        policy.RequireClaim("admin", "true"));
});
```

This creates an "Admin" policy that requires the JWT to contain an `admin` claim with the value `"true"`.

### Apply the Policy

Restrict write operations to admins:

```csharp
[Authorize("Admin")]
[HttpPost(ApiEndpoints.Movies.Create)]
public async Task<IActionResult> Create([FromBody] CreateMovieRequest request,
    CancellationToken token)
{
    var movie = request.ToMovie();
    await _movieService.CreateAsync(movie, token);
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

    return Ok();
}
```

Now:
- An authenticated user without the `admin` claim gets **403 Forbidden** on write endpoints
- An unauthenticated user gets **401 Unauthorized**
- Anyone can read movies

## Extracting User Identity

For user-specific features — like movie ratings — we need to know which user is making the request. The user's identity is encoded in the JWT claims.

### Getting the User ID from Claims

```csharp
[HttpPut("api/movies/{id:guid}/rating")]
[Authorize]
public async Task<IActionResult> RateMovie([FromRoute] Guid id,
    [FromBody] RateMovieRequest request, CancellationToken token)
{
    var userId = HttpContext.User.Claims
        .SingleOrDefault(x => x.Type == "userid");

    if (userId is null)
        return Unauthorized();

    // Pass userId.Value to the service
    var result = await _ratingService.RateMovieAsync(
        id, Guid.Parse(userId.Value), request.Rating, token);

    if (!result)
        return NotFound();

    return Ok();
}
```

The `HttpContext.User` property is populated automatically by the JWT middleware. You can extract any claim from the token payload.

### Passing User Context to the Service Layer

For a cleaner approach, create an extension method or use `IHttpContextAccessor`:

```csharp
// Extension method approach
public static Guid? GetUserId(this HttpContext context)
{
    var userId = context.User.Claims
        .SingleOrDefault(x => x.Type == "userid");

    if (userId is null)
        return null;

    return Guid.Parse(userId.Value);
}
```

Usage:

```csharp
var userId = HttpContext.GetUserId();
```

## Generating Tokens for Testing

For testing, you will need a way to generate JWTs. A minimal token generator:

```csharp
// Identity.API/Controllers/TokenController.cs (separate project)
[HttpPost("token")]
public IActionResult GenerateToken([FromBody] TokenRequest request)
{
    // In production, validate credentials against a user store
    var tokenHandler = new JwtSecurityTokenHandler();
    var key = Encoding.UTF8.GetBytes(_config["Jwt:Secret"]!);

    var claims = new List<Claim>
    {
        new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        new(JwtRegisteredClaimNames.Sub, request.UserId),
        new(JwtRegisteredClaimNames.Email, request.Email),
        new("userid", request.UserId)
    };

    if (request.IsAdmin)
    {
        claims.Add(new Claim("admin", "true"));
    }

    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(claims),
        Expires = DateTime.UtcNow.AddHours(8),
        Issuer = _config["Jwt:Issuer"],
        Audience = _config["Jwt:Audience"],
        SigningCredentials = new SigningCredentials(
            new SymmetricSecurityKey(key),
            SecurityAlgorithms.HmacSha256Signature)
    };

    var token = tokenHandler.CreateToken(tokenDescriptor);
    return Ok(new { Token = tokenHandler.WriteToken(token) });
}
```

> **Tip:** In a real application, authentication would be handled by a dedicated identity service (e.g., IdentityServer, Auth0, Azure AD B2C). The Movies API only needs to validate tokens, not issue them.

### Testing with the HTTP File

```http
### Get a token (from your identity service)
POST https://localhost:5002/token
Content-Type: application/json

{
  "userId": "d8a1e7c0-4f3b-4e5a-9c2d-1a2b3c4d5e6f",
  "email": "admin@example.com",
  "isAdmin": true
}

### Create a movie (requires admin token)
POST https://localhost:5001/api/movies
Content-Type: application/json
Authorization: Bearer {{token}}

{
  "title": "Inception",
  "year": 2010,
  "genre": ["Action", "Sci-Fi", "Thriller"]
}

### Get all movies (no auth required)
GET https://localhost:5001/api/movies
```

## Summary

We have added a complete authentication and authorization layer:

- **JWT authentication** validates tokens without database lookups
- **`[Authorize]`** and **`[AllowAnonymous]`** control access at the endpoint level
- **Claims-based policies** provide fine-grained authorization (admin vs regular user)
- **User identity extraction** enables user-specific features

The key insight is the separation: authentication (proving identity) and authorization (granting permissions) are distinct concerns, and ASP.NET Core's middleware pipeline handles them elegantly.

## What's Next?

In [Part 5: Filtering, Sorting, and Pagination](2025-09-dotnet-rest-api-part5-features.md), we will add query parameters for filtering movies by title and year, dynamic sorting by any field, and paginated responses with metadata.

## References

- [ASP.NET Core Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/)
- [JWT.io — JSON Web Token debugger](https://jwt.io/)
- [Claims-based authorization in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/claims)
- [Policy-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)
