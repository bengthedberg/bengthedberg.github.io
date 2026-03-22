---
layout: post
title: "CQRS & Domain Events in .NET — Part 1: Domain Events for Loosely Coupled Systems"
date: 2026-05-28 00:00:00 +0000
tags:
  - dotnet
  - cqrs
  - domain-events
  - ddd
  - ef-core
  - mediatr
series: "CQRS & Domain Events in .NET"
series_part: 1
---


## Series Overview

This is a four-part series exploring how to implement CQRS and domain events in .NET, from foundational concepts to production-ready validation pipelines.

1. **Part 1: Domain Events for Loosely Coupled Systems** (this article)
2. [Part 2: Building a Custom Domain Events Dispatcher](2026-06-cqrs-domain-events-part2-dispatcher.md)
3. [Part 3: CQRS Pattern — Replacing MediatR](2026-06-cqrs-domain-events-part3-cqrs-pattern.md)
4. [Part 4: CQRS Validation with Pipeline Behaviors and FluentValidation](2026-06-cqrs-domain-events-part4-validation.md)


## Introduction

In software engineering, "coupling" describes how much different parts of a system depend on each other. Tightly coupled code means a change in one place ripples through the rest of the system. Loosely coupled code means each piece can evolve independently.

Domain events are a tactical pattern from Domain-Driven Design (DDD) that help you build loosely coupled systems. The idea is simple: when something meaningful happens in your domain, you raise an event. Other components subscribe to that event and react accordingly, without the originating code needing to know about them.

In this article, you will learn what domain events are, how they differ from integration events, how to implement and raise them from your entities, how to publish them transactionally with EF Core, and how to handle them with MediatR.

## What Are Domain Events?

An event represents something that has happened in the past. It is a fact — immutable and unchangeable.

A **domain event** is an event that occurred within your domain, and other parts of the domain should be aware of it. Domain events allow you to express side effects explicitly and provide a better separation of concerns. They are an ideal mechanism for triggering side effects across multiple aggregates.

For example, when a student completes a course, you might want to send a notification, update a leaderboard, and issue a certificate. Without domain events, all of that logic ends up crammed into one method. With domain events, the `Complete()` method simply raises a `CourseCompletedDomainEvent`, and separate handlers take care of the rest.

One important responsibility falls on you: ensuring that publishing a domain event is transactional. As you will see, this is easier said than done.

## Domain Events vs. Integration Events

You may have heard of integration events and wondered how they differ from domain events. Semantically, both represent something that occurred in the past. However, their **intent** is different, and understanding that distinction matters.

**Domain events:**

- Published and consumed within a single domain
- Sent using an in-memory message bus
- Can be processed synchronously or asynchronously

**Integration events:**

- Consumed by other subsystems (microservices, bounded contexts)
- Sent with a message broker over a queue
- Processed completely asynchronously

When deciding which type of event to publish, think about the intent and who should be handling the event. Domain events can also be used to *generate* integration events, bridging the gap between in-process and cross-service communication.

## Implementing Domain Events

A clean approach to implementing domain events is to create an `IDomainEvent` abstraction that implements MediatR's `INotification` interface. This gives you access to MediatR's built-in publish-subscribe support, allowing a single event to fan out to multiple handlers.

```csharp
using MediatR;

public interface IDomainEvent : INotification
{
}
```

When designing concrete domain events, keep these constraints in mind:

- **Immutability** — domain events are facts and should be immutable
- **Fat vs. thin events** — decide how much information each event carries
- **Past tense naming** — use names like `CourseCompleted`, not `CompleteCourse`

```csharp
public class CourseCompletedDomainEvent : IDomainEvent
{
    public Guid CourseId { get; init; }
}
```

## Raising Domain Events from Entities

After defining your domain events, you need a way to raise them from within the domain. A natural approach is to create an `Entity` base class that stores raised events in an internal collection. Only entities are allowed to raise domain events, and you can enforce this by making the `RaiseDomainEvent` method `protected`.

```csharp
public abstract class Entity : IEntity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyList<IDomainEvent> GetDomainEvents()
    {
        return _domainEvents.ToList();
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
}
```

Now your entities can inherit from `Entity` and raise events as part of their domain logic:

