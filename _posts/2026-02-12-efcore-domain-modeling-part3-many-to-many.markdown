---
layout: post
title: "Introduction to EF Core and Domain Modeling — Part 3: Many-to-Many Relationships"
date: 2026-02-12 00:00:00 +0000
categories: [.NET]
tags:
  - efcore
  - dotnet
  - domain-modeling
  - entity-framework
  - csharp
series: "Introduction to EF Core and Domain Modeling"
series_part: 3
---


## Series Overview

This is a 4-part series on Entity Framework Core and domain modeling in .NET:

1. [Setup and Your First DbContext](/posts/efcore-domain-modeling-part1-setup/) — What is EF Core, project setup, DbContext, dependency injection, migrations
2. [One-to-One and One-to-Many Relationships](/posts/efcore-domain-modeling-part2-relationships/) — Modeling relationships between entities with the Fluent API
3. **Many-to-Many Relationships** (this article) — Join tables, implicit and explicit approaches, unidirectional navigation
4. [Seed Data and Putting It All Together](/posts/efcore-domain-modeling-part4-seed-data/) — Populating your database, complete model example, next steps


## Introduction

In [Part 2](/posts/efcore-domain-modeling-part2-relationships/), we modelled relationships where one entity references another directly through a foreign key. Many-to-many relationships are fundamentally different: they cannot be represented with a single foreign key. Instead, they require an additional **join table** that sits between the two entities.

Think of real-world examples: a `Post` can have many `Tags`, and a `Tag` can be applied to many `Posts`. A `Student` can enrol in many `Courses`, and a `Course` has many `Students`. Neither side "owns" the other — the relationship is symmetric.

In this article, we'll look at three ways EF Core handles many-to-many relationships, from the most explicit to the most concise.

## The Explicit Join Entity Approach

The traditional way to model many-to-many is to create an explicit join entity with two one-to-many relationships:

```csharp
public class Post
{
    public int Id { get; set; }
    public List<PostTag> PostTags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }
    public List<PostTag> PostTags { get; } = [];
}

public class PostTag
{
    public int PostsId { get; set; }
    public int TagsId { get; set; }
    public Post Post { get; set; } = null!;
    public Tag Tag { get; set; } = null!;
}
```

In this mapping there is no many-to-many relationship at the C# level — only two one-to-many relationships, one for each foreign key in the join table. To get from a `Post` to its `Tags`, you navigate through `PostTags`:

```csharp
var tags = post.PostTags.Select(pt => pt.Tag);
```

This approach is **not wrong**, but it doesn't reflect the intent. The join table represents a single many-to-many relationship, not two independent one-to-many relationships.

### When to Use the Explicit Approach

Use an explicit join entity when the **join itself carries data**. For example, if you need to track *when* a tag was applied to a post:

```csharp
public class PostTag
{
    public int PostsId { get; set; }
    public int TagsId { get; set; }
    public Post Post { get; set; } = null!;
    public Tag Tag { get; set; } = null!;
    public DateTime TaggedOn { get; set; } // Payload on the join
}
```

Without the explicit entity, there's nowhere to put `TaggedOn`.

## The Implicit (EF-Managed) Approach

EF Core can manage the join entity transparently, without a .NET class defined for it. This is what is typically meant by "many-to-many" in EF Core:

```csharp
public class Post
{
    public int Id { get; set; }
    public List<Tag> Tags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }
    public List<Post> Posts { get; } = [];
}
```

That's it. No `PostTag` class. EF Core's model building conventions detect the two collection navigations and automatically:

1. Create a shadow join entity type
2. Create a `PostTag` join table in the database
3. Set up the composite primary key and foreign keys

Navigation is direct — `post.Tags` gives you the tags, `tag.Posts` gives you the posts. No intermediate entity to traverse.

This is the preferred approach for **pure associations** where the join carries no additional data.

## Unidirectional Many-to-Many

*Available in EF Core 7 and later.*

It is not necessary to include a navigation on both sides. If only `Post` needs to know about its `Tags`, but `Tag` doesn't need to know about its `Posts`:

```csharp
public class Post
{
    public int Id { get; set; }
    public List<Tag> Tags { get; } = [];
}

public class Tag
{
    public int Id { get; set; }
}
```

EF Core needs a hint that this is many-to-many rather than one-to-many. Configure it with `HasMany`/`WithMany`, passing no argument on the side without a navigation:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>()
        .HasMany(e => e.Tags)
        .WithMany();
}
```

This is useful when the reverse navigation would just add noise to the `Tag` entity. Not every entity needs to know about every relationship it participates in.

## What the Database Actually Looks Like

Regardless of which C# approach you use (explicit, implicit, or unidirectional), the database schema is the same:

```sql
CREATE TABLE "Posts" (
    "Id" INTEGER NOT NULL CONSTRAINT "PK_Posts" PRIMARY KEY AUTOINCREMENT);

CREATE TABLE "Tags" (
    "Id" INTEGER NOT NULL CONSTRAINT "PK_Tags" PRIMARY KEY AUTOINCREMENT);

CREATE TABLE "PostTag" (
    "PostId" INTEGER NOT NULL,
    "TagsId" INTEGER NOT NULL,
    CONSTRAINT "PK_PostTag" PRIMARY KEY ("PostId", "TagsId"),
    CONSTRAINT "FK_PostTag_Posts_PostId" FOREIGN KEY ("PostId")
        REFERENCES "Posts" ("Id") ON DELETE CASCADE,
    CONSTRAINT "FK_PostTag_Tags_TagsId" FOREIGN KEY ("TagsId")
        REFERENCES "Tags" ("Id") ON DELETE CASCADE);
```

The join table has:

- A **composite primary key** (`PostId`, `TagsId`) — each combination can exist only once
- **Two foreign keys** with cascade delete — if a `Post` or `Tag` is deleted, the join table rows are cleaned up automatically
- **No additional columns** (unless you used an explicit join entity with payload)

## Choosing Your Approach

| Approach | When to Use |
|---|---|
| **Implicit (EF-managed)** | Pure associations with no extra data on the join. This is the default choice. |
| **Unidirectional** | Same as implicit, but one side doesn't need the reverse navigation. Keeps entities cleaner. |
| **Explicit join entity** | The join carries payload (dates, ordering, metadata). You need to query or manipulate the join data directly. |

Start with the implicit approach. If you later discover the join needs payload, you can refactor to an explicit join entity — EF Core migrations will handle the schema transition.

## Conclusion

Many-to-many relationships are conceptually straightforward but require a join table that EF Core can manage for you. The implicit approach keeps your C# model clean, while the explicit join entity gives you full control when the relationship itself carries data.

In [Part 4](/posts/efcore-domain-modeling-part4-seed-data/), we'll seed the database with initial data and bring together all the relationship types from this series into a complete domain model.

## References

- [Many-to-Many Relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships/many-to-many)
- [Relationships in EF Core](https://learn.microsoft.com/en-us/ef/core/modeling/relationships/)
