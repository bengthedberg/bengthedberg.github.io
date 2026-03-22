---
layout: post
title: ".NET Testing — Part 3: Integration Testing with Testcontainers"
date: 2026-03-19 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - testing
  - testcontainers
  - docker
  - integration-testing
  - postgresql
  - csharp
series: ".NET Testing"
series_part: 3
---


## Series Overview

This is a 3-part series on testing .NET applications:

1. [Getting Started with xUnit.net](/posts/dotnet-testing-part1-xunit/) — Project setup, writing tests, data-driven tests, fixtures, parallelism, and advanced features
2. [Writing Readable Tests with Fluent Assertions](/posts/dotnet-testing-part2-fluent-assertions/) — Natural-language assertions, clearer failure messages, and custom assertions
3. **Integration Testing with Testcontainers** (this article) — Testing against real databases using Docker containers and CI setup


## The Problem with Mocking Infrastructure

Unit tests with mocked dependencies are fast and isolated, but they come with a fundamental risk: **your mocks might not behave like the real thing**. A mocked database won't catch SQL syntax errors, constraint violations, or driver-specific quirks. In-memory database providers skip features like stored procedures, indexes, and concurrency.

Testcontainers solves this by giving you disposable Docker containers that run real services. Your tests talk to the same PostgreSQL, Redis, or RabbitMQ that runs in production — no mocks, no in-memory substitutes.

## What Is Testcontainers?

