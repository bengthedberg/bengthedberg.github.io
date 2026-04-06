---
layout: post
title: "Deploy to AWS with S3, CloudFront & CDK"
date: 2026-03-20 00:00:00 +0000
tags:
  - react
  - aws
  - s3
  - cloudfront
  - cdk
  - deployment
series: "Building Modern React SPAs"
series_part: 6
---


## Series Overview

1. [TypeScript Patterns for React](/posts/react-spa-part1-typescript-patterns/)
2. [Type-Safe Routing with TanStack Router](/posts/react-spa-part2-tanstack-router/)
3. [Server State with TanStack Query](/posts/react-spa-part3-tanstack-query/)
4. [Styling with Tailwind CSS v4 & shadcn/ui](/posts/react-spa-part4-tailwind-shadcn/)
5. [Modular Feature Architecture](/posts/react-spa-part5-architecture/)
6. **Deploy to AWS with S3, CloudFront & CDK** (this post)

## Introduction

With our React SPA built using TanStack Router, TanStack Query, Tailwind CSS, and a modular feature architecture, the final step is getting it into production. This post covers deploying to AWS using S3 for static file hosting, CloudFront as a CDN with HTTPS, and AWS CDK for infrastructure-as-code.

The architecture is straightforward: the browser hits CloudFront, which serves files from a private S3 bucket via Origin Access Control (OAC). Client-side routing is handled by custom error responses that redirect back to `index.html`, letting TanStack Router manage navigation on the client.

## Architecture Overview

```
Browser --HTTPS--> CloudFront (CDN, TLS 1.2+, HTTP/2+3)
                        |
                        |  Origin Access Control (OAC)
                        |  signs every request with SigV4
                        v
                   S3 Bucket (PRIVATE, block all public access)
                        |
                   Static assets: index.html, JS, CSS, images
```

Key design decisions:

- The S3 bucket stays **fully private** -- no public access, no static website hosting enabled. CloudFront uses Origin Access Control (OAC) to sign requests using SigV4 short-term credentials. This is the AWS-recommended approach, replacing the legacy Origin Access Identity (OAI).
- Client-side routing is handled via **CloudFront custom error responses** that redirect 403/404 errors back to `index.html`, allowing TanStack Router to handle navigation.
- CDK's **BucketDeployment** construct uploads the built React app to S3 and automatically invalidates the CloudFront cache.

## Prerequisites

| Tool | Minimum Version | Purpose |
|------|----------------|---------|
| Node.js | v18+ | Runtime for Vite, CDK, and npm |
| AWS CLI | v2 | Authenticate and interact with AWS |
| AWS CDK CLI | v2.150+ | Synthesize and deploy CDK stacks |
| TypeScript | v5.3+ | Required by TanStack Router |

### AWS Account Setup

```bash
# Configure credentials
aws configure

# Verify
aws sts get-caller-identity
```

## Project Structure

```
my-app/
├── frontend/                    # React TypeScript SPA (Vite + TanStack)
│   ├── src/
│   │   ├── routes/              #   TanStack Router file-based routes
│   │   ├── features/            #   Feature modules
│   │   ├── shared/              #   Shared code
│   │   └── main.tsx             #   Entry point
│   ├── package.json
│   └── vite.config.ts
│
├── infra/                       # CDK TypeScript project
│   ├── bin/
│   │   └── infra.ts             #   CDK App entry point
│   ├── lib/
│   │   └── spa-stack.ts         #   Main infrastructure stack
│   ├── cdk.json
│   └── package.json
│
└── .github/
    └── workflows/
        └── deploy.yml           # CI/CD pipeline (optional)
```

## Phase 1: Frontend Build Configuration

### Vite Configuration

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { tanstackRouter } from '@tanstack/router-plugin/vite';
import path from 'path';

