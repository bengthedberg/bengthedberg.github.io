---
layout: post
title: "Advanced AWS CDK Patterns: CI/CD with GitHub Actions"
date: 2026-04-10 00:00:00 +0000
categories: [AWS]
tags:
  - aws
  - cdk
  - github-actions
  - cicd
  - oidc
  - deployment
series: "Advanced AWS CDK Patterns"
series_part: 2
---


## Series Overview

1. [Multi-Stack Patterns and Cross-Stack References](/posts/aws-cdk-advanced-part1-multi-stack/)
2. **CI/CD with GitHub Actions** (you are here)
3. [Event-Driven Order Processing](/posts/aws-cdk-advanced-part3-order-processing/)

## Introduction

Deploying AWS CDK stacks manually from a developer's laptop works for prototyping, but it breaks down quickly in a team setting. You need reproducible builds, approval gates for production, and keyless authentication that does not rely on long-lived credentials sitting in a secrets store.

This post walks through building a production-grade CI/CD pipeline for CDK using GitHub Actions. We will cover OIDC-based AWS authentication (no access keys), multi-environment promotion with approval gates, a layered secrets management strategy, and reusable workflow patterns that work for both TypeScript and C# CDK projects.

## Architecture Overview

The deployment pipeline follows a standard promotion model: build and test, deploy to dev, deploy to staging, run integration tests, then deploy to production with manual approval.

```
┌──────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                         │
├──────────────────────────────────────────────────────────────────┤
│   Push to main ──► GitHub Actions Workflow                       │
│                        │                                         │
│                        ▼                                         │
│                ┌──────────────┐                                  │
│                │   Build &    │                                  │
│                │   cdk synth  │                                  │
│                │   + tests    │                                  │
│                └──────┬───────┘                                  │
│                       │                                          │
│            ┌──────────┼──────────┐                               │
│            ▼          ▼          ▼                                │
│     ┌──────────┐ ┌─────────┐ ┌──────────┐                       │
│     │   Dev    │ │ Staging │ │   Prod   │                        │
│     │ (auto)   │ │ (auto)  │ │ (manual  │                        │
│     │          │ │         │ │ approval)│                        │
│     └──────────┘ └─────────┘ └──────────┘                       │
└──────────────────────────────────────────────────────────────────┘
```

Each environment targets a separate AWS account. Dev and staging deploy automatically after tests pass. Production requires manual approval through GitHub's built-in environment protection rules.

## AWS OIDC Authentication

The first rule of CDK CI/CD: never store AWS access keys in GitHub Secrets. Use OIDC federation instead. GitHub's OIDC provider issues short-lived tokens that AWS STS exchanges for temporary credentials, scoped to a single workflow run and automatically expired.

### Step 1: Create the OIDC Provider in AWS

Run this once in every AWS account you want to deploy to:

```bash
aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"
```

### Step 2: Create the IAM Role

Create one role per account with a trust policy that restricts which repository and branches can assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

For production, tighten the `sub` condition to restrict access to a specific GitHub Environment:

```json
"token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:environment:production"
```

This ensures that only workflows running in the `production` GitHub Environment can assume the production role.

### Step 3: Attach Permissions and Bootstrap CDK

The role needs permissions to deploy CloudFormation stacks and manage all resources your CDK stacks create. For production, narrow the policies to specific resource ARNs rather than using broad wildcard permissions.

Bootstrap CDK in each account:

```bash
cdk bootstrap aws://111111111111/us-east-1 \
  --trust 111111111111 \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

## GitHub Environments and Protection Rules

GitHub Environments provide approval gates, environment-scoped secrets, and deployment tracking. Create three environments in your repository under **Settings > Environments**:

- **`development`** -- No protection rules. Deploys automatically on every push to main.
- **`staging`** -- Optional branch restriction to `main`. Deploys automatically after dev succeeds.
- **`production`** -- Required reviewers (1-2 team leads), deployment branch restricted to `main`, optional 5-minute wait timer.

When a workflow job specifies `environment: production`, GitHub pauses execution and waits for reviewer approval before the job runs. The job also gets access to that environment's scoped secrets.

## Secrets Management Strategy

A CDK + GitHub Actions setup involves three layers of secrets, each serving a different purpose.

### Layer 1: GitHub Secrets (Deployment-Time Configuration)

These are values the CI/CD pipeline needs to authenticate and configure deployments. They never reach your application code at runtime.

| Secret | Scope | Purpose |
|--------|-------|---------|
| `AWS_ROLE_ARN` | Per environment | OIDC role for that account |
| `AWS_ACCOUNT_ID` | Per environment | Target AWS account |
| `AWS_REGION` | Repository-wide | Deployment region |

### Layer 2: AWS Secrets Manager (Application Runtime Secrets)

Database passwords, API keys, and third-party tokens that your Lambda or ECS task needs at runtime. These are created and managed by CDK, never stored in GitHub:

```typescript
const dbSecret = new secretsmanager.Secret(this, "DbPassword", {
  secretName: `/${envName}/app/db-password`,
  generateSecretString: {
    secretStringTemplate: JSON.stringify({ username: "admin" }),
    generateStringKey: "password",
    excludePunctuation: true,
  },
});

