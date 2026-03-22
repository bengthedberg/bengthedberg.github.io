---
layout: post
title: "Connecting GitHub Actions to AWS the Right Way: OIDC Federation"
date: 2025-12-18 00:00:00 +0000
tags:
  - aws
  - github-actions
  - cicd
  - oidc
  - security
  - iam
---


## Introduction

If your GitHub Actions workflows deploy to AWS, you need credentials. The old approach — storing long-lived IAM access keys as GitHub secrets — works, but it's a security liability. Keys don't expire automatically, they can leak, and rotating them is a manual chore.

The modern approach is **OIDC federation**: GitHub proves its identity to AWS using short-lived tokens, and AWS grants temporary credentials in return. No secrets stored in GitHub. No keys to rotate. No credentials that can leak in logs.

This article walks through how it works and how to set it up with a CloudFormation template you can deploy today.

## How OIDC Federation Works

The flow has three actors: your GitHub Actions workflow, GitHub's OIDC provider, and AWS IAM.

1. **GitHub Actions requests a token** — When your workflow runs, GitHub generates a signed JWT (JSON Web Token) containing claims about the workflow: which repository triggered it, which branch, which environment.

2. **The workflow presents the token to AWS** — Using the `aws-actions/configure-aws-credentials` action, the workflow calls `sts:AssumeRoleWithWebIdentity`, passing the GitHub-signed JWT.

3. **AWS validates and grants credentials** — AWS checks the token's signature against GitHub's OIDC provider (which you registered as a trusted identity provider in IAM). If the token is valid and the claims match your role's trust policy conditions, AWS issues temporary credentials (typically valid for 1 hour).

The result: your workflow gets credentials that are scoped, short-lived, and automatically rotated on every run. Nothing is stored in GitHub secrets.

## Step-by-Step Setup

### Step 1: Register GitHub as an OIDC Provider in AWS

You need to tell AWS to trust tokens signed by GitHub. This is a one-time setup per AWS account — all repositories in your GitHub organisation can share the same provider.

Deploy the CloudFormation template below. It creates:
- An **OIDC identity provider** pointing to `token.actions.githubusercontent.com`
- An **IAM role** that GitHub Actions can assume

### Step 2: Deploy the CloudFormation Template

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: >
  GitHub OIDC Provider and IAM Role for GitHub Actions.
  See: https://github.com/aws-actions/configure-aws-credentials

Parameters:
  GitHubOrg:
    Type: String
    Description: Your GitHub organisation or username
  RepositoryName:
    Type: String
    Default: "*"
    Description: >
      Repository name to allow (use * for all repos in the org).
      For tighter security, specify a single repo name.
  OIDCProviderArn:
    Description: >
      ARN of an existing GitHub OIDC Provider.
      Leave empty to create a new one.
    Default: ""
    Type: String

Conditions:
  CreateOIDCProvider: !Equals
    - !Ref OIDCProviderArn
    - ""

Resources:
  GitHubOIDC:
    Type: AWS::IAM::OIDCProvider
    Condition: CreateOIDCProvider
    Properties:
      Url: https://token.actions.githubusercontent.com
      ClientIdList:
        - sts.amazonaws.com
      ThumbprintList:
        - 6938fd4d98bab03faadb97b34396831e3780aea1
        - 1c58a3a8518e8759bf075b76b750d4f2df264fcd

  GitHubActionsRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: GitHubActions
      MaxSessionDuration: 3600
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action: sts:AssumeRoleWithWebIdentity
            Principal:
              Federated: !If
                - CreateOIDCProvider
                - !Ref GitHubOIDC
                - !Ref OIDCProviderArn
            Condition:
              StringEquals:
                token.actions.githubusercontent.com:aud: sts.amazonaws.com
              StringLike:
                token.actions.githubusercontent.com:sub: !Sub repo:${GitHubOrg}/${RepositoryName}:*
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AdministratorAccess

