---
layout: post
title: ".NET Testing — Part 2: Writing Readable Tests with Fluent Assertions"
date: 2026-03-12 00:00:00 +0000
tags:
  - dotnet
  - testing
  - fluent-assertions
  - csharp
  - unit-testing
series: ".NET Testing"
series_part: 2
---


## Series Overview

This is a 3-part series on testing .NET applications:

1. [Getting Started with xUnit.net](2026-03-dotnet-testing-part1-xunit.md) — Project setup, writing tests, data-driven tests, fixtures, parallelism, and advanced features
2. **Writing Readable Tests with Fluent Assertions** (this article) — Natural-language assertions, clearer failure messages, and custom assertions
3. [Integration Testing with Testcontainers](2026-03-dotnet-testing-part3-testcontainers.md) — Testing against real databases using Docker containers and CI setup


## Why Fluent Assertions?

It's not enough to know how to write tests — you need to write *readable* tests. When a test fails six months from now, the assertion should immediately tell you what went wrong and what was expected.

Fluent Assertions is a library that gives you three things:

1. **Readable assertions** that chain naturally and read like English
2. **Clearer failure messages** that include variable names and context
3. **A rich API** with more assertion methods than xUnit's built-in `Assert`

## Installation

```shell
dotnet add package FluentAssertions
```

## The Readability Difference

Consider this standard xUnit assertion block:

```csharp
Assert.Equal("9780321146533", book.ISBN);
Assert.Equal('3', book.ISBNCheckDigit);
Assert.Equal("Kent Beck", book.Author);
Assert.Equal("Test Driven Development: By Example", book.Name);
```

Now with Fluent Assertions:

```csharp
book.ISBN.Should().Be("9780321146533");
book.ISBNCheckDigit.Should().Be('3');
book.Author.Should().Be("Kent Beck");
book.Name.Should().Be("Test Driven Development: By Example");
```

These read almost like English sentences: *"The book's ISBN should be 9780321146533."*

## Better Failure Messages

This is where Fluent Assertions really shines. A failing xUnit assertion:

```csharp
bool saveOperationResult = false;
Assert.True(saveOperationResult);
```

Produces:

```
Assert.True() Failure
Expected: True
Actual: False
```

The same test with Fluent Assertions:

```csharp
bool saveOperationResult = false;
saveOperationResult.Should().BeTrue();
```

Produces:

```
Expected saveOperationResult to be true, but found False.
```

The variable name is included automatically. You can add even more context with the `because` parameter:

```csharp
int itemCount = 5;
itemCount.Should().Be(10, "because the cart should contain both user's and gift items");
```

```
Expected itemCount to be 10 because the cart should contain both user's and gift items, but found 5.
```

## Assertion Reference

### Basic — All Types

```csharp
sut.Should().BeNull();
sut.Should().NotBeNull();
sut.Should().BeOfType<Customer>();
sut.Should().Be(otherCustomer);
```

### Strings

**Null / empty checks:**

```csharp
theString.Should().BeNull();
theString.Should().NotBeNull();
theString.Should().BeEmpty();
theString.Should().NotBeEmpty("because the string is not empty");
theString.Should().HaveLength(5);
theString.Should().BeNullOrWhiteSpace();
theString.Should().NotBeNullOrWhiteSpace();
```

**Casing:**

```csharp
theString.Should().BeUpperCased();
theString.Should().BeLowerCased();
```

**Content:**

```csharp
theString.Should().Be("exact match");
theString.Should().BeEquivalentTo("CASE INSENSITIVE MATCH");
theString.Should().BeOneOf("option1", "option2");

theString.Should().Contain("substring");
theString.Should().Contain("x", Exactly.Once());
theString.Should().Contain("x", AtLeast.Twice());
theString.Should().ContainAll("must", "have", "all");
theString.Should().ContainAny("any", "of", "these");
theString.Should().NotContain("nope");

theString.Should().StartWith("prefix");
theString.Should().EndWith("suffix");
theString.Should().StartWithEquivalentOf("PREFIX"); // case-insensitive
```

**Pattern matching:**

```csharp
emailAddress.Should().Match("*@*.com");          // wildcard
someString.Should().MatchRegex("h.*\\sworld.$"); // regex
```

### Booleans

```csharp
theBoolean.Should().BeTrue();
theBoolean.Should().BeFalse("it's set to false");
theBoolean.Should().Be(otherBoolean);
```

### Numeric Types

```csharp
number.Should().Be(42);
number.Should().BePositive();
number.Should().BeNegative();
number.Should().BeGreaterThan(10);
number.Should().BeLessThanOrEqualTo(100);
number.Should().BeInRange(1, 100);
```