handler.addEnvironment("DB_SECRET_ARN", dbSecret.secretArn);
dbSecret.grantRead(handler);
```

Notice that only the secret's ARN is passed as an environment variable, not the actual secret value. The Lambda reads the secret at runtime via the SDK.

### Layer 3: SSM Parameter Store (Non-Secret Configuration)

For configuration that varies by environment but is not secret -- feature flags, endpoint URLs, capacity settings:

```typescript
new ssm.StringParameter(this, "ApiUrl", {
  parameterName: `/${envName}/app/api-url`,
  stringValue: api.url,
});
```

### Decision Guide

| Data | Where | Why |
|------|-------|-----|
| OIDC role ARN | GitHub Environment Secret | Needed by GitHub Actions, not by app |
| Database password | AWS Secrets Manager | Rotatable, auditable, runtime access |
| API endpoint URL | SSM Parameter Store | Non-secret, environment-specific config |
| Feature flag | SSM Parameter Store | Easy to change without redeploy |
| Third-party API key | AWS Secrets Manager | Secret, needs rotation |
| VPC CIDR | CDK config file (checked into repo) | Infrastructure constant, not secret |

## Environment Configuration

Keep environment-specific settings in a typed configuration file checked into your repository:

```typescript
export interface EnvironmentConfig {
  envName: string;
  account: string;
  region: string;
  vpcCidr: string;
  lambdaMemory: number;
  enableAlarms: boolean;
  retainData: boolean;
  domainName?: string;
}

export const environments: Record<string, EnvironmentConfig> = {
  dev: {
    envName: "dev",
    account: "111111111111",
    region: "us-east-1",
    vpcCidr: "10.0.0.0/16",
    lambdaMemory: 256,
    enableAlarms: false,
    retainData: false,
  },
  staging: {
    envName: "staging",
    account: "222222222222",
    region: "us-east-1",
    vpcCidr: "10.1.0.0/16",
    lambdaMemory: 512,
    enableAlarms: true,
    retainData: true,
  },
  prod: {
    envName: "prod",
    account: "333333333333",
    region: "us-east-1",
    vpcCidr: "10.2.0.0/16",
    lambdaMemory: 1024,
    enableAlarms: true,
    retainData: true,
    domainName: "api.example.com",
  },
};
```

The CDK entry point allows CI to override the account ID via context, making the GitHub Environment Secret the single source of truth for account targeting:

```typescript
const envName = app.node.tryGetContext("env") ?? "dev";
const accountOverride = app.node.tryGetContext("account");

const config: EnvironmentConfig = {
  ...environments[envName],
  ...(accountOverride ? { account: accountOverride } : {}),
};
```

## GitHub Actions Workflows

### Reusable Composite Action

A composite action eliminates duplication across environments. It handles OIDC authentication, dependency installation (TypeScript or .NET), synthesis, diffing, and deployment:

```yaml
# .github/actions/cdk-deploy/action.yml
name: "CDK Deploy"
description: "Synthesize, diff, and deploy CDK stacks"

inputs:
  environment:
    description: "Target environment (dev, staging, prod)"
    required: true
  aws-role-arn:
    description: "IAM role ARN for OIDC"
    required: true
  aws-region:
    description: "AWS region"
    required: true
  aws-account-id:
    description: "AWS account ID"
    required: true
  working-directory:
    required: false
    default: "infra"

