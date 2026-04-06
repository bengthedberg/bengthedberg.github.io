---
layout: post
title: "C# and .NET on AWS: A Practical Guide to Compute, Services, and Infrastructure"
date: 2026-01-25 00:00:00 +0000
tags:
  - dotnet
  - aws
  - csharp
  - lambda
  - ecs
  - cdk
---


## Introduction

AWS is a first-class platform for .NET workloads. Whether you are building serverless APIs with Lambda, containerized microservices with ECS/Fargate, or managing infrastructure with the CDK, AWS provides deep .NET integration at every layer. The AWS SDK for .NET v4 offers idiomatic C# APIs for virtually every AWS service, and the tooling ecosystem includes Visual Studio extensions, CLI tools, and guided deployment wizards.

This guide covers the most common patterns for running C# .NET workloads on AWS: choosing the right compute model, working with core AWS services, defining infrastructure as code, and applying security and cost optimization best practices. All examples target .NET 10, the latest supported runtime.

## The AWS SDK for .NET

### Installation and Setup

The SDK is distributed as individual NuGet packages, one per service, all depending on `AWSSDK.Core`:

```bash
dotnet add package AWSSDK.S3
dotnet add package AWSSDK.DynamoDBv2
dotnet add package AWSSDK.SQS
dotnet add package AWSSDK.Extensions.NETCore.Setup
```

Register AWS service clients via dependency injection in ASP.NET Core:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDefaultAWSOptions(
    builder.Configuration.GetAWSOptions());
builder.Services.AddAWSService<IAmazonS3>();
builder.Services.AddAWSService<IAmazonDynamoDB>();

var app = builder.Build();
```

Then inject clients into your services:

```csharp
public class StorageService
{
    private readonly IAmazonS3 _s3;

    public StorageService(IAmazonS3 s3) => _s3 = s3;

    public async Task UploadAsync(string bucket, string key, Stream data)
    {
        await _s3.PutObjectAsync(new PutObjectRequest
        {
            BucketName = bucket,
            Key = key,
            InputStream = data
        });
    }
}
```

### SDK v4 Key Changes

Version 4 is the current major release (v3 entered maintenance mode in March 2026). The most important behavioral change is that collection properties now default to `null` instead of empty collections. Always check for `null` when reading responses, or use the compatibility flag `AWSConfigs.InitializeCollections = true` during migration.

### Best Practices

- Always use `await` -- never `.Result` or `.Wait()`, which can deadlock.
- Reuse client instances -- they are thread-safe and manage connection pooling internally.
- Handle errors specifically using pattern matching on `AmazonServiceException.ErrorCode`.

## Choosing a Compute Model

Before diving into code, here is a decision matrix for .NET workloads on AWS:

| Criteria | Lambda | ECS/Fargate | Elastic Beanstalk | EC2 | App Runner |
|----------|--------|-------------|-------------------|-----|------------|
| Ops overhead | Minimal | Low-Medium | Low | High | Minimal |
| Cold starts | Possible | No | No | No | No |
| Max execution | 15 min | Unlimited | Unlimited | Unlimited | Unlimited |
| Scaling | Per-request | Per-task | Per-instance | Manual/ASG | Auto |
| Best for | Event-driven, APIs | Microservices | Simple web apps | Legacy, full control | Simple APIs |
| Windows | No | Yes (EC2 launch type) | Yes | Yes | No |

**When to choose what:**

- **Lambda** -- short-lived, event-driven workloads: API endpoints, file processing, queue consumers.
- **ECS/Fargate** -- microservices, long-running background workers, complex networking.
- **Elastic Beanstalk** -- simplest deployment for traditional web apps.
- **EC2** -- legacy .NET Framework apps needing Windows, GPU workloads.
- **App Runner** -- simple containerized APIs with scale-to-zero.

## AWS Lambda (Serverless)

### Getting Started

```bash
dotnet new install Amazon.Lambda.Templates
dotnet tool install -g Amazon.Lambda.Tools

dotnet new lambda.EmptyFunction -n MyFunction
cd MyFunction/src/MyFunction
```

```csharp
// Function.cs
using Amazon.Lambda.Core;

