---
layout: post
title: "Advanced AWS CDK Patterns: Multi-Stack Patterns and Cross-Stack References"
date: 2026-04-01 00:00:00 +0000
tags:
  - aws
  - cdk
  - infrastructure-as-code
  - multi-stack
  - typescript
series: "Advanced AWS CDK Patterns"
series_part: 1
---


## Series Overview

1. **Multi-Stack Patterns and Cross-Stack References** (you are here)
2. [CI/CD with GitHub Actions](/posts/aws-cdk-advanced-part2-github-actions/)
3. [Event-Driven Order Processing](/posts/aws-cdk-advanced-part3-order-processing/)

## Introduction

As your AWS infrastructure grows beyond a handful of resources, managing everything in a single CDK stack becomes impractical. CloudFormation has a hard limit of 500 resources per stack, but long before you hit that ceiling, you will encounter problems with deployment times, blast radius, and team ownership boundaries.

Multi-stack architectures solve these problems by decomposing your infrastructure into independently deployable units. In this post, we will explore when and how to split your CDK application into multiple stacks, examine the four primary patterns for cross-stack communication, and walk through complete TypeScript and C# examples you can adapt for your own projects.

## Why Multiple Stacks?

There are five compelling reasons to break a monolithic CDK application into separate stacks:

- **Blast radius isolation** -- a bad deployment in your API stack does not affect your database stack.
- **Independent lifecycles** -- your VPC changes far less often than your Lambda code, so they should not force each other to redeploy.
- **Team ownership** -- different teams can own, version, and deploy their stacks independently.
- **AWS limits** -- the 500-resource CloudFormation limit is real, and large stacks hit it sooner than you expect.
- **Cross-account separation** -- production databases in one account, compute in another, each with its own security boundary.

## The CDK App, Stack, and Construct Hierarchy

Every CDK application follows a three-level hierarchy:

```
App (cdk.App)
 ├── NetworkStack        (VPC, subnets, NAT gateways)
 ├── DataStack           (RDS, DynamoDB, S3)
 ├── ComputeStack        (ECS, Lambda, API Gateway)
 └── MonitoringStack     (CloudWatch, SNS, dashboards)
```

The `App` is the orchestration root. Each stack maps one-to-one to a CloudFormation stack and is the unit of deployment. Constructs are the building blocks inside stacks. Crucially, each stack can target a different AWS account or region, making the App the natural place to wire together a multi-account architecture.

## Project Structure

For teams working on related stacks, a monorepo layout keeps everything in sync:

```
infra/
├── bin/
│   └── app.ts                    # CDK entry point
├── lib/
│   ├── constructs/               # Shared L3 constructs
│   │   ├── secure-bucket.ts
│   │   └── tagged-vpc.ts
│   ├── network-stack.ts
│   ├── data-stack.ts
│   ├── compute-stack.ts
│   └── monitoring-stack.ts
├── config/
│   ├── dev.ts
│   ├── staging.ts
│   └── prod.ts
├── test/
│   ├── network-stack.test.ts
│   └── data-stack.test.ts
├── cdk.json
├── tsconfig.json
└── package.json
```

When different teams own different stacks, a polyrepo layout is more appropriate. In that case, stacks live in separate repositories and communicate through exported CloudFormation outputs consumed via `Fn.importValue` or SSM Parameter Store.

## Stack Communication Patterns

There are four primary ways stacks share data, each with a different coupling level:

| Pattern | When to Use | Coupling Level |
|---|---|---|
| **Direct references (props)** | Stacks in the same app, same account/region | Tight |
| **CfnOutput + Fn.importValue** | Stacks in the same account/region, different apps | Medium |
| **SSM Parameter Store** | Cross-account, cross-app, or polyrepo | Loose |
| **Shared constructs library** | Common patterns across many stacks | None (code reuse) |

### Pattern 1: Direct Reference via Props

The producing stack exposes a property. The consuming stack accepts it as a constructor prop. CDK automatically creates a CloudFormation export/import under the hood. This is the simplest and most type-safe approach when all stacks live in the same CDK app.

### Pattern 2: CfnOutput + Fn.importValue

The producing stack creates an explicit `CfnOutput` with an `exportName`. The consuming stack reads it with `Fn.importValue`. This works across separately deployed CDK apps as long as they share the same account and region. Be careful though: once a CloudFormation export is consumed by another stack, you cannot update or delete it until all consumers are updated first. This creates a deployment lock.

### Pattern 3: SSM Parameter Store

The producing stack writes values to SSM. The consuming stack reads from SSM at deploy time or runtime, even from a different account. This is the most flexible pattern and the recommended approach for polyrepo setups. SSM parameters survive stack deletes and redeployments, making them more resilient than CloudFormation exports.

