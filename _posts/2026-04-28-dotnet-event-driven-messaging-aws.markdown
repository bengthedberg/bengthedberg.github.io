---
layout: post
title: "Event-Driven Messaging with .NET 10 on AWS: SQS, SNS, and EventBridge"
date: 2026-04-28 00:00:00 +0000
categories: [.NET, AWS]
tags:
  - dotnet
  - aws
  - sqs
  - sns
  - event-driven
  - messaging
---


## Introduction

Event-driven architecture decouples services by communicating through messages rather than direct calls. When a service publishes an event, any number of consumers can react independently, enabling resilient, scalable systems where each component can evolve, deploy, and scale on its own schedule.

AWS provides three messaging services that compose together naturally: **SQS** for point-to-point queuing, **SNS** for pub/sub fan-out, and **EventBridge** for content-based event routing. The **AWS Message Processing Framework for .NET** (`AWS.Messaging`) wraps these services in a high-level, opinionated framework that handles serialization, message routing, visibility management, and message deletion so you can focus on business logic.

This post covers the full lifecycle of building event-driven .NET applications on AWS: publishing and consuming messages, FIFO ordering, fan-out patterns, idempotency, distributed tracing with OpenTelemetry, and production readiness.

## Architecture Overview

Before writing code, understand how SQS, SNS, and EventBridge compose in a well-designed system:

```
┌──────────────┐
│  Your App    │  publishes domain events
│  (Producer)  │
└──────┬───────┘
       │
       ▼
┌──────────────────┐       ┌──────────────────┐
│  EventBridge     │──────▶│  SNS Topic       │  fan-out
│  (Event Router)  │       │  (Pub/Sub)       │
└──────┬───────────┘       └───┬──────────┬───┘
       │                      │          │
       ▼                      ▼          ▼
┌──────────────┐      ┌──────────┐  ┌──────────┐
│ SQS Queue A  │      │ SQS Q B  │  │ SQS Q C  │
│ (Orders)     │      │ (Emails) │  │ (Audit)  │
└──────┬───────┘      └────┬─────┘  └────┬─────┘
       ▼                   ▼             ▼
┌──────────────┐   ┌──────────────┐ ┌──────────────┐
│  ECS Worker  │   │   Lambda     │ │  ECS Worker  │
└──────────────┘   └──────────────┘ └──────────────┘
```

### Service Selection Guide

| Service | Role | Delivery | Best For |
|---------|------|----------|----------|
| SQS | Point-to-point queue | Pull-based, at-least-once (standard) or exactly-once (FIFO) | Decoupling services, buffering, load leveling |
| SNS | Pub/sub fan-out | Push-based to multiple subscribers | Broadcasting one event to many consumers |
| EventBridge | Event router | Push-based with content-based filtering | Routing events by type/content to different targets |

The common layered pattern is EventBridge (routes) to SNS (fans out) to SQS (buffers) to your consumer (processes).

## Project Setup

We will build two applications: an API that publishes messages and a background worker that consumes them.

```bash
# Publisher
dotnet new webapi -n OrderApi --framework net10.0
cd OrderApi
dotnet add package AWS.Messaging
dotnet add package AWS.Messaging.Telemetry.OpenTelemetry
dotnet add package AWSSDK.SQS
dotnet add package AWSSDK.SimpleNotificationService

# Consumer
dotnet new worker -n OrderWorker --framework net10.0
cd OrderWorker
dotnet add package AWS.Messaging
dotnet add package AWSSDK.SQS
```

### Define Message Types

Create shared message contracts as plain C# records:

```csharp
public record OrderPlaced
{
    public required string OrderId { get; init; }
    public required string CustomerId { get; init; }
    public required decimal TotalAmount { get; init; }
    public required List<OrderItem> Items { get; init; }
    public DateTime PlacedAt { get; init; } = DateTime.UtcNow;
}

public record OrderItem
{
    public required string Sku { get; init; }
    public required int Quantity { get; init; }
    public required decimal UnitPrice { get; init; }
}

public record OrderShipped
{
    public required string OrderId { get; init; }
    public required string TrackingNumber { get; init; }
    public required string Carrier { get; init; }
}
```

## Publishing Messages

### Configure the Publisher

```csharp
// OrderApi/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAWSMessageBus(bus =>
{
    bus.AddSQSPublisher<OrderPlaced>(
        "https://sqs.us-east-1.amazonaws.com/123456789012/OrderQueue");
    bus.AddSNSPublisher<OrderShipped>(
        "arn:aws:sns:us-east-1:123456789012:OrderEvents");
    bus.AddEventBridgePublisher<PaymentProcessed>(
        "arn:aws:events:us-east-1:123456789012:event-bus/ecommerce-bus");
});

var app = builder.Build();

app.MapPost("/orders", async (OrderPlaced order, IMessagePublisher publisher) =>
{
    await publisher.PublishAsync(order);
    return Results.Accepted($"/orders/{order.OrderId}");
});

app.Run();
```