runs:
  using: "composite"
  steps:
    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ inputs.aws-role-arn }}
        aws-region: ${{ inputs.aws-region }}

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: "20"
        cache: "npm"
        cache-dependency-path: "${{ inputs.working-directory }}/package-lock.json"

    - name: Install dependencies
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: npm ci

    - name: Install CDK CLI
      shell: bash
      run: npm install -g aws-cdk

    - name: CDK Diff
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        cdk diff --all \
          -c env=${{ inputs.environment }} \
          -c account=${{ inputs.aws-account-id }} || true

    - name: CDK Deploy
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        cdk deploy --all --require-approval never \
          -c env=${{ inputs.environment }} \
          -c account=${{ inputs.aws-account-id }} \
          --outputs-file cdk-outputs.json

    - name: Upload CDK Outputs
      uses: actions/upload-artifact@v4
      with:
        name: cdk-outputs-${{ inputs.environment }}
        path: ${{ inputs.working-directory }}/cdk-outputs.json
        retention-days: 5
```

### Main Deployment Pipeline

The main workflow chains build, test, and deployment across all three environments:

```yaml
# .github/workflows/deploy.yml
name: Deploy Infrastructure

on:
  push:
    branches: [main]
    paths:
      - "infra/**"
      - "src/**"
      - ".github/workflows/deploy.yml"
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "dev"
        type: choice
        options: [dev, staging, prod]

permissions:
  id-token: write
  contents: read

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: "infra/package-lock.json"
      - name: Install dependencies
        working-directory: infra
        run: npm ci
      - name: Lint
        working-directory: infra
        run: npm run lint
      - name: Unit tests
        working-directory: infra
        run: npm test
      - name: CDK Synth
        working-directory: infra
        run: npx cdk synth --all -c env=dev --quiet

  deploy-dev:
    name: Deploy to Dev
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Dev
        uses: ./.github/actions/cdk-deploy
        with:
          environment: dev
          aws-role-arn: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
          aws-account-id: ${{ secrets.AWS_ACCOUNT_ID }}

  deploy-staging:
    name: Deploy to Staging
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        uses: ./.github/actions/cdk-deploy
        with:
          environment: staging
          aws-role-arn: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
          aws-account-id: ${{ secrets.AWS_ACCOUNT_ID }}

  integration-tests:
    name: Integration Tests
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Download Staging Outputs
        uses: actions/download-artifact@v4
        with:
          name: cdk-outputs-staging
      - name: Run integration tests
        run: |
          API_URL=$(jq -r '."staging-MyApp-Compute".ApiUrlOutput' cdk-outputs.json)
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${API_URL}items")
          if [ "$HTTP_STATUS" != "200" ]; then
            echo "Health check failed with status $HTTP_STATUS"
            exit 1
          fi

  deploy-prod:
    name: Deploy to Production
    needs: integration-tests
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        uses: ./.github/actions/cdk-deploy
        with:
          environment: prod
          aws-role-arn: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
          aws-account-id: ${{ secrets.AWS_ACCOUNT_ID }}
```

Key design decisions in this workflow:

- **Two-phase validation**: PRs only run `build-and-test`. Deployments only happen on pushes to `main`.
- **`concurrency` block**: Prevents two concurrent CDK deploys to the same stack, which would corrupt each other.
- **`--require-approval never`**: Skips the interactive approval prompt in CI. The approval gate comes from the GitHub Environment protection rule instead.
- **`cdk bootstrap` is not in the workflow**: Bootstrap is an account-level one-time operation, not a per-deployment step.

### Pull Request Diff Workflow

Show infrastructure changes on every PR so reviewers can catch surprises before merging:

```yaml
# .github/workflows/pr-diff.yml
name: CDK Diff on PR

on:
  pull_request:
    branches: [main]
    paths: ["infra/**", "src/**"]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  cdk-diff:
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: "infra/package-lock.json"
      - working-directory: infra
        run: npm ci
      - run: npm install -g aws-cdk
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: CDK Diff
        id: diff
        working-directory: infra
        run: |
          set +e
          DIFF_OUTPUT=$(cdk diff --all -c env=dev 2>&1)
          echo "diff<<EOF" >> $GITHUB_OUTPUT
          echo "$DIFF_OUTPUT" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      - name: Post diff as PR comment
        uses: actions/github-script@v7
        with:
          script: |
            const diff = `${{ steps.diff.outputs.diff }}`;
            const body = `## CDK Diff\n\n\`\`\`\n${diff}\n\`\`\``;
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });
            const botComment = comments.find(c => c.body.includes("## CDK Diff"));
            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body,
              });
            }