[Testcontainers](https://testcontainers.com/) provides lightweight APIs for bootstrapping integration tests with real services wrapped in Docker containers. The .NET library integrates directly with xUnit's lifecycle, so containers start before your tests and are cleaned up after.

## Building a Complete Example

Let's build a small service that stores customers in PostgreSQL, then write an integration test that runs against a real database.

### Step 1: Create the Solution

```shell
dotnet new sln -o TestcontainersDemo
cd TestcontainersDemo

# Application project
dotnet new classlib -o CustomerService
dotnet sln add ./CustomerService/CustomerService.csproj

# Test project
dotnet new xunit -o CustomerService.Tests
dotnet sln add ./CustomerService.Tests/CustomerService.Tests.csproj
dotnet add ./CustomerService.Tests/CustomerService.Tests.csproj reference ./CustomerService/CustomerService.csproj

# Database driver
dotnet add ./CustomerService/CustomerService.csproj package Npgsql
```

### Step 2: Define the Domain Model

```csharp
namespace CustomerService.Model;

public readonly record struct Customer(long Id, string Name);
```

### Step 3: Create a Database Connection Provider

```csharp
using System.Data.Common;
using Npgsql;

namespace CustomerService.Database;

public sealed class DbConnectionProvider
{
    private readonly string _connectionString;

    public DbConnectionProvider(string connectionString)
    {
        _connectionString = connectionString;
    }

    public DbConnection GetConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }
}
```

### Step 4: Implement the Service

```csharp
using CustomerService.Database;
using CustomerService.Model;

namespace CustomerService;

public sealed class CustomerService
{
    private readonly DbConnectionProvider _dbConnectionProvider;

    public CustomerService(DbConnectionProvider dbConnectionProvider)
    {
        _dbConnectionProvider = dbConnectionProvider;
        CreateCustomersTable();
    }

    public IEnumerable<Customer> GetCustomers()
    {
        IList<Customer> customers = new List<Customer>();

        using var connection = _dbConnectionProvider.GetConnection();
        using var command = connection.CreateCommand();
        command.CommandText = "SELECT id, name FROM customers";
        command.Connection?.Open();

        using var dataReader = command.ExecuteReader();
        while (dataReader.Read())
        {
            var id = dataReader.GetInt64(0);
            var name = dataReader.GetString(1);
            customers.Add(new Customer(id, name));
        }

        return customers;
    }

    public void Create(Customer customer)
    {
        using var connection = _dbConnectionProvider.GetConnection();
        using var command = connection.CreateCommand();

        var id = command.CreateParameter();
        id.ParameterName = "@id";
        id.Value = customer.Id;

        var name = command.CreateParameter();
        name.ParameterName = "@name";
        name.Value = customer.Name;

        command.CommandText = "INSERT INTO customers (id, name) VALUES(@id, @name)";
        command.Parameters.Add(id);
        command.Parameters.Add(name);
        command.Connection?.Open();
        command.ExecuteNonQuery();
    }

    private void CreateCustomersTable()
    {
        using var connection = _dbConnectionProvider.GetConnection();
        using var command = connection.CreateCommand();
        command.CommandText = "CREATE TABLE IF NOT EXISTS customers " +
            "(id BIGINT NOT NULL, name VARCHAR NOT NULL, PRIMARY KEY (id))";
        command.Connection?.Open();
        command.ExecuteNonQuery();
    }
}
```

The service handles its own schema creation with `CREATE TABLE IF NOT EXISTS`. In a real application you'd use migrations, but this keeps the example focused.

## Writing the Integration Test

Add the Testcontainers PostgreSQL module to the test project:

```shell
dotnet add ./CustomerService.Tests/CustomerService.Tests.csproj package Testcontainers.PostgreSql
```

Now write the test:

```csharp
using CustomerService.Database;
using CustomerService.Model;
using Testcontainers.PostgreSql;

namespace CustomerService.Tests;

public sealed class CustomerServiceTest : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:15-alpine")
        .Build();

    public Task InitializeAsync()
    {
        return _postgres.StartAsync();
    }

    public Task DisposeAsync()
    {
        return _postgres.DisposeAsync().AsTask();
    }

    [Fact]
    public void ShouldReturnTwoCustomers()
    {
        // Given
        var customerService = new CustomerService(
            new DbConnectionProvider(_postgres.GetConnectionString()));

        // When
        customerService.Create(new Customer(1, "George"));
        customerService.Create(new Customer(2, "John"));
        var customers = customerService.GetCustomers();

        // Then
        Assert.Equal(2, customers.Count());
    }
}
```

### How It Works

1. **`PostgreSqlBuilder`** declares a Postgres container using the `postgres:15-alpine` Docker image
2. **`IAsyncLifetime.InitializeAsync`** starts the container before any tests run — Testcontainers pulls the image automatically if it's not cached locally
3. **`_postgres.GetConnectionString()`** returns a connection string pointing to the ephemeral container with a random port
4. **`IAsyncLifetime.DisposeAsync`** stops and removes the container after all tests complete

### Running the Test

Make sure Docker is running, then:

```shell
dotnet test
```

You'll see output like:

```
[testcontainers.org 00:00:00.04] Connected to Docker:
  Host: unix:///var/run/docker.sock
  Server Version: 27.1.1
  Operating System: Docker Desktop

[testcontainers.org 00:00:00.11] Docker container 220c2681e3e0 created
[testcontainers.org 00:00:00.13] Start Docker container 220c2681e3e0
[testcontainers.org 00:00:01.45] Docker container a26577c95780 ready

Passed!  - Failed: 0, Passed: 1, Skipped: 0, Total: 1
```

## Using Fixtures for Performance

Starting a new container per test class adds overhead. For tests that can share a database, use `IClassFixture<T>` (covered in [Part 1](/posts/dotnet-testing-part1-xunit/)):

```csharp
public sealed class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:15-alpine")
        .Build();

    public string ConnectionString => _postgres.GetConnectionString();

    public Task InitializeAsync() => _postgres.StartAsync();
    public Task DisposeAsync() => _postgres.DisposeAsync().AsTask();
}

public class CustomerTests : IClassFixture<PostgresFixture>
{
    private readonly PostgresFixture _fixture;
    public CustomerTests(PostgresFixture fixture) => _fixture = fixture;

    [Fact]
    public void ShouldCreateCustomer()
    {
        var sut = new CustomerService(
            new DbConnectionProvider(_fixture.ConnectionString));
        // ...
    }
}
```

## Setting Up CI with GitHub Actions

Testcontainers works out of the box in CI environments that have Docker. For even faster execution, you can use [Testcontainers Cloud](https://testcontainers.com/cloud/), which offloads container execution to a managed service.

### GitHub Actions Workflow

```yaml
name: Build and Test
on:
  push:
  pull_request:
    branches: [main]
    paths-ignore:
      - "README.md"
      - ".gitignore"

env:
  DOTNET_VERSION: "8.0.x"

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Testcontainers Cloud Client
        uses: atomicjar/testcontainers-cloud-setup-action@v1
        with:
          token: ${{ secrets.TC_CLOUD_TOKEN }}

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Install dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: >
          dotnet test --no-restore --verbosity normal
          --collect:"XPlat Code Coverage"
          --logger trx
          --results-directory coverage

      - name: Code Coverage Summary
        uses: irongut/CodeCoverageSummary@v1.3.0
        with:
          filename: "coverage/*/coverage.cobertura.xml"
          badge: true
          format: "markdown"
          output: "both"

      - name: Add Coverage PR Comment
        uses: marocchino/sticky-pull-request-comment@v2
        if: github.event_name == 'pull_request'
        with:
          recreate: true
          path: code-coverage-results.md

      - name: Write to Job Summary
        run: cat code-coverage-results.md >> $GITHUB_STEP_SUMMARY

      - name: Terminate Testcontainers Cloud sessions
        uses: atomicjar/testcontainers-cloud-setup-action@v1
        with:
          action: terminate
```

Key points:
- **Testcontainers Cloud** is optional but speeds up CI by running containers remotely — sign up for a free account and store the token as `TC_CLOUD_TOKEN`
- **Code coverage** is collected via `coverlet.collector` (already included in xUnit projects) and summarized as a PR comment
- The Testcontainers Cloud session is terminated at the end to free resources

## When to Use Testcontainers vs Mocks

| Use Testcontainers when... | Use mocks when... |
|----------------------------|-------------------|
| Testing database queries and migrations | Testing business logic in isolation |
| Verifying SQL constraints and indexes | Testing code that doesn't touch infrastructure |
| Integration testing API endpoints | Unit testing with fast feedback loops |
| Validating message queue consumers | Testing error handling for specific scenarios |

The best test suites use both: fast unit tests with mocks for business logic, and Testcontainers for integration tests that verify the real infrastructure behaves as expected.

## Series Recap

Across this 3-part series, we've built a complete .NET testing toolkit:

1. **[xUnit.net](/posts/dotnet-testing-part1-xunit/)** — The foundation: test structure, data-driven tests, fixtures, parallel execution
2. **[Fluent Assertions](/posts/dotnet-testing-part2-fluent-assertions/)** — Readable assertions with better failure messages
3. **Testcontainers** — Real infrastructure in your test suite, from local development to CI

Together, these tools let you write tests that are readable, reliable, and actually test what matters.

## References

- [Testcontainers for .NET](https://dotnet.testcontainers.org/)
- [Testcontainers PostgreSQL module](https://dotnet.testcontainers.org/modules/postgres/)
- [Testcontainers Cloud](https://testcontainers.com/cloud/)
- [Npgsql — .NET PostgreSQL driver](https://www.npgsql.org/)