The framework serializes your .NET object to JSON, wraps it in a CloudEvents-compatible envelope, and publishes to the configured destination.

### Service-Specific Publishers

When you need FIFO group IDs, message attributes, or EventBridge source/detail-type, inject the typed publisher:

```csharp
// SQS FIFO publisher
app.MapPost("/orders/fifo", async (
    OrderPlaced order, ISQSPublisher sqsPublisher) =>
{
    await sqsPublisher.SendAsync(order, new SQSOptions
    {
        MessageGroupId = order.CustomerId,
        MessageDeduplicationId = order.OrderId
    });
    return Results.Accepted();
});

// EventBridge publisher with source and detail type
app.MapPost("/payments", async (
    PaymentProcessed payment, IEventBridgePublisher ebPublisher) =>
{
    await ebPublisher.PublishAsync(payment, new EventBridgeOptions
    {
        Source = "com.myapp.payments",
        DetailType = "PaymentProcessed"
    });
    return Results.Ok();
});
```

### Batch Publishing

For high-throughput scenarios, batch publishing reduces API calls:

```csharp
app.MapPost("/orders/batch", async (
    List<OrderPlaced> orders, ISQSPublisher sqsPublisher) =>
{
    var response = await sqsPublisher.SendBatchAsync(orders);
    return Results.Ok(new { response.Successful.Count, Failed = response.Failed.Count });
});
```

## Consuming Messages

### Message Handlers

Create a handler class that implements `IMessageHandler<T>`:

```csharp
public class OrderPlacedHandler : IMessageHandler<OrderPlaced>
{
    private readonly ILogger<OrderPlacedHandler> _logger;
    private readonly IOrderRepository _orderRepo;

    public OrderPlacedHandler(
        ILogger<OrderPlacedHandler> logger, IOrderRepository orderRepo)
    {
        _logger = logger;
        _orderRepo = orderRepo;
    }

    public async Task<MessageProcessStatus> HandleAsync(
        MessageEnvelope<OrderPlaced> messageEnvelope,
        CancellationToken token = default)
    {
        var order = messageEnvelope.Message;

        _logger.LogInformation(
            "Processing order {OrderId} for customer {CustomerId}",
            order.OrderId, order.CustomerId);

        try
        {
            await _orderRepo.SaveAsync(order, token);
            return MessageProcessStatus.Success();  // deletes from SQS
        }
        catch (DuplicateOrderException)
        {
            _logger.LogWarning("Order {OrderId} already exists", order.OrderId);
            return MessageProcessStatus.Success();  // idempotent skip
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process order {OrderId}", order.OrderId);
            return MessageProcessStatus.Failed();   // retries later
        }
    }
}
```

### Configure the Consumer

```csharp
// OrderWorker/Program.cs
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddAWSMessageBus(bus =>
{
    bus.AddSQSPoller(
        "https://sqs.us-east-1.amazonaws.com/123456789012/OrderQueue",
        options =>
        {
            options.MaxNumberOfConcurrentMessages = 10;
            options.WaitTimeSeconds = 20;       // long polling
            options.VisibilityTimeout = 30;
        });
    bus.AddMessageHandler<OrderPlacedHandler, OrderPlaced>();
    bus.AddMessageHandler<OrderShippedHandler, OrderShipped>();
});

var host = builder.Build();
host.Run();
```

### How the Poller Works

1. The poller calls `ReceiveMessage` with long polling.
2. Each message is deserialized using the CloudEvents envelope to determine the .NET type.
3. The framework resolves the correct `IMessageHandler<T>` from the DI container.
4. While the handler runs, the framework automatically extends message visibility to prevent duplicate processing.
5. On `Success`, the message is deleted. On `Failed` or exception, the message returns to the queue.

## FIFO Queues and Ordering

Use FIFO queues when message ordering matters or you need exactly-once delivery within a deduplication window:

```csharp
public async Task PlaceOrderAsync(OrderPlaced order)
{
    await _publisher.SendAsync(order, new SQSOptions
    {
        MessageGroupId = order.CustomerId,        // ordered per customer
        MessageDeduplicationId = order.OrderId     // 5-minute dedup window
    });
}
```