export default defineConfig({
  plugins: [
    tanstackRouter({
      target: 'react',
      autoCodeSplitting: true,
    }),
    react(),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  build: {
    sourcemap: false,
    outDir: 'dist',
    chunkSizeWarningLimit: 500,
  },
});
```

The `autoCodeSplitting: true` option automatically wraps each route component in `React.lazy()`, so only the code needed for the current route is downloaded.

### Verify the Build

```bash
cd frontend
npm run build
npm run preview
# Test navigation, deep links, and data loading
```

## Phase 2: CDK Infrastructure Setup

### Initialize the CDK Project

```bash
mkdir infra && cd infra
cdk init app --language typescript
```

CDK v2 consolidates all construct libraries into the single `aws-cdk-lib` package.

### Bootstrap the AWS Environment

Bootstrapping creates a `CDKToolkit` CloudFormation stack with the resources CDK needs for deployment. Each AWS account + region pair must be bootstrapped once:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
cdk bootstrap aws://$ACCOUNT_ID/us-east-1 --termination-protection
```

## Phase 3: The CDK Stack

### CDK App Entry Point

```typescript
// infra/bin/infra.ts
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { SpaStack } from '../lib/spa-stack';

const app = new cdk.App();

new SpaStack(app, 'ReactSpaStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
  tags: {
    Project: 'react-spa',
    ManagedBy: 'cdk',
  },
});
```

### The Main Stack

```typescript
// infra/lib/spa-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as s3deploy from 'aws-cdk-lib/aws-s3-deployment';
import * as path from 'path';

export class SpaStack extends cdk.Stack {
  public readonly distribution: cloudfront.Distribution;
  public readonly siteBucket: s3.Bucket;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1. S3 BUCKET -- Private, no website hosting
    this.siteBucket = new s3.Bucket(this, 'SiteBucket', {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    // 2. SECURITY RESPONSE HEADERS POLICY
    const responseHeadersPolicy = new cloudfront.ResponseHeadersPolicy(
      this, 'SecurityHeaders', {
        responseHeadersPolicyName: `${id}-security-headers`,
        securityHeadersBehavior: {
          contentTypeOptions: { override: true },
          frameOptions: {
            frameOption: cloudfront.HeadersFrameOption.DENY,
            override: true,
          },
          referrerPolicy: {
            referrerPolicy:
              cloudfront.HeadersReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN,
            override: true,
          },
          strictTransportSecurity: {
            accessControlMaxAge: cdk.Duration.days(365),
            includeSubdomains: true,
            preload: true,
            override: true,
          },
          xssProtection: {
            protection: true,
            modeBlock: true,
            override: true,
          },
        },
      }
    );

    // 3. CLOUDFRONT DISTRIBUTION WITH OAC
    this.distribution = new cloudfront.Distribution(this, 'SiteDistribution', {
      comment: 'React SPA with TanStack Router + Query',
      defaultBehavior: {
        origin: origins.S3BucketOrigin.withOriginAccessControl(this.siteBucket),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
        responseHeadersPolicy,
        compress: true,
      },
      defaultRootObject: 'index.html',
      errorResponses: [
        {
          httpStatus: 403,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
          ttl: cdk.Duration.minutes(5),
        },
        {
          httpStatus: 404,
          responseHttpStatus: 200,
          responsePagePath: '/index.html',
          ttl: cdk.Duration.minutes(5),
        },
      ],
      minimumProtocolVersion: cloudfront.SecurityPolicyProtocol.TLS_V1_2_2021,
      httpVersion: cloudfront.HttpVersion.HTTP2_AND_3,
      enableIpv6: true,
      priceClass: cloudfront.PriceClass.PRICE_CLASS_100,
    });

    // 4. DEPLOY FRONTEND BUILD TO S3 + INVALIDATE
    new s3deploy.BucketDeployment(this, 'DeploySite', {
      sources: [
        s3deploy.Source.asset(path.join(__dirname, '../../frontend/dist')),
      ],
      destinationBucket: this.siteBucket,
      distribution: this.distribution,
      distributionPaths: ['/*'],
      memoryLimit: 256,
    });

    // 5. STACK OUTPUTS
    new cdk.CfnOutput(this, 'CloudFrontURL', {
      value: `https://${this.distribution.distributionDomainName}`,
      description: 'CloudFront Distribution URL',
    });

    new cdk.CfnOutput(this, 'CloudFrontDistributionId', {
      value: this.distribution.distributionId,
      description: 'Distribution ID (for manual invalidations)',
    });

    new cdk.CfnOutput(this, 'S3BucketName', {
      value: this.siteBucket.bucketName,
    });
  }
}
```

### Understanding Each Construct

**S3 Bucket** -- Private, no website hosting. OAC requires the S3 REST API endpoint, not the website endpoint. Do not enable "static website hosting."

**CloudFront Distribution with OAC** -- `S3BucketOrigin.withOriginAccessControl()` replaces the deprecated `S3Origin` (which used legacy OAI). It automatically creates an OAC, attaches it, and updates the bucket policy.

**Error Responses for SPA Routing** -- When TanStack Router navigates to `/posts/42`, S3 returns 403 (no such object). CloudFront rewrites this to `/index.html` with status 200, and TanStack Router handles the path client-side.

**BucketDeployment** -- Uses a Lambda to sync files from `frontend/dist/` to S3 and invalidate CloudFront cache. Increase `memoryLimit` for large bundles.

**Response Headers Policy** -- Adds HSTS, X-Frame-Options, X-Content-Type-Options, XSS-Protection, and Referrer-Policy to all responses.

## Phase 4: Build and Deploy

```bash
# 1. Build the frontend
cd frontend
npm run build
cd ..

# 2. Synthesize and review the CloudFormation template
cd infra
npx cdk synth

# 3. Preview changes
npx cdk diff

# 4. Deploy
npx cdk deploy

