---
layout: post
title: "Introduction to EF Core and Domain Modeling — Part 4: Seed Data and Putting It All Together"
date: 2026-02-26 00:00:00 +0000
tags:
  - efcore
  - dotnet
  - domain-modeling
  - entity-framework
  - csharp
series: "Introduction to EF Core and Domain Modeling"
series_part: 4
---


## Series Overview

This is a 4-part series on Entity Framework Core and domain modeling in .NET:

1. [Setup and Your First DbContext](2026-01-efcore-domain-modeling-part1-setup.md) — What is EF Core, project setup, DbContext, dependency injection, migrations
2. [One-to-One and One-to-Many Relationships](2026-02-efcore-domain-modeling-part2-relationships.md) — Modeling relationships between entities with the Fluent API
3. [Many-to-Many Relationships](2026-02-efcore-domain-modeling-part3-many-to-many.md) — Join tables, implicit and explicit approaches, unidirectional navigation
4. **Seed Data and Putting It All Together** (this article) — Populating your database, complete model example, next steps


## Introduction

Over the previous three parts, we've built up a domain model with entities and relationships. But a schema without data is just structure. In this final part, we'll cover how to populate the database with initial data using EF Core's seed data feature, then bring the entire series together in a complete model example.

## Seed Data with HasData()

EF Core lets you associate seed data with entity types as part of the model configuration. When you add or update a migration, EF Core automatically computes the required insert, update, or delete operations to bring the data in line with your configuration.

The simplest example — seeding lookup data:

```csharp
modelBuilder.Entity<Country>(b =>
{
    b.Property(x => x.Name).IsRequired();
    b.HasData(
        new Country { CountryId = 1, Name = "USA" },
        new Country { CountryId = 2, Name = "Canada" },
        new Country { CountryId = 3, Name = "Mexico" });
});
```

Key rules for `HasData()`:

- **Primary keys must be specified explicitly** — even if they're auto-generated in normal use. EF Core uses the PK to determine whether a row should be inserted, updated, or deleted in future migrations.
- **Navigation properties are ignored** — you cannot set `post.Blog = myBlog`. Instead, set foreign key values directly.
- **Seed data is tied to migrations** — it's part of the schema management, not runtime logic. For large datasets or dynamic data, use a separate seeding service at application startup.

## Seeding Related Entities

When your entities have relationships, seed them by setting foreign key values — not navigation properties:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Seed blogs
    modelBuilder.Entity<Blog>().HasData(
        new Blog { Id = 1, Name = "Tech Blog", Url = "https://techblog.example.com" },
        new Blog { Id = 2, Name = "Travel Blog", Url = "https://travelblog.example.com" }
    );

    // Seed blog headers (one-to-one with Blog)
    modelBuilder.Entity<BlogHeader>().HasData(
        new BlogHeader { Id = 1, BlogId = 1, Title = "Welcome to Tech Blog", Subtitle = "All things code" },
        new BlogHeader { Id = 2, BlogId = 2, Title = "Adventures Await", Subtitle = "Stories from the road" }
    );

    // Seed posts (one-to-many with Blog)
    modelBuilder.Entity<Post>().HasData(
        new Post { Id = 1, BlogId = 1, Title = "Getting Started with EF Core" },
        new Post { Id = 2, BlogId = 1, Title = "Domain Modeling Tips" },
        new Post { Id = 3, BlogId = 2, Title = "Hiking in Norway" }
    );

    // Seed tags
    modelBuilder.Entity<Tag>().HasData(
        new Tag { Id = 1, Name = "efcore" },
        new Tag { Id = 2, Name = "dotnet" },
        new Tag { Id = 3, Name = "travel" }
    );
}
```

### Seeding Many-to-Many Join Data

For implicit many-to-many relationships (where EF Core manages the join table), you seed the join table using the anonymous type approach:

```csharp
// Seed the PostTag join table
modelBuilder.Entity<Post>()
    .HasMany(e => e.Tags)
    .WithMany(e => e.Posts)
    .UsingEntity(j => j.HasData(
        new { PostsId = 1, TagsId = 1 },  // "Getting Started with EF Core" + "efcore"
        new { PostsId = 1, TagsId = 2 },  // "Getting Started with EF Core" + "dotnet"
        new { PostsId = 2, TagsId = 1 },  // "Domain Modeling Tips" + "efcore"
        new { PostsId = 2, TagsId = 2 },  // "Domain Modeling Tips" + "dotnet"
        new { PostsId = 3, TagsId = 3 }   // "Hiking in Norway" + "travel"
    ));
