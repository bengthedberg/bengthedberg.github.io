---
layout: post
title: "AWS Networking Architecture & Cost Optimization"
date: 2026-04-06 00:00:00 +0000
categories: [AWS]
tags:
  - aws
  - networking
  - vpc
  - cost-optimization
  - nat-gateway
  - direct-connect
---


## Summary

Building in the cloud requires dual mastery: understanding how to connect services and understanding how those connections impact your bottom line. While basic networking components like VPCs and subnets provide the structural "fence" for your infrastructure, the traffic flowing through them can lead to significant monthly surprises if not architected with cost in mind.

This guide breaks down AWS networking into its free and paid layers, quantifies the real costs, and provides architectural strategies to optimize spend without sacrificing security.

## 1. The Core Infrastructure (The "Free" Layer)

One of the most important things to understand is that AWS does **not** charge for the structural blueprints of your network. You can build a complex architecture without an hourly fee for the following components:

### VPCs and Subnets

There is no charge for creating a Virtual Private Cloud or adding subnets to it. You can create multiple VPCs per region and divide them into as many subnets as your architecture requires — all at zero cost.

```
# Create a VPC — no charge
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create subnets across AZs — no charge
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.2.0/24 --availability-zone us-east-1b
```

### Routing and Gateways

Attaching Route Tables, Internet Gateways, or Virtual Private Gateways does not incur an hourly charge. You can create as many route tables as you need and associate them with subnets freely.

### Security Layers

Both security mechanisms are free to configure and use:

| Component | Scope | Stateful | Rules | Cost |
|-----------|-------|----------|-------|------|
| **Network ACLs (NACLs)** | Subnet level | No — must define inbound AND outbound rules | Evaluated in number order, first match wins | Free |
| **Security Groups** | Instance/ENI level | Yes — return traffic is automatically allowed | All rules evaluated, allow-only | Free |

### What This Means

You should never hesitate to build a well-structured network with multiple subnets, route tables, and security layers. The complexity of your network design has **no direct cost** — only the traffic and managed services within it do.

## 2. The Cost of Connectivity: NAT Gateways and Egress

The "pay as you go" model means you are billed heavily for **managed services** and **data leaving** the AWS network. This is where most networking cost surprises happen.

### The NAT Gateway "Tax"

A NAT Gateway allows instances in a private subnet to reach the internet for updates, API calls, and package downloads — without being exposed to inbound threats.

**The problem: NAT Gateways are expensive.**

#### Hourly Cost

| Configuration | Monthly Base Cost |
|---------------|-------------------|
| Single NAT Gateway (one AZ) | ~$32/month |
| High availability (3 AZs) | ~$100/month |
| HA across 2 regions | ~$200/month |

That is the cost **just for the NAT Gateway to exist** — before a single byte flows through it.

#### Data Processing Cost

Every gigabyte that passes through the NAT Gateway incurs a processing charge:

| Volume | NAT Processing Cost |
|--------|-------------------|
| 100 GB/month | ~$4.50 |
| 1 TB/month | ~$45 |
| 10 TB/month | ~$450 |

#### Real-World Example: Software Updates

Consider 50 EC2 instances in private subnets, each downloading a 500 MB OS update monthly:

```
50 instances × 500 MB = 25 GB through NAT Gateway
NAT processing: 25 GB × $0.045 = $1.13
NAT hourly: $0.045/hr × 730 hrs = $32.85
Total: ~$34/month just for updates
```

Now multiply that by daily container pulls, API calls to external services, and log shipping. Costs escalate quickly.

### Internet Egress: The Hidden Multiplier

Traffic **entering** your VPC from the internet is free, but **egress** (traffic leaving) is expensive:

| Traffic Direction | Cost |
|-------------------|------|
| Internet → VPC (ingress) | **Free** |
| VPC → Internet (egress) | ~$90/TB |
| VPC → Internet via NAT Gateway | ~$136/TB |
| Same AZ traffic | **Free** |
| Cross-AZ traffic | ~$10/TB (each direction) |
| Cross-region traffic | ~$20/TB |

#### Cost Breakdown: 1 TB Leaving via NAT Gateway

