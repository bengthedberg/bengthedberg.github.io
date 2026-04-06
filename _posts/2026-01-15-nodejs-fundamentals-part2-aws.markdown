---
layout: post
title: "Node.js Fundamentals Part 2: Deploying and Running Node.js on AWS"
date: 2026-01-15 00:00:00 +0000
tags:
  - nodejs
  - aws
  - lambda
  - serverless
  - typescript
series: "Node.js Fundamentals"
series_part: 2
---


## Series Overview

- [Part 1: Node.js Core Concepts and Building Blocks](/posts/nodejs-fundamentals-part1-core/)
- **Part 2: Deploying and Running Node.js on AWS** (this post)

## Introduction

In Part 1, we covered Node.js core concepts: the event loop, modules, streams, and building REST APIs with Express. Now it is time to take those skills to the cloud. AWS provides a rich set of services that pair naturally with Node.js, from serverless functions to managed databases and message queues.

This post walks through the most common AWS services you will use with Node.js: Lambda for serverless compute, DynamoDB for NoSQL storage, S3 for object storage, SQS and SNS for messaging, and Powertools for production-grade observability. Each section includes practical code examples using the AWS SDK v3.

## Setting Up the AWS SDK v3

The AWS SDK v3 for Node.js uses modular packages -- you install only what you need:

```bash
npm install @aws-sdk/client-s3 @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
npm install @aws-sdk/client-lambda @aws-sdk/client-sqs @aws-sdk/client-sns
npm install @aws-sdk/client-secrets-manager
```

### Configuring Credentials

The SDK looks for credentials in this order:

1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. Shared credentials file (`~/.aws/credentials`)
3. IAM role (when running on AWS infrastructure)

```js
import { S3Client } from "@aws-sdk/client-s3";

// The SDK picks up credentials automatically from the environment
const s3 = new S3Client({ region: "us-east-1" });
```

Never hardcode credentials. Use environment variables for local development and IAM roles when running on AWS.

## AWS Lambda with Node.js

Lambda lets you run code without provisioning servers. You pay only for the compute time you consume.

### Basic Lambda Handler

```js
// index.mjs
export const handler = async (event, context) => {
  console.log("Event received:", JSON.stringify(event, null, 2));

  const name = event.queryStringParameters?.name || "World";

  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      message: `Hello, ${name}!`,
      requestId: context.awsRequestId,
      timestamp: new Date().toISOString(),
    }),
  };
};
```

### Deploying with Dependencies

When your function needs npm packages, bundle everything into a zip:

```json
{
  "name": "my-lambda",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "uuid": "^9.0.0",
    "dayjs": "^1.11.10"
  }
}
```

```bash
cd my-lambda
npm install
zip -r function.zip .

aws lambda create-function \
  --function-name my-node-function \
  --runtime nodejs20.x \
  --handler index.handler \
  --role arn:aws:iam::YOUR_ACCOUNT_ID:role/lambda-execution-role \
  --zip-file fileb://function.zip
```

### API Gateway Integration

API Gateway creates RESTful endpoints that trigger Lambda functions. A single handler can route multiple HTTP methods:

```js
export const handler = async (event) => {
  const { httpMethod, pathParameters, queryStringParameters, body } = event;

  const headers = {
    "Content-Type": "application/json",
    "Access-Control-Allow-Origin": "*",
  };

  try {
    switch (httpMethod) {
      case "GET":
        if (pathParameters?.id) {
          return {
            statusCode: 200, headers,
            body: JSON.stringify({ item: { id: pathParameters.id } }),
          };
        }
        return {
          statusCode: 200, headers,
          body: JSON.stringify({ items: [], count: 0 }),
        };

      case "POST":
        const newItem = JSON.parse(body);
        return {
          statusCode: 201, headers,
          body: JSON.stringify({ message: "Created", item: newItem }),
        };

      default:
        return {
          statusCode: 405, headers,
          body: JSON.stringify({ error: `Method ${httpMethod} not allowed` }),
        };
    }
  } catch (error) {
    return {
      statusCode: 500, headers,
      body: JSON.stringify({ error: "Internal server error" }),
    };
  }
};
```

For a simpler alternative, **Lambda Function URLs** give you an HTTPS endpoint without API Gateway:

