# Public IP vs Private IP in AWS EC2 — In Depth

---

## The Core Concept — Why Two Types of IP?

Every device that communicates over a network needs an address — just like every house needs a street address for mail delivery. But not every server needs to be reachable by the entire internet. Some servers should be hidden, only reachable internally.

This is exactly why AWS gives EC2 instances two separate IP systems — one for the outside world, one for internal communication.

> Think of it like a large office building. The building has one public street address anyone can find and visit. But inside, every room has an internal extension number that only people inside the building can call. The street address = Public IP. The internal extension = Private IP.

---

## IP Address Fundamentals

Before diving into AWS specifics, you need to understand what makes an IP "public" or "private" at a fundamental level.

### The Internet is Built on Public IPs

Every device that communicates on the public internet must have a **globally unique** IP address. No two devices anywhere on earth can share the same public IP at the same time. Public IPs are:
- Assigned and managed by **IANA** (Internet Assigned Numbers Authority) globally
- Distributed to ISPs (Internet Service Providers) and cloud providers like AWS
- **Globally routable** — any router anywhere in the world knows how to forward traffic to a public IP
- Owned and managed by AWS in the context of EC2 — you don't bring your own public IP (unless you use Elastic IP)

### Private IP Ranges — Reserved by International Agreement

Certain IP ranges have been internationally reserved and are **never routed on the public internet**. Every router on the internet is configured to drop traffic destined for these ranges. They exist only within private networks.

The three private IP ranges defined by **RFC 1918** are:

| Class | Range | Total Addresses | Typical Use |
|---|---|---|---|
| Class A | `10.0.0.0` – `10.255.255.255` | 16.7 million | Large enterprise networks, AWS VPCs |
| Class B | `172.16.0.0` – `172.31.255.255` | 1 million | Medium networks, AWS default VPC |
| Class C | `192.168.0.0` – `192.168.255.255` | 65,536 | Home routers, small office networks |

**You do not need permission to use these ranges inside your own private network.** AWS uses them for your VPC (Virtual Private Cloud) — specifically, the AWS default VPC uses the `172.31.0.0/16` range.

The key rule: **private IPs cannot be accessed from the internet, and internet traffic cannot be sent to a private IP directly.** The internet routing tables simply have no entry for these ranges — the traffic gets dropped.

---

## Private IP in AWS EC2 — The Mandatory Address

Every single EC2 instance, without exception, gets a **private IP address** the moment it launches. You cannot launch an EC2 instance without a private IP. It is assigned automatically from the CIDR block of the subnet your instance is placed in.

**Properties of EC2 Private IP:**
- Assigned from your VPC's IP range (e.g., `172.31.0.0/16` in the default VPC)
- **Permanent for the lifetime of the instance** — it never changes, even if you stop and start the instance
- Used for all communication between resources inside AWS
- Not visible to or reachable from the internet
- Free — no cost associated with private IPs

**What private IP is used for:**

When your web server needs to query your database, it does not go out to the internet and come back. It communicates directly over the private network:

```
Web Server (172.31.10.5)  ──private network──→  Database (172.31.20.8)

This traffic never leaves AWS's internal network.
It is fast, free, and secure.
```

This is the correct and recommended way for all internal AWS communication — EC2 to RDS, EC2 to ElastiCache, EC2 to another EC2, etc.

---

## Public IP in AWS EC2 — The Optional Address

A public IP is **optional** — you decide at launch time whether your instance needs one. It makes your instance reachable directly from the internet.

**Properties of EC2 Public IP (Dynamic):**
- Assigned from AWS's pool of public IP addresses
- **Ephemeral — it changes every time you stop and start the instance.** If you stop your instance tonight and start it tomorrow, it will have a completely different public IP.
- Lost permanently when instance is terminated
- No direct cost for a standard dynamic public IP (though as of February 2024, AWS charges $0.005/hour for all public IPv4 addresses)
- Enables inbound internet traffic to reach your instance

**The stop/start IP change problem:**

