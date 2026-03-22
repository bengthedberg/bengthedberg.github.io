---
layout: post
title: "Introduction to EF Core and Domain Modeling — Part 2: One-to-One and One-to-Many Relationships"
date: 2026-02-05 00:00:00 +0000
tags:
  - efcore
  - dotnet
  - domain-modeling
  - entity-framework
  - csharp
series: "Introduction to EF Core and Domain Modeling"
series_part: 2
---


## Series Overview

This is a 4-part series on Entity Framework Core and domain modeling in .NET:

1. [Setup and Your First DbContext](/posts/efcore-domain-modeling-part1-setup/) — What is EF Core, project setup, DbContext, dependency injection, migrations
2. **One-to-One and One-to-Many Relationships** (this article) — Modeling relationships between entities with the Fluent API
3. [Many-to-Many Relationships](/posts/efcore-domain-modeling-part3-many-to-many/) — Join tables, implicit and explicit approaches, unidirectional navigation
4. [Seed Data and Putting It All Together](/posts/efcore-domain-modeling-part4-seed-data/) — Populating your database, complete model example, next steps


## Introduction

Real domain models are not isolated entities — they have relationships. A blog has posts. A post has an author. An order has line items. EF Core models these relationships through **navigation properties** and **foreign keys**, and can discover many of them automatically through conventions.

In this article we'll cover the two most fundamental relationship types: **one-to-many** and **one-to-one**. We start with one-to-many because it's the most common relationship you'll encounter.

## Key Terminology

Before diving into code, let's establish the vocabulary EF Core uses:

- **Principal entity** — the "parent" side of the relationship. It can exist independently.
- **Dependent entity** — the "child" side. It holds the foreign key and depends on the principal.
- **Foreign key (FK)** — a property on the dependent that references the principal's primary key.
- **Navigation property** — a C# property that lets you traverse the relationship:
  - **Reference navigation** — points to a single related entity (e.g. `Post.Blog`)
  - **Collection navigation** — points to many related entities (e.g. `Blog.Posts`)
- **Required relationship** — the FK is non-nullable; the dependent must always reference a principal.
- **Optional relationship** — the FK is nullable; the dependent can exist without a principal.

## One-to-Many Relationships

One-to-many relationships are the most common in domain modeling. A single entity is associated with any number of other entities. For example, a `Blog` can have many `Posts`, but each `Post` belongs to only one `Blog`.

### Required One-to-Many

```csharp
// Principal (parent)
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; } = new List<Post>(); // Collection navigation containing dependents
}

// Dependent (child)
public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; } // Required foreign key property
    public Blog Blog { get; set; } = null!; // Required reference navigation to principal
}
```

The `BlogId` property is a non-nullable `int`, making this a **required** relationship — every `Post` must belong to a `Blog`.

> **Note**: A required relationship ensures that every dependent entity must be associated with some principal entity. However, a principal entity can always exist without any dependent entities. That is, a required relationship does not indicate that there will always be at least one dependent entity. There is no way in the EF model, and also no standard way in a relational database, to ensure that a principal is associated with a certain number of dependents. If this is needed, it must be implemented in application (business) logic.

EF Core discovers this relationship by convention — no configuration needed. But if you want to be explicit (or if convention doesn't pick it up), use the Fluent API:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(e => e.Posts)
        .WithOne(e => e.Blog)
        .HasForeignKey(e => e.BlogId)
        .IsRequired();
}
```

You can configure the same relationship starting from the dependent side instead:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>()
        .HasOne(e => e.Blog)
        .WithMany(e => e.Posts)
        .HasForeignKey(e => e.BlogId)
        .IsRequired();
}
```

Neither of these options is better than the other; they both result in exactly the same configuration.

### Optional One-to-Many

Make the foreign key and navigation nullable, and the dependent can exist without a principal:

```csharp
// Principal (parent)
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; } = new List<Post>(); // Collection navigation containing dependents
}

// Dependent (child)
public class Post
{
    public int Id { get; set; }
    public int? BlogId { get; set; } // Optional foreign key property
    public Blog? Blog { get; set; } // Optional reference navigation to principal
}
```

This makes the relationship "optional" — a `Post` can exist without being related to any `Blog`.

Explicit configuration:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(e => e.Posts)
        .WithOne(e => e.Blog)
        .HasForeignKey(e => e.BlogId)
        .IsRequired(false);
}
```

### One-to-Many with Shadow Foreign Key

You can remove both the foreign key property and the navigation from the dependent, keeping the entity class clean:

```csharp
// Principal (parent)
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; } = new List<Post>(); // Collection navigation containing dependents
}