```

### Destroy Workflow with Safety Checks

The destroy workflow is intentionally gated: it requires typing the environment name to confirm and omits `prod` from the dropdown entirely:

```yaml
# .github/workflows/destroy.yml
name: Destroy Infrastructure

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to destroy"
        required: true
        type: choice
        options: [dev, staging]
      confirm:
        description: "Type the environment name to confirm"
        required: true
        type: string

jobs:
  destroy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    if: github.event.inputs.confirm == github.event.inputs.environment
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
        working-directory: infra
      - run: npm install -g aws-cdk
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: CDK Destroy
        working-directory: infra
        run: |
          cdk destroy --all --force \
            -c env=${{ github.event.inputs.environment }} \
            -c account=${{ secrets.AWS_ACCOUNT_ID }}
```

## Advanced Patterns

### Passing Secrets to CDK at Deploy Time

When a value is both secret and needed at CDK synth time (such as a certificate ARN or hosted zone ID), pass it via CDK context:

```yaml
- name: CDK Deploy
  env:
    HOSTED_ZONE_ID: ${{ secrets.HOSTED_ZONE_ID }}
    CERTIFICATE_ARN: ${{ secrets.CERTIFICATE_ARN }}
  run: |
    cdk deploy --all --require-approval never \
      -c hostedZoneId=$HOSTED_ZONE_ID \
      -c certificateArn=$CERTIFICATE_ARN
```

Then in your CDK code:

```typescript
const hostedZoneId = this.node.tryGetContext("hostedZoneId");
const certificate = acm.Certificate.fromCertificateArn(
  this, "Cert", this.node.tryGetContext("certificateArn")
);
```

### Cross-Stack Outputs Between Workflow Jobs

When a later job needs outputs from an earlier deployment:

```yaml
deploy-dev:
  outputs:
    api-url: ${{ steps.extract.outputs.api-url }}
  steps:
    - name: Extract outputs
      id: extract
      run: |
        API_URL=$(jq -r '."dev-MyApp-Compute".ApiUrlOutput' infra/cdk-outputs.json)
        echo "api-url=$API_URL" >> $GITHUB_OUTPUT

smoke-test:
  needs: deploy-dev
  steps:
    - run: curl -f "${{ needs.deploy-dev.outputs.api-url }}items"
```

## Troubleshooting

**"Unable to assume role" during OIDC auth:** The trust policy `sub` condition does not match. Check that the repository name, branch, and environment in the condition match exactly. Use `StringLike` with wildcards during debugging, then tighten for production.

**"CDK bootstrap version mismatch":** Your deployed bootstrap stack is older than what the CDK CLI expects. Re-run `cdk bootstrap` in the target account with the latest CLI version.

**"Export cannot be deleted as it is in use":** Another stack imports a value you are changing. Deploy the consuming stack first to remove the import, then deploy the producing stack.

## Security Best Practices

1. **Never store AWS keys in GitHub Secrets.** OIDC is the only acceptable pattern.
2. **Scope OIDC trust policies tightly.** Use `environment:production` in the `sub` condition for prod accounts.
3. **Separate AWS accounts per environment.** A compromised dev role cannot touch prod.
4. **Pin action versions** to commit SHAs for maximum security.
5. **Never echo secrets in logs.** Use `add-mask` for derived values that might contain tokens.
6. **Use `RemovalPolicy.RETAIN`** on stateful resources in staging and production.

## Conclusion

A well-designed CDK CI/CD pipeline eliminates the risks of manual deployments while maintaining the safety gates that production workloads demand. OIDC authentication removes the burden of credential rotation. GitHub Environments provide approval workflows without external tooling. And a layered secrets strategy keeps sensitive values where they belong -- deployment-time configuration in GitHub, runtime secrets in AWS Secrets Manager, and non-secret configuration in SSM Parameter Store.

The reusable composite action pattern keeps your workflows DRY across environments and makes it straightforward to support both TypeScript and C# CDK projects. Combined with the PR diff workflow, your team gets infrastructure change visibility at code review time, long before changes reach production.

In the next post, we will apply these patterns to build a complete event-driven order processing system using SQS, Lambda, and DynamoDB.

## References

- [GitHub OIDC with AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
- [GitHub Environments for Deployment Protection](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
- [CDK Pipelines -- Continuous Delivery](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html)
- [AWS CDK Bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html)