Outputs:
  RoleARN:
    Description: ARN of the IAM role for GitHub Actions
    Value: !GetAtt GitHubActionsRole.Arn
```

Deploy it:

```bash
aws cloudformation deploy \
  --template-file github-oidc.yaml \
  --stack-name github-oidc \
  --parameter-overrides GitHubOrg=your-org-name \
  --capabilities CAPABILITY_NAMED_IAM
```

### Step 3: Configure Your GitHub Actions Workflow

In your workflow, use the `aws-actions/configure-aws-credentials` action to assume the role:

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-arn: arn:aws:iam::123456789012:role/GitHubActions
          aws-region: ap-southeast-2

      - name: Verify credentials
        run: aws sts get-caller-identity

      - name: Deploy
        run: |
          # Your deployment commands here
          cdk deploy --require-approval never
```

The critical line is `permissions: id-token: write` — without it, the workflow can't request an OIDC token from GitHub.

## Understanding the Trust Policy

The trust policy on the IAM role controls *which* GitHub workflows can assume it. Let's break down the conditions:

```yaml
Condition:
  StringEquals:
    token.actions.githubusercontent.com:aud: sts.amazonaws.com
  StringLike:
    token.actions.githubusercontent.com:sub: !Sub repo:${GitHubOrg}/${RepositoryName}:*
```

- **`aud` (audience)**: Must be `sts.amazonaws.com` — ensures the token was intended for AWS
- **`sub` (subject)**: Matches the repository and optionally the branch, environment, or event type

### Tightening the Subject Claim

The `sub` claim format is: `repo:<org>/<repo>:<context>`

You can restrict access beyond just the repository:

| Pattern | Allows |
|---------|--------|
| `repo:my-org/*:*` | Any repo in the org, any branch |
| `repo:my-org/my-repo:*` | Specific repo, any branch |
| `repo:my-org/my-repo:ref:refs/heads/main` | Only the `main` branch |
| `repo:my-org/my-repo:environment:production` | Only the `production` environment |
| `repo:my-org/my-repo:pull_request` | Only pull request events |

For production deployments, always restrict to a specific branch or environment rather than using wildcards.

## Security Considerations

### Don't Use AdministratorAccess in Production

The template above attaches `AdministratorAccess` for simplicity. In practice, create a custom policy with only the permissions your workflow needs:

```yaml
ManagedPolicyArns:
  # Instead of AdministratorAccess, use specific policies:
  - arn:aws:iam::aws:policy/AmazonS3FullAccess
  - arn:aws:iam::aws:policy/AWSCloudFormationFullAccess
  - arn:aws:iam::aws:policy/AWSLambda_FullAccess
```

Or better yet, write an inline policy scoped to the exact resources your deployment touches.

### One Role Per Deployment Scope

Consider creating separate roles for different trust levels:
- `GitHubActions-Deploy-Prod` — restricted to `main` branch, limited permissions
- `GitHubActions-Deploy-Dev` — broader access, any branch
- `GitHubActions-ReadOnly` — for workflows that only need to read from AWS (e.g., integration tests)

### Session Duration

The template sets `MaxSessionDuration: 3600` (1 hour). This is usually sufficient for CI/CD workflows. If your deployments take longer, increase it — but keep it as short as practical.

## Key Takeaways

1. **Stop using long-lived IAM keys in GitHub secrets.** OIDC federation is more secure, requires no rotation, and is straightforward to set up.
2. **The OIDC provider is per-account, the role is per-use-case.** Register the provider once, then create as many roles as you need with different trust policies and permissions.
3. **Restrict the subject claim.** Don't use `*` wildcards in production — scope to specific repos, branches, or environments.
4. **Apply least-privilege to the IAM role.** `AdministratorAccess` is fine for getting started, but scope it down before going to production.
5. **Remember `permissions: id-token: write`** in your workflow — it's the most common reason OIDC auth fails.

## References

- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub Docs: Configuring OIDC in AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [AWS Docs: Creating OIDC Identity Providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
