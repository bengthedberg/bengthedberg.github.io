---
layout: post
title: "Understanding TCP/IP Networking: IP Addresses, Subnets, and Routing"
date: 2025-08-07 00:00:00 +0000
tags:
  - networking
  - tcp-ip
  - subnets
  - ip-addresses
  - routing
---


## Why Developers Should Understand TCP/IP

Every application you build communicates over a network. Whether you're configuring cloud infrastructure, debugging connectivity issues, setting up VPCs, or understanding why a request times out, TCP/IP knowledge is foundational. This article breaks down the core concepts: IP addresses, subnet masks, network classes, default gateways, and private address ranges.


## IP Addresses: Network and Host

An IP address is a **32-bit number** that uniquely identifies a device on a TCP/IP network. It's written in **dotted-decimal notation** — four numbers (octets) separated by periods:

```
192.168.123.132
```

Each octet represents 8 bits, so the full address in binary is:

```
11000000.10101000.01111011.10000100
```

### Two Parts of Every IP Address

Every IP address is split into two logical parts:

| Part | Purpose | Example |
|------|---------|---------|
| **Network address** | Identifies which network the device belongs to | `192.168.123.0` |
| **Host address** | Identifies the specific device within that network | `0.0.0.132` |

Routers use the **network** portion to forward packets to the correct network. Once the packet arrives at the destination network, the **host** portion identifies the specific device.

This is analogous to postal mail: the network address is the street name, and the host address is the house number.


## Subnet Masks: Separating Network from Host

The division between network and host isn't fixed — it's determined by the **subnet mask**.

A subnet mask is another 32-bit number. In binary, it's always a sequence of `1`s followed by `0`s:

- The `1` bits mark the **network** portion
- The `0` bits mark the **host** portion

### Example

```
11000000.10101000.01111011.10000100  — IP address   (192.168.123.132)
11111111.11111111.11111111.00000000  — Subnet mask  (255.255.255.0)
```

The first 24 bits (all `1`s in the mask) are the network address. The remaining 8 bits (all `0`s) are the host address:

```
11000000.10101000.01111011.00000000  — Network address (192.168.123.0)
00000000.00000000.00000000.10000100  — Host address    (0.0.0.132)
```

With this mask, any device with an IP starting with `192.168.123.x` is on the same network. The last octet (0–255) identifies individual hosts within it.

### How the Mask Works (Bitwise AND)

To extract the network address, perform a **bitwise AND** between the IP address and the subnet mask:

```
  192.168.123.132   →  11000000.10101000.01111011.10000100
& 255.255.255.0     →  11111111.11111111.11111111.00000000
= 192.168.123.0     →  11000000.10101000.01111011.00000000
```

This operation zeroes out the host bits, leaving only the network address.

### Common Subnet Masks

| Decimal | Binary | Network bits | Hosts per subnet |
|---------|--------|-------------|-----------------|
| `255.0.0.0` | `11111111.00000000.00000000.00000000` | 8 | 16,777,214 |
| `255.255.0.0` | `11111111.11111111.00000000.00000000` | 16 | 65,534 |
| `255.255.255.0` | `11111111.11111111.11111111.00000000` | 24 | 254 |
| `255.255.255.192` | `11111111.11111111.11111111.11000000` | 26 | 62 |
| `255.255.255.224` | `11111111.11111111.11111111.11100000` | 27 | 30 |

The number of usable hosts is `2^(host bits) - 2` (subtract the network address and broadcast address).


## CIDR Notation

Instead of writing out the full subnet mask, **CIDR notation** appends the number of network bits after a slash:

| CIDR | Subnet Mask | Meaning |
|------|-------------|---------|
| `/8` | `255.0.0.0` | First 8 bits are the network |
| `/16` | `255.255.0.0` | First 16 bits are the network |
| `/24` | `255.255.255.0` | First 24 bits are the network |
| `/26` | `255.255.255.192` | First 26 bits are the network |
| `/32` | `255.255.255.255` | Single host |

`192.168.123.0/24` means "the network 192.168.123.0 with a 24-bit mask" — equivalent to subnet mask `255.255.255.0`.

You'll see CIDR notation everywhere in cloud configurations (AWS VPCs, security groups, route tables).


## Network Classes

Historically, IP addresses were grouped into **classes** based on their first octet. While classful addressing has been largely replaced by CIDR, the terminology persists and is useful to understand:

| Class | First Octet Range | Default Subnet Mask | Network/Host Split | Example |
|-------|-------------------|--------------------|--------------------|---------|
| **A** | 1–126 | `255.0.0.0` (`/8`) | 8 / 24 bits | `10.52.36.11` |
| **B** | 128–191 | `255.255.0.0` (`/16`) | 16 / 16 bits | `172.16.52.63` |
| **C** | 192–223 | `255.255.255.0` (`/24`) | 24 / 8 bits | `192.168.123.132` |

