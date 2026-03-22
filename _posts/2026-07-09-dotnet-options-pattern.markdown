---
layout: post
title: "Configuration with the Options Pattern in .NET"
date: 2026-07-09 00:00:00 +0000
tags:
  - dotnet
  - configuration
  - options-pattern
  - csharp
---


# Configuration with the Options Pattern in .NET

Every non-trivial application needs configuration -- API keys, endpoint URLs, feature flags, connection strings. In .NET, the most common starting point is `appsettings.json`, but how you *consume* that configuration in your code makes all the difference. Sprinkling `IConfiguration["SomeKey"]` calls throughout your services leads to stringly-typed, error-prone code that is hard to validate and refactor.

The **Options pattern** is the recommended way to bind configuration sections to strongly typed C# classes, with built-in support for validation, reloading, and dependency injection. Let's see how it works.

## Why the Options Pattern?

Without the Options pattern, reading configuration often looks like this:

```csharp
var apiKey = configuration["Adyen:ApiKey"];
```

This has several problems:

- **No compile-time safety** -- a typo in the key silently returns `null`.
- **No validation** -- you only discover missing or malformed values at runtime, often deep inside business logic.
- **Scattered access** -- every class that needs config must know the exact key paths.

The Options pattern solves all three by giving you a POCO class that the framework populates, validates, and injects for you.

## Step 1: Define Your Configuration Section

Start with a section in `appsettings.json` that groups related settings:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "Adyen": {
    "ApiBaseUrl": "",
    "ReturnUrl": "",
    "ThemeId": "",
    "MerchantAccount": "",
    "ApiKey": ""
  }
}
```

## Step 2: Create an Options Class

Create a class whose properties mirror the JSON section. A `SectionName` constant makes registration cleaner and avoids magic strings:

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyApp.NameSpace;

public class AdyenOptions
{
    public const string SectionName = "Adyen";

    [Required]
    [RegularExpression(@"^[a-zA-Z''-'\s]{1,40}$")]
    public string ApiKey { get; set; } = string.Empty;

    [Required, Url]
    public string ApiBaseUrl { get; set; } = string.Empty;

    [Required]
    [RegularExpression(@"^[a-zA-Z''-'\s]{1,40}$")]
    public string MerchantAccount { get; set; } = string.Empty;

    [Required, Url]
    public string ReturnUrl { get; set; } = string.Empty;

    public string? ThemeId { get; set; }
}
```

Notice the `DataAnnotations` attributes. These are not just for documentation -- the framework can enforce them at startup, which we will configure next.

## Step 3: Register and Validate

In your `Program.cs` (or wherever you configure services), bind the options class to the configuration section and enable validation:

```csharp
services.AddOptions<AdyenOptions>()
    .BindConfiguration(AdyenOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Let's unpack this chain:

- **`AddOptions<T>()`** -- returns an `OptionsBuilder<T>` that lets you configure how the options object is built.
- **`BindConfiguration(sectionName)`** -- binds the specified `appsettings.json` section to the class properties.
- **`ValidateDataAnnotations()`** -- enables validation using the `DataAnnotations` attributes on your properties (`[Required]`, `[Url]`, `[RegularExpression]`, etc.).
- **`ValidateOnStart()`** -- runs validation when the application starts rather than waiting until the options are first accessed. This is critical for catching misconfiguration early. Without it, you might not discover a missing API key until the first payment request fails.

## Step 4: Inject and Use

Now any service can receive the configuration through dependency injection via `IOptions<T>`:

```csharp
public class PaymentService
{
    private readonly AdyenOptions _adyenOptions;

    public PaymentService(IOptions<AdyenOptions> adyenOptions)
    {
        _adyenOptions = adyenOptions.Value;
    }
}
```

The `.Value` property gives you the populated, validated `AdyenOptions` instance. Your service never needs to know about `IConfiguration`, JSON keys, or parsing logic.

## IOptions vs. IOptionsSnapshot vs. IOptionsMonitor

The `IOptions<T>` interface comes in three flavors, each suited to different scenarios:

| Interface             | Lifetime   | Reloads on change? | Use when                          |
|-----------------------|------------|---------------------|-----------------------------------|
| `IOptions<T>`         | Singleton  | No                  | Configuration is static           |
| `IOptionsSnapshot<T>` | Scoped     | Yes, per request    | Config can change; need per-request values |
| `IOptionsMonitor<T>`  | Singleton  | Yes, via callback   | Singleton services that need live updates  |

For most web applications, `IOptions<T>` is sufficient. Use `IOptionsSnapshot<T>` when you want hot-reload behavior in scoped services, and `IOptionsMonitor<T>` when singleton services need to react to configuration changes.

## Key Takeaways

- The Options pattern replaces fragile string-based configuration access with strongly typed, validated classes.
- Use `DataAnnotations` on your options class and call `ValidateOnStart()` to catch configuration errors at application startup.
- Choose between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>` based on whether your service needs to react to configuration changes at runtime.
- Keep a `SectionName` constant on your options class to avoid magic strings during registration.

## References

- [Options pattern in ASP.NET Core (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options)
- [Configuration in ASP.NET Core (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration)
- [Options validation (Microsoft Docs)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options#options-validation)
- [IOptionsMonitor (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.options.ioptionsmonitor-1)
