---
layout: post
title: ".NET Testing — Part 1: Getting Started with xUnit.net"
date: 2026-03-05 00:00:00 +0000
tags:
  - dotnet
  - testing
  - xunit
  - csharp
  - unit-testing
series: ".NET Testing"
series_part: 1
---


## Series Overview

This is a 3-part series on testing .NET applications:

1. **Getting Started with xUnit.net** (this article) — Project setup, writing tests, data-driven tests, fixtures, parallelism, and advanced features
2. [Writing Readable Tests with Fluent Assertions](2026-03-dotnet-testing-part2-fluent-assertions.md) — Natural-language assertions, clearer failure messages, and custom assertions
3. [Integration Testing with Testcontainers](2026-03-dotnet-testing-part3-testcontainers.md) — Testing against real databases using Docker containers and CI setup


## Why xUnit.net?

xUnit.net is the most widely-used testing framework in the .NET ecosystem. It was built by the original creators of NUnit with a focus on simplicity, extensibility, and modern .NET idioms. If you're writing tests for a .NET application today, xUnit is the default choice.

## Setting Up Your First Test Project

Create a new xUnit test project with the .NET CLI:

```shell
dotnet new xunit -o ./tests/MyApp.Unit.Tests -n MyApp.Unit.Tests
```

This scaffolds a class library project with the essential NuGet packages:

| Package | Purpose |
|---------|---------|
| [xunit](https://github.com/xunit/xunit) | The testing framework itself |
| [xunit.runner.visualstudio](https://github.com/xunit/visualstudio.xunit) | Visual Studio / VS Code test runner |
| [Microsoft.NET.Test.Sdk](https://github.com/microsoft/vstest) | The VSTest platform that runs your tests |
| [coverlet.collector](https://github.com/coverlet-coverage/coverlet) | Code coverage collection |

## Writing Your First Test

Every method annotated with `[Fact]` is a test:

```csharp
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_1and2_gives3()
    {
        var result = Add(1, 2);
        Assert.Equal(3, result);
    }
}
```

The class must be `public`, but no attributes are needed on the class itself. Run your tests with:

```shell
dotnet test
```

## Skipping a Test

Add the `Skip` property to temporarily disable a test:

```csharp
[Fact(Skip = "Waiting on API endpoint to be deployed")]
public void CallsExternalApi()
{
    Assert.False(true);
}
```

## Data-Driven Tests with Theory

The real power of xUnit comes from `[Theory]` — a single test method that runs against multiple data sets. There are four ways to provide data:

```csharp
[Theory]
[InlineData(1, 2, 3)]              // Constant values
[MemberData(nameof(AdditionData))] // From a static property/method
[ClassData(typeof(AdditionTestData))] // From a class implementing IEnumerable<object[]>
public void Add_ReturnsCorrectSum(int a, int b, int expected)
{
    Assert.Equal(expected, a + b);
}
```

### InlineData — Simple constants

Best for a handful of cases with primitive values. The data lives right next to the test.

### MemberData — Static properties or methods

Use `TheoryData<T>` for strong typing:

```csharp
public static TheoryData<int, int, int> AdditionData => new()
{
    { 18, 24, 42 },
    { 6, 7, 13 },
};
```

### ClassData — Reusable data classes

When the same data set feeds multiple test methods:

```csharp
public class AdditionTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 9, 1, 10 };
        yield return new object[] { 9, 10, 19 };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

### Custom DataAttribute

For full control, inherit from `DataAttribute`:

```csharp
private class CustomDataAttribute : DataAttribute
{
    public override IEnumerable<object[]> GetData(MethodInfo testMethod)
    {
        yield return new object[] { 2, 3, 5 };
    }
}
```

## The Built-in Assert API

xUnit provides a comprehensive set of assertions:

```csharp
// Equality
Assert.Equal(42, value);
Assert.NotEqual(42, value);

// Booleans
Assert.True(condition, "should be true");
Assert.False(condition, "should be false");

// Strings
Assert.Equal("expected", str, ignoreCase: false);
Assert.StartsWith("prefix", str);
Assert.EndsWith("suffix", str);
Assert.Matches("[0-9]+", str);

// Collections
Assert.Empty(collection);
Assert.Single(collection);
Assert.Contains(item, collection);
Assert.All(collection, item => Assert.InRange(item, 0, 10));
Assert.Collection(collection,
    item => Assert.Equal(1, item),
    item => Assert.Equal(2, item));

// Exceptions
var ex = Assert.Throws<ArgumentException>(() => sut.BadMethod());
Assert.Equal("message", ex.Message);

var ex2 = await Assert.ThrowsAsync<InvalidOperationException>(
    () => sut.BadMethodAsync());

// Events
Assert.Raises<EventArgs>(
    handler => obj.MyEvent += handler,
    handler => obj.MyEvent -= handler,
    () => obj.RaiseEvent());
```

While these work fine, part 2 of this series covers **Fluent Assertions** — a library that makes your assertions read like plain English.

## Setup and Teardown

xUnit uses constructor/disposable patterns instead of `[SetUp]`/`[TearDown]` attributes. This is more idiomatic C# and clearer about lifecycle.

### Per-Test Setup — Constructor and IDisposable

xUnit creates a **new instance of the test class for every test**, so the constructor runs before each test:

```csharp
public class OrderTests : IDisposable, IAsyncLifetime
{
    public OrderTests()
    {
        // Runs before each test
    }

    public void Dispose()
    {
        // Runs after each test
    }

    // For async setup/teardown:
    public Task InitializeAsync() => Task.CompletedTask;
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public void Test1() { }

    [Fact]
    public void Test2() { }
}
```

### Per-Class Setup — IClassFixture

When setup is expensive (e.g., creating a database connection), share it across all tests in a class:

```csharp
public class DatabaseFixture : IDisposable, IAsyncLifetime
{
    public DatabaseFixture()
    {
        // Called once before all tests in the class
    }

    public void Dispose()
    {
        // Called once after all tests in the class
    }

    public Task InitializeAsync() => Task.CompletedTask;
    public Task DisposeAsync() => Task.CompletedTask;
}

public class OrderTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public OrderTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Test1() { }
}
```

### Cross-Class Setup — Collection Fixtures

Share a fixture across multiple test classes using collections:

```csharp
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class OrderTests
{
    private readonly DatabaseFixture _fixture;
    public OrderTests(DatabaseFixture fixture) => _fixture = fixture;
}

[Collection("Database")]
public class CustomerTests
{
    private readonly DatabaseFixture _fixture;
    public CustomerTests(DatabaseFixture fixture) => _fixture = fixture;
}
```

## Parallel Execution

By default, xUnit runs **collections in parallel** and creates one collection per class. Understanding this is important for test performance.

Two classes with no explicit collection run in parallel:

```csharp
// These two classes run in parallel (~10 seconds total)
public class UnitTest1
{
    [Fact]
    public async Task Test1() => await Task.Delay(5000);
    [Fact]
    public async Task Test2() => await Task.Delay(5000);
}

public class UnitTest2
{
    [Fact]
    public void Test3() => Thread.Sleep(5000);
    [Fact]
    public void Test4() => Thread.Sleep(5000);
}
```

Placing them in the same collection forces sequential execution:

```csharp
// Same collection = sequential (~20 seconds total)
[Collection("Sequential")]
public class UnitTest1 { ... }

[Collection("Sequential")]
public class UnitTest2 { ... }
```

Use explicit collections when tests share state that isn't thread-safe.

## Categorising Tests with Traits

Traits let you tag and filter tests:

```csharp
internal static class TestCategories
{
    public const string Category = "Category";

    public static class Values
    {
        public const string Integration = "Integration";
        public const string Unit = "Unit";
    }
}

[Trait(TestCategories.Category, TestCategories.Values.Integration)]
public class ApiIntegrationTests
{
    [Fact]
    [Trait("Issue", "123")]
    public void CallsExternalApi() { }
}
```

Then filter at the command line:

```shell
dotnet test --filter "Category=Integration"    # Run only integration tests
dotnet test --filter "Category!=Integration"   # Exclude integration tests
```

## Diagnostic Output with ITestOutputHelper

`Console.WriteLine` is swallowed by the test runner. Use `ITestOutputHelper` instead — especially valuable for debugging CI failures:

```csharp
public class OrderTests
{
    private readonly ITestOutputHelper _output;

    public OrderTests(ITestOutputHelper output) => _output = output;

    [Fact]
    public void ProcessOrder()
    {
        _output.WriteLine("Processing order with ID: 42");
        // ...
    }
}
```

## Customising Test Display Names

By default, xUnit shows the method name. Override it with `DisplayName`:

```csharp
[Fact(DisplayName = "1 + 1 = 2")]
public void Test_that_1_plus_1_eq_2()
{
    Assert.Equal(2, 1 + 1);
}
```

Or configure it project-wide with `xunit.runner.json` at the test project root:

```json
{
  "$schema": "https://xunit.net/schema/current/xunit.runner.schema.json",
  "methodDisplay": "method",
  "methodDisplayOptions": "replaceUnderscoreWithSpace,useOperatorMonikers,useEscapeSequences"
}
```

With this config, `Test_that_1_X2B_1_eq_3` displays as `Test that 1 + 1 = 3`.

## Dynamically Skipping Tests

Create a custom attribute to conditionally skip tests based on the environment:

```csharp
public sealed class IgnoreOnWindowsFactAttribute : FactAttribute
{
    public IgnoreOnWindowsFactAttribute()
    {
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
        {
            Skip = "Ignored on Windows";
        }
    }
}

public class PlatformTests
{
    [IgnoreOnWindowsFact]
    public void UnixOnlyTest() { }
}
```

## Multi-Targeting

Run tests against multiple .NET versions by editing the project file:

```xml
<PropertyGroup>
    <TargetFrameworks>net8.0;net9.0</TargetFrameworks>
</PropertyGroup>
```

```shell
dotnet test                    # Runs against all targets
dotnet test --framework net8.0 # Runs against a specific target
```

## What's Next?

xUnit gives you a solid foundation, but the built-in `Assert` API can be verbose and produce unhelpful failure messages. In [Part 2](2026-03-dotnet-testing-part2-fluent-assertions.md), we'll add **Fluent Assertions** to make tests read like English and get much better error output when things fail.

## References

- [xUnit.net documentation](https://xunit.net/)
- [xUnit GitHub repository](https://github.com/xunit/xunit)
- [.NET testing best practices](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