```bash
aws lambda create-function-url-config \
  --function-name my-node-function \
  --auth-type NONE
```

## DynamoDB with Node.js

DynamoDB is AWS's fully managed NoSQL database. The `lib-dynamodb` package provides a higher-level document client that handles type marshalling automatically.

### CRUD Operations

```js
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient, PutCommand, GetCommand,
  UpdateCommand, DeleteCommand, QueryCommand,
} from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({ region: "us-east-1" });
const docClient = DynamoDBDocumentClient.from(client, {
  marshallOptions: { removeUndefinedValues: true },
});

const TABLE_NAME = "Users";

// CREATE
async function createUser(user) {
  await docClient.send(new PutCommand({
    TableName: TABLE_NAME,
    Item: {
      userId: user.id,
      email: user.email,
      name: user.name,
      createdAt: new Date().toISOString(),
    },
    ConditionExpression: "attribute_not_exists(userId)",
  }));
}

// READ
async function getUser(userId) {
  const { Item } = await docClient.send(new GetCommand({
    TableName: TABLE_NAME,
    Key: { userId },
  }));
  return Item || null;
}

// UPDATE
async function updateUser(userId, updates) {
  const expressionParts = [];
  const expressionValues = {};
  const expressionNames = {};

  Object.entries(updates).forEach(([key, value], index) => {
    expressionParts.push(`#attr${index} = :val${index}`);
    expressionNames[`#attr${index}`] = key;
    expressionValues[`:val${index}`] = value;
  });

  const { Attributes } = await docClient.send(new UpdateCommand({
    TableName: TABLE_NAME,
    Key: { userId },
    UpdateExpression: `SET ${expressionParts.join(", ")}`,
    ExpressionAttributeNames: expressionNames,
    ExpressionAttributeValues: expressionValues,
    ReturnValues: "ALL_NEW",
  }));
  return Attributes;
}

// QUERY by index
async function getUserByEmail(email) {
  const { Items } = await docClient.send(new QueryCommand({
    TableName: TABLE_NAME,
    IndexName: "email-index",
    KeyConditionExpression: "email = :email",
    ExpressionAttributeValues: { ":email": email },
  }));
  return Items?.[0] || null;
}
```

## S3 Operations

S3 is AWS's object storage service, commonly used for file uploads, static assets, and backups.

### Core Operations

```js
import {
  S3Client, PutObjectCommand, GetObjectCommand,
  DeleteObjectCommand, ListObjectsV2Command,
} from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: "us-east-1" });
const BUCKET = "my-app-bucket";

// Upload a file
async function uploadFile(key, content, contentType) {
  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    Body: content,
    ContentType: contentType,
  }));
}

// Download and read as string
async function readS3JSON(key) {
  const response = await s3.send(new GetObjectCommand({
    Bucket: BUCKET, Key: key,
  }));
  const text = await response.Body.transformToString();
  return JSON.parse(text);
}

// Generate a presigned URL for temporary download access
async function getPresignedUrl(key, expiresIn = 3600) {
  return await getSignedUrl(s3, new GetObjectCommand({
    Bucket: BUCKET, Key: key,
  }), { expiresIn });
}
```

### Streaming Large File Uploads

For large files, use the `Upload` class from `@aws-sdk/lib-storage`, which handles multipart uploads automatically:

```js
import { Upload } from "@aws-sdk/lib-storage";
import { createReadStream } from "fs";

async function uploadLargeFile(key, filePath) {
  const upload = new Upload({
    client: s3,
    params: { Bucket: BUCKET, Key: key, Body: createReadStream(filePath) },
    queueSize: 4,
    partSize: 10 * 1024 * 1024, // 10 MB per part
  });

  upload.on("httpUploadProgress", (progress) => {
    console.log(`Upload: ${Math.round((progress.loaded / progress.total) * 100)}%`);
  });

  await upload.done();
}
```

## SQS -- Message Queues

SQS decouples components of your application using message queues.

### Sending and Receiving Messages

```js
import {
  SQSClient, SendMessageCommand,
  ReceiveMessageCommand, DeleteMessageCommand,
} from "@aws-sdk/client-sqs";

