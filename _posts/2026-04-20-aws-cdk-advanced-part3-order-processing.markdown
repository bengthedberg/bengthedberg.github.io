---
layout: post
title: "Advanced AWS CDK Patterns: Event-Driven Order Processing"
date: 2026-04-20 00:00:00 +0000
tags:
  - aws
  - cdk
  - sns
  - sqs
  - lambda
  - event-driven
  - serverless
series: "Advanced AWS CDK Patterns"
series_part: 3
---


## Series Overview

1. [Multi-Stack Patterns and Cross-Stack References](/posts/aws-cdk-advanced-part1-multi-stack/)
2. [CI/CD with GitHub Actions](/posts/aws-cdk-advanced-part2-github-actions/)
3. **Event-Driven Order Processing** (you are here)

## Introduction

The SQS-to-Lambda-to-DynamoDB pipeline is one of the most battle-tested serverless patterns on AWS. An SQS queue buffers incoming events, a Lambda function processes each batch, and DynamoDB stores the results. A dead letter queue catches failures for later investigation.

This pattern provides three critical properties that synchronous architectures lack: the producer does not need to know anything about the consumer (loose coupling), traffic bursts are absorbed by the queue rather than overwhelming downstream services (backpressure), and if the Lambda fails, the message stays in the queue and gets retried automatically (durability).

In this final post of the series, we will build a complete order processing system from scratch, covering infrastructure definition with CDK, Lambda handler implementation with idempotency guarantees, unit testing, GitHub Actions CI/CD, and production hardening.

## Architecture

```
[Producer] ──► [SQS: OrderCreatedQueue]
                        │
                        ▼
              [Lambda: OrderProcessor]
                        │
                        ▼
              [DynamoDB: OrdersTable]
                        │
                        ▼
              [Dead Letter Queue (DLQ)]
                 (failed messages)
```

### Why SQS Over the Alternatives?

We chose SQS as the event source for specific reasons. Here is how it compares to the alternatives:

- **EventBridge** is a better fit when you need content-based routing (routing `OrderCreated` to one Lambda and `OrderCancelled` to another) or when you have many consumers for the same event. For a single-consumer pipeline, SQS is simpler, cheaper, and gives you native batching and DLQ support without additional rules configuration.

- **Kinesis Data Streams** is designed for high-throughput, ordered event streaming (clickstream data or IoT telemetry). It uses a shard-based model where you pay for provisioned throughput even at idle. For order processing where reliable delivery matters more than strict ordering, SQS is more cost-effective and operationally simpler.

- **SNS to SQS fan-out** is what you would add if multiple services need to react to `OrderCreated` events (send a confirmation email, update inventory, charge payment). Our design uses a single consumer, but the pattern extends naturally by placing an SNS topic in front of the SQS queue.

## Project Setup

Initialize a new CDK project and install the event sources module:

```bash
mkdir order-processing-cdk && cd order-processing-cdk
npx cdk init app --language typescript
npm install @aws-cdk/aws-lambda-event-sources
```

This produces the standard CDK project structure:

```
order-processing-cdk/
├── bin/
│   └── order-processing-cdk.ts
├── lib/
│   └── order-processing-cdk-stack.ts
├── test/
│   └── order-processing-cdk.test.ts
├── cdk.json
├── tsconfig.json
└── package.json
```

## Infrastructure Stack

The stack defines five resources: a dead letter queue, a main SQS queue, a DynamoDB table with a global secondary index, a Lambda function, and the wiring that connects them.