```
Day 1: Your website is at 54.123.45.67
You stop the instance to save costs overnight
Day 2: You start it again → new IP: 54.198.76.21
Your DNS record is wrong, your users can't find you
```

This is a real operational problem. The solution is **Elastic IP**.

---

## Elastic IP — The Static Public IP

An **Elastic IP (EIP)** is a **static, permanent public IP address** that you own in your AWS account until you explicitly release it.

**Key properties:**
- Does not change when you stop/start/reboot instances
- Can be **remapped instantly** from one instance to another — if your instance fails, you detach the EIP and attach it to a replacement instance. Users experience no IP change.
- Can be moved across AZs (unlike private IPs which are subnet-specific)
- **Cost model:** Free while attached to a running instance. **Charged ($0.005/hour) when allocated but NOT attached** — AWS does this to discourage hoarding of public IPs, which are a globally scarce resource.

**When to use Elastic IP:**
- Production servers that need a consistent public IP
- When your DNS record must point to a stable address
- Failover scenarios where you need to move an IP between instances instantly

**When NOT to use Elastic IP:**
- Most modern architectures use a **Load Balancer** with a DNS name instead of a fixed IP — more scalable and resilient
- Dev/test environments where IP stability doesn't matter

---

## The Architecture Decision — Who Gets a Public IP?

This is a fundamental cloud architecture decision. Here is the standard three-tier architecture pattern:

```
Internet
    ↓
┌─────────────────────────────────────────────────┐
│  Load Balancer (has public DNS/IP)              │  ← PUBLIC FACING
│  This is the ONLY thing the internet talks to  │
└─────────────────────────────────────────────────┘
    ↓ (via private IP)
┌─────────────────────────────────────────────────┐
│  Web / Application Servers (EC2)                │  ← PRIVATE ONLY
│  No public IP                                  │  (no direct internet access)
│  Only reachable from Load Balancer             │
└─────────────────────────────────────────────────┘
    ↓ (via private IP)
┌─────────────────────────────────────────────────┐
│  Database Server (RDS)                          │  ← PRIVATE ONLY
│  No public IP                                  │  (absolutely no internet access)
│  Only reachable from App Servers               │
└─────────────────────────────────────────────────┘
```

**Why does the database NEVER get a public IP?**

If your database had a public IP, it would be exposed to the entire internet. Hackers, bots, and vulnerability scanners constantly probe public IPs on ports like 3306 (MySQL), 5432 (PostgreSQL), 1433 (SQL Server). One misconfigured security group rule and your entire data set is compromised. Keeping databases on private IP only — combined with security groups that only allow traffic from app servers — is the foundation of cloud security.

---

## The Bastion Host — Accessing Private Instances

Here is the problem: if your database or internal app server has no public IP, how do you SSH into it to perform maintenance, troubleshooting, or configuration?

You cannot reach it directly from your laptop. The solution is a **Bastion Host** (also called a Jump Server or Jump Box).

**What is a Bastion Host?**

A Bastion Host is a special-purpose EC2 instance with a public IP whose only job is to act as a secure gateway into your private network. It has extremely restricted security group rules — usually only your company's office IP can SSH into it on port 22.

```
Your Laptop
    │
    │  SSH (port 22, your IP only)
    ↓
Bastion Host (has public IP: 54.123.45.67)
    │
    │  SSH (port 22, internal private network only)
    ↓
App Server (private IP: 172.31.10.5)  ← No public IP
    │
    │  MySQL (port 3306, private network only)
    ↓
Database Server (private IP: 172.31.20.8)  ← No public IP
```

**Step-by-step process:**

```bash
# Step 1: SSH into the Bastion Host from your laptop
ssh -i "my-key.pem" ubuntu@54.123.45.67

# Step 2: From inside Bastion Host, SSH to private instance
ssh -i "my-key.pem" ubuntu@172.31.10.5

# You are now inside a server that has no public IP
# The internet cannot reach this server directly
```

