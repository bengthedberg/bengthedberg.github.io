---
layout: post
title: "Building AWS Applications with .NET 10: A Clean Architecture Template"
date: 2026-01-08 00:00:00 +0000
tags:
  - aws
  - dotnet
  - cdk
  - clean-architecture
  - lambda
  - github-actions
  - cicd
---


## Introduction

This guide walks through building a production-ready AWS application using .NET 10 and Clean Architecture. By the end, you'll have a solution template that includes:

- **Infrastructure as Code** with AWS CDK
- **Clean Architecture** — domain, infrastructure, and Lambda layers
- **CI/CD** with GitHub Actions and OIDC authentication
- **Local debugging** for Lambda functions
- **Code quality** enforcement with `.editorconfig` and `dotnet format`
- **Project templates** so you can scaffold new solutions instantly

### Principles

- Infrastructure as code using [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- Domain-centric architecture using [Clean Architecture](https://ardalis.com/clean-architecture-asp-net-core/)
- Consistent coding styles enforced by [.editorconfig](https://editorconfig.org/) and [dotnet format](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-format)
- Source control with [GitHub](https://github.com) and [trunk-based development](https://trunkbaseddevelopment.com/)
- CI/CD as code using [GitHub Actions](https://docs.github.com/en/actions/using-workflows/about-workflows)
- Local Lambda testing with [SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-test-and-debug.html)
- Reusable [project templates](https://learn.microsoft.com/en-us/dotnet/core/tools/custom-templates) for new solutions


## 1. Initial Structure

Create the solution and folder structure:

```bash
mkdir AWS.Application
cd AWS.Application
dotnet new sln
mkdir src tests docs
dotnet new gitignore
```

Initialise the repository:

```bash
git init --initial-branch=main
git add .
git commit -m 'initial solution'
```


## 2. Infrastructure as Code with CDK

### Create the CDK Project

The `cdk init` command must run in an empty directory:

```bash
mkdir src/AWS.Application.CDK
cd src/AWS.Application.CDK
cdk init --generate-only app --language csharp
cd ../..
```

Restructure to fit our solution layout:

```bash
mv src/AWS.Application.CDK/.gitignore .
mv src/AWS.Application.CDK/cdk.json .
mv src/AWS.Application.CDK/src/AwsApplicationCdk/* src/AWS.Application.CDK/
rm -rf src/AWS.Application.CDK/src

# Rename files
mv src/AWS.Application.CDK/AwsApplicationCdk.csproj src/AWS.Application.CDK/AWS.Application.CDK.csproj
mv src/AWS.Application.CDK/AwsApplicationCdkStack.cs src/AWS.Application.CDK/ApplicationStack.cs

# Fix namespaces and references
find . -name "cdk.json" -exec sed -i '' 's/AwsApplicationCdk/AWS.Application.CDK/g' {} +
find . -name "*.cs" -exec sed -i '' 's/AwsApplicationCdkStack/ApplicationStack/g' {} +
find . -name "*.cs" -exec sed -i '' 's/namespace AwsApplicationCdk/namespace AWS.Application.CDK/g' {} +

# Add all projects to solution
dotnet sln add $(find . -name "*.csproj")
```

### Add Configuration

Create `src/AWS.Application.CDK/appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.Hosting.Lifetime": "Warning"
    }
  },
  "AWS": {
    "AccountId": "#{AWS_ACCOUNT_ID}",
    "Region": "#{AWS_REGION}"
  },
  "Debug": "false",
  "DumpConfig": "false",
  "App": {
    "Name": "shared",
    "Project": "AWS.Application",
    "Department": "",
    "Company": "",
    "Version": "1.0.0",
    "CostCenter": "n/a",
    "Environment": "sbx"
  }
}
```

Include the config file in build output. Add to `src/AWS.Application.CDK/AWS.Application.CDK.csproj`:

```xml
<ItemGroup>
  <None Update="appsettings.json">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </None>
</ItemGroup>
```

### ApplicationStack with Tagging

Update `src/AWS.Application.CDK/ApplicationStack.cs` to apply consistent resource tags:

```csharp
using Amazon.CDK;
using Constructs;
using Microsoft.Extensions.Configuration;

namespace AWS.Application.CDK;

public class ApplicationStack(Construct scope, string id, IStackProps props, IConfiguration config)
    : Stack(scope, id, props)
{
    // Called after construction to add constructs and tags
    public ApplicationStack Build()
    {
        // Add service constructs here

        // Tag all resources in the stack
        Tags.Of(this).Add("project", config["App:Project"]);
        Tags.Of(this).Add("department", config["App:Department"]);
        Tags.Of(this).Add("company", config["App:Company"]);
        Tags.Of(this).Add("application", config["App:Name"]);
        Tags.Of(this).Add("costCentre", config["App:CostCenter"]);
        Tags.Of(this).Add("version", config["App:Version"]);
        Tags.Of(this).Add("environment", config["App:Environment"]);

        return this;
    }
}
```

### Program.cs with Configuration

Update `src/AWS.Application.CDK/Program.cs`:

```csharp
using System.Diagnostics;
using Amazon.CDK;
using Microsoft.Extensions.Configuration;
using Environment = Amazon.CDK.Environment;

var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false)
    .AddEnvironmentVariables()
    .Build();

if (config.GetValue<bool>("Debug"))
    Debugger.Launch();

if (config.GetValue<bool>("DumpConfig"))
    Console.WriteLine(config.GetDebugView());

var app = new App();

var env = new Environment
{
    Account = GetConfigOrEnv(config["AWS:AccountId"], "#{AWS_ACCOUNT_ID}",
        "CDK_DEPLOY_ACCOUNT", "CDK_DEFAULT_ACCOUNT"),
    Region = GetConfigOrEnv(config["AWS:Region"], "#{AWS_REGION}",
        "CDK_DEPLOY_REGION", "CDK_DEFAULT_REGION")
};

new ApplicationStack(app, "ApplicationStack", new StackProps { Env = env }, config)
    .Build();

app.Synth();

static string? GetConfigOrEnv(string? configValue, string placeholder, params string[] envVars)
{
    if (!string.IsNullOrEmpty(configValue) && !configValue.Equals(placeholder, StringComparison.OrdinalIgnoreCase))
        return configValue;

    foreach (var envVar in envVars)
    {
        var value = System.Environment.GetEnvironmentVariable(envVar);
        if (!string.IsNullOrEmpty(value)) return value;
    }
    return null;
}
```

> **TIP**: Set `Debug` to `true` in `appsettings.json` to attach a debugger during CDK synthesis. Set `DumpConfig` to `true` to print all resolved configuration values.

### Install Required Packages

```bash
dotnet add src/AWS.Application.CDK package Microsoft.Extensions.Configuration.Json
dotnet add src/AWS.Application.CDK package Microsoft.Extensions.Hosting
```

Verify it builds:

```bash
dotnet build
```

### CDK Unit Tests

```bash
dotnet new xunit -f net10.0 -o tests/AWS.Application.CDK.UnitTests
dotnet sln add tests/AWS.Application.CDK.UnitTests
dotnet add tests/AWS.Application.CDK.UnitTests reference src/AWS.Application.CDK
dotnet add tests/AWS.Application.CDK.UnitTests package NSubstitute
dotnet add tests/AWS.Application.CDK.UnitTests package Amazon.CDK.Lib
dotnet add tests/AWS.Application.CDK.UnitTests package Constructs
```

Add `InternalsVisibleTo` in `src/AWS.Application.CDK/AWS.Application.CDK.csproj`:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="$(AssemblyName).UnitTests" />
</ItemGroup>
```

Create `tests/AWS.Application.CDK.UnitTests/ApplicationStackTest.cs`:

```csharp
using Amazon.CDK;
using NSubstitute;
using Microsoft.Extensions.Configuration;
using Environment = Amazon.CDK.Environment;

namespace AWS.Application.CDK.Tests;

public class ApplicationStackTest
{
    private readonly IConfiguration _configuration = Substitute.For<IConfiguration>();

    [Fact]
    public void EnsureStackHasProperties()
    {
        _configuration["App:Environment"].Returns("dev");
        _configuration["App:Project"].Returns("project");
        _configuration["App:Department"].Returns("department");
        _configuration["App:Company"].Returns("company");
        _configuration["App:Name"].Returns("name");
        _configuration["App:CostCenter"].Returns("costcenter");
        _configuration["App:Version"].Returns("1.0.0");

        var app = new App();
        var env = new Environment
        {
            Account = "1234567890",
            Region = "ap-southeast-2"
        };

        var stack = new ApplicationStack(app, "ApplicationStack",
            new StackProps { Env = env }, _configuration).Build();

        Assert.Equal("ApplicationStack", stack.StackName);
        Assert.Equal("1234567890", stack.Account);
        Assert.Equal("ap-southeast-2", stack.Region);
    }
}
```

```bash
dotnet test
git add .
git commit -m 'add cdk project with corresponding unit tests'
```


## 3. Domain Layer (Clean Architecture)

The Core project contains domain logic with no dependencies on infrastructure or AWS.

```bash
dotnet new classlib -f net10.0 -o src/AWS.Application.Core
dotnet new xunit -f net10.0 -o tests/AWS.Application.Core.UnitTests
dotnet add tests/AWS.Application.Core.UnitTests reference src/AWS.Application.Core
dotnet sln add $(find . -name "*.csproj")
```

Add `InternalsVisibleTo` to `src/AWS.Application.Core/AWS.Application.Core.csproj`:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="$(AssemblyName).UnitTests" />
</ItemGroup>
```

Create the domain structure:

```bash
mkdir -p src/AWS.Application.Core/{DTOs,Entities,Enums,Exceptions,Interfaces,Services}
rm src/AWS.Application.Core/Class1.cs
```

### DTOs

`src/AWS.Application.Core/DTOs/ProcessDataRequest.cs`:

```csharp
namespace AWS.Application.Core.DTOs;

public record ProcessDataRequest
{
    public string ProcessDate { get; init; } = string.Empty;
    public string ProcessId { get; init; } = string.Empty;
}
```

`src/AWS.Application.Core/DTOs/ProcessDataResponse.cs`:

```csharp
namespace AWS.Application.Core.DTOs;

public record ProcessDataResponse
{
    public string ProcessedDate { get; init; } = string.Empty;
    public int ProcessedRecords { get; init; }
}
```

### Entities

`src/AWS.Application.Core/Entities/DataItem.cs`:

```csharp
using AWS.Application.Core.Enums;

namespace AWS.Application.Core.Entities;

public class DataItem
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTimeOffset CreatedDate { get; set; }
    public PriorityStatus Status { get; set; }
}
```

### Enums

`src/AWS.Application.Core/Enums/PriorityStatus.cs`:

```csharp
namespace AWS.Application.Core.Enums;

public enum PriorityStatus
{
    Normal,
    Low,
    Urgent
}
```

### Exceptions

`src/AWS.Application.Core/Exceptions/DomainException.cs`:

```csharp
namespace AWS.Application.Core.Exceptions;

public class DomainException : Exception
{
    public DomainException(string? message) : base(message) { }
    public DomainException(string? message, Exception? innerException) : base(message, innerException) { }
    private DomainException() { }
}
```

### Interfaces

`src/AWS.Application.Core/Interfaces/IDataService.cs`:

```csharp
using AWS.Application.Core.Entities;

namespace AWS.Application.Core.Interfaces;

public interface IDataService
{
    Task<IEnumerable<DataItem>> GetAllDataItems(DateTimeOffset date);
}
```

`src/AWS.Application.Core/Interfaces/IDomainService.cs`:

```csharp
using AWS.Application.Core.DTOs;

namespace AWS.Application.Core.Interfaces;

public interface IDomainService
{
    Task<ProcessDataResponse> Process(ProcessDataRequest request);
}
```

### Services

`src/AWS.Application.Core/Services/DomainService.cs`:

```csharp
using AWS.Application.Core.DTOs;
using AWS.Application.Core.Exceptions;
using AWS.Application.Core.Interfaces;
using static System.Globalization.CultureInfo;

namespace AWS.Application.Core.Services;

public class DomainService(IDataService dataService) : IDomainService
{
    public async Task<ProcessDataResponse> Process(ProcessDataRequest request)
    {
        if (string.IsNullOrWhiteSpace(request.ProcessDate))
            throw new DomainException("Process date is null");

        var processDate = DateTimeOffset.ParseExact(request.ProcessDate, "yyyy-MM-dd", InvariantCulture);
        var items = await dataService.GetAllDataItems(processDate);

        return new ProcessDataResponse
        {
            ProcessedDate = request.ProcessDate,
            ProcessedRecords = items.Count()
        };
    }
}
```

### Domain Unit Tests

```bash
dotnet add tests/AWS.Application.Core.UnitTests package NSubstitute
rm tests/AWS.Application.Core.UnitTests/UnitTest1.cs
```

`tests/AWS.Application.Core.UnitTests/DomainServiceTests.cs`:

```csharp
using AWS.Application.Core.DTOs;
using AWS.Application.Core.Entities;
using AWS.Application.Core.Enums;
using AWS.Application.Core.Exceptions;
using AWS.Application.Core.Interfaces;
using AWS.Application.Core.Services;
using NSubstitute;

namespace AWS.Application.Core.UnitTests;

public class DomainServiceTests
{
    [Fact]
    public async Task DomainProcess_SuccessfulRequest()
    {
        var dataService = Substitute.For<IDataService>();
        dataService.GetAllDataItems(Arg.Any<DateTimeOffset>())
            .Returns(callInfo =>
            [
                new DataItem
                {
                    Id = 1,
                    Name = "MyItem",
                    CreatedDate = callInfo.Arg<DateTimeOffset>(),
                    Status = PriorityStatus.Normal
                }
            ]);

        var domainService = new DomainService(dataService);
        var result = await domainService.Process(new ProcessDataRequest
        {
            ProcessDate = "2026-01-31",
            ProcessId = "10ABC"
        });

        Assert.Equal("2026-01-31", result.ProcessedDate);
        Assert.Equal(1, result.ProcessedRecords);
    }

    [Fact]
    public async Task DomainProcess_InvalidDateException()
    {
        var dataService = Substitute.For<IDataService>();
        var domainService = new DomainService(dataService);

        await Assert.ThrowsAsync<DomainException>(() =>
            domainService.Process(new ProcessDataRequest { ProcessId = "10ABC" }));
    }
}
```

```bash
dotnet test
git add .
git commit -m 'add domain class library with unit tests'
```


## 4. Infrastructure Layer

The infrastructure project implements the interfaces defined in Core.

```bash
dotnet new classlib -f net10.0 -o src/AWS.Application.Infrastructure
dotnet sln add src/AWS.Application.Infrastructure
dotnet add src/AWS.Application.Infrastructure reference src/AWS.Application.Core
dotnet new xunit -f net10.0 -o tests/AWS.Application.Infrastructure.UnitTests
dotnet sln add tests/AWS.Application.Infrastructure.UnitTests
dotnet add tests/AWS.Application.Infrastructure.UnitTests reference src/AWS.Application.Infrastructure
dotnet sln add $(find . -name "*.csproj")
```

Add `InternalsVisibleTo` to `src/AWS.Application.Infrastructure/AWS.Application.Infrastructure.csproj`:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="$(AssemblyName).UnitTests" />
</ItemGroup>
```

Install packages:

```bash
dotnet add src/AWS.Application.Infrastructure package Microsoft.Extensions.Configuration
dotnet add src/AWS.Application.Infrastructure package Microsoft.Extensions.DependencyInjection
```

Setup:

```bash
mkdir src/AWS.Application.Infrastructure/Services
rm src/AWS.Application.Infrastructure/Class1.cs
```

### Data Service Implementation

`src/AWS.Application.Infrastructure/Services/DataService.cs`:

```csharp
using AWS.Application.Core.Interfaces;
using AWS.Application.Core.Entities;
using Microsoft.Extensions.Configuration;

namespace AWS.Application.Infrastructure.Services;

public class DataService(IConfiguration configuration) : IDataService
{
    // TODO: Replace with actual database/repository implementation
    internal static readonly Dictionary<int, DataItem> Items = [];
    internal readonly string TableName = configuration["TableName"] ?? string.Empty;

    public async Task<IEnumerable<DataItem>> GetAllDataItems(DateTimeOffset date)
    {
        var result = Items.Values.ToList();
        await Task.Delay(100);
        return result;
    }
}
```

### Dependency Injection Registration

`src/AWS.Application.Infrastructure/ConfigureServices.cs`:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using AWS.Application.Core.Interfaces;
using AWS.Application.Core.Services;
using AWS.Application.Infrastructure.Services;

namespace AWS.Application.Infrastructure;

public static class ConfigureServices
{
    public static IServiceCollection AddImplementation(
        this IServiceCollection services, IConfiguration configuration)
    {
        services.AddScoped<IDataService, DataService>();
        services.AddScoped<IDomainService, DomainService>();
        return services;
    }
}
```

### Infrastructure Unit Tests

```bash
dotnet add tests/AWS.Application.Infrastructure.UnitTests package NSubstitute
dotnet add tests/AWS.Application.Infrastructure.UnitTests package Microsoft.Extensions.Configuration
rm tests/AWS.Application.Infrastructure.UnitTests/UnitTest1.cs
```

`tests/AWS.Application.Infrastructure.UnitTests/DataServiceTests.cs`:

```csharp
using System.Globalization;
using NSubstitute;
using Microsoft.Extensions.Configuration;
using AWS.Application.Infrastructure.Services;

namespace AWS.Application.Infrastructure.UnitTests;

public class DataServiceTests
{
    private readonly IConfiguration _configuration = Substitute.For<IConfiguration>();

    public DataServiceTests()
    {
        _configuration["TableName"].Returns("TestDataTable");
    }

    [Fact]
    public void DataService_Configured_Success()
    {
        var dataService = new DataService(_configuration);
        Assert.Equal("TestDataTable", dataService.TableName);
    }

    [Fact]
    public async Task DataService_Process_Success()
    {
        var dataService = new DataService(_configuration);
        var response = await dataService.GetAllDataItems(
            DateTimeOffset.ParseExact("2026-05-03", "yyyy-MM-dd", CultureInfo.InvariantCulture));
        Assert.Empty(response);
    }
}
```

```bash
dotnet test
git add .
git commit -m 'add infrastructure project with unit tests'
```


## 5. Adding a Lambda Function

### Create the Lambda Project

```bash
cd src
dotnet new lambda.EmptyFunction --name AWS.Application.Lambda.Simple --region ap-southeast-2 --profile default
cd ..
```

Restructure to match our layout:

```bash
mv src/AWS.Application.Lambda.Simple/src/AWS.Application.Lambda.Simple/* src/AWS.Application.Lambda.Simple/
rm -rf src/AWS.Application.Lambda.Simple/src
mkdir -p tests/AWS.Application.Lambda.Simple.Tests
mv src/AWS.Application.Lambda.Simple/test/AWS.Application.Lambda.Simple.Tests/* tests/AWS.Application.Lambda.Simple.Tests/
rm -rf src/AWS.Application.Lambda.Simple/test
dotnet sln add $(find . -name "*.csproj")
```

### Wire Up Domain Logic

Add references and packages:

```bash
dotnet add src/AWS.Application.Lambda.Simple reference src/AWS.Application.Core
dotnet add src/AWS.Application.Lambda.Simple reference src/AWS.Application.Infrastructure
dotnet add src/AWS.Application.Lambda.Simple package Microsoft.Extensions.Configuration
dotnet add src/AWS.Application.Lambda.Simple package Microsoft.Extensions.Configuration.Json
dotnet add src/AWS.Application.Lambda.Simple package Microsoft.Extensions.Configuration.EnvironmentVariables
```

`src/AWS.Application.Lambda.Simple/Startup.cs`:

```csharp
using System.Reflection;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using AWS.Application.Infrastructure;

namespace AWS.Application.Lambda.Simple;

public static class Startup
{
    public static ServiceProvider? Services { get; private set; }

    public static void ConfigureServices()
    {
        IConfiguration configuration = new ConfigurationBuilder()
            .AddEnvironmentVariables()
            .SetBasePath(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location) ?? string.Empty)
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .Build();

        var serviceCollection = new ServiceCollection();
        serviceCollection.AddSingleton(configuration);
        serviceCollection.AddImplementation(configuration);

        Services = serviceCollection.BuildServiceProvider();
    }
}
```

`src/AWS.Application.Lambda.Simple/Function.cs`:

```csharp
using System.Runtime.CompilerServices;
using Amazon.Lambda.Core;
using AWS.Application.Core.DTOs;
using AWS.Application.Core.Interfaces;
using Microsoft.Extensions.DependencyInjection;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
[assembly: InternalsVisibleTo("AWS.Application.Lambda.Simple.Tests")]

namespace AWS.Application.Lambda.Simple;

public class Function
{
    private readonly IDomainService? _client;

    // Lambda runtime requires a parameterless constructor
    public Function() : this(null) { }

    // Internal constructor for unit testing with mocked services
    internal Function(IDomainService? client = null)
    {
        Startup.ConfigureServices();
        _client = client ?? Startup.Services?.GetService<IDomainService>();
    }

    public async Task<ProcessDataResponse> FunctionHandler(ProcessDataRequest input, ILambdaContext context)
    {
        return await _client!.Process(input);
    }
}
```

### Lambda Unit Tests

```bash
dotnet add tests/AWS.Application.Lambda.Simple.Tests package NSubstitute
dotnet add tests/AWS.Application.Lambda.Simple.Tests package Microsoft.Extensions.Configuration
```

`tests/AWS.Application.Lambda.Simple.Tests/FunctionTest.cs`:

```csharp
using Amazon.Lambda.TestUtilities;
using AWS.Application.Core.DTOs;
using AWS.Application.Core.Entities;
using AWS.Application.Core.Enums;
using AWS.Application.Core.Interfaces;
using AWS.Application.Core.Services;
using NSubstitute;

namespace AWS.Application.Lambda.Simple.Tests;

public class FunctionTest
{
    [Fact]
    public async Task SimpleLambda_Returns_Success()
    {
        var dataService = Substitute.For<IDataService>();
        dataService.GetAllDataItems(Arg.Any<DateTimeOffset>())
            .Returns(callInfo =>
            [
                new DataItem
                {
                    Id = 1,
                    Name = "MyItem",
                    CreatedDate = callInfo.Arg<DateTimeOffset>(),
                    Status = PriorityStatus.Normal
                }
            ]);

        var domainService = new DomainService(dataService);
        var function = new Function(domainService);
        var context = new TestLambdaContext();

        var result = await function.FunctionHandler(
            new ProcessDataRequest { ProcessDate = "2026-05-22", ProcessId = "123" },
            context);

        Assert.Equal("2026-05-22", result.ProcessedDate);
    }
}
```

```bash
dotnet test
git add .
git commit -m 'add lambda function with domain logic'
```

### CDK Construct for the Lambda

Create `src/AWS.Application.CDK/Constructs/SimpleLambdaConstruct.cs`:

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using Amazon.CDK.AWS.Logs;
using Constructs;
using Microsoft.Extensions.Configuration;
using AssetOptions = Amazon.CDK.AWS.S3.Assets.AssetOptions;

namespace AWS.Application.CDK.Constructs;

public class SimpleLambdaConstruct : Construct
{
    public SimpleLambdaConstruct(Construct scope, string id, IConfiguration config)
        : base(scope, id)
    {
        var environment = config["App:Environment"];

        var function = new Function(this, "MyFunction", new FunctionProps
        {
            FunctionName = $"simplelambda-{environment}",
            Runtime = Runtime.DOTNET_10,
            Handler = "AWS.Application.Lambda.Simple::AWS.Application.Lambda.Simple.Function::FunctionHandler",
            Description = "Simple Lambda function with domain logic",
            Timeout = Duration.Seconds(300),
            MemorySize = 512,
            Architecture = Architecture.ARM_64,
            LogRetention = RetentionDays.ONE_WEEK,
            Tracing = Tracing.PASS_THROUGH,
            Environment = new Dictionary<string, string>
            {
                ["ENVIRONMENT"] = environment ?? "dev"
            },
            ReservedConcurrentExecutions = 10,
            Code = Code.FromAsset("./", new AssetOptions
            {
                Bundling = new BundlingOptions
                {
                    Image = Runtime.DOTNET_10.BundlingImage,
                    Command =
                    [
                        "bash", "-c", string.Join(" && ",
                        [
                            "cd /asset-input",
                            "export DOTNET_CLI_HOME=\"/tmp/DOTNET_CLI_HOME\"",
                            "export PATH=\"$PATH:/tmp/DOTNET_CLI_HOME/.dotnet/tools\"",
                            "dotnet tool install -g Amazon.Lambda.Tools",
                            "dotnet lambda package -pl ./src/AWS.Application.Lambda.Simple -o output.zip -c Release -f net10.0 -farch arm64",
                            "unzip -o -d /asset-output output.zip"
                        ])
                    ]
                }
            })
        });
    }
}
```

Update `ApplicationStack.Build()` to include the construct:

```csharp
public ApplicationStack Build()
{
    _ = new SimpleLambdaConstruct(this, "Lambda", config);

    // Tag all resources...
}
```

Update the CDK unit test to skip bundling:

```csharp
app.Node.SetContext("aws:cdk:bundling-stacks", Array.Empty<string>());
```

### Local Lambda Debugging

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Lambda (macOS/Linux)",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "sln build",
      "program": "${env:HOME}/.dotnet/tools/dotnet-lambda-test-tool-10.0",
      "args": ["--port", "5050"],
      "cwd": "${workspaceFolder}/src/AWS.Application.Lambda.Simple",
      "console": "internalConsole",
      "stopAtEntry": false,
      "internalConsoleOptions": "openOnSessionStart"
    },
    {
      "name": "Debug Lambda (Windows)",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "sln build",
      "program": "${env:USERPROFILE}/.dotnet/tools/dotnet-lambda-test-tool-10.0.exe",
      "args": ["--port", "5050"],
      "cwd": "${workspaceFolder}/src/AWS.Application.Lambda.Simple",
      "console": "internalConsole",
      "stopAtEntry": false,
      "internalConsoleOptions": "openOnSessionStart"
    }
  ]
}
```

### Lambda Sandbox Deployment Config

`src/AWS.Application.Lambda.Simple/aws-lambda-tools-defaults.json`:

```json
{
  "profile": "default",
  "region": "ap-southeast-2",
  "configuration": "Release",
  "function-architecture": "arm64",
  "function-runtime": "dotnet10",
  "function-memory-size": 256,
  "function-timeout": 30,
  "function-name": "simplelambda-sbx",
  "function-handler": "AWS.Application.Lambda.Simple::AWS.Application.Lambda.Simple.Function::FunctionHandler"
}
```


## 6. CI/CD with GitHub Actions

### Build Workflow

`.github/workflows/build.yml`:

```yaml
name: Build

on:
  push:
    paths-ignore:
      - "**/*.md"
      - "**/*.gitignore"
      - "**/*.gitattributes"

jobs:
  cdk:
    name: "Validate CDK"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - uses: actions/setup-node@v4
        with:
          node-version: "22"

      - run: npm install -g aws-cdk
      - run: cdk synth --validation

  inspect:
    name: "Inspect Code"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - run: dotnet format --verify-no-changes

  test:
    name: "Test Application"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - run: dotnet restore
      - run: dotnet build --configuration Release --no-restore
      - run: dotnet test --configuration Release --no-build --logger "trx;LogFileName=test-results.trx" || true

      - name: Test Report
        uses: dorny/test-reporter@v1
        if: success() || failure()
        with:
          name: Test Results
          path: "**/test-results.trx"
          reporter: dotnet-trx
          fail-on-error: true
```

### Deploy Workflow

`.github/workflows/deploy.yml` — uses OIDC for keyless AWS authentication and a reusable job to avoid duplicating steps per environment:

```yaml
name: AWS CDK Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        description: Environment To Deploy
        options: [dev, uat, prd]
        required: true
      region:
        type: choice
        description: AWS Region
        options: [ap-southeast-2]
        required: true

run-name: Deploy to ${{ inputs.environment }} - ${{ inputs.region }}

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment == 'dev' && 'Development' || inputs.environment == 'uat' && 'UAT' || 'Production' }}
    # Block production deploys from non-main branches
    if: inputs.environment != 'prd' || github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Inject environment config
        run: |
          cd src/AWS.Application.CDK
          sed -i "s/#{AWS_ACCOUNT_ID}/${{ secrets.AWS_ACCOUNT_ID }}/g" appsettings.json
          sed -i "s/#{AWS_REGION}/${{ inputs.region }}/g" appsettings.json
          jq '.App.Environment = "${{ inputs.environment }}"' appsettings.json > tmp.json && mv tmp.json appsettings.json

      - run: dotnet build --configuration Release

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_IAM_ROLE }}
          aws-region: ${{ inputs.region }}

      - run: npm install -g aws-cdk
      - run: cdk synth
      - run: cdk deploy --require-approval never
```

> **Key improvement over the original**: This uses a single job with conditional environment mapping instead of duplicating the entire job three times. The `microsoft/variable-substitution` action (deprecated) is replaced with `sed` and `jq`.

### GitHub Repository Setup

Each environment needs these secrets configured:
- `AWS_IAM_ROLE` — the OIDC role ARN (see [Connecting GitHub Actions to AWS](../2025-12-connecting-github-actions-with-aws.md))
- `AWS_ACCOUNT_ID` — the target AWS account

Set up branch protection on `main`:
1. Require pull requests with at least 1 approval
2. Require status checks to pass (Build workflow)
3. Require branches to be up to date before merging
4. For Production: restrict deployment to `main` branch only


## 7. Developer Experience

### VS Code Tasks

`.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "sln build",
      "command": "dotnet",
      "type": "process",
      "group": { "kind": "build", "isDefault": true },
      "args": ["build", "${workspaceFolder}/AWS.Application.sln",
               "/property:GenerateFullPaths=true", "/consoleloggerparameters:NoSummary"],
      "problemMatcher": "$msCompile"
    },
    {
      "label": "sln test",
      "command": "dotnet",
      "type": "process",
      "group": { "kind": "test", "isDefault": true },
      "args": ["test", "${workspaceFolder}/AWS.Application.sln"],
      "problemMatcher": "$msCompile"
    },
    {
      "label": "sln format verify",
      "command": "dotnet",
      "type": "process",
      "args": ["format", "--verify-no-changes"],
      "problemMatcher": "$msCompile"
    },
    {
      "label": "sln format",
      "command": "dotnet",
      "type": "process",
      "args": ["format"],
      "problemMatcher": "$msCompile"
    },
    {
      "label": "cdk synth",
      "command": "npx",
      "type": "shell",
      "args": ["cdk", "synth", "ApplicationStack"],
      "problemMatcher": []
    },
    {
      "label": "cdk deploy",
      "command": "npx",
      "type": "shell",
      "args": ["cdk", "deploy", "ApplicationStack"],
      "problemMatcher": []
    },
    {
      "label": "cdk destroy",
      "command": "npx",
      "type": "shell",
      "args": ["cdk", "destroy", "ApplicationStack"],
      "problemMatcher": []
    },
    {
      "label": "lambda deploy simple",
      "command": "dotnet",
      "type": "process",
      "options": { "cwd": "${workspaceFolder}/src/AWS.Application.Lambda.Simple" },
      "args": ["lambda", "deploy-function"],
      "problemMatcher": "$msCompile"
    }
  ]
}
```

### Recommended Extensions

`.vscode/extensions.json`:

```json
{
  "recommendations": [
    "ms-dotnettools.csdevkit",
    "amazonwebservices.aws-toolkit-vscode",
    "kreativ-software.csharpextensions",
    "adrianwilczynski.namespace",
    "ms-azuretools.vscode-docker",
    "editorconfig.editorconfig",
    "github.vscode-github-actions",
    "eamodio.gitlens",
    "pkief.material-icon-theme",
    "josefpihrt-vscode.roslynator",
    "gruntfuggly.todo-tree",
    "redhat.vscode-yaml"
  ]
}
```

### Project Template

Make the entire solution a reusable `dotnet new` template. Create `.template.config/template.json`:

```json
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Your Name",
  "classifications": ["AWS", "CDK", "Clean Architecture"],
  "identity": "AWS.Application",
  "name": "AWS CDK Application (.NET 10)",
  "description": "A full AWS CDK application with Clean Architecture, Lambda, and CI/CD",
  "shortName": "AWSApplication",
  "sourceName": "AWS.Application",
  "tags": {
    "language": "C#",
    "type": "project"
  },
  "preferNameDirectory": true,
  "sources": [
    {
      "source": "./",
      "target": "./",
      "exclude": [
        "**/[Bb]in/**", "**/[Oo]bj/**",
        ".template.config/**/*",
        "**/*.filelist", "**/*.user", "**/*.lock.json",
        "cdk.out/**/*", "publish/**/*",
        ".vs/**/*", ".idea/**/*", ".git/**/*", "docs/**/*"
      ]
    }
  ]
}
```

Install and use:

```bash
# Install the template from the repo
dotnet new install .

# List available templates
dotnet new list

# Create a new solution
dotnet new AWSApplication -n "MyCompany.OrderService"
```

The `sourceName` parameter means every occurrence of `AWS.Application` in filenames and content is replaced with your chosen name.


## 8. Final Solution Structure

```
AWS.Application/
├── .github/workflows/
│   ├── build.yml
│   └── deploy.yml
├── .template.config/
│   └── template.json
├── .vscode/
│   ├── extensions.json
│   ├── launch.json
│   └── tasks.json
├── src/
│   ├── AWS.Application.CDK/           # Infrastructure as Code
│   │   ├── Constructs/
│   │   │   └── SimpleLambdaConstruct.cs
│   │   ├── ApplicationStack.cs
│   │   ├── Program.cs
│   │   └── appsettings.json
│   ├── AWS.Application.Core/          # Domain layer (no dependencies)
│   │   ├── DTOs/
│   │   ├── Entities/
│   │   ├── Enums/
│   │   ├── Exceptions/
│   │   ├── Interfaces/
│   │   └── Services/
│   ├── AWS.Application.Infrastructure/ # Infrastructure implementations
│   │   └── Services/
│   └── AWS.Application.Lambda.Simple/  # Lambda entry point
│       ├── Function.cs
│       └── Startup.cs
├── tests/
│   ├── AWS.Application.CDK.UnitTests/
│   ├── AWS.Application.Core.UnitTests/
│   ├── AWS.Application.Infrastructure.UnitTests/
│   └── AWS.Application.Lambda.Simple.Tests/
├── .editorconfig
├── .gitignore
├── cdk.json
└── AWS.Application.sln
```

## Key Takeaways

1. **Clean Architecture keeps your domain portable.** The Core project has zero AWS dependencies — it can be tested, reused, and reasoned about independently.

2. **CDK with .NET configuration is powerful.** Using `IConfiguration` in your CDK project means you can inject environment-specific values from `appsettings.json`, environment variables, or your CI/CD pipeline.

3. **OIDC > static credentials.** The deploy workflow uses `aws-actions/configure-aws-credentials@v4` with `id-token: write` — no long-lived AWS keys stored in GitHub.

4. **Trunk-based development scales.** One main branch, short-lived feature branches, automated checks on every push, and environment-gated deployments.

5. **Template everything.** The `.template.config` turns your battle-tested solution into a `dotnet new` template. New projects start with CI/CD, testing, CDK, and coding standards already in place.

6. **Invest in developer experience.** VS Code tasks, recommended extensions, and launch configs for local Lambda debugging eliminate friction and make the "right way" the easy way.

## What Changed from .NET 6

If you're migrating an existing .NET 6 AWS application:

| Area | .NET 6 | .NET 10 |
|------|--------|---------|
| Target framework | `net6.0` | `net10.0` |
| Lambda runtime | `Runtime.DOTNET_6` | `Runtime.DOTNET_10` |
| Lambda tools config | `dotnet6` | `dotnet10` |
| Test tool | `dotnet-lambda-test-tool-6.0` | `dotnet-lambda-test-tool-10.0` |
| C# features | C# 10 | C# 14 — primary constructors, collection expressions |
| Mocking library | Moq | NSubstitute (recommended — no telemetry concerns) |
| GitHub Actions | `@v2`/`@v3` | `@v4` |
| Node.js (for CDK CLI) | 16 | 22 |
| Template CLI | `dotnet new --install` | `dotnet new install` |
| Variable substitution | `microsoft/variable-substitution@v1` | `sed`/`jq` (action is deprecated) |

## References

- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [Clean Architecture with ASP.NET Core](https://ardalis.com/clean-architecture-asp-net-core/)
- [AWS Lambda .NET Guide](https://docs.aws.amazon.com/lambda/latest/dg/lambda-csharp.html)
- [GitHub Actions OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Trunk-Based Development](https://trunkbaseddevelopment.com/)
- [NSubstitute](https://nsubstitute.github.io/)