```typescript
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import * as sqs from "aws-cdk-lib/aws-sqs";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";
import { SqsEventSource } from "aws-cdk-lib/aws-lambda-event-sources";

export class OrderProcessingStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1. Dead Letter Queue
    const dlq = new sqs.Queue(this, "OrderCreatedDLQ", {
      queueName: "OrderCreatedDLQ",
      retentionPeriod: cdk.Duration.days(14),
    });

    // 2. Main SQS Queue
    const orderQueue = new sqs.Queue(this, "OrderCreatedQueue", {
      queueName: "OrderCreatedQueue",
      visibilityTimeout: cdk.Duration.seconds(60),
      retentionPeriod: cdk.Duration.days(7),
      deadLetterQueue: {
        queue: dlq,
        maxReceiveCount: 3,
      },
    });

    // 3. DynamoDB Table
    const ordersTable = new dynamodb.Table(this, "OrdersTable", {
      tableName: "Orders",
      partitionKey: {
        name: "orderId",
        type: dynamodb.AttributeType.STRING,
      },
      sortKey: {
        name: "createdAt",
        type: dynamodb.AttributeType.STRING,
      },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      pointInTimeRecovery: true,
    });

    ordersTable.addGlobalSecondaryIndex({
      indexName: "CustomerIndex",
      partitionKey: {
        name: "customerId",
        type: dynamodb.AttributeType.STRING,
      },
      sortKey: {
        name: "createdAt",
        type: dynamodb.AttributeType.STRING,
      },
    });

    // 4. Lambda Function
    const orderProcessor = new lambda.Function(this, "OrderProcessorFn", {
      functionName: "OrderProcessor",
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: "index.handler",
      code: lambda.Code.fromAsset("lambda/order-processor/dist"),
      timeout: cdk.Duration.seconds(30),
      memorySize: 256,
      environment: {
        TABLE_NAME: ordersTable.tableName,
        DLQ_URL: dlq.queueUrl,
      },
      tracing: lambda.Tracing.ACTIVE,
    });

    // 5. Wire Everything Together
    ordersTable.grantReadWriteData(orderProcessor);
    dlq.grantSendMessages(orderProcessor);

    orderProcessor.addEventSource(
      new SqsEventSource(orderQueue, {
        batchSize: 10,
        maxBatchingWindow: cdk.Duration.seconds(5),
        reportBatchItemFailures: true,
      })
    );

    // 6. Outputs
    new cdk.CfnOutput(this, "QueueUrl", {
      value: orderQueue.queueUrl,
      description: "URL of the OrderCreated SQS queue",
    });

    new cdk.CfnOutput(this, "TableName", {
      value: ordersTable.tableName,
    });

    new cdk.CfnOutput(this, "DLQUrl", {
      value: dlq.queueUrl,
    });
  }
}
```

## Design Decisions

Every configuration choice in this stack has specific reasoning behind it. Understanding these trade-offs will help you adapt the pattern to your own use cases.

### Standard SQS vs FIFO Queue

We chose a Standard queue over a FIFO queue. The key trade-off:

| Property | Standard Queue | FIFO Queue |
|---|---|---|
| Throughput | Nearly unlimited | 300 msg/s (3,000 with batching) |
| Ordering | Best-effort | Strict per message group |
| Duplicates | Possible (at-least-once) | Exactly-once available |
| Cost | $0.40/million requests | $0.50/million requests |

For order processing, strict ordering is typically unnecessary because each order is independent. Our Lambda handles duplicates via a DynamoDB `ConditionExpression` (idempotency), making the at-least-once delivery of Standard queues perfectly acceptable. If you needed strict ordering for state changes on the same order, you would switch to FIFO with `orderId` as the message group ID.

### Visibility Timeout and Lambda Timeout

The visibility timeout (60 seconds) is how long SQS hides a message from other consumers after a Lambda picks it up. The critical rule is: **visibility timeout must be at least twice the Lambda timeout**.

Our Lambda timeout is 30 seconds, so we set visibility to 60 seconds. If you set them equal, a Lambda that hits its timeout at exactly 30 seconds would have its message become visible at the same moment, causing a race condition where a new invocation picks it up immediately -- potentially resulting in duplicate processing.

The downside: if a message genuinely fails after 1 second of processing, it will not be retried for another 59 seconds. For latency-sensitive workloads, you might prefer a shorter timeout with fewer retries.

### Dead Letter Queue (maxReceiveCount: 3)

After 3 failed processing attempts, messages move to the DLQ rather than being retried indefinitely. This prevents poison messages -- malformed events that will never succeed but consume Lambda invocations forever.

Why 3 attempts? One retry is too few (transient DynamoDB throttling or network blips would lose messages). Five or more retries waste compute on genuinely bad messages. Three gives transient errors two chances to resolve while quarantining permanent failures quickly.

The 14-day DLQ retention gives your operations team two weeks to investigate and replay failed messages. Consider adding a CloudWatch alarm on `ApproximateNumberOfMessagesVisible > 0` on the DLQ for immediate notification when messages land there.

### Batch Processing

