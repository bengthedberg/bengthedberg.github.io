---
layout: post
title: "AWS Networking Basics for Programmers"
date: 2026-04-05 00:00:00 +0000
categories: [AWS]
tags:
  - aws
  - networking
  - vpc
  - subnets
  - security-groups
---


## Summary

A hands-on guide to AWS networking fundamentals — the concepts every developer needs to understand before deploying applications to AWS. Based on the video [AWS Networking Basics For Programmers | Hands On](https://www.youtube.com/watch?v=2doSoMN2xvI) by Travis Media.

## VPC (Virtual Private Cloud)

A VPC is your own isolated network within AWS. Think of it as your private data center in the cloud.

- Every AWS account gets a **default VPC** in each region
- You define the IP range using **CIDR notation** (e.g., `10.0.0.0/16` gives you 65,536 addresses)
- All resources you launch (EC2, RDS, Lambda, etc.) live inside a VPC
- VPCs are **region-scoped** — they span all Availability Zones in a region

### Creating a VPC

```
AWS Console → VPC → Create VPC
- Name: my-app-vpc
- IPv4 CIDR block: 10.0.0.0/16
```

### CIDR Notation Quick Reference

| CIDR | Addresses | Use case |
|------|-----------|----------|
| `/16` | 65,536 | Large VPC |
| `/20` | 4,096 | Medium subnet |
| `/24` | 256 | Small subnet |
| `/28` | 16 | Minimal subnet |

## Subnets

Subnets divide your VPC into smaller network segments, each in a specific Availability Zone.

### Public Subnets

- Have a route to an **Internet Gateway**
- Resources here can be reached from the internet (if allowed by security groups)
- Use for: load balancers, bastion hosts, NAT gateways

```
Subnet: public-subnet-1
AZ: us-east-1a
CIDR: 10.0.1.0/24
```

### Private Subnets

- **No direct route** to the internet
- Resources here are isolated from inbound internet traffic
- Can reach the internet outbound via a **NAT Gateway** in a public subnet
- Use for: application servers, databases, internal services

```
Subnet: private-subnet-1
AZ: us-east-1a
CIDR: 10.0.10.0/24
```

### Best Practice: Multi-AZ

Always create subnets in at least **2 Availability Zones** for high availability:

```
public-subnet-1   → us-east-1a  (10.0.1.0/24)
public-subnet-2   → us-east-1b  (10.0.2.0/24)
private-subnet-1  → us-east-1a  (10.0.10.0/24)
private-subnet-2  → us-east-1b  (10.0.11.0/24)
```

## Internet Gateway (IGW)

Provides a path for traffic between your VPC and the internet.

- One IGW per VPC
- Must be **attached** to the VPC
- Public subnets route `0.0.0.0/0` traffic through the IGW

## NAT Gateway

Allows resources in **private subnets** to reach the internet (for updates, API calls, etc.) without being reachable from the internet.

- Deployed in a **public subnet**
- Requires an **Elastic IP**
- Private subnet route table points `0.0.0.0/0` to the NAT Gateway
- Costs money even when idle — consider NAT instances for dev environments

```
Private subnet route table:
  10.0.0.0/16  → local
  0.0.0.0/0    → nat-gateway-id
```

## Route Tables

Route tables control where network traffic is directed.

- Every subnet is associated with a route table
- If not explicitly associated, it uses the **main route table**

### Public Subnet Route Table

| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | igw-xxxxx |

### Private Subnet Route Table

| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | nat-xxxxx |

## Security Groups

Security groups act as a **virtual firewall** for your resources. They control inbound and outbound traffic at the instance level.

- **Stateful** — if you allow inbound traffic, the response is automatically allowed out
- **Default deny** — all inbound traffic is denied unless you create a rule
- Can reference other security groups (not just IP ranges)

### Common Patterns

**Web server security group:**

| Type | Port | Source | Description |
|------|------|--------|-------------|
| HTTP | 80 | `0.0.0.0/0` | Public web traffic |
| HTTPS | 443 | `0.0.0.0/0` | Public web traffic |
| SSH | 22 | `your-ip/32` | Admin access |

**Database security group:**

| Type | Port | Source | Description |
|------|------|--------|-------------|
| PostgreSQL | 5432 | `sg-webserver` | Only from app servers |

### Security Groups vs NACLs

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance | Subnet |
| Rules | Allow only | Allow and Deny |
| Stateful | Yes | No |
| Evaluation | All rules | Rules in order |
| Default | Deny all inbound | Allow all |

## Network ACLs (NACLs)

Network ACLs provide an additional layer of security at the **subnet level**.

- **Stateless** — you must create rules for both inbound and outbound
- Rules are evaluated in **number order** (lowest first)
- Default NACL allows all traffic
- Useful for blocking specific IPs or IP ranges

## VPC Peering

Connects two VPCs so they can communicate using private IP addresses.

- Works across accounts and regions
- No transitive peering — if A peers with B and B peers with C, A cannot reach C through B
- CIDR ranges must not overlap

## VPC Endpoints

Allow resources in your VPC to connect to AWS services **without going through the internet**.

### Gateway Endpoints (free)

- S3 and DynamoDB only
- Added to route tables

### Interface Endpoints (cost per hour + data)

- Most other AWS services (SQS, SNS, Secrets Manager, etc.)
- Creates an ENI in your subnet

## Typical Architecture

```
                    Internet
                       │
                  ┌────┴────┐
                  │   IGW   │
                  └────┬────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────┴────┐   ┌────┴────┐       │
    │ Public  │   │ Public  │       │
    │Subnet 1a│   │Subnet 1b│       │
    │  (ALB)  │   │  (ALB)  │       │
    │  (NAT)  │   │         │       │
    └────┬────┘   └────┬────┘       │
         │             │            │
    ┌────┴────┐   ┌────┴────┐       │
    │ Private │   │ Private │       │
    │Subnet 1a│   │Subnet 1b│       │
    │  (App)  │   │  (App)  │       │
    └────┬────┘   └────┬────┘       │
         │             │            │
    ┌────┴────┐   ┌────┴────┐       │
    │ Private │   │ Private │       │
    │Subnet 1a│   │Subnet 1b│       │
    │  (DB)   │   │  (DB)   │       │
    └─────────┘   └─────────┘       │
         │                          │
         └──── VPC (10.0.0.0/16) ───┘
```

## Key Takeaways

- Always use a **custom VPC** for production — don't rely on the default
- Place application servers and databases in **private subnets**
- Use **security groups** to control access between tiers (reference SG IDs, not IPs)
- Deploy across **multiple AZs** for high availability
- Use **VPC endpoints** for AWS service access to avoid NAT Gateway costs and improve security
- Keep CIDR blocks organized and documented — overlapping ranges break peering

## References

- [AWS Networking Basics For Programmers | Hands On](https://www.youtube.com/watch?v=2doSoMN2xvI) — Travis Media
- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [VPC Subnets Guide](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [NAT Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
