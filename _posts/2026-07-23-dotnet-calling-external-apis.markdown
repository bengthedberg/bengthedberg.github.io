---
layout: post
title: "Calling External APIs in .NET the Right Way"
date: 2026-07-23 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - http
  - httpclient
  - csharp
  - resilience
---


# Calling External APIs in .NET the Right Way

Almost every modern application needs to talk to external services -- payment providers, identity platforms, third-party data sources. In .NET, the primary tool for this job is `HttpClient`. It has a clean API, first-class JSON support, and is easy to get started with.

Unfortunately, it is also easy to get wrong.

Getting `HttpClient` usage right is not just about clean code -- it is about avoiding production outages caused by **port exhaustion** and **stale DNS resolution**. In this article we will walk through the different approaches, from the simplest (and most problematic) to the recommended best practice.

## The Simple Way (and Why It Is Dangerous)

The most intuitive approach is to create a new `HttpClient` for each request:

```csharp
public class GitHubService
{
    private readonly GitHubSettings _settings;

    public GitHubService(IOptions<GitHubSettings> settings)
    {
        _settings = settings.Value;
    }

    public async Task<GitHubUser?> GetUserAsync(string username)
    {
        using var client = new HttpClient();

        client.DefaultRequestHeaders.Add("Authorization", _settings.GitHubToken);
        client.DefaultRequestHeaders.Add("User-Agent", _settings.UserAgent);
        client.BaseAddress = new Uri("https://api.github.com");

        GitHubUser? user = await client
            .GetFromJsonAsync<GitHubUser>($"users/{username}");

        return user;
    }
}
```

This looks clean, but it has a serious problem. Each `HttpClient` instance uses its own connection pool. When the `HttpClient` is disposed, the underlying TCP connections enter a `TIME_WAIT` state and hold onto their ports for up to 240 seconds. Under load, your application will exhaust the available ports, leading to `SocketException` errors.

**Key insight:** `HttpClient` instances are meant to be **long-lived** and reused.

## The Smart Way: IHttpClientFactory

Instead of managing `HttpClient` lifetimes yourself, let `IHttpClientFactory` handle it:

```csharp
public class GitHubService
{
    private readonly GitHubSettings _settings;
    private readonly IHttpClientFactory _factory;

    public GitHubService(
        IOptions<GitHubSettings> settings,
        IHttpClientFactory factory)
    {
        _settings = settings.Value;
        _factory = factory;
    }

    public async Task<GitHubUser?> GetUserAsync(string username)
    {
        using var client = _factory.CreateClient();

        client.DefaultRequestHeaders.Add("Authorization", _settings.GitHubToken);
        client.DefaultRequestHeaders.Add("User-Agent", _settings.UserAgent);
        client.BaseAddress = new Uri("https://api.github.com");

        GitHubUser? user = await client
            .GetFromJsonAsync<GitHubUser>($"users/{username}");

        return user;
    }
}
```

The magic happens behind the scenes. The expensive part of `HttpClient` is its `HttpMessageHandler`, which owns the connection pool. `IHttpClientFactory` **caches and reuses** message handlers across `HttpClient` instances, preventing port exhaustion while also periodically recycling handlers to pick up DNS changes.

With this approach, the `HttpClient` instances themselves are **short-lived** -- you create them, use them, and dispose them. The factory manages the underlying handler pool.

Register it in your DI container with:

```csharp
builder.Services.AddHttpClient();
```

## Reducing Duplication: Named Clients

The factory approach solves the resource management problem, but we are still configuring headers and base address on every call. **Named clients** let you centralize that configuration:

```csharp
services.AddHttpClient("github", (serviceProvider, client) =>
{
    var settings = serviceProvider
        .GetRequiredService<IOptions<GitHubSettings>>().Value;

    client.DefaultRequestHeaders.Add("Authorization", settings.GitHubToken);
    client.DefaultRequestHeaders.Add("User-Agent", settings.UserAgent);

    client.BaseAddress = new Uri("https://api.github.com");
});
```

Now the service code is much cleaner:

```csharp
public class GitHubService
{
    private readonly IHttpClientFactory _factory;

    public GitHubService(IHttpClientFactory factory)
    {
        _factory = factory;
    }

    public async Task<GitHubUser?> GetUserAsync(string username)
    {
        using var client = _factory.CreateClient("github");

        GitHubUser? user = await client
            .GetFromJsonAsync<GitHubUser>($"users/{username}");

        return user;
    }
}
```

The downside is the magic string `"github"` -- a typo gives you an unconfigured client with no compile-time warning.

## The Best Way: Typed Clients

**Typed clients** eliminate the magic string by binding an `HttpClient` directly to a specific service class:

```csharp
services.AddHttpClient<GitHubService>((serviceProvider, client) =>
{
    var settings = serviceProvider
        .GetRequiredService<IOptions<GitHubSettings>>().Value;

    client.DefaultRequestHeaders.Add("Authorization", settings.GitHubToken);
    client.DefaultRequestHeaders.Add("User-Agent", settings.UserAgent);

    client.BaseAddress = new Uri("https://api.github.com");
});
```

The service receives a pre-configured `HttpClient` through constructor injection. No factory, no magic strings:

```csharp
public class GitHubService
{
    private readonly HttpClient _client;

    public GitHubService(HttpClient client)
    {
        _client = client;
    }

    public async Task<GitHubUser?> GetUserAsync(string username)
    {
        GitHubUser? user = await _client
            .GetFromJsonAsync<GitHubUser>($"users/{username}");

        return user;
    }
}
```

Under the hood, this still uses named clients (the name is the type name). The `AddHttpClient<T>` call also registers `GitHubService` in the DI container with a **transient** lifetime.

## The Singleton Trap

There is an important caveat with typed clients: they are registered as **transient**. If you inject a typed client into a **singleton** service, the `HttpClient` will be captured and cached for the lifetime of the application. This defeats the handler recycling mechanism and prevents the client from picking up DNS changes.

The fix is to configure `SocketsHttpHandler` with `PooledConnectionLifetime` and disable the factory's own handler recycling:

```csharp
services.AddHttpClient<GitHubService>((serviceProvider, client) =>
{
    var settings = serviceProvider
        .GetRequiredService<IOptions<GitHubSettings>>().Value;

    client.DefaultRequestHeaders.Add("Authorization", settings.GitHubToken);
    client.DefaultRequestHeaders.Add("User-Agent", settings.UserAgent);

    client.BaseAddress = new Uri("https://api.github.com");
})
.ConfigurePrimaryHttpMessageHandler(() =>
{
    return new SocketsHttpHandler()
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(15)
    };
})
.SetHandlerLifetime(Timeout.InfiniteTimeSpan);
```

With this configuration, the `SocketsHttpHandler` handles connection pooling and periodic recycling internally, so it is safe to hold onto the `HttpClient` indefinitely.

## Which Approach Should You Use?

| Scenario | Recommended approach |
|----------|---------------------|
| Quick prototype or console app | `IHttpClientFactory` (basic) |
| Multiple services calling the same API | Named client |
| Dedicated service per external API | Typed client |
| Typed client in a singleton service | Typed client + `SocketsHttpHandler` |

For most ASP.NET Core applications, **typed clients** are the way to go. They provide the cleanest API, compile-time safety, and proper resource management. Just be mindful of the singleton trap.

## Key Takeaways

- Never create `HttpClient` instances with `new HttpClient()` in production code. You risk port exhaustion.
- Use `IHttpClientFactory` to get proper handler pooling and DNS rotation.
- **Typed clients** give you the cleanest developer experience by injecting a pre-configured `HttpClient` directly.
- If you must use a typed client in a singleton, configure `SocketsHttpHandler` with `PooledConnectionLifetime` to handle connection recycling.

## References

- [IHttpClientFactory guidelines (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)
- [Use IHttpClientFactory to implement resilient HTTP requests (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)
- [HttpClient recommended use (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines#recommended-use)
- [SocketsHttpHandler class (Microsoft Docs)](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.socketshttphandler)