The SQS event source mapping delivers messages to Lambda in batches. We configured it to wait up to 5 seconds to collect up to 10 messages before invoking the Lambda.

Batching amortizes per-invocation overhead (cold start, SDK initialization, DynamoDB connection setup) across multiple messages. Processing 10 messages in one invocation is cheaper and faster than 10 separate invocations.

The `reportBatchItemFailures: true` setting is critical. Without it, if your Lambda throws an error, the entire batch is considered failed and all 10 messages go back to the queue -- even the 9 that succeeded. With partial batch failure reporting, your Lambda returns a list of failed `messageId`s, and only those specific messages are retried.

The trade-off is latency: batching adds up to 5 seconds of delay. If you need sub-second processing, set `maxBatchingWindow` to 0 and `batchSize` to 1.

### DynamoDB Key Design

We use `orderId` as the partition key and `createdAt` as the sort key. This composite key supports two access patterns efficiently: fetching a specific order by ID (point read) and querying all versions of an order sorted by time (range query). The Global Secondary Index (CustomerIndex) enables querying all orders for a given customer.

On-Demand billing (`PAY_PER_REQUEST`) is the right default for most workloads. You pay nothing at idle and scale automatically. Once your traffic stabilizes, switching to provisioned mode with auto-scaling can cut costs by 50-85%.

### Lambda Runtime and Memory

Node.js 20.x provides fast cold starts (~200ms) and native async/await for the I/O-heavy DynamoDB calls. At 256 MB, Lambda allocates enough CPU for efficient SDK serialization without overpaying. X-Ray tracing is enabled to provide latency breakdowns for debugging production issues.

## Lambda Handler

The handler processes SQS messages in batches with idempotency guarantees and partial batch failure reporting:

```typescript
import { SQSEvent, SQSBatchResponse, SQSBatchItemFailure } from "aws-lambda";
import { DynamoDB } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocument } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDB({});
const docClient = DynamoDBDocument.from(client);
const TABLE_NAME = process.env.TABLE_NAME!;

interface OrderCreatedEvent {
  orderId: string;
  customerId: string;
  items: Array<{
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
  }>;
  totalAmount: number;
  currency: string;
  shippingAddress: {
    street: string;
    city: string;
    state: string;
    zip: string;
    country: string;
  };
  createdAt: string;
}

export async function handler(event: SQSEvent): Promise<SQSBatchResponse> {
  const batchItemFailures: SQSBatchItemFailure[] = [];

  for (const record of event.Records) {
    try {
      const order: OrderCreatedEvent = JSON.parse(record.body);

      console.log(`Processing order: ${order.orderId}`, {
        customerId: order.customerId,
        totalAmount: order.totalAmount,
        itemCount: order.items.length,
      });

      validateOrder(order);

      const enrichedOrder = {
        ...order,
        status: "CONFIRMED",
        processedAt: new Date().toISOString(),
        itemCount: order.items.length,
        sqsMessageId: record.messageId,
        ttl: Math.floor(Date.now() / 1000) + 90 * 24 * 60 * 60,
      };

      await docClient.put({
        TableName: TABLE_NAME,
        Item: enrichedOrder,
        ConditionExpression: "attribute_not_exists(orderId)",
      });

      console.log(`Order ${order.orderId} saved successfully.`);
    } catch (error: any) {
      if (error.name === "ConditionalCheckFailedException") {
        console.warn(
          `Duplicate order detected: ${record.messageId}. Skipping.`
        );
        continue;
      }

      console.error(`Failed to process message ${record.messageId}:`, error);
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures };
}

function validateOrder(order: OrderCreatedEvent): void {
  if (!order.orderId) throw new Error("Missing orderId");
  if (!order.customerId) throw new Error("Missing customerId");
  if (!order.items || order.items.length === 0) throw new Error("Empty items");
  if (order.totalAmount <= 0) throw new Error("Invalid totalAmount");
}
```

### Key Implementation Details

**Idempotency via ConditionExpression.** The `ConditionExpression: "attribute_not_exists(orderId)"` ensures that if the same order is processed twice (which can happen with SQS at-least-once delivery), the second write silently fails with `ConditionalCheckFailedException`. We catch that specific error and skip the message without reporting it as a failure. For multi-step operations that include external side effects (payment APIs), consider the Powertools for AWS Lambda Idempotency utility instead.