```

The property names (`PostsId`, `TagsId`) must match the shadow property names EF Core generates for the join table. These are derived from the collection navigation property names.

## The Complete Domain Model

Let's bring together everything from this series into a single, complete example. This model includes all relationship types we've covered.

### Entities

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;

    // One-to-one: Blog has one BlogHeader
    public BlogHeader? Header { get; set; }

    // One-to-many: Blog has many Posts
    public ICollection<Post> Posts { get; } = new List<Post>();
}

public class BlogHeader
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Subtitle { get; set; } = string.Empty;

    // One-to-one: BlogHeader belongs to one Blog
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    // One-to-many: Post belongs to one Blog
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;

    // Many-to-many: Post has many Tags
    public List<Tag> Tags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Many-to-many: Tag has many Posts
    public List<Post> Posts { get; } = [];
}
```

### DbContext

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();
    public DbSet<Tag> Tags => Set<Tag>();
    public DbSet<BlogHeader> BlogHeaders => Set<BlogHeader>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // One-to-one: Blog -> BlogHeader
        modelBuilder.Entity<Blog>()
            .HasOne(e => e.Header)
            .WithOne(e => e.Blog)
            .HasForeignKey<BlogHeader>(e => e.BlogId)
            .IsRequired();

        // One-to-many: Blog -> Posts
        modelBuilder.Entity<Blog>()
            .HasMany(e => e.Posts)
            .WithOne(e => e.Blog)
            .HasForeignKey(e => e.BlogId)
            .IsRequired();

        // Many-to-many: Post <-> Tag (EF Core manages the join table)
        modelBuilder.Entity<Post>()
            .HasMany(e => e.Tags)
            .WithMany(e => e.Posts);

        // Seed data
        modelBuilder.Entity<Blog>().HasData(
            new Blog { Id = 1, Name = "Tech Blog", Url = "https://techblog.example.com" }
        );

        modelBuilder.Entity<BlogHeader>().HasData(
            new BlogHeader { Id = 1, BlogId = 1, Title = "Welcome", Subtitle = "All things code" }
        );

        modelBuilder.Entity<Post>().HasData(
            new Post { Id = 1, BlogId = 1, Title = "Getting Started with EF Core" },
            new Post { Id = 2, BlogId = 1, Title = "Domain Modeling Tips" }
        );

        modelBuilder.Entity<Tag>().HasData(
            new Tag { Id = 1, Name = "efcore" },
            new Tag { Id = 2, Name = "dotnet" }
        );

        modelBuilder.Entity<Post>()
            .HasMany(e => e.Tags)
            .WithMany(e => e.Posts)
            .UsingEntity(j => j.HasData(
                new { PostsId = 1, TagsId = 1 },
                new { PostsId = 1, TagsId = 2 },
                new { PostsId = 2, TagsId = 1 }
            ));
    }
}
```

### Generating the Migration

```bash
dotnet ef migrations add CompleteModel
dotnet ef database update
```

Review the generated migration to confirm EF Core created:

- A `Blogs` table with `Id`, `Name`, `Url`
- A `BlogHeaders` table with a required FK to `Blogs`
- A `Posts` table with a required FK to `Blogs`
- A `Tags` table
- A `PostTag` join table with a composite PK and two FKs
- Insert statements for all seed data

## What to Explore Next

This series covered the fundamentals of EF Core and domain modeling. Here are some topics to explore as your models grow more sophisticated:

- **Value objects and owned types** — model complex properties (e.g. `Address`) as part of an entity without giving them their own table
- **Table-per-hierarchy (TPH) inheritance** — map an inheritance hierarchy to a single table with a discriminator column
- **Global query filters** — automatically apply `WHERE` clauses (e.g. soft-delete, multi-tenancy) to every query
- **Compiled queries** — pre-compile frequently used LINQ queries for better performance
- **Raw SQL and SQL queries** — drop to raw SQL when LINQ isn't enough
- **Interceptors** — hook into EF Core's pipeline for auditing, logging, or modifying commands

## Conclusion

Over four articles, we've gone from an empty project to a complete domain model with one-to-one, one-to-many, and many-to-many relationships, all configured through the Fluent API and populated with seed data. EF Core handles the translation between your C# domain and the relational database — letting you focus on modelling the problem rather than writing SQL.

The key takeaways:

1. **Start with conventions** — EF Core discovers most relationships automatically
2. **Use the Fluent API** when conventions aren't enough — it gives you full control
3. **Choose the right relationship type** for your domain, not your database
4. **Seed data** keeps your development database consistent and your migrations reproducible

## References

- [Data Seeding](https://learn.microsoft.com/en-us/ef/core/modeling/data-seeding)
- [EF Core Documentation](https://learn.microsoft.com/en-us/ef/core/)
- [Migrations Overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Advanced Modeling](https://learn.microsoft.com/en-us/ef/core/modeling/)