const sqs = new SQSClient({ region: "us-east-1" });
const QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789/my-queue";

async function sendMessage(payload) {
  const result = await sqs.send(new SendMessageCommand({
    QueueUrl: QUEUE_URL,
    MessageBody: JSON.stringify(payload),
    MessageAttributes: {
      eventType: { DataType: "String", StringValue: payload.type || "default" },
    },
  }));
  return result.MessageId;
}

async function pollMessages() {
  const { Messages = [] } = await sqs.send(new ReceiveMessageCommand({
    QueueUrl: QUEUE_URL,
    MaxNumberOfMessages: 10,
    WaitTimeSeconds: 20,       // long polling (recommended)
    VisibilityTimeout: 60,
  }));

  for (const message of Messages) {
    try {
      const body = JSON.parse(message.Body);
      // Process the message...

      await sqs.send(new DeleteMessageCommand({
        QueueUrl: QUEUE_URL,
        ReceiptHandle: message.ReceiptHandle,
      }));
    } catch (error) {
      console.error("Processing failed:", error);
      // Message reappears after VisibilityTimeout
    }
  }
}
```

### SQS-Triggered Lambda

When SQS triggers a Lambda function, you receive a batch of records. Use partial batch failure reporting to retry only the records that failed:

```js
export const handler = async (event) => {
  const failed = [];

  for (const record of event.Records) {
    try {
      const body = JSON.parse(record.body);
      await processMessage(body);
    } catch (error) {
      console.error(`Failed: ${record.messageId}`, error);
      failed.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures: failed };
};
```

## SNS -- Pub/Sub Notifications

SNS sends messages to multiple subscribers (SQS queues, Lambda functions, email, HTTP endpoints):

```js
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";

const sns = new SNSClient({ region: "us-east-1" });
const TOPIC_ARN = "arn:aws:sns:us-east-1:123456789:my-notifications";

async function publishNotification(subject, message, attributes = {}) {
  await sns.send(new PublishCommand({
    TopicArn: TOPIC_ARN,
    Subject: subject,
    Message: JSON.stringify(message),
    MessageAttributes: Object.fromEntries(
      Object.entries(attributes).map(([k, v]) => [
        k, { DataType: "String", StringValue: String(v) },
      ])
    ),
  }));
}
```

## Secrets Manager and Parameter Store

Never hardcode secrets. Use Secrets Manager for complex secrets (database credentials, API keys) and Parameter Store for configuration values:

```js
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const secretsClient = new SecretsManagerClient({ region: "us-east-1" });
const secretCache = new Map();

async function getSecret(secretName) {
  if (secretCache.has(secretName)) return secretCache.get(secretName);

  const response = await secretsClient.send(
    new GetSecretValueCommand({ SecretId: secretName })
  );

  const secret = JSON.parse(response.SecretString);
  secretCache.set(secretName, secret);
  return secret;
}

// Usage
const dbConfig = await getSecret("prod/database/credentials");
```

## Powertools for AWS Lambda

Powertools for AWS Lambda (TypeScript) is an official AWS toolkit that implements serverless best practices for logging, tracing, metrics, idempotency, and more.

### Structured Logging

```js
import { Logger } from "@aws-lambda-powertools/logger";
import middy from "@middy/core";
import { injectLambdaContext } from "@aws-lambda-powertools/logger/middleware";

const logger = new Logger({
  serviceName: "order-service",
  logLevel: "INFO",
});

const lambdaHandler = async (event) => {
  logger.info("Order received", {
    orderId: event.orderId,
    customerId: event.customerId,
  });

  const result = await processOrder(event);
  return { statusCode: 200, body: JSON.stringify(result) };
};

export const handler = middy(lambdaHandler).use(
  injectLambdaContext(logger, { clearState: true })
);
```

### Distributed Tracing

```js
import { Tracer } from "@aws-lambda-powertools/tracer";
import { captureLambdaHandler } from "@aws-lambda-powertools/tracer/middleware";

const tracer = new Tracer({ serviceName: "order-service" });
const dynamoClient = tracer.captureAWSv3Client(new DynamoDBClient({}));

const lambdaHandler = async (event) => {
  tracer.putAnnotation("orderId", event.orderId);
  tracer.putMetadata("orderDetails", { items: event.items });
  return await processOrder(event);
};

export const handler = middy(lambdaHandler).use(captureLambdaHandler(tracer));
```

### Custom Metrics

```js
import { Metrics, MetricUnit } from "@aws-lambda-powertools/metrics";
import { logMetrics } from "@aws-lambda-powertools/metrics/middleware";

const metrics = new Metrics({
  namespace: "OrderService",
  serviceName: "order-service",
});

const lambdaHandler = async (event) => {
  metrics.addMetric("OrdersReceived", MetricUnit.Count, 1);
  const start = Date.now();
  const result = await processOrder(event);
  metrics.addMetric("ProcessingTime", MetricUnit.Milliseconds, Date.now() - start);
  return { statusCode: 200, body: JSON.stringify(result) };
};

export const handler = middy(lambdaHandler).use(
  logMetrics(metrics, { captureColdStartMetric: true })
);
```

### Idempotency

Prevent duplicate execution when Lambda retries your function:

```js
import { makeIdempotent } from "@aws-lambda-powertools/idempotency";
import { DynamoDBPersistenceLayer } from "@aws-lambda-powertools/idempotency/dynamodb";

const persistenceStore = new DynamoDBPersistenceLayer({
  tableName: "IdempotencyTable",
});

const processPayment = async (event) => {
  const result = await chargeCustomer(event.orderId, event.amount);
  return { statusCode: 200, body: JSON.stringify(result) };
};

export const handler = makeIdempotent(processPayment, {
  persistenceStore,
  config: new IdempotencyConfig({
    eventKeyJmesPath: "orderId",
    expiresAfterSeconds: 3600,
  }),
});
```

### Batch Processing

Handle partial failures when processing SQS batches:

```js
import { BatchProcessor, EventType, processPartialResponse } from "@aws-lambda-powertools/batch";

const processor = new BatchProcessor(EventType.SQS);

const recordHandler = async (record) => {
  const order = JSON.parse(record.body);
  await fulfillOrder(order);
};

export const handler = async (event, context) => {
  return processPartialResponse(event, recordHandler, processor, { context });
};
```

## Deploying to EC2

For long-running Node.js servers, EC2 gives you full control. Use PM2 for process management and Nginx as a reverse proxy:

```bash
#!/bin/bash
# EC2 User Data script
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

npm install -g pm2
cd /opt/myapp
npm ci --production
pm2 start index.js --name "my-app" -i max
pm2 startup systemd
pm2 save
```

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Best Practices

### Security
- Use IAM roles instead of access keys on AWS infrastructure.
- Store secrets in Secrets Manager or Parameter Store.
- Enable encryption at rest for DynamoDB, S3, and SQS.

### Performance
- Reuse SDK clients across Lambda invocations by declaring them outside the handler.
- Enable keep-alive on HTTP connections.
- Use DynamoDB `ProjectionExpression` to fetch only needed attributes.

### Cost Optimization
- Use Lambda for bursty or low-traffic workloads; EC2 or Fargate for steady traffic.
- Set DynamoDB to on-demand mode for unpredictable workloads.
- Enable S3 lifecycle policies to transition old objects to cheaper tiers.

### Recommended Project Structure

```
my-aws-app/
├── src/
│   ├── handlers/          # Lambda handlers
│   ├── services/          # Business logic
│   ├── clients/           # AWS SDK client singletons
│   └── utils/             # Logger, validation
├── tests/
├── infra/                 # CloudFormation / CDK templates
└── package.json
```

## Conclusion

Node.js and AWS are a natural pairing. Lambda provides effortless scaling for event-driven workloads, DynamoDB handles NoSQL storage with single-digit millisecond latency, and S3 offers virtually unlimited object storage. SQS and SNS decouple your services for resilience, while Powertools brings production-grade observability with minimal boilerplate.

The key to success is choosing the right compute model for your workload, following AWS security best practices (IAM roles, encrypted secrets, least privilege), and using structured logging and tracing from day one.

## References

- [AWS SDK for JavaScript v3 Documentation](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/)
- [Powertools for AWS Lambda (TypeScript)](https://docs.powertools.aws.dev/lambda/typescript/latest/)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/)