### Dates and Times

Fluent Assertions includes expressive date builders:

```csharp
var theDatetime = 1.March(2010).At(22, 15).AsLocal();

theDatetime.Should().Be(1.March(2010).At(22, 15));
theDatetime.Should().BeAfter(1.February(2010));
theDatetime.Should().BeBefore(2.March(2010));
theDatetime.Should().BeSameDateAs(1.March(2010).At(22, 16));

// Relative comparisons
theDatetime.Should().BeLessThan(10.Minutes()).Before(otherDatetime);
theDatetime.Should().BeWithin(2.Hours()).After(otherDatetime);
theDatetime.Should().BeExactly(24.Hours()).Before(appointment);

// Parts
theDatetime.Should().HaveYear(2010);
theDatetime.Should().HaveMonth(3);
theDatetime.Should().HaveDay(1);
theDatetime.Should().HaveHour(22);
```

### Object Equivalence

`BeEquivalentTo` compares property values — the objects don't even need to be the same type. This is perfect for comparing domain models to DTOs:

```csharp
customer.Should().BeEquivalentTo(customerDto);
```

Key distinction:
- `Be()` uses `Object.Equals()` — reference equality by default
- `BeEquivalentTo()` compares property values — structural equality

### Collections

```csharp
collection.Should().NotBeEmpty();
collection.Should().HaveCount(3);
collection.Should().OnlyHaveUniqueItems();

collection.Should().Equal("first", "second", "third");        // order matters
collection.Should().BeEquivalentTo("third", "first", "second"); // order doesn't matter

collection.Should().Contain("first").And.HaveElementAt(2, "third");
collection.Should().ContainInOrder("first", "second", "third");
collection.Should().StartWith("first");
collection.Should().EndWith("third");

collection.Should().BeInAscendingOrder();
collection.Should().BeInDescendingOrder();
```

### Exceptions

```csharp
// Synchronous
Action act = () => sut.BadMethod();
act.Should().Throw<ArgumentException>();
act.Should().NotThrow<NullReferenceException>();

// Asynchronous
Func<Task> act = () => sut.BadMethodAsync();
await act.Should().ThrowAsync<ArgumentNullException>();
await act.Should().NotThrowAsync();
```

## Writing Custom Assertions

You can extend Fluent Assertions with domain-specific assertions. For example, validating Australian mobile numbers:

```csharp
public static class CustomAssertions
{
    public static void BeAValidMobileNumber(
        this StringAssertions assertions,
        string because = "the string should be a valid mobile number",
        params object[] becauseArgs)
    {
        var regex = new Regex(
            @"^(\+?\(61\)|\(\+?61\)|\+?61|\(0[1-9]\)|0[1-9])?( ?-?[0-9]){7,9}$");

        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(regex.IsMatch(assertions.Subject))
            .FailWith(
                "Expected {context:string} to be a valid mobile number{reason}, " +
                "but found {0} is not valid.",
                assertions.Subject);
    }
}
```

Use it like any other assertion:

```csharp
[Fact]
public void ValidMobileNumber_ShouldPass()
{
    string mobile = "+61412345678";
    mobile.Should().BeAValidMobileNumber(
        "because it is a valid Australian mobile number");
}
```

The pattern is:

1. Create a static extension method on the appropriate `*Assertions` class (e.g., `StringAssertions`, `NumericAssertions<T>`)
2. Use `Execute.Assertion` to build the failure message with `BecauseOf` and `FailWith`
3. Use `{context:string}` to include the variable name and `{reason}` to include the "because" text

## Practical Tips

1. **Always add `because` for non-obvious assertions** — your future self will thank you when a CI build fails at 2 AM
2. **Use `BeEquivalentTo` for DTOs and API responses** — it's more forgiving than `Equal` and handles type differences gracefully
3. **Chain assertions** with `.And` for cleaner tests: `result.Should().NotBeNull().And.BeOfType<Order>()`
4. **Don't over-assert** — one concept per test. Fluent Assertions makes it tempting to chain everything, but focused tests are easier to debug

## What's Next?

Fluent Assertions makes your unit tests readable and your failure messages actionable. But what about integration tests that need real infrastructure? In [Part 3](2026-03-dotnet-testing-part3-testcontainers.md), we'll use **Testcontainers** to spin up real PostgreSQL databases in Docker for true integration testing — no mocks required.

## References

- [Fluent Assertions documentation](https://fluentassertions.com/introduction)
- [Fluent Assertions GitHub](https://github.com/fluentassertions/fluentassertions)
