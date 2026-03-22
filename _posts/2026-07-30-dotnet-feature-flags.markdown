---
layout: post
title: "Feature Flags in .NET: Ship Faster with Confidence"
date: 2026-07-30 00:00:00 +0000
tags:
  - dotnet
  - feature-flags
  - feature-management
  - csharp
---


# Feature Flags in .NET: Ship Faster with Confidence

Imagine being able to deploy new code to production without immediately exposing it to users. Or gradually rolling out a feature to 5% of your audience, then 20%, then everyone -- all without a single redeployment. That is the power of feature flags.

Feature flags are a software development technique that lets you wrap application features in conditional statements, toggling them on or off at runtime. They decouple deployment from release, giving you fine-grained control over what users see and when.

In this article, we will cover:

- Getting started with feature flags in .NET
- Using feature filters for phased rollouts
- How feature flags enable trunk-based development
- Running A/B tests with feature flags

## Getting Started with Feature Flags in .NET

Microsoft provides an official [Feature Management](https://github.com/microsoft/FeatureManagement-Dotnet) library that integrates seamlessly with the .NET configuration system. Install it via NuGet:

```powershell
Install-Package Microsoft.FeatureManagement
```

Register the feature management services in your application's dependency injection container:

```csharp
builder.Services.AddFeatureManagement();
```

### Defining Feature Flags

Feature flags are built on top of the .NET configuration system, so any configuration provider (JSON files, environment variables, Azure App Configuration) can serve as the backing store. The simplest approach is to define them in `appsettings.json`:

```json
"FeatureManagement": {
  "ClipArticleContent": false
}
```

By convention, flags go under the `FeatureManagement` section, though you can customize this when calling `AddFeatureManagement`.

For referencing flag names in code, you have two options. Microsoft recommends enums with `nameof`, but using a static class with constants is often simpler:

```csharp
// Using enums
public enum FeatureFlags
{
    ClipArticleContent = 1
}

// Using constants (simpler in practice)
public static class FeatureFlags
{
    public const string ClipArticleContent = "ClipArticleContent";
}
```

### Checking Feature Flag State

Use the `IFeatureManager` service to check whether a flag is enabled. Here is an example in a minimal API endpoint:

```csharp
app.MapGet("articles/{id}", async (
    Guid id,
    IGetArticle query,
    IFeatureManager featureManager) =>
{
    var article = query.Execute(id);

    if (await featureManager.IsEnabledAsync(FeatureFlags.ClipArticleContent))
    {
        article.Content = article.Content.Substring(0, 50);
    }

    return Results.Ok(article);
});
```

You can also gate entire controllers or endpoints declaratively using the `FeatureGate` attribute:

```csharp
[FeatureGate(FeatureFlags.EnableArticlesApi)]
public class ArticlesController : Controller
{
   // This entire controller is only accessible when the flag is enabled
}
```

## Feature Filters and Phased Rollouts

Simple on/off flags are useful, but real-world scenarios often demand more nuance. The `Microsoft.FeatureManagement` library includes built-in feature filters for dynamic rules:

- **`Microsoft.Percentage`** -- Enables a flag for a percentage of evaluations
- **`Microsoft.TimeWindow`** -- Enables a flag during a specific time range
- **`Microsoft.Targeting`** -- Enables a flag for specific users or groups

### Percentage-Based Rollouts

Here is how to define a feature flag that activates 50% of the time:

```json
"FeatureFlags": {
  "ShowArticlePreview": {
    "EnabledFor": [
      {
        "Name": "Percentage",
        "Parameters": {
          "Value": 50
        }
      }
    ]
  }
}
```

Register the filter in your service configuration:

```csharp
builder.Services.AddFeatureManagement().AddFeatureFilter<PercentageFilter>();
```

One caveat: with a simple percentage filter, the same user might see different behavior across requests. For consistent experiences, consider caching the flag state per session or using the `TargetingFilter` instead.

### Targeted Rollouts

The `TargetingFilter` is the most powerful built-in filter. It lets you enable features for specific users or groups and gradually increase the rollout percentage. This is the foundation of phased rollouts -- you start with a small percentage of users and slowly expand while monitoring system health.

## Trunk-Based Development and Feature Flags

[Trunk-based development](https://trunkbaseddevelopment.com/) is a Git branching strategy where all developers work in short-lived branches or directly on the main branch. It avoids the "merge hell" caused by long-lived feature branches.

Feature flags are what make trunk-based development practical. Here is why:

1. You push incomplete work to the trunk continuously
2. The incomplete feature stays hidden behind a disabled feature flag
3. When the feature is complete and tested, you enable the flag
4. If something goes wrong in production, you disable the flag instantly -- no rollback needed

This approach keeps the trunk always releasable and gives you a fast path to both deploying and reverting changes.

## A/B Testing with Feature Flags

A/B testing (or split testing) takes feature flags further by randomly assigning users to different variants and measuring which performs better against a conversion goal.

For example, you might test whether removing a hero image from a landing page increases newsletter signups. A feature flag controls which version each user sees, and an analytics platform like [PostHog](https://posthog.com/) measures the results.

The workflow looks like this:

1. Define two (or more) variants behind feature flags
2. Randomly assign users to variants
3. Measure conversion rates per variant
4. Declare a winner based on statistical significance
5. Roll out the winning variant to everyone

## Key Takeaways

- Feature flags let you deploy code without releasing it, giving you a safety net for production changes
- The `Microsoft.FeatureManagement` library integrates with the .NET configuration system and requires minimal setup
- Built-in feature filters enable percentage-based rollouts, time windows, and user targeting
- Feature flags are essential for trunk-based development, keeping the main branch always releasable
- Combined with analytics, feature flags power A/B testing for data-driven product decisions

## References

- [Microsoft.FeatureManagement GitHub Repository](https://github.com/microsoft/FeatureManagement-Dotnet)
- [Microsoft Feature Management Documentation](https://learn.microsoft.com/en-us/azure/azure-app-configuration/use-feature-flags-dotnet-core)
- [PercentageFilter API Reference](https://learn.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement.featurefilters.percentagefilter)
- [TargetingFilter API Reference](https://learn.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement.featurefilters.targetingfilter)
- [Trunk-Based Development](https://trunkbaseddevelopment.com/)
- [PostHog - Product Analytics and A/B Testing](https://posthog.com/)