**SDK client initialization outside the handler.** The DynamoDB client is created at the module level, not inside the handler function. Lambda reuses the execution environment across invocations, so the client and its TCP connection persist between calls. Initializing inside the handler would add 50-100ms of latency per invocation.

**TTL for automatic cleanup.** The `ttl` attribute is set 90 days in the future. When you enable TTL on the DynamoDB table, expired items are deleted automatically at no cost. Note that TTL deletions are best-effort and can be delayed up to 48 hours after expiry.

**Structured logging.** The second argument to `console.log` is a JSON-serializable object that CloudWatch Logs Insights can query directly, enabling queries like `fields @timestamp, orderId, totalAmount | filter customerId = "CUST-42"`.

## Building the Lambda

Since the Lambda is written in TypeScript, we use esbuild to bundle it to JavaScript. Create `lambda/order-processor/package.json`:

```json
{
  "name": "order-processor",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "build": "esbuild index.ts --bundle --platform=node --target=node20 --outfile=dist/index.js --external:@aws-sdk/*"
  },
  "devDependencies": {
    "@types/aws-lambda": "^8.10.145",
    "esbuild": "^0.24.0"
  },
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.700.0",
    "@aws-sdk/lib-dynamodb": "^3.700.0"
  }
}
```

We externalize `@aws-sdk/*` because the Node.js 20.x Lambda runtime already includes AWS SDK v3. Bundling it would add approximately 5 MB to the deployment package for no benefit and would actually increase cold start time. With externalization, the bundle is typically under 50 KB.

## Unit Tests

CDK unit tests verify that your infrastructure code produces the expected CloudFormation resources:

```typescript
import * as cdk from "aws-cdk-lib";
import { Template, Match } from "aws-cdk-lib/assertions";
import { OrderProcessingStack } from "../lib/order-processing-cdk-stack";

describe("OrderProcessingStack", () => {
  let template: Template;

  beforeAll(() => {
    const app = new cdk.App();
    const stack = new OrderProcessingStack(app, "TestStack");
    template = Template.fromStack(stack);
  });

  test("creates the SQS queue with a DLQ", () => {
    template.hasResourceProperties("AWS::SQS::Queue", {
      QueueName: "OrderCreatedQueue",
      VisibilityTimeout: 60,
    });
    template.hasResourceProperties("AWS::SQS::Queue", {
      QueueName: "OrderCreatedDLQ",
    });
  });

  test("creates the DynamoDB table with correct keys", () => {
    template.hasResourceProperties("AWS::DynamoDB::Table", {
      TableName: "Orders",
      KeySchema: [
        { AttributeName: "orderId", KeyType: "HASH" },
        { AttributeName: "createdAt", KeyType: "RANGE" },
      ],
      BillingMode: "PAY_PER_REQUEST",
    });
  });

  test("creates the Lambda function with correct config", () => {
    template.hasResourceProperties("AWS::Lambda::Function", {
      FunctionName: "OrderProcessor",
      Runtime: "nodejs20.x",
      MemorySize: 256,
      Timeout: 30,
    });
  });

  test("Lambda has event source mapping to SQS", () => {
    template.hasResourceProperties(
      "AWS::Lambda::EventSourceMapping",
      Match.objectLike({
        BatchSize: 10,
        FunctionResponseTypes: ["ReportBatchItemFailures"],
      })
    );
  });

  test("Lambda has IAM permissions for DynamoDB", () => {
    template.hasResourceProperties("AWS::IAM::Policy", {
      PolicyDocument: {
        Statement: Match.arrayWith([
          Match.objectLike({
            Action: Match.arrayWith([
              "dynamodb:BatchWriteItem",
              "dynamodb:PutItem",
            ]),
            Effect: "Allow",
          }),
        ]),
      },
    });
  });
});
```

These tests catch regressions like accidentally changing the billing mode or losing the SQS event source mapping. For a complete testing strategy, add snapshot tests, Lambda unit tests with mocked DynamoDB calls, and integration tests against a deployed environment.

## GitHub Actions CI/CD

The workflow builds the Lambda, runs tests, validates the CDK template, and deploys on push to main using OIDC authentication (/posts/as covered in detail in [Part 2](2026-04-aws-cdk-advanced-part2-github-actions/)):