```
Internet egress:     $0.09/GB × 1,024 GB = $92.16
NAT processing:      $0.045/GB × 1,024 GB = $46.08
                                    Total = $138.24/TB
```

That is **$138 per terabyte** for data leaving your private subnet to the internet. A video streaming service, API-heavy application, or data pipeline can easily hit multiple terabytes per month.

### Cost Comparison: Traffic Paths

```
┌──────────────────────────────────────────────────────────┐
│                    Cost per TB                           │
│                                                          │
│  Same AZ:          $0     ████                           │
│  Cross-AZ:         $20    ████████                       │
│  Internet egress:  $90    ████████████████████████████    │
│  NAT + egress:     $136   ████████████████████████████████│
└──────────────────────────────────────────────────────────┘
```

## 3. High-Volume Data Transfer: Direct Connect vs. VPN

When connecting on-premises data centres to AWS, the choice of technology depends on your data volume and reliability requirements.

### Site-to-Site VPN

A Site-to-Site VPN creates an encrypted tunnel over the public internet between your on-premises network and your VPC.

- **How it works**: IPSec tunnel over the internet
- **Setup time**: Minutes to hours
- **Bandwidth**: Limited by your internet connection
- **Redundancy**: Can set up dual tunnels to different AZs

```
On-Premises Router ──── IPSec Tunnel ──── AWS VPN Gateway
                     (over internet)           │
                                          Your VPC
```

### AWS Direct Connect

Direct Connect provides a dedicated physical connection between your data centre and AWS, bypassing the public internet entirely.

- **How it works**: Dedicated fibre connection through a Direct Connect partner
- **Setup time**: Weeks to months (physical provisioning)
- **Bandwidth**: 1 Gbps, 10 Gbps, or 100 Gbps dedicated
- **Redundancy**: Requires separate connections for HA

```
On-Premises Router ──── Dedicated Fibre ──── AWS Direct Connect
                     (private network)            │
                                             Your VPC
```

### Cost Comparison

| Feature | Site-to-Site VPN | Direct Connect |
|---------|-----------------|----------------|
| **Fixed monthly cost** | ~$36/month | $240 – $1,800+/month |
| **Data transfer cost** | ~$90/TB (internet egress) | ~$21 – $24/TB |
| **Setup time** | Minutes | Weeks |
| **Encryption** | Built-in (IPSec) | Optional (MACsec or VPN overlay) |
| **Latency** | Variable (internet) | Consistent and low |
| **Bandwidth** | Shared (internet) | Dedicated |

### Break-Even Analysis

```
Monthly data transfer: X TB

VPN cost:    $36 + ($90 × X)
DX cost:     $240 + ($22 × X)

Break-even:  $36 + $90X = $240 + $22X
             $68X = $204
             X ≈ 3 TB/month
```

**Rule of thumb**: If you transfer more than **3 TB/month** between on-premises and AWS, Direct Connect is likely cheaper. Below that, VPN wins on both cost and simplicity.

| Monthly Transfer | VPN Cost | Direct Connect Cost | Winner |
|------------------|----------|--------------------|----|
| 1 TB | $126 | $262 | VPN |
| 3 TB | $306 | $306 | Tie |
| 5 TB | $486 | $350 | Direct Connect |
| 10 TB | $936 | $460 | Direct Connect |
| 50 TB | $4,536 | $1,340 | Direct Connect |

## 4. Architectural Trade-offs: Security vs. Budget

There is a frequent tension between following "best practices" and minimizing costs. Understanding these trade-offs allows you to make informed decisions based on your specific risk tolerance and budget.

### The "No-NAT" Approach

To avoid NAT Gateway fees entirely, some architects make all subnets "public" (giving them a route to the Internet Gateway) but simply **refuse to assign public IP addresses** to backend instances.

```
┌─────────────────────────────────────┐
│  "Public" Subnet (with IGW route)   │
│                                     │
│  ┌──────────┐    ┌──────────┐      │
│  │ Instance │    │ Instance │      │
│  │ (no pub  │    │ (no pub  │      │
│  │  IP)     │    │  IP)     │      │
│  └──────────┘    └──────────┘      │
│                                     │
│  Route: 0.0.0.0/0 → igw-xxx       │
└─────────────────────────────────────┘
```