**Class A** networks have few network addresses but millions of hosts per network. **Class C** networks have many network addresses but only 254 hosts each. Class B sits in between.

> **Note**: 127.x.x.x is reserved for loopback (`127.0.0.1` is `localhost`). Classes D (224–239, multicast) and E (240–255, experimental) exist but aren't used for standard addressing.


## Default Gateways

When a device wants to communicate with another device, it first determines whether the destination is **local** (same subnet) or **remote** (different subnet).

The process:

1. Apply the subnet mask to **both** the source IP and destination IP (bitwise AND)
2. Compare the resulting network addresses
3. If they **match** — the destination is local; send directly on the subnet
4. If they **don't match** — the destination is remote; forward the packet to the **default gateway**

The **default gateway** is a router on your local subnet that knows how to forward packets to other networks. It's the "exit door" from your subnet to the rest of the network.

### Example

Your device: `192.168.1.100` with mask `255.255.255.0` and gateway `192.168.1.1`

**Sending to `192.168.1.50`** (local):
```
192.168.1.100 AND 255.255.255.0 = 192.168.1.0
192.168.1.50  AND 255.255.255.0 = 192.168.1.0
→ Same network → send directly
```

**Sending to `10.0.0.5`** (remote):
```
192.168.1.100 AND 255.255.255.0 = 192.168.1.0
10.0.0.5      AND 255.255.255.0 = 10.0.0.0
→ Different network → forward to gateway 192.168.1.1
```

The router at `192.168.1.1` then consults its routing table to determine where to send the packet next.


## Private IP Addresses

Not every device needs a globally unique, publicly routable IP address. **RFC 1918** reserves three address ranges for private networks:

| Name | CIDR Block | Address Range | Addresses | Classful Description |
|------|-----------|---------------|-----------|---------------------|
| 24-bit block | `10.0.0.0/8` | `10.0.0.0` – `10.255.255.255` | 16,777,216 | Single Class A |
| 20-bit block | `172.16.0.0/12` | `172.16.0.0` – `172.31.255.255` | 1,048,576 | 16 Class B blocks |
| 16-bit block | `192.168.0.0/16` | `192.168.0.0` – `192.168.255.255` | 65,536 | 256 Class C blocks |

### Key properties of private addresses

- **Not routable on the public internet** — routers discard packets with private source/destination addresses
- **Free to use** — no registration required
- **Reusable** — different organisations can use the same private ranges without conflict
- **Require NAT for internet access** — Network Address Translation maps private addresses to a public IP for outbound traffic

### Where you'll see them

- **Home networks**: Your router typically assigns addresses from `192.168.0.0/24` or `192.168.1.0/24`
- **AWS VPCs**: Default VPC uses `172.31.0.0/16`; custom VPCs commonly use `10.0.0.0/16`
- **Docker networks**: Default bridge uses `172.17.0.0/16`
- **Kubernetes**: Pod networks typically use `10.244.0.0/16` or similar


## IPv6: The Future (and Present)

IPv4's 32-bit address space provides approximately 4.3 billion addresses — not enough for the modern internet. **IPv6** expands the address size to **128 bits**, providing roughly 3.4 x 10^38 addresses.

IPv6 addresses are written as eight groups of four hexadecimal digits:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

Leading zeros can be omitted, and consecutive groups of zeros can be replaced with `::`:

```
2001:db8:85a3::8a2e:370:7334
```

While IPv6 adoption continues to grow, most developers still work primarily with IPv4 in cloud and enterprise environments. Understanding IPv4 thoroughly remains essential.


## Quick Reference

| Concept | Key Point |
|---------|-----------|
| **IP address** | 32-bit number identifying a device on a network |
| **Subnet mask** | Defines which bits are network vs. host |
| **CIDR notation** | Shorthand for subnet mask (`/24` = `255.255.255.0`) |
| **Network classes** | Historical grouping by first octet (A: 1–126, B: 128–191, C: 192–223) |
| **Default gateway** | Router that forwards packets to other networks |
| **Private addresses** | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — not internet-routable |
| **NAT** | Translates private addresses to public for internet access |
| **IPv6** | 128-bit addresses replacing IPv4's 32-bit space |

## References

- [RFC 791 — Internet Protocol](https://www.rfc-editor.org/rfc/rfc791)
- [RFC 1918 — Private Address Space](https://www.rfc-editor.org/rfc/rfc1918)
- [CIDR — Classless Inter-Domain Routing](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)
- [IPv6 — Wikipedia](https://en.wikipedia.org/wiki/IPv6)