```yaml
name: CDK Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - name: Build Lambda
        working-directory: lambda/order-processor
        run: |
          npm ci
          npm run build
      - run: npx tsc --noEmit
      - run: npm test
      - run: npx cdk synth --quiet

  deploy:
    needs: build-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - name: Build Lambda
        working-directory: lambda/order-processor
        run: |
          npm ci
          npm run build
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION || 'us-east-1' }}
      - run: npx cdk deploy --require-approval never --outputs-file cdk-outputs.json
      - run: cat cdk-outputs.json
```

## Testing End-to-End

After deploying, verify the full pipeline by sending a test message to SQS:

```bash
aws sqs send-message \
  --queue-url "<QUEUE_URL>" \
  --message-body '{
    "orderId": "ORD-20260420-001",
    "customerId": "CUST-42",
    "items": [
      {
        "productId": "PROD-100",
        "productName": "Mechanical Keyboard",
        "quantity": 1,
        "unitPrice": 149.99
      }
    ],
    "totalAmount": 149.99,
    "currency": "USD",
    "shippingAddress": {
      "street": "123 Dev Lane",
      "city": "Portland",
      "state": "OR",
      "zip": "97201",
      "country": "US"
    },
    "createdAt": "2026-04-20T14:30:00Z"
  }'
```

Verify the order was stored in DynamoDB:

```bash
aws dynamodb get-item \
  --table-name Orders \
  --key '{"orderId": {"S": "ORD-20260420-001"}, "createdAt": {"S": "2026-04-20T14:30:00Z"}}'
```

Test idempotency by sending the same message again. The Lambda should log "Duplicate order detected" and the DynamoDB item should remain unchanged. Test the DLQ by sending a malformed message and verifying it appears in the DLQ after 3 failed attempts.

## Production Hardening Checklist

When moving from development to production, address these areas:

### Data Protection
- Set `removalPolicy: RETAIN` on DynamoDB and SQS to prevent accidental data loss.
- Enable KMS encryption on SQS and DynamoDB.
- Enable Point-in-Time Recovery and consider on-demand backups before major deployments.

### Security
- Replace broad managed policies with least-privilege custom IAM policies.
- Use KMS to encrypt sensitive Lambda environment variables at rest.
- Place Lambda in a VPC if it needs access to private resources.

### Observability
- Add CloudWatch alarms for DLQ message depth, Lambda error rate, Lambda duration P99, and DynamoDB throttled requests.
- Create a CloudWatch dashboard combining Lambda, SQS, and DynamoDB metrics.
- Implement structured logging with correlation IDs using Powertools Logger.

### Scalability
- Set `reservedConcurrentExecutions` on the Lambda to prevent traffic spikes from exhausting your account's concurrency limit.
- Use `maxConcurrency` on the SQS event source mapping if DynamoDB cannot keep up with the processing rate.
- Monitor DynamoDB consumed capacity and switch to provisioned mode with auto-scaling once traffic patterns stabilize.

## Conclusion

The SQS-to-Lambda-to-DynamoDB pattern provides a robust foundation for event-driven processing on AWS. The queue acts as a buffer that decouples producers from consumers, handles backpressure naturally, and guarantees message delivery through automatic retries and dead letter queues.

The key takeaways from this implementation are:

- Use `ConditionExpression` on DynamoDB writes for simple idempotency; upgrade to Powertools Idempotency for multi-step operations.
- Always set the visibility timeout to at least twice the Lambda timeout.
- Enable `reportBatchItemFailures` to prevent successful messages from being reprocessed when a batch partially fails.
- Externalize `@aws-sdk/*` from your Lambda bundle since the runtime already includes it.
- Initialize SDK clients outside the handler function to benefit from execution environment reuse.

Combined with the multi-stack patterns from [Part 1](/posts/aws-cdk-advanced-part1-multi-stack/) and the CI/CD pipeline from [Part 2](/posts/aws-cdk-advanced-part2-github-actions/), you now have a complete toolkit for building, deploying, and operating production-grade serverless applications on AWS.

## References

- [AWS CDK v2 Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Using Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [DynamoDB Condition Expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html)
- [Powertools for AWS Lambda (TypeScript)](https://docs.powertools.aws.dev/lambda/typescript/latest/)
- [Serverless Land -- SQS to Lambda Pattern](https://serverlessland.com/patterns/sqs-lambda-cdk)
- [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)
