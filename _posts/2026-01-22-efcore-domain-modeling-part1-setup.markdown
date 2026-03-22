---
layout: post
title: "Introduction to EF Core and Domain Modeling — Part 1: Setup and Your First DbContext"
date: 2026-01-22 00:00:00 +0000
tags:
  - efcore
  - dotnet
  - domain-modeling
  - entity-framework
  - csharp
series: "Introduction to EF Core and Domain Modeling"
series_part: 1
---


## Series Overview

This is a 4-part series on Entity Framework Core and domain modeling in .NET:

1. **Setup and Your First DbContext** (this article) — What is EF Core, project setup, DbContext, dependency injection, migrations
2. [One-to-One and One-to-Many Relationships](2026-02-efcore-domain-modeling-part2-relationships.md) — Modeling relationships between entities with the Fluent API
3. [Many-to-Many Relationships](2026-02-efcore-domain-modeling-part3-many-to-many.md) — Join tables, implicit and explicit approaches, unidirectional navigation
4. [Seed Data and Putting It All Together](2026-02-efcore-domain-modeling-part4-seed-data.md) — Populating your database, complete model example, next steps


## What Is EF Core?

Entity Framework Core (EF Core) is an **object-relational mapper (ORM)** for .NET. It lets you work with a database using C# objects instead of writing raw SQL. You define your domain model as plain C# classes, and EF Core handles translating between those objects and database tables.

Why use an ORM?

- **Productivity** — Write C# instead of SQL for most data access. LINQ queries are type-checked at compile time.
- **Abstraction** — Switch database providers (SQL Server, PostgreSQL, SQLite) with minimal code changes.
- **Migrations** — EF Core tracks schema changes in code and generates migration scripts automatically.
- **Domain focus** — Your C# classes represent your domain, not your database schema. EF Core bridges the gap.

EF Core is not a replacement for SQL knowledge. Complex queries, performance tuning, and database-specific features still benefit from understanding what happens at the SQL level. But for the majority of CRUD operations and domain modeling, EF Core removes significant boilerplate.

## Prerequisites

To follow along with this series, you need:

- **.NET SDK** (8.0 or later) — [Download from dot.net](https://dot.net)
- **A database** — SQL Server, PostgreSQL, or SQLite. We'll use SQL Server in examples, but the concepts apply to all providers.
- **An IDE** — Visual Studio, VS Code with the C# Dev Kit, or JetBrains Rider

### NuGet Packages

Install the core EF packages into your project:

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer   # or .Npgsql, .Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design      # for migrations tooling
```

You also need the EF Core CLI tools:

```bash
dotnet tool install --global dotnet-ef
```

## Creating Your First DbContext

The `DbContext` is the central class in EF Core. It represents a session with the database and provides:

- **`DbSet<T>` properties** — each one maps a C# class to a database table
- **Change tracking** — EF Core monitors modifications to your entities
- **Query translation** — LINQ queries on `DbSet<T>` are translated to SQL
- **Save operations** — `SaveChanges()` persists tracked changes to the database

Here is a minimal DbContext:

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs => Set<Blog>();
}
```

The constructor accepts a `DbContextOptions<ApplicationDbContext>` parameter. This is how configuration (connection string, provider, etc.) is passed in — typically through dependency injection.

## Your First Entity

Let's define a simple `Blog` entity:

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;
}
```

EF Core uses **conventions** to infer the database schema from your C# classes:

- A property named `Id` or `<TypeName>Id` (e.g. `BlogId`) is automatically configured as the **primary key**
- `string` properties map to `nvarchar(max)` in SQL Server (configurable with Fluent API or data annotations)
- The table name defaults to the `DbSet` property name (`Blogs`)

No configuration needed — EF Core discovers this entity through the `DbSet<Blog>` property on the DbContext.

## Registering with Dependency Injection

ASP.NET Core applications are configured using dependency injection. EF Core plugs into this with `AddDbContext` in `Program.cs`:

```csharp
var connectionString =
    builder.Configuration.GetConnectionString("DefaultConnection")
        ?? throw new InvalidOperationException("Connection string"
        + "'DefaultConnection' not found.");

builder.Services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(connectionString),
    ServiceLifetime.Scoped,
    ServiceLifetime.Singleton);
```

Add the connection string in `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=BlogDb;Trusted_Connection=True;"
  }
}
```

This registers `ApplicationDbContext` as a **scoped service** — a new instance is created for each HTTP request and disposed when the request ends. This aligns naturally with the **unit-of-work pattern**: each request gets its own DbContext, tracks changes, calls `SaveChanges()`, and the context is cleaned up.

### Using the DbContext

With DI registration in place, inject `ApplicationDbContext` into your controllers or services through constructor injection:

```csharp
public class BlogController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public BlogController(ApplicationDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<IActionResult> GetBlogs()
    {
        var blogs = await _context.Blogs.ToListAsync();
        return Ok(blogs);
    }
}
```

Each request gets its own `ApplicationDbContext` instance, performs a unit of work, and the context is disposed when the request ends.

## Migrations: Creating the Database

EF Core **migrations** track changes to your model and generate the SQL to update the database schema. The workflow is straightforward:

### 1. Create a migration

```bash
dotnet ef migrations add InitialCreate
```

This generates a migration file in a `Migrations/` folder. The file contains `Up()` and `Down()` methods — `Up` applies the change, `Down` reverts it.

### 2. Apply the migration

```bash
dotnet ef database update
```

This executes the pending migrations against your database, creating tables, columns, and constraints as defined by your model.

### 3. Iterate

As you change your entities (add properties, relationships, etc.), repeat the cycle:

```bash
dotnet ef migrations add AddBlogDescription
dotnet ef database update
```

EF Core compares the current model to the last migration snapshot and generates only the delta.

> **Tip**: Always review the generated migration code before applying it. EF Core is good at inferring intent, but destructive changes (dropping columns, renaming tables) deserve a human review.

## Conclusion

You now have a working EF Core setup: a DbContext registered with dependency injection, a simple entity mapped by convention, and migrations to manage your schema. This foundation is what everything else in the series builds on.

In [Part 2](2026-02-efcore-domain-modeling-part2-relationships.md), we'll add relationships between entities — one-to-many and one-to-one — and learn how to configure them with the Fluent API.

## References

- [EF Core Documentation](https://learn.microsoft.com/en-us/ef/core/)
- [DbContext Lifetime, Configuration, and Initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [Migrations Overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