**SSH Agent Forwarding — cleaner approach:**
Instead of storing your private key on the Bastion Host (a security risk), you use SSH agent forwarding — your local SSH key is forwarded through the tunnel without being copied to the bastion:

```bash
ssh -A -i "my-key.pem" ubuntu@54.123.45.67
# -A enables agent forwarding
# Once inside bastion, your key is available without copying it there
ssh ubuntu@172.31.10.5
```

**Bastion Host security best practices:**
- Only allow SSH from specific IP addresses (your office/home), never `0.0.0.0/0`
- Use a tiny instance type (t3.micro) — it does almost nothing, just forwards SSH
- Enable CloudTrail logging to audit who accessed it and when
- Consider using **AWS Systems Manager Session Manager** as a modern replacement — no bastion needed, no port 22 open at all

---

## AWS Systems Manager Session Manager — The Modern Alternative

Session Manager is AWS's modern replacement for the bastion host pattern. It allows you to get a shell into any EC2 instance with **zero open ports, no public IP, and no SSH keys**.

How it works: An SSM agent (pre-installed on Amazon Linux and Ubuntu) communicates outbound to the AWS Systems Manager service over HTTPS (port 443). You connect through the AWS console or CLI — no inbound ports needed at all.

```
Your Laptop
    ↓
AWS Console / AWS CLI
    ↓
AWS Systems Manager Service
    ↓ (outbound HTTPS from instance, no inbound ports)
EC2 Instance (no public IP, no port 22 open)
```

Benefits over Bastion Host:
- No EC2 bastion to maintain, patch, and pay for
- No security group rules for SSH
- All sessions logged automatically to CloudTrail and S3
- Works even if the instance has no internet access (via VPC endpoints)

---

## IPv4 vs IPv6 in AWS

Modern AWS also supports **IPv6** alongside the traditional IPv4.

**IPv4** — The classic format: `192.168.1.1` (32-bit, ~4.3 billion addresses). Running out globally — hence the cost for public IPv4 addresses.

**IPv6** — The new format: `2600:1f18:24e:b700::1` (128-bit, effectively unlimited addresses). All IPv6 addresses are **public by default** — there are no "private" IPv6 ranges in the traditional sense. AWS uses security groups and network ACLs instead of IP scarcity for access control.

In AWS, your VPC can have an IPv6 CIDR block, and instances can get both private IPv4 and public IPv6 addresses simultaneously.

---

## Complete Comparison — Everything at a Glance

| Aspect | Private IP | Public IP (Dynamic) | Elastic IP | IPv6 |
|---|---|---|---|---|
| Mandatory? | Yes — every instance | No — optional | No — you allocate | No — optional |
| Changes on stop/start? | Never | Yes — new IP every time | Never | Never |
| Internet accessible? | No | Yes | Yes | Yes |
| Cost | Free | $0.005/hr (from Feb 2024) | Free if attached, $0.005/hr if unattached | Free |
| IP range | 10.x, 172.16-31.x, 192.168.x | AWS public pool | AWS public pool | 128-bit hex |
| Use case | Internal communication | Dev/test, temporary access | Production, stable endpoints | Modern internet traffic |
| Lost on termination? | Yes | Yes | No — stays in your account | Yes |
| Can be moved between instances? | No | No | Yes — instantly | No |

---

## Architecture Best Practices Summary

| Rule | Why |
|---|---|
| Databases should NEVER have a public IP | Eliminates internet exposure of your most valuable asset |
| App servers behind a load balancer don't need public IPs | LB handles all internet traffic, app servers stay private |
| Use Elastic IP for anything production-facing that needs a fixed IP | Prevents IP change on stop/start breaking DNS and connections |
| Use Bastion Host or Session Manager for private instance access | Secure, auditable, no need to expose instances publicly |
| Restrict SSH/RDP source to specific IPs, never 0.0.0.0/0 | Eliminates brute-force attack surface on public-facing management ports |
| Use private IPs for all internal AWS service communication | Faster, free, more secure — traffic stays within AWS network |