| Aspect | Standard Queue | FIFO Queue |
|--------|---------------|------------|
| Throughput | Nearly unlimited | 300 msg/s (70K with high-throughput mode) |
| Ordering | Best-effort | Strict within message group |
| Deduplication | None (at-least-once) | 5-minute window (exactly-once) |
| Cost | $0.40/million | $0.50/million |

## Fan-Out with SNS

The classic fan-out pattern publishes one event to an SNS topic, which delivers to multiple SQS queues:

```csharp
// Publisher sends to SNS
builder.Services.AddAWSMessageBus(bus =>
{
    bus.AddSNSPublisher<OrderPlaced>(
        "arn:aws:sns:us-east-1:123456789012:OrderEvents");
});

// Each downstream service consumes from its own queue
// Email service
builder.Services.AddAWSMessageBus(bus =>
{
    bus.AddSQSPoller(".../EmailNotificationQueue");
    bus.AddMessageHandler<SendOrderConfirmationHandler, OrderPlaced>();
});

// Inventory service
builder.Services.AddAWSMessageBus(bus =>
{
    bus.AddSQSPoller(".../InventoryQueue");
    bus.AddMessageHandler<UpdateInventoryHandler, OrderPlaced>();
});
```

Use SNS subscription filter policies to deliver only relevant messages to each queue, avoiding unnecessary processing.

## Event Routing with EventBridge

EventBridge routes events based on content-matching rules. Define rules with CDK:

```csharp
// Route high-value orders to a VIP queue
var highValueRule = new Rule(this, "HighValueOrderRule", new RuleProps
{
    EventBus = eventBus,
    EventPattern = new EventPattern
    {
        Source = new[] { "com.myapp.orders" },
        DetailType = new[] { "OrderPlaced" },
        Detail = new Dictionary<string, object>
        {
            ["totalAmount"] = new object[] {
                new Dictionary<string, object> { ["numeric"] = new object[] { ">=", 1000 } }
            }
        }
    }
});
highValueRule.AddTarget(new SqsQueue(vipQueue));
```

### EventBridge vs. SNS

| Criteria | EventBridge | SNS |
|----------|-------------|-----|
| Filtering | Content-based on any JSON field | Attribute-based filter policies |
| Event archive & replay | Yes | No |
| Schema registry | Yes (auto-discovery) | No |
| Throughput | Tens of thousands/s | ~30M msgs/s per topic |

Use EventBridge as the top-level router for domain events. Use SNS for high-throughput fan-out when you do not need content-based routing.

## Idempotency Patterns

AWS messaging services provide at-least-once delivery. Your handlers must be idempotent -- processing the same message twice should produce the same result.

### DynamoDB Idempotency Table

The most robust approach uses DynamoDB with conditional writes:

```csharp
public class IdempotencyService
{
    private readonly IAmazonDynamoDB _dynamoDb;

    public async Task<bool> TryClaimAsync(string idempotencyKey)
    {
        try
        {
            await _dynamoDb.PutItemAsync(new PutItemRequest
            {
                TableName = "ProcessedMessages",
                Item = new Dictionary<string, AttributeValue>
                {
                    ["PK"] = new() { S = idempotencyKey },
                    ["ProcessedAt"] = new() { S = DateTime.UtcNow.ToString("O") },
                    ["TTL"] = new() { N = DateTimeOffset.UtcNow
                        .AddDays(7).ToUnixTimeSeconds().ToString() }
                },
                ConditionExpression = "attribute_not_exists(PK)"
            });
            return true;  // first time, safe to process
        }
        catch (ConditionalCheckFailedException)
        {
            return false; // already processed, skip
        }
    }
}
```

Wire it into your handler:

```csharp
public async Task<MessageProcessStatus> HandleAsync(
    MessageEnvelope<OrderPlaced> envelope, CancellationToken token = default)
{
    var order = envelope.Message;
    string key = $"order-placed:{order.OrderId}";

    if (!await _idempotency.TryClaimAsync(key))
    {
        _logger.LogInformation("Order {OrderId} already processed", order.OrderId);
        return MessageProcessStatus.Success();
    }

    await _orderRepo.SaveAsync(order, token);
    return MessageProcessStatus.Success();
}
```

### Choosing an Idempotency Strategy

| Strategy | Complexity | Best For |
|----------|-----------|----------|
| FIFO deduplication | Low | Short dedup windows (5 min) |
| DynamoDB idempotency table | Medium | Most production systems |
| Database constraints | Low | CRUD-heavy operations |
| Powertools Idempotency | Low | Lambda functions |

## OpenTelemetry and Distributed Tracing

In event-driven systems, a single request triggers a message that may be processed minutes later by a different service. OpenTelemetry context propagation connects these traces across service boundaries.

### Publisher Configuration