**Pros:**
- Eliminates ~$32–100/month NAT Gateway base cost
- Eliminates $45/TB NAT processing charges
- Simpler architecture

**Cons:**
- Risk of administrative error — if someone accidentally assigns a public IP (or enables `auto-assign public IP` on the subnet), the instance becomes **instantly exposed** to the internet
- Violates the principle of defence in depth
- May not pass security audits or compliance requirements (PCI-DSS, HIPAA, SOC 2)
- No outbound internet access unless you assign a public IP or Elastic IP

**When this is acceptable:**
- Development and test environments
- Small projects with a single operator
- Non-sensitive workloads
- Learning and experimentation

**When this is NOT acceptable:**
- Production environments handling customer data
- Regulated industries
- Multi-team environments where subnet configuration is shared

### Caching and Proxies: Reducing NAT Traffic

Instead of eliminating the NAT Gateway, you can dramatically reduce the traffic flowing through it:

#### Package Caching Proxy

Instead of 100 Linux instances each downloading updates through the NAT Gateway, one server downloads the update once and distributes it locally.

```
WITHOUT proxy:
  100 instances × 200 MB update = 20 GB through NAT
  Cost: 20 GB × $0.045 = $0.90 per update cycle

WITH Squid proxy:
  1 download × 200 MB through NAT = 0.2 GB
  99 instances download from proxy (local traffic = free)
  Cost: 0.2 GB × $0.045 = $0.009 per update cycle
  Savings: 99%
```

Tools for this:
- **Squid** — general HTTP caching proxy
- **Apt-Cacher-NG** — Debian/Ubuntu package cache
- **Nexus Repository** — Maven, npm, Docker, and more
- **AWS CodeArtifact** — managed package repository (npm, Maven, PyPI, NuGet)

#### ECR Pull-Through Cache

For Docker images, use **ECR pull-through cache** to avoid pulling from Docker Hub through your NAT Gateway repeatedly:

```bash
# Configure once — subsequent pulls use the cache
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix docker-hub \
  --upstream-registry-url registry-1.docker.io
```

#### S3 and DynamoDB Gateway Endpoints

Gateway endpoints route traffic to S3 and DynamoDB **without going through the NAT Gateway** — and they are **free**.

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-xxx
```

**Impact**: If your application reads/writes heavily to S3 (logs, backups, data lakes), a gateway endpoint can eliminate hundreds of dollars in monthly NAT processing charges.

#### Interface Endpoints for AWS Services

For services like SQS, SNS, Secrets Manager, and CloudWatch, **interface endpoints** keep traffic on the AWS private network:

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.sqs \
  --subnet-ids subnet-xxx
```

**Cost**: ~$7.20/month per endpoint per AZ + $10/TB processed. Compare this to NAT Gateway costs — for high-volume AWS service calls, interface endpoints can be significantly cheaper.

### Decision Matrix

| Scenario | Recommended Approach | Monthly Overhead |
|----------|---------------------|-----------------|
| Dev/test, < $100 budget | Public subnets, no NAT, strict SGs | ~$0 |
| Small production, low traffic | Single NAT Gateway + S3 endpoint | ~$35 |
| Standard production, multi-AZ | NAT per AZ + gateway endpoints + caching proxy | ~$100–150 |
| High security, high traffic | NAT per AZ + interface endpoints + Direct Connect | ~$500+ |
| Massive data transfer (50+ TB) | Direct Connect + all endpoint types | Custom |

## 5. Proactive Cost Management

Because a misconfigured script, a runaway container, or a forgotten test environment can burn through a three-month budget in weeks, monitoring is essential.

### Budget Alerts

Set thresholds to receive notifications before spend gets out of control:

```bash
# Create a budget with email alert at $100
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "NetworkingBudget",
    "BudgetLimit": {"Amount": "100", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "Service": ["Amazon Virtual Private Cloud", "AWS Direct Connect"]
    }
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{
      "SubscriptionType": "EMAIL",
      "Address": "alerts@example.com"
    }]
  }]'
```

**Recommended thresholds:**
- 50% — early warning
- 80% — investigate immediately
- 100% — action required

### Automated Zombie Resource Audits

Use scripts to check for resources left running after training, testing, or experimentation:

```bash
#!/bin/bash
# Find NAT Gateways that might be forgotten
echo "=== NAT Gateways ==="
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --query 'NatGateways[].{Id:NatGatewayId,VPC:VpcId,Subnet:SubnetId,Created:CreateTime}' \
  --output table

# Find unattached Elastic IPs (charged when not associated)
echo "=== Unused Elastic IPs ==="
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==`null`].{IP:PublicIp,AllocId:AllocationId}' \
  --output table

# Find VPN connections
echo "=== VPN Connections ==="
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[?State==`available`].{Id:VpnConnectionId,VGW:VpnGatewayId}' \
  --output table

# Find Interface VPC Endpoints (charged hourly)
echo "=== Interface Endpoints ==="
aws ec2 describe-vpc-endpoints \
  --query 'VpcEndpoints[?VpcEndpointType==`Interface`].{Id:VpcEndpointId,Service:ServiceName,State:State}' \
  --output table
```

### AWS Cost Explorer Filters

Use Cost Explorer to isolate networking costs:

1. **Group by**: Service → filter to "Amazon Virtual Private Cloud"
2. **Group by**: Usage Type → look for:
   - `NatGateway-Hours` — base NAT cost
   - `NatGateway-Bytes` — NAT processing
   - `DataTransfer-Out-Bytes` — internet egress
   - `DataTransfer-Regional-Bytes` — cross-AZ traffic

### Peer Review Checklist

Significant architectural changes should be peer-reviewed. Include these networking cost questions:

- [ ] Are there NAT Gateways? Is the traffic volume justified?
- [ ] Are S3/DynamoDB Gateway Endpoints configured? (free alternative to NAT for these services)
- [ ] Is cross-AZ traffic minimized? (co-locate related services in the same AZ where possible)
- [ ] Are there idle or test resources that should be cleaned up?
- [ ] Is egress traffic optimized? (CloudFront for static content, compression, caching)
- [ ] For data transfer > 3 TB/month: has Direct Connect been evaluated?

## Cost Optimization Cheat Sheet

| Action | Savings | Effort |
|--------|---------|--------|
| Add S3 Gateway Endpoint | Eliminate S3 NAT processing ($45/TB) | 5 minutes |
| Add DynamoDB Gateway Endpoint | Eliminate DynamoDB NAT processing | 5 minutes |
| Use CloudFront for static content | Reduce egress from $90/TB to $85/TB + caching | 1 hour |
| Deploy caching proxy for updates | Reduce NAT traffic by 90%+ | 2 hours |
| Use ECR pull-through cache | Eliminate Docker Hub pulls through NAT | 30 minutes |
| Single NAT instead of per-AZ | Save ~$64/month (less HA) | 10 minutes |
| Switch to Direct Connect (>3 TB) | Save $68/TB on data transfer | Weeks |
| Clean up zombie resources | Variable — often $50–500/month | 1 hour |

## Key Takeaways

- **VPCs, subnets, route tables, IGWs, security groups, and NACLs are free** — never compromise on network design to save money
- **NAT Gateways are the biggest networking cost trap** — $32+/month base + $45/TB processing
- **Always configure S3 and DynamoDB Gateway Endpoints** — they are free and eliminate NAT charges for these services
- **Internet egress is ~$90/TB, or ~$136/TB through NAT** — architect to minimize outbound data
- **Direct Connect breaks even at ~3 TB/month** vs. VPN for on-premises connectivity
- **The "no-NAT" public subnet approach** saves money but introduces risk — acceptable for dev/test, not for production with sensitive data
- **Set billing alerts immediately** — a misconfigured resource can cost more than your entire month's budget
- **Audit regularly** for zombie NAT Gateways, unused Elastic IPs, and forgotten interface endpoints

## References

- [AWS Networking Basics For Programmers | Hands On](https://www.youtube.com/watch?v=2doSoMN2xvI) — Travis Media
- [AWS VPC Pricing](https://aws.amazon.com/vpc/pricing/)
- [AWS Data Transfer Pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer)
- [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/#NAT_Gateway)
- [AWS Direct Connect Pricing](https://aws.amazon.com/directconnect/pricing/)
- [VPC Endpoints Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