// Dependent (child)
public class Post
{
    public int Id { get; set; }
}
```

EF Core discovers this as an optional relationship by convention. Since there is nothing in the code to indicate it should be required, you need explicit configuration:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasMany(e => e.Posts)
        .WithOne()
        .HasForeignKey("BlogId")
        .IsRequired();
}
```

EF Core creates a **shadow property** called `BlogId` on the `Post` entity. It exists in the EF model and the database, but not in your C# class. This is useful when you want a clean domain model that doesn't expose infrastructure concerns like foreign keys.

## One-to-One Relationships

One-to-one relationships are used when one entity is associated with at most one other entity. For example, a `Blog` has one `BlogHeader`, and that `BlogHeader` belongs to a single `Blog`.

### Required One-to-One

```csharp
// Principal (parent)
public class Blog
{
    public int Id { get; set; }
    public BlogHeader? Header { get; set; } // Reference navigation to dependent
}

// Dependent (child)
public class BlogHeader
{
    public int Id { get; set; }
    public int BlogId { get; set; } // Required foreign key property
    public Blog Blog { get; set; } = null!; // Required reference navigation to principal
}
```

Explicit configuration:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasOne(e => e.Header)
        .WithOne(e => e.Blog)
        .HasForeignKey<BlogHeader>(e => e.BlogId)
        .IsRequired();
}
```

Note the `HasForeignKey<BlogHeader>` — with one-to-one relationships, you must tell EF Core which side holds the foreign key, since both sides have reference navigations.

### Choosing Principal vs Dependent

It's not always obvious which side should be the principal and which should be the dependent. Consider:

- **Existing database?** The table with the foreign key column must map to the dependent type.
- **Logical dependency** — If one entity cannot logically exist without the other, it's the dependent. A `BlogHeader` makes no sense without a `Blog`, so `BlogHeader` is the dependent.
- **Parent/child** — In a natural parent/child relationship, the child is usually the dependent.

### Optional One-to-One

Make the foreign key and navigation nullable:

```csharp
// Principal (parent)
public class Blog
{
    public int Id { get; set; }
    public BlogHeader? Header { get; set; } // Reference navigation to dependent
}

// Dependent (child)
public class BlogHeader
{
    public int Id { get; set; }
    public int? BlogId { get; set; } // Optional foreign key property
    public Blog? Blog { get; set; } // Optional reference navigation to principal
}
```

This means a `BlogHeader` can exist without being associated with any `Blog`.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasOne(e => e.Header)
        .WithOne(e => e.Blog)
        .HasForeignKey<BlogHeader>(e => e.BlogId)
        .IsRequired(false);
}
```

### One-to-One with Shadow Foreign Key

Remove both the foreign key and the navigation from the dependent:

```csharp
// Principal (parent)
public class Blog
{
    public int Id { get; set; }
    public BlogHeader? Header { get; set; } // Reference navigation to dependent
}

// Dependent (child)
public class BlogHeader
{
    public int Id { get; set; }
}
```

This requires configuration to indicate the principal and dependent ends:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .HasOne(e => e.Header)
        .WithOne()
        .HasForeignKey<BlogHeader>("BlogId")
        .IsRequired();
}
```

## Convention vs Explicit Configuration

EF Core discovers relationships automatically when:

- There are navigation properties on one or both sides
- There is a foreign key property that follows naming conventions (`<NavigationPropertyName>Id` or `<PrincipalTypeName>Id`)

You need explicit Fluent API configuration when:

- The relationship is **ambiguous** (multiple relationships between the same types)
- The foreign key property **doesn't follow conventions**
- You're using **shadow foreign keys** (no FK property in the entity class)
- You need to specify **required vs optional** when it can't be inferred from nullability
- You want to configure **cascade delete behaviour**

As a general rule: start with conventions, and add Fluent API configuration only when EF Core can't figure out your intent. The generated migration will tell you exactly what EF Core inferred — review it to confirm.

## Conclusion

You now know how to model the two most fundamental relationship types in EF Core. One-to-many covers the majority of real-world associations, while one-to-one handles cases where two entities have a strict pairing.

In [Part 3](/posts/efcore-domain-modeling-part3-many-to-many/), we'll tackle the more nuanced many-to-many relationship — including when to let EF Core manage the join table transparently and when to define an explicit join entity.

## References

- [Relationships in EF Core](https://learn.microsoft.com/en-us/ef/core/modeling/relationships/)
- [One-to-Many Relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships/one-to-many)
- [One-to-One Relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships/one-to-one)
- [Fluent API Configuration](https://learn.microsoft.com/en-us/ef/core/modeling/)