# 5. Verify: open the CloudFrontURL output in your browser
#    Test: navigate to /about, refresh the page
#    Test: navigate to /posts, verify data loads
```

For subsequent code-only deployments:

```bash
cd frontend && npm run build && cd ../infra && npx cdk deploy
```

## Phase 5: Custom Domain with SSL (Optional)

```typescript
import * as acm from 'aws-cdk-lib/aws-certificatemanager';
import * as route53 from 'aws-cdk-lib/aws-route53';
import * as route53targets from 'aws-cdk-lib/aws-route53-targets';

const domainName = 'app.example.com';

const hostedZone = route53.HostedZone.fromLookup(this, 'HostedZone', {
  domainName: 'example.com',
});

const certificate = new acm.Certificate(this, 'SiteCertificate', {
  domainName,
  validation: acm.CertificateValidation.fromDns(hostedZone),
});

// Add to Distribution props:
//   domainNames: [domainName],
//   certificate,

new route53.ARecord(this, 'SiteAliasRecord', {
  zone: hostedZone,
  recordName: domainName,
  target: route53.RecordTarget.fromAlias(
    new route53targets.CloudFrontTarget(this.distribution)
  ),
});
```

CloudFront requires ACM certificates in `us-east-1`. Either deploy the stack to `us-east-1` or create a separate certificate stack in that region.

## Phase 6: CI/CD Pipeline (Optional)

### GitHub Actions with OIDC

```yaml
# .github/workflows/deploy.yml
name: Deploy React SPA
on:
  push:
    branches: [main]

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: |
            frontend/package-lock.json
            infra/package-lock.json

      - name: Build frontend
        working-directory: frontend
        run: |
          npm ci
          npm run build

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsDeployRole
          aws-region: us-east-1

      - name: Deploy CDK
        working-directory: infra
        run: |
          npm ci
          npx cdk deploy --require-approval never
```

## Troubleshooting Guide

### 403 on Deep Links (e.g., /about, /posts/42)

**Cause:** S3 returns 403 for non-existent objects in a private bucket.
**Fix:** Ensure CloudFront error responses map 403 and 404 to `/index.html` with status 200.

### routeTree.gen.ts Not Generated

**Cause:** The Vite dev server is not running, or the TanStack Router plugin is not configured.
**Fix:** Ensure `tanstackRouter()` is listed BEFORE `react()` in `vite.config.ts` plugins. Start the dev server with `npm run dev`.

### Stale Content After Deployment

**Fix:** Ensure `distributionPaths: ['/*']` is set on BucketDeployment. For manual invalidation:

```bash
aws cloudfront create-invalidation --distribution-id DIST_ID --paths "/*"
```

### BucketDeployment Lambda Timeout

**Fix:** Increase `memoryLimit` to 512 or use `aws s3 sync` in CI/CD instead.

### "S3Origin is deprecated" Warnings

**Fix:** Replace `new origins.S3Origin(bucket)` with `origins.S3BucketOrigin.withOriginAccessControl(bucket)`.

## Production Hardening Checklist

- S3 bucket: Change to `RemovalPolicy.RETAIN`, remove `autoDeleteObjects`
- Add WAF on CloudFront for bot protection and rate limiting
- Set Cache-Control: long `max-age` on hashed assets, `no-cache` on `index.html`
- Enable CloudFront access logging to a separate S3 bucket
- Configure CloudWatch alarms for 4xx/5xx error rates
- Add Content-Security-Policy header tailored to your app
- Set up environment separation (dev/staging/prod stacks or accounts)
- Enable bootstrap termination protection on production
- Tune TanStack Query `staleTime` and `gcTime` per query
- Add TanStack Router error boundaries on routes

## Conclusion

This concludes the "Building Modern React SPAs" series. We have covered the full stack: TypeScript patterns for type-safe components, TanStack Router for compile-time safe navigation, TanStack Query for declarative server-state management, Tailwind CSS v4 and shadcn/ui for a themeable design system, a modular feature architecture that scales with your team, and finally AWS infrastructure with S3, CloudFront, and CDK.

Together, these tools and patterns form a production-ready foundation for building modern single-page applications. The key theme throughout has been type safety and developer experience -- catching errors at compile time, eliminating boilerplate, and providing clear patterns that make the codebase predictable as it grows.

## References

- [AWS CDK v2 Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [CDK Bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html)
- [CloudFront Origins (CDK)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cloudfront_origins-readme.html)
- [S3 Bucket (CDK)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html)
- [BucketDeployment (CDK)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3_deployment-readme.html)
- [CloudFront OAC L2 Construct RFC](/posts/https://github.com/aws/aws-cdk-rfcs/blob/main/text/0617-cloudfront-oac-l2/)
- [AWS Blog: CDK L2 for CloudFront OAC](https://aws.amazon.com/blogs/devops/a-new-aws-cdk-l2-construct-for-amazon-cloudfront-origin-access-control-oac/)
- [Deploy React SPA to S3+CloudFront (AWS Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/deploy-a-react-based-single-page-application-to-amazon-s3-and-cloudfront.html)
- [GitHub OIDC for AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Vite Build Options](https://vite.dev/config/build-options.html)