### Pattern 4: Shared Constructs Library

You publish an npm or NuGet package with reusable L3 constructs. There is no runtime coupling -- just shared infrastructure patterns that enforce organizational standards like encryption, tagging, and access controls.

## TypeScript Implementation

### Entry Point

The entry point is where you instantiate all stacks and wire them together with explicit dependencies:

```typescript
#!/usr/bin/env node
import "source-map-support/register";
import * as cdk from "aws-cdk-lib";
import { NetworkStack } from "../lib/network-stack";
import { DataStack } from "../lib/data-stack";
import { ComputeStack } from "../lib/compute-stack";
import { MonitoringStack } from "../lib/monitoring-stack";
import { getConfig } from "../config";

const app = new cdk.App();

const envName = app.node.tryGetContext("env") ?? "dev";
const config = getConfig(envName);

const envAccount: cdk.Environment = {
  account: config.accountId,
  region: config.region,
};

const networkStack = new NetworkStack(app, `${config.prefix}-Network`, {
  env: envAccount,
  cidr: config.vpcCidr,
  maxAzs: config.maxAzs,
});

const dataStack = new DataStack(app, `${config.prefix}-Data`, {
  env: envAccount,
  vpc: networkStack.vpc,
  dbName: config.dbName,
});
dataStack.addDependency(networkStack);

const computeStack = new ComputeStack(app, `${config.prefix}-Compute`, {
  env: envAccount,
  vpc: networkStack.vpc,
  table: dataStack.table,
  dbSecret: dataStack.dbSecret,
  dbEndpoint: dataStack.dbEndpoint,
});
computeStack.addDependency(dataStack);

const monitoringStack = new MonitoringStack(app, `${config.prefix}-Monitoring`, {
  env: envAccount,
  apiGateway: computeStack.api,
  lambdaFunctions: computeStack.functions,
  alarmEmail: config.alarmEmail,
});
monitoringStack.addDependency(computeStack);

app.synth();
```

Notice the use of `addDependency()` to make deployment order explicit. While CDK can often infer order from cross-stack references, explicit dependencies prevent subtle ordering bugs.

### Network Stack

The network stack creates the VPC and publishes its ID through both a CloudFormation export and an SSM parameter, supporting both same-app and cross-app consumption:

```typescript
import * as cdk from "aws-cdk-lib";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import { Construct } from "constructs";

export interface NetworkStackProps extends cdk.StackProps {
  cidr: string;
  maxAzs: number;
}

export class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.IVpc;

  constructor(scope: Construct, id: string, props: NetworkStackProps) {
    super(scope, id, props);

    this.vpc = new ec2.Vpc(this, "Vpc", {
      ipAddresses: ec2.IpAddresses.cidr(props.cidr),
      maxAzs: props.maxAzs,
      natGateways: 1,
      subnetConfiguration: [
        { name: "Public", subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
        { name: "Private", subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
        { name: "Isolated", subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 24 },
      ],
    });

    new cdk.CfnOutput(this, "VpcIdOutput", {
      value: this.vpc.vpcId,
      exportName: `${cdk.Stack.of(this).stackName}-VpcId`,
      description: "VPC ID for cross-stack reference",
    });

    new cdk.aws_ssm.StringParameter(this, "VpcIdParam", {
      parameterName: "/infra/network/vpc-id",
      stringValue: this.vpc.vpcId,
      description: "VPC ID published for cross-app lookup",
    });
  }
}
```

### Data Stack

The data stack receives the VPC via its props interface and creates both a DynamoDB table and an Aurora PostgreSQL cluster:

```typescript
import * as cdk from "aws-cdk-lib";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as rds from "aws-cdk-lib/aws-rds";
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";
import { Construct } from "constructs";

export interface DataStackProps extends cdk.StackProps {
  vpc: ec2.IVpc;
  dbName: string;
}

export class DataStack extends cdk.Stack {
  public readonly table: dynamodb.ITable;
  public readonly dbSecret: secretsmanager.ISecret;
  public readonly dbEndpoint: string;

  constructor(scope: Construct, id: string, props: DataStackProps) {
    super(scope, id, props);

    this.table = new dynamodb.Table(this, "AppTable", {
      partitionKey: { name: "PK", type: dynamodb.AttributeType.STRING },
      sortKey: { name: "SK", type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      pointInTimeRecoverySpecification: { pointInTimeRecoveryEnabled: true },
    });

    const dbCluster = new rds.DatabaseCluster(this, "DbCluster", {
      engine: rds.DatabaseClusterEngine.auroraPostgres({
        version: rds.AuroraPostgresEngineVersion.VER_15_4,
      }),
      writer: rds.ClusterInstance.serverlessV2("Writer"),
      vpc: props.vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      defaultDatabaseName: props.dbName,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    this.dbSecret = dbCluster.secret!;
    this.dbEndpoint = dbCluster.clusterEndpoint.hostname;
  }
}
```