```csharp
public class Course : Entity
{
    public Guid Id { get; private set; }
    public CourseStatus Status { get; private set; }
    public DateTime? CompletedOnUtc { get; private set; }

    public void Complete()
    {
        Status = CourseStatus.Completed;
        CompletedOnUtc = DateTime.UtcNow;

        RaiseDomainEvent(new CourseCompletedDomainEvent { CourseId = this.Id });
    }
}
```

Notice how the `Complete()` method focuses on the state change and simply announces what happened. It does not know or care about what the system does in response.

## Publishing Domain Events with EF Core

An elegant solution for publishing domain events is to leverage EF Core's role as a Unit of Work. Since EF Core tracks all entities involved in the current transaction, you can gather domain events from those entities and publish them at the right moment.

The simplest approach is to override `SaveChangesAsync`:

```csharp
public class ApplicationDbContext : DbContext
{
    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        var result = await base.SaveChangesAsync(cancellationToken);

        await PublishDomainEventsAsync();

        return result;
    }
}
```

### When to Publish: Before or After SaveChanges?

This is the most important design decision in this entire setup. You have two options:

1. **Before `SaveChangesAsync`** — domain events are part of the same transaction. You get immediate consistency, but handlers that fail will roll back the entire operation.
2. **After `SaveChangesAsync`** — domain events run in a separate transaction. You get eventual consistency, but handlers can fail independently.

Publishing after saving changes is often the more practical choice. Eventual consistency is acceptable for most side effects (sending emails, updating caches), and it avoids coupling the success of your core operation to every handler.

However, this introduces a risk: if a handler fails, your database may be inconsistent. The **Outbox pattern** solves this by persisting domain events as outbox messages in the same transaction as your business data. A background job then processes them reliably, giving you both atomicity and eventual consistency.

### The PublishDomainEventsAsync Method

Here is what the publishing method looks like internally:

```csharp
private async Task PublishDomainEventsAsync()
{
    var domainEvents = ChangeTracker
        .Entries<Entity>()
        .Select(entry => entry.Entity)
        .SelectMany(entity =>
        {
            var domainEvents = entity.GetDomainEvents();
            entity.ClearDomainEvents();
            return domainEvents;
        })
        .ToList();

    foreach (var domainEvent in domainEvents)
    {
        await _publisher.Publish(domainEvent);
    }
}
```

This method iterates over all tracked entities, collects their domain events, clears them to prevent duplicate publishing, and then dispatches each event through MediatR's publisher.

## Handling Domain Events

With the plumbing in place, handling domain events is straightforward. You define a class that implements `INotificationHandler<T>` with your domain event type as the generic argument.

Here is a handler for `CourseCompletedDomainEvent` that publishes an integration event to notify external systems:

```csharp
public class CourseCompletedDomainEventHandler
    : INotificationHandler<CourseCompletedDomainEvent>
{
    private readonly IBus _bus;

    public CourseCompletedDomainEventHandler(IBus bus)
    {
        _bus = bus;
    }

    public async Task Handle(
        CourseCompletedDomainEvent domainEvent,
        CancellationToken cancellationToken)
    {
        await _bus.Publish(
            new CourseCompletedIntegrationEvent(domainEvent.CourseId),
            cancellationToken);
    }
}
```

You can add as many handlers as you need for a single event. Each handler focuses on a single concern — sending an email, updating analytics, publishing an integration event — keeping your codebase clean and extensible.

## Summary

Domain events are a foundational pattern for building loosely coupled systems. They let you separate core domain logic from side effects, keeping each piece focused and independently testable.

Using EF Core as a Unit of Work and MediatR for publish-subscribe gives you a practical, production-ready implementation without reinventing the wheel. The key decision is *when* to publish: before or after saving changes, and whether to introduce the Outbox pattern for transactional guarantees.

In this article we relied on MediatR for dispatching. But what if you want to eliminate that dependency and build your own lightweight dispatcher? That is exactly what we will cover next.

## What's Next?

In [Part 2: Building a Custom Domain Events Dispatcher](2026-06-cqrs-domain-events-part2-dispatcher.md), we will build a custom, strongly-typed domain event dispatcher from scratch — no third-party libraries required for the core dispatching logic.

## References

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [MediatR GitHub Repository](https://github.com/jbogard/MediatR)
- [Microsoft — Domain Events: Design and Implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation)
- [Outbox Pattern — Reliable Messaging](https://www.milanjovanovic.tech/blog/implementing-the-outbox-pattern)
- [EF Core — SaveChanges and Interceptors](https://learn.microsoft.com/en-us/ef/core/saving/)