[assembly: LambdaSerializer(typeof(
    Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace MyFunction;

public class Function
{
    public string FunctionHandler(string input, ILambdaContext context)
    {
        context.Logger.LogInformation($"Processing: {input}");
        return input.ToUpper();
    }
}
```

```bash
dotnet lambda deploy-function MyFunction \
    --function-role arn:aws:iam::123456789012:role/lambda-role \
    --runtime dotnet10
```

### File-Based Apps (.NET 10)

.NET 10 introduces file-based Lambda functions -- a single `.cs` file with no `.csproj`:

```csharp
// MyLambdaFunction.cs
#r "nuget: Amazon.Lambda.RuntimeSupport"
#r "nuget: Amazon.Lambda.Serialization.SystemTextJson"

using Amazon.Lambda.Core;
using Amazon.Lambda.RuntimeSupport;

var handler = (string input, ILambdaContext context) =>
{
    context.Logger.LogInformation($"Processing: {input}");
    return input.ToUpper();
};

await LambdaBootstrapBuilder
    .Create(handler, new Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer())
    .Build()
    .RunAsync();
```

### ASP.NET Core on Lambda

Run a full ASP.NET Core API as a Lambda function with a single line of code:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

// This single line makes it Lambda-compatible
builder.Services.AddAWSLambdaHosting(LambdaEventSource.RestApi);

var app = builder.Build();
app.MapControllers();
app.Run();
```

This lets you develop and test locally as a normal ASP.NET Core app, then deploy to Lambda without code changes.

### Native AOT

Native AOT compiles your function to native code, eliminating JIT compilation and reducing cold start times by up to 86%:

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <StripSymbols>true</StripSymbols>
</PropertyGroup>
```

Trade-offs: sub-second cold starts and lower memory, but no runtime reflection, larger deployment packages, and longer build times.

### Lambda Annotations Framework

Write Lambda functions more naturally with routing and serialization handled automatically:

```csharp
using Amazon.Lambda.Annotations;
using Amazon.Lambda.Annotations.APIGateway;

public class Functions
{
    [LambdaFunction]
    [HttpApi(LambdaHttpMethod.Get, "/users/{id}")]
    public async Task<User> GetUser(string id, ILambdaContext context)
    {
        return await _userService.GetByIdAsync(id);
    }
}
```

## Amazon ECS with Fargate

ECS with Fargate is ideal for microservices, long-running APIs, and workloads needing container orchestration without managing EC2 instances.

### Dockerfile for a .NET API

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApi.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Key advantages of ECS/Fargate: no EC2 instances to manage, fine-grained VPC networking, sidecar container support, and built-in blue/green deployments. The main trade-off is more setup complexity compared to Elastic Beanstalk or App Runner.

## Working with Common AWS Services

### Amazon S3

```csharp
public class S3Service
{
    private readonly IAmazonS3 _s3;

    public S3Service(IAmazonS3 s3) => _s3 = s3;

    public async Task<string> UploadFileAsync(
        string bucket, string key, Stream content, string contentType)
    {
        await _s3.PutObjectAsync(new PutObjectRequest
        {
            BucketName = bucket, Key = key,
            InputStream = content, ContentType = contentType
        });
        return $"s3://{bucket}/{key}";
    }

    public string GetPresignedUrl(string bucket, string key, int expiryMinutes = 60)
    {
        return _s3.GetPreSignedURL(new GetPreSignedUrlRequest
        {
            BucketName = bucket, Key = key,
            Expires = DateTime.UtcNow.AddMinutes(expiryMinutes)
        });
    }
}
```

For large files, use the Transfer Utility which handles multipart uploads automatically.

### Amazon DynamoDB

DynamoDB offers three programming models. The **Object Persistence Model** is recommended for most applications:

```csharp
using Amazon.DynamoDBv2.DataModel;

[DynamoDBTable("Users")]
public class User
{
    [DynamoDBHashKey]
    public string Id { get; set; }

    [DynamoDBProperty]
    public string Name { get; set; }

    [DynamoDBProperty]
    public string Email { get; set; }
}

// Usage
var context = new DynamoDBContext(_dynamoDb);
var user = await context.LoadAsync<User>("user-123");
await context.SaveAsync(new User { Id = "user-456", Name = "Alice" });
```

| Model | When to Use |
|-------|-------------|
| Low-level API | Maximum control, complex queries, batch operations |
| Document Model | Flexible schemas, unknown attributes at compile time |
| Object Persistence | Well-defined data models, CRUD-heavy applications |

### Amazon SQS

```csharp
public class QueueService
{
    private readonly IAmazonSQS _sqs;
    private readonly string _queueUrl;

    public async Task SendAsync<T>(T message)
    {
        await _sqs.SendMessageAsync(new SendMessageRequest
        {
            QueueUrl = _queueUrl,
            MessageBody = JsonSerializer.Serialize(message),
        });
    }

    public async Task<List<Message>> ReceiveAsync(int maxMessages = 10)
    {
        var response = await _sqs.ReceiveMessageAsync(new ReceiveMessageRequest
        {
            QueueUrl = _queueUrl,
            MaxNumberOfMessages = maxMessages,
            WaitTimeSeconds = 20  // long polling
        });
        return response.Messages;
    }
}
```

### Amazon Cognito -- Authentication

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority =
            $"https://cognito-idp.{region}.amazonaws.com/{userPoolId}";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            ValidateAudience = false
        };
    });
```

## Infrastructure as Code with AWS CDK

The CDK lets you define infrastructure in C#, the same language as your application:

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using Amazon.CDK.AWS.APIGateway;

public class MyApiStack : Stack
{
    public MyApiStack(Construct scope, string id) : base(scope, id)
    {
        var function = new Function(this, "MyFunction", new FunctionProps
        {
            Runtime = Runtime.DOTNET_10,
            Handler = "MyFunction::MyFunction.Function::FunctionHandler",
            Code = Code.FromAsset("./src/MyFunction/bin/Release/net10.0/publish"),
            MemorySize = 256,
            Timeout = Duration.Seconds(30)
        });

        var api = new RestApi(this, "MyApi");
        api.Root.AddProxy(new ProxyResourceOptions
        {
            DefaultIntegration = new LambdaIntegration(function)
        });
    }
}
```

### CDK vs. Alternatives

| Tool | Strengths | Best For |
|------|-----------|----------|
| AWS CDK | Write IaC in C#; high-level constructs; type safety | All-in on AWS, C# teams |
| CloudFormation | Native AWS; no extra tooling | Simple stacks, existing templates |
| Terraform | Multi-cloud; large ecosystem | Multi-cloud or existing Terraform usage |
| Pulumi | True programming languages; multi-cloud | .NET teams needing multi-cloud |

## CI/CD Pipelines

### GitHub Actions

```yaml
name: Deploy .NET Lambda
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - run: dotnet tool install -g Amazon.Lambda.Tools
      - run: cd src/MyFunction && dotnet lambda deploy-function MyFunction
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: us-east-1
```

### AWS Deploy Tool

The Deploy Tool provides guided deployments from the CLI:

```bash
dotnet tool install -g aws.deploy.tools
dotnet aws deploy
```

It analyzes your project, recommends deployment targets (Lambda, ECS, Beanstalk, App Runner), and handles the provisioning.

## Observability and Monitoring

### Structured Logging

```csharp
logger.LogInformation("Processing order {OrderId} for {CustomerId}",
    order.Id, order.CustomerId);
```

### OpenTelemetry with AWS X-Ray

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddAWSInstrumentation()
            .AddOtlpExporter();
    });
```

### Powertools for AWS Lambda (.NET)

Powertools provides structured logging, metrics, and tracing with minimal code:

```csharp
[Logging(LogEvent = true)]
[Tracing]
[Metrics(Namespace = "MyApp")]
public async Task<APIGatewayProxyResponse> Handler(
    APIGatewayProxyRequest request, ILambdaContext context)
{
    Logger.LogInformation("Processing request");
    Metrics.AddMetric("OrdersProcessed", 1, MetricUnit.Count);
    return new APIGatewayProxyResponse { StatusCode = 200 };
}
```

## Security Best Practices

### IAM Least Privilege

Every Lambda function, ECS task, and EC2 instance should have an IAM role with only the permissions it needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/uploads/*"
  }]
}
```

### Credential Management

Never hardcode credentials. Use the default credential chain:

1. Environment variables
2. Shared credentials file (`~/.aws/credentials`)
3. IAM role -- **preferred in production**
4. SSO / IAM Identity Center -- **preferred for developer machines**

### Secrets

Store secrets in AWS Secrets Manager or SSM Parameter Store. Use the ASP.NET Core configuration provider to load them at startup:

```csharp
builder.Configuration.AddSecretsManager(configurator: options =>
{
    options.SecretFilter = entry => entry.Name.StartsWith("myapp/");
    options.KeyGenerator = (_, key) => key.Replace("myapp/", "").Replace("/", ":");
});
```

## Cost Optimization

### Lambda
- Right-size memory using AWS Lambda Power Tuning.
- Use ARM64 (Graviton) for 20% lower cost.
- Use Native AOT to reduce execution time and memory.

### ECS/Fargate
- Use Fargate Spot for fault-tolerant workloads (up to 70% discount).
- Right-size task CPU and memory based on CloudWatch utilization data.

### General
- Use AWS Cost Explorer and Budgets for tracking.
- Consider Savings Plans for predictable workloads.
- Leverage S3 Intelligent-Tiering for automatic storage cost optimization.

## Quick-Reference Decision Guide

**"I want to build a REST API"**
-- Lambda + API Gateway (low traffic) or ECS/Fargate + ALB (high/steady traffic).

**"I have a legacy .NET Framework 4.x app"**
-- EC2 with Windows Server, or modernize to .NET 10 and move to containers.

**"I need background job processing"**
-- Lambda (short jobs under 15 min) or ECS/Fargate (long-running workers).

**"I want the simplest possible deployment"**
-- Elastic Beanstalk or App Runner.

**"I need a message-driven architecture"**
-- SQS + Lambda, or SQS + ECS with the AWS Message Processing Framework.

**"I want to define infrastructure in C#"**
-- AWS CDK.

## Conclusion

Running .NET on AWS is a mature, well-supported path. The combination of the AWS SDK for .NET v4, deep Lambda integration with Native AOT and file-based apps, container orchestration with ECS/Fargate, and infrastructure-as-code with the CDK gives .NET developers a complete cloud-native toolkit. Start with the compute model that fits your workload, use IAM roles and encrypted secrets from day one, and invest in observability with OpenTelemetry and Powertools to keep your production systems healthy.

## References

- [AWS SDK for .NET Developer Guide](https://docs.aws.amazon.com/sdk-for-net/)
- [AWS Lambda .NET Documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-csharp.html)
- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/)
- [Powertools for AWS Lambda (.NET)](https://docs.powertools.aws.dev/lambda/dotnet/)
- [AWS .NET Deploy Tool](https://aws.github.io/aws-dotnet-deploy/)