```csharp
builder.Services.AddAWSMessageBus(bus =>
{
    bus.AddSQSPublisher<OrderPlaced>(queueUrl);
    bus.AddSNSPublisher<OrderShipped>(topicArn);
});

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService("order-api", serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAWSMessagingInstrumentation()
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddAWSInstrumentation()
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://localhost:4317"))
    );
```

### Consumer Configuration

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService("order-worker", serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAWSMessagingInstrumentation()
        .AddAWSInstrumentation()
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://localhost:4317"))
    );
```

When `AWS.Messaging` publishes a message, it injects the W3C Trace Context into the CloudEvents envelope. The consumer extracts it and links its span to the publisher, creating end-to-end traces:

```
[order-api: POST /orders]
  └── [order-api: SQS.SendMessage]
        └── [order-worker: SQS.ProcessMessage]
              └── [order-worker: DynamoDB.PutItem]
```

### Choosing a Tracing Backend

| Backend | Best For |
|---------|----------|
| AWS X-Ray | AWS-native teams, deep Lambda/ECS integration |
| Grafana Tempo | Multi-cloud, open-source teams |
| Jaeger | Self-hosted environments |
| Datadog / New Relic | Enterprise with existing contracts |

## Lambda Integration

The framework integrates with Lambda for serverless consumers:

```csharp
public class Function
{
    private readonly ILambdaMessaging _messaging;

    public Function()
    {
        var services = new ServiceCollection();
        services.AddAWSMessageBus(bus =>
        {
            bus.AddLambdaMessageProcessor();
            bus.AddMessageHandler<OrderPlacedHandler, OrderPlaced>();
        });
        _messaging = services.BuildServiceProvider()
            .GetRequiredService<ILambdaMessaging>();
    }

    public async Task FunctionHandler(SQSEvent sqsEvent, ILambdaContext context)
    {
        await _messaging.ProcessLambdaEventAsync(sqsEvent, context);
    }
}
```

| Factor | Lambda Consumer | Long-Running Consumer (ECS) |
|--------|----------------|----------------------------|
| Scaling | Automatic per-batch | Configure concurrency |
| Cold starts | Yes | No |
| Max runtime | 15 minutes | Unlimited |
| Cost (low volume) | Very cheap | Pay for idle compute |
| Cost (high volume) | Can be expensive | More cost-effective |

## Error Handling and Dead-Letter Queues

Every production queue should have a dead-letter queue. After a message exceeds the maximum receive count, SQS moves it to the DLQ:

```bash
aws sqs create-queue --queue-name OrderQueue-DLQ

aws sqs set-queue-attributes \
    --queue-url .../OrderQueue \
    --attributes '{
        "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:...:OrderQueue-DLQ\",\"maxReceiveCount\":\"3\"}"
    }'
```

Distinguish between transient and permanent errors in your handler:

```csharp
try
{
    await _orderService.ProcessAsync(order, token);
    return MessageProcessStatus.Success();
}
catch (TransientException)
{
    return MessageProcessStatus.Failed();  // let SQS retry
}
catch (ValidationException ex)
{
    // Permanent error -- publish to error topic and remove from queue
    await _errorPublisher.PublishAsync(new OrderProcessingFailed
    {
        OrderId = order.OrderId, Reason = ex.Message
    });
    return MessageProcessStatus.Success();
}
```

## Scalability and Performance Tuning

### SQS Poller Configuration

```csharp
bus.AddSQSPoller(queueUrl, options =>
{
    options.MaxNumberOfConcurrentMessages = 20;    // higher = more throughput
    options.WaitTimeSeconds = 20;                  // long polling
    options.VisibilityTimeout = 60;                // 6x expected handler duration
    options.VisibilityTimeoutExtensionThreshold = 5;
});
```

### Scaling by Compute Type

**Lambda:** SQS automatically scales concurrency based on queue depth. Configure `ReservedConcurrentExecutions` to cap scaling and `BatchSize` for throughput tuning.

**ECS/Fargate:** Scale tasks based on the `ApproximateNumberOfMessagesVisible` CloudWatch metric using Application Auto Scaling with target tracking.

### Throughput Optimization Checklist

| Action | Impact |
|--------|--------|
| Long polling (`WaitTimeSeconds = 20`) | Reduces empty polls, saves cost |
| Batch publishing with `SendBatchAsync` | Up to 10x fewer API calls |
| Tune `MaxNumberOfConcurrentMessages` | Higher throughput per host |
| Enable FIFO high-throughput mode | Up to 70K msgs/s |
| Compress large payloads | Reduced SQS costs (charged per 64KB) |

## Testing

### Unit Testing Handlers

Test handlers in isolation by constructing a `MessageEnvelope` directly:

```csharp
[Fact]
public async Task HandleAsync_ValidOrder_ReturnsSuccess()
{
    var mockRepo = new Mock<IOrderRepository>();
    var handler = new OrderPlacedHandler(
        NullLogger<OrderPlacedHandler>.Instance, mockRepo.Object);

    var envelope = new MessageEnvelope<OrderPlaced>
    {
        Message = new OrderPlaced
        {
            OrderId = "ORD-001", CustomerId = "CUST-123",
            TotalAmount = 99.99m, Items = new List<OrderItem>()
        }
    };

    var result = await handler.HandleAsync(envelope);

    Assert.Equal(MessageProcessStatus.Success(), result);
    mockRepo.Verify(x => x.SaveAsync(It.IsAny<OrderPlaced>(), default), Times.Once);
}
```

### Integration Testing with LocalStack

Use Testcontainers to run LocalStack for real SQS/DynamoDB integration tests:

```csharp
public class MessagingIntegrationTests : IAsyncLifetime
{
    private readonly LocalStackContainer _localStack;