The key design decision here is using `RemovalPolicy.RETAIN` on stateful resources. This ensures that a `cdk destroy` or a failed deployment never accidentally deletes your data.

### Compute Stack

The compute stack demonstrates how CDK handles cross-stack IAM automatically. When you call `props.table.grantReadWriteData(handler)` and the table lives in a different stack, CDK generates the correct IAM policies and CloudFormation exports behind the scenes:

```typescript
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as apigateway from "aws-cdk-lib/aws-apigateway";
import { Construct } from "constructs";

export class ComputeStack extends cdk.Stack {
  public readonly api: apigateway.RestApi;
  public readonly functions: lambda.IFunction[];

  constructor(scope: Construct, id: string, props: ComputeStackProps) {
    super(scope, id, props);

    const handler = new lambda.Function(this, "ApiHandler", {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: "index.handler",
      code: lambda.Code.fromAsset("lambda/api"),
      vpc: props.vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      environment: {
        TABLE_NAME: props.table.tableName,
        DB_SECRET_ARN: props.dbSecret.secretArn,
        DB_ENDPOINT: props.dbEndpoint,
      },
    });

    props.table.grantReadWriteData(handler);
    props.dbSecret.grantRead(handler);

    this.functions = [handler];

    this.api = new apigateway.RestApi(this, "Api", {
      restApiName: "MultiStackApi",
      deployOptions: { stageName: "v1" },
    });

    const items = this.api.root.addResource("items");
    items.addMethod("GET", new apigateway.LambdaIntegration(handler));
    items.addMethod("POST", new apigateway.LambdaIntegration(handler));
  }
}
```

### Monitoring Stack

The monitoring stack consumes resources from the compute stack and sets up CloudWatch alarms with SNS notifications:

```typescript
import * as cdk from "aws-cdk-lib";
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as sns from "aws-cdk-lib/aws-sns";
import * as snsSubscriptions from "aws-cdk-lib/aws-sns-subscriptions";
import * as cwActions from "aws-cdk-lib/aws-cloudwatch-actions";
import { Construct } from "constructs";

export class MonitoringStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: MonitoringStackProps) {
    super(scope, id, props);

    const alarmTopic = new sns.Topic(this, "AlarmTopic");
    alarmTopic.addSubscription(
      new snsSubscriptions.EmailSubscription(props.alarmEmail)
    );

    const apiErrorAlarm = new cloudwatch.Alarm(this, "Api5xxAlarm", {
      metric: props.apiGateway.metricServerError({
        period: cdk.Duration.minutes(5),
        statistic: "Sum",
      }),
      threshold: 10,
      evaluationPeriods: 2,
      alarmDescription: "API Gateway 5xx errors exceeded threshold",
    });
    apiErrorAlarm.addAlarmAction(new cwActions.SnsAction(alarmTopic));

    for (const fn of props.lambdaFunctions) {
      const errorAlarm = new cloudwatch.Alarm(this, `${fn.node.id}ErrorAlarm`, {
        metric: fn.metricErrors({
          period: cdk.Duration.minutes(5),
          statistic: "Sum",
        }),
        threshold: 5,
        evaluationPeriods: 2,
      });
      errorAlarm.addAlarmAction(new cwActions.SnsAction(alarmTopic));
    }
  }
}
```

## Consuming Resources from a Separate CDK App

When a completely different CDK application needs resources from your stacks, you have three options:

```typescript
export class ConsumerStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Option A: CloudFormation import (same account/region)
    const vpcId = cdk.Fn.importValue("Dev-Network-VpcId");

    // Option B: SSM at deploy time (works cross-account with proper IAM)
    const dbEndpoint = ssm.StringParameter.valueForStringParameter(
      this, "/infra/data/db-endpoint"
    );

    // Option C: SSM at synth time (value baked into template)
    const vpcIdFromSsm = ssm.StringParameter.valueFromLookup(
      this, "/infra/network/vpc-id"
    );
  }
}
```

Option B is the most flexible for cross-account scenarios. Option C bakes the value into the CloudFormation template at synthesis time, which means it will not pick up changes until the next `cdk synth`.

## Cross-Account and Cross-Region Deployments

A single CDK app can target multiple accounts and regions:

```typescript
const prodEnv = { account: "222222222222", region: "us-east-1" };
const drEnv   = { account: "222222222222", region: "us-west-2" };

const prodNetwork = new NetworkStack(app, "Prod-Network", {
  env: prodEnv,
  cidr: "10.1.0.0/16",
  maxAzs: 3,
});

const drNetwork = new NetworkStack(app, "DR-Network", {
  env: drEnv,
  cidr: "10.2.0.0/16",
  maxAzs: 2,
});
```

For cross-account SSM reads, the producing account must share the parameter via a resource policy, and the consuming account's deployment role needs `ssm:GetParameter` permission on the producing account's parameter.

## Shared Constructs Library

To enforce organizational standards across teams, publish reusable L3 constructs as an npm or NuGet package:

```typescript
import * as cdk from "aws-cdk-lib";
import * as s3 from "aws-cdk-lib/aws-s3";
import { Construct } from "constructs";

export class SecureBucket extends Construct {
  public readonly bucket: s3.IBucket;

  constructor(scope: Construct, id: string, props: SecureBucketProps = {}) {
    super(scope, id);

    this.bucket = new s3.Bucket(this, "Bucket", {
      versioned: props.versioned ?? true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      enforceSSL: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });
  }
}
```

Every team that uses `SecureBucket` automatically gets encryption, public access blocking, SSL enforcement, and data retention -- without having to remember each setting.

## Pipeline Orchestration with CDK Pipelines

For automated multi-environment deployments, CDK Pipelines provides a self-mutating pipeline that deploys your stacks in the correct order:

```typescript
const pipeline = new CodePipeline(this, "Pipeline", {
  pipelineName: "MultiStackPipeline",
  synth: new ShellStep("Synth", {
    input: CodePipelineSource.gitHub("your-org/your-repo", "main"),
    commands: ["npm ci", "npx cdk synth"],
  }),
  crossAccountKeys: true,
});

pipeline.addStage(new AppStage(this, "Dev", {
  env: { account: "111111111111", region: "us-east-1" },
}));

pipeline.addStage(
  new AppStage(this, "Prod", {
    env: { account: "222222222222", region: "us-east-1" },
  }),
  {
    pre: [new cdk.pipelines.ManualApprovalStep("PromoteToProd")],
  }
);
```

The `AppStage` class wraps all your application stacks (Network, Data, Compute) and is deployed as a unit to each environment.

## Best Practices and Common Pitfalls

### Do

- **Use `addDependency()`** to make deployment order explicit.
- **Set `env` on every stack.** Without it, CDK generates environment-agnostic templates that cannot use `Vpc.fromLookup` or account-specific features.
- **Use SSM for polyrepo communication.** It survives stack deletes and redeployments.
- **Tag everything** at the `App` level so tags cascade to all stacks.
- **Pin construct library versions.** A breaking change in an L2 construct can cascade across all stacks.
- **Use `RemovalPolicy.RETAIN`** on stateful resources so a stack delete does not destroy data.

### Do Not

- **Do not create circular dependencies.** If Stack A references Stack B and vice versa, CDK will fail. Refactor the shared resource into a third stack.
- **Do not use `Fn.importValue` for values that change.** Once consumed, CloudFormation exports are locked until all consumers update. Prefer SSM for volatile values.
- **Do not put everything in one stack.** The 500-resource limit is real, and monolith stacks have long deploy times and large blast radius.
- **Do not hard-code account IDs in stack code.** Use configuration files or context variables.
- **Do not skip `cdk diff` before `cdk deploy`.** Always review what will change, especially with cross-stack references.

### Debugging Cross-Stack Reference Errors

The most common error you will encounter:

```
Export Dev-Network-VpcId cannot be deleted as it is in use by Dev-Data
```

This means you are trying to change or delete an exported value that another stack still imports. The fix is to deploy the consuming stack first to remove the import, then deploy the producing stack.

## Conclusion

Multi-stack CDK architectures give you fine-grained control over blast radius, deployment cadence, and team ownership. The four communication patterns -- direct props, CloudFormation exports, SSM Parameter Store, and shared constructs -- cover every scenario from same-app convenience to cross-account flexibility.

Start by identifying the natural boundaries in your infrastructure (network, data, compute, monitoring), create a stack for each, and wire them together with explicit dependencies. As your system grows, lean on SSM Parameter Store for loose coupling and shared constructs libraries for organizational consistency.

In the next post in this series, we will build a production-ready CI/CD pipeline using GitHub Actions to deploy these multi-stack applications across multiple environments.

## References

- [AWS CDK v2 Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [CDK API Reference -- Construct Library](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-construct-library.html)
- [CDK Pipelines -- Continuous Delivery](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html)
- [CDK Environments](https://docs.aws.amazon.com/cdk/v2/guide/environments.html)
- [CloudFormation Resource Limits](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)
- [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