    public MessagingIntegrationTests()
    {
        _localStack = new LocalStackBuilder()
            .WithServices(LocalStackService.SQS, LocalStackService.DynamoDB)
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _localStack.StartAsync();
        var sqsClient = new AmazonSQSClient(new AmazonSQSConfig
        {
            ServiceURL = _localStack.GetConnectionString()
        });
        await sqsClient.CreateQueueAsync("TestQueue");
    }
}
```

## Production Checklist

Before going live, verify each of these items:

- Dead-letter queues configured on every SQS queue (maxReceiveCount = 3-5)
- Idempotency implemented in every handler
- Visibility timeout set to 6x expected handler duration
- Long polling enabled (WaitTimeSeconds = 20)
- OpenTelemetry configured with `AddAWSMessagingInstrumentation()`
- Structured logging with correlation IDs (OrderId, MessageId)
- Alarms on queue depth, DLQ messages, and handler error rates
- IAM least privilege -- producers can only send, consumers can only receive/delete
- Encryption enabled (SQS SSE, SNS SSE)
- VPC endpoints for SQS/SNS if running in a VPC

### Key Monitoring Metrics

| Metric | Alert Threshold |
|--------|----------------|
| Queue depth (`ApproximateNumberOfMessagesVisible`) | > 1000 sustained |
| DLQ messages | > 0 |
| Message age (`ApproximateAgeOfOldestMessage`) | > 300 seconds |
| Handler latency (OpenTelemetry span) | p99 > handler SLA |
| Handler errors | > 1% error rate |

## Quick Reference

```
Publishing:     IMessagePublisher.PublishAsync(message)
                ISQSPublisher.SendAsync(message, sqsOptions)
                ISNSPublisher.PublishAsync(message, snsOptions)
                IEventBridgePublisher.PublishAsync(message, ebOptions)

Consuming:      IMessageHandler<T>.HandleAsync(envelope, token)
                → MessageProcessStatus.Success()  // deletes message
                → MessageProcessStatus.Failed()   // retries later

Configuration:  builder.Services.AddAWSMessageBus(bus => {
                    bus.AddSQSPublisher<T>(queueUrl);
                    bus.AddSNSPublisher<T>(topicArn);
                    bus.AddEventBridgePublisher<T>(busArn);
                    bus.AddSQSPoller(queueUrl, options => { ... });
                    bus.AddMessageHandler<THandler, TMessage>();
                });
```

## Conclusion

The AWS Message Processing Framework for .NET removes the boilerplate of building event-driven systems on AWS. It provides a clean abstraction over SQS, SNS, and EventBridge with automatic serialization, message routing, visibility management, and first-class OpenTelemetry support. Combined with idempotency patterns, dead-letter queues, and proper monitoring, you can build production-grade messaging systems that are resilient, observable, and scalable.

Start with a simple SQS publisher and consumer, add SNS fan-out as you grow, and introduce EventBridge when you need content-based routing across many services. The framework supports all three patterns with the same handler model, making it straightforward to evolve your architecture over time.

## References

- [AWS Message Processing Framework for .NET](https://github.com/awslabs/aws-dotnet-messaging)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/)
- [Amazon SNS Developer Guide](https://docs.aws.amazon.com/sns/latest/dg/)
- [Amazon EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/)
- [OpenTelemetry .NET Documentation](https://opentelemetry.io/docs/languages/dotnet/)
- [Powertools for AWS Lambda (.NET)](https://docs.powertools.aws.dev/lambda/dotnet/)
