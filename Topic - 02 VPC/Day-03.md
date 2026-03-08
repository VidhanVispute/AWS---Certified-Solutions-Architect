# AWS VPC — Part 2: Subnets, CIDR Design, and High Availability Architecture

---

## Why Subnets Exist — The Segmentation Problem

When you create a VPC with a CIDR block like `10.0.0.0/16`, you have a pool of 65,536 IP addresses. But a flat network where every single resource — your public-facing load balancers, your application servers, your databases, your bastion hosts — all sit in the same undivided IP space is an architectural and security disaster.

Think about what a real production system at a company like Swiggy looks like. They have:
- Load balancers that must be reachable from the public internet
- Application servers that should only be reachable from the load balancers, not the internet
- Databases that should only be reachable from the application servers, not the load balancers, and certainly not the internet
- Internal admin tools reachable only via VPN

If all of these live in the same flat network, enforcing these access boundaries becomes extremely difficult — you'd have to rely entirely on security groups and NACLs for every single rule, with no structural network-level separation. One misconfigured security group and your database is reachable from the internet.

**Subnets solve this by dividing your VPC's IP space into named, purpose-bound segments, each with its own routing behavior.** The routing behavior — specifically, whether a subnet has a route to an Internet Gateway — is what makes a subnet "public" or "private." This is the most important conceptual point of this entire topic and we'll come back to it repeatedly.

---

## What a Subnet Actually Is

A subnet (short for subnetwork) is a **range of IP addresses within your VPC, tied to a single Availability Zone**. Every subnet you create is a subdivision of your VPC's CIDR block. Resources launched into a subnet get a private IP address from that subnet's range.

The two immovable rules of subnets:

**Rule 1: A subnet cannot span multiple Availability Zones.** A subnet is permanently bound to exactly one AZ. If you need resources in three AZs, you need at minimum three subnets. This is not a limitation — it's by design. It forces you to think explicitly about AZ placement and build HA into your architecture structurally, not as an afterthought.

**Rule 2: Subnet CIDR blocks must be subsets of the VPC CIDR and cannot overlap with each other.** If your VPC is `10.0.0.0/16`, a subnet can be `10.0.1.0/24` or `10.0.4.0/22`, but it cannot be `192.168.0.0/24` (outside the VPC range) and two subnets cannot both claim `10.0.1.0/24`.

---

## The 5 Reserved IPs Per Subnet — AWS Takes Its Cut

This is a detail that catches people off guard in both real projects and interviews. In every subnet you create, **AWS automatically reserves 5 IP addresses** from your range. They are not available for your resources. Here's exactly which ones and why:

For a subnet `10.0.1.0/24` (256 total addresses):

```
10.0.1.0   → Network address (standard, identifies the subnet itself)
10.0.1.1   → AWS reserves for the VPC Router
10.0.1.2   → AWS reserves for DNS resolution (Amazon DNS server)
10.0.1.3   → AWS reserves for future use
10.0.1.255 → Broadcast address (AWS doesn't support broadcast, but reserves it)

Usable IPs: 256 - 5 = 251
```

For a `/28` subnet (the smallest allowed), you get 16 total IPs minus 5 reserved = **11 usable IPs**. This minimum was specifically set because 11 IPs is enough for very small use cases like a NAT Gateway subnet.

> ⚠️ **Interview Trap:** "How many usable IPs does a /24 subnet give you?" The naive answer is 254 (subtracting network and broadcast like in traditional networking). The AWS answer is **251** because AWS reserves 3 additional addresses. Always subtract 5, not 2.

---

## Designing a VPC CIDR and Subnet Plan — The Right Way

This is where most tutorials fail you — they show you how to click through the console without teaching you how to think about IP address planning. Let's fix that.

### Step 1: Choose Your VPC CIDR

Your VPC CIDR choice must account for three things:

**Future growth.** If you pick `/24` (256 IPs) and your company grows from 10 to 200 engineers each deploying microservices with their own EC2 instances and RDS clusters, you will run out of IPs. In practice, most production VPCs use `/16` (65,536 IPs) for large environments or `/20` (4,096 IPs) for medium ones.

**On-premises connectivity.** If you ever want to connect your AWS VPC to your office network or data center via VPN or Direct Connect, the two networks cannot overlap in IP range. If your office uses `10.0.0.0/8`, don't use that range for your VPC. This is why many organizations use `172.16.0.0/12` or specific `10.x.x.x/16` ranges that don't conflict with their on-premises allocations.

**Multi-VPC / multi-account architectures.** If you have a dev VPC, staging VPC, and prod VPC, and you want to peer them or connect them via Transit Gateway, none of them can have overlapping CIDRs. Plan a distinct range for each environment upfront.

A common enterprise pattern:
```
Production VPC:   10.0.0.0/16
Staging VPC:      10.1.0.0/16
Development VPC:  10.2.0.0/16
```

### Step 2: Design Your Subnet Tiers

A standard three-tier architecture in AWS looks like this:

```
VPC: 10.0.0.0/16
│
├── PUBLIC TIER (load balancers, NAT Gateways, bastion hosts)
│   ├── public-subnet-1a  →  10.0.1.0/24  (AZ: ap-south-1a)
│   ├── public-subnet-1b  →  10.0.2.0/24  (AZ: ap-south-1b)
│   └── public-subnet-1c  →  10.0.3.0/24  (AZ: ap-south-1c)
│
├── PRIVATE TIER (application servers, ECS tasks)
│   ├── private-subnet-1a →  10.0.11.0/24 (AZ: ap-south-1a)
│   ├── private-subnet-1b →  10.0.12.0/24 (AZ: ap-south-1b)
│   └── private-subnet-1c →  10.0.13.0/24 (AZ: ap-south-1c)
│
└── DATABASE TIER (RDS, ElastiCache — no outbound internet at all)
    ├── db-subnet-1a      →  10.0.21.0/24 (AZ: ap-south-1a)
    ├── db-subnet-1b      →  10.0.22.0/24 (AZ: ap-south-1b)
    └── db-subnet-1c      →  10.0.23.0/24 (AZ: ap-south-1c)
```

This gives you 9 subnets, 3 AZs, 3 tiers. The numbering convention (`1.x`, `11.x`, `21.x`) is deliberate — it makes the tier and AZ immediately obvious from the IP address alone. When you're debugging at 2am and see `10.0.11.45` in a log, you immediately know it's a private-tier resource in AZ-a.

---

## Public vs Private Subnet — The Routing Definition

Here's the most important distinction in all of VPC networking, and the one most commonly misunderstood:

**A subnet is "public" or "private" purely based on its route table — specifically, whether it has a route to an Internet Gateway.**

There is no checkbox in the subnet settings labeled "public" or "private." What makes a subnet public is that its associated route table has a route like `0.0.0.0/0 → igw-xxxxxxxx` (Internet Gateway). What makes a subnet private is the absence of that route (or having a route to a NAT Gateway instead).

```
Public Subnet Route Table:
┌─────────────────┬──────────────────────┐
│ Destination     │ Target               │
├─────────────────┼──────────────────────┤
│ 10.0.0.0/16     │ local                │  ← VPC-internal traffic
│ 0.0.0.0/0       │ igw-0a1b2c3d4e5f     │  ← Internet traffic
└─────────────────┴──────────────────────┘

Private Subnet Route Table:
┌─────────────────┬──────────────────────┐
│ Destination     │ Target               │
├─────────────────┼──────────────────────┤
│ 10.0.0.0/16     │ local                │  ← VPC-internal traffic only
└─────────────────┴──────────────────────┘
(No internet route = no internet access, inbound or outbound)
```

The `local` route is automatically present in every route table and cannot be deleted. It means: "all traffic destined for any IP within the VPC's CIDR stays within the VPC and is routed internally." This is what allows your application server in a private subnet to talk to your RDS database in the database subnet — both are within the VPC, so the `local` route handles it.

> ⚠️ **Interview Trap:** "Does putting an EC2 instance in a public subnet automatically make it internet-accessible?" No — three things must all be true: (1) the subnet must have a route to an Internet Gateway, (2) the EC2 instance must have a public IP or Elastic IP assigned, and (3) the Security Group must allow the relevant inbound traffic. All three gates must be open.

---

## Auto-Assign Public IP — Subnet-Level Setting

Each subnet has a setting called **"Auto-assign public IPv4 address"**. When enabled, every EC2 instance launched into that subnet automatically gets a public IP address (in addition to its private IP). When disabled, instances get only a private IP unless you explicitly assign an Elastic IP.

For public subnets (load balancers, bastion hosts), you typically enable this. For private and database subnets, you must leave it disabled — a database instance with a public IP is a security violation regardless of whether the security group blocks access.

---

## The Full Architecture — Visualized

Let's put it all together. Here's what a proper HA production VPC looks like structurally:

```
Region: ap-south-1 (Mumbai)
VPC: 10.0.0.0/16
│
├─────────────────────────────────────────────────────────┐
│  AZ: ap-south-1a          AZ: ap-south-1b               │
│                                                         │
│  ┌──────────────────┐     ┌──────────────────┐          │
│  │  Public Subnet   │     │  Public Subnet   │          │
│  │  10.0.1.0/24     │     │  10.0.2.0/24     │          │
│  │                  │     │                  │          │
│  │  [ALB node]      │     │  [ALB node]      │          │
│  │  [NAT Gateway]   │     │                  │          │
│  └────────┬─────────┘     └────────┬─────────┘          │
│           │                        │                    │
│           ▼                        ▼                    │
│  ┌──────────────────┐     ┌──────────────────┐          │
│  │  Private Subnet  │     │  Private Subnet  │          │
│  │  10.0.11.0/24    │     │  10.0.12.0/24    │          │
│  │                  │     │                  │          │
│  │  [App Server 1]  │     │  [App Server 2]  │          │
│  └────────┬─────────┘     └────────┬─────────┘          │
│           │                        │                    │
│           ▼                        ▼                    │
│  ┌──────────────────┐     ┌──────────────────┐          │
│  │  DB Subnet       │     │  DB Subnet       │          │
│  │  10.0.21.0/24    │     │  10.0.22.0/24    │          │
│  │                  │     │                  │          │
│  │  [RDS Primary]   │     │  [RDS Standby]   │          │
│  └──────────────────┘     └──────────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │
         ▼
  [Internet Gateway]
         │
         ▼
    [Internet]
```

Traffic flow for a user request: Internet → Internet Gateway → ALB in public subnet → App Server in private subnet → RDS in DB subnet. At no point does internet traffic directly reach the app servers or databases. If an app server needs to download a package update from the internet (outbound only), it goes through the NAT Gateway sitting in the public subnet — which we'll cover in a dedicated topic.

---

## Hands-On: Creating a Custom VPC and Subnets

Here's the exact sequence of steps and what each configuration choice means.

### Creating the VPC

In the AWS Console under VPC → Your VPCs → Create VPC, choose **"VPC only"** (not "VPC and more" — the wizard hides too much from you while learning).

```
Name tag:        prod-vpc
IPv4 CIDR block: 10.0.0.0/16
IPv6 CIDR block: No IPv6 CIDR block (unless you need it)
Tenancy:         Default (unless you need dedicated hardware — same as EC2 tenancy)
```

When you click Create, AWS provisions the VPC along with a **default route table**, a **default NACL**, and a **default security group** for that VPC. No subnets are created yet — you do that separately.

### Creating Subnets

VPC → Subnets → Create Subnet. Select your VPC first, then add subnet configurations:

```
Subnet 1:
  Name:              public-subnet-1a
  Availability Zone: ap-south-1a
  IPv4 CIDR:         10.0.1.0/24

Subnet 2:
  Name:              public-subnet-1b
  Availability Zone: ap-south-1b
  IPv4 CIDR:         10.0.2.0/24

Subnet 3:
  Name:              private-subnet-1a
  Availability Zone: ap-south-1a
  IPv4 CIDR:         10.0.11.0/24
```

AWS lets you add multiple subnets in a single "Create Subnet" flow — you don't need to repeat the flow six times.

After creating public subnets, enable auto-assign public IP on them:
Subnets → Select public subnet → Actions → Edit subnet settings → Enable auto-assign public IPv4 address.

### The IP Conflict Problem From the Video — Solved

The video runs into a failure when trying to create a second subnet because the IP ranges conflict. This happens when someone tries to use the same CIDR for two subnets, or tries to create a subnet whose range overlaps with an existing one.

The fix is simple: **plan your ranges upfront so they don't overlap.**

```
VPC CIDR: 192.168.0.0/20  (4096 IPs: 192.168.0.0 – 192.168.15.255)

Subnet 1: 192.168.0.0/24  (IPs: 192.168.0.0 – 192.168.0.255)   ✓
Subnet 2: 192.168.1.0/24  (IPs: 192.168.1.0 – 192.168.1.255)   ✓ (no overlap)
Subnet 3: 192.168.0.0/24  (IPs: 192.168.0.0 – 192.168.0.255)   ✗ (duplicates Subnet 1)
Subnet 3: 192.168.16.0/24 (IPs: 192.168.16.x – outside VPC)    ✗ (outside VPC CIDR)
```

The key mental model: your VPC CIDR is a bucket. Subnets are containers you fit inside that bucket. The containers cannot overlap and cannot extend outside the bucket.

---

## Subnet Sizing Guidance

Choosing the right subnet size is a balance between having enough IPs and not wasting address space.

A `/24` subnet gives you 251 usable IPs. For most application tiers this is sufficient. For database tiers where you might have 3 RDS instances and a few ElastiCache nodes, even a `/27` (27 usable IPs) would technically be enough — but going too small creates operational risk if you need to add resources. For public subnets where you might only run 1–2 NAT Gateways and a load balancer, a `/28` (11 IPs) would work, but gives you almost no room. A `/24` per subnet as a standard makes planning easy and is the most common production choice.

---

## Comparison Table: Public vs Private vs Database Subnet

| Dimension | Public Subnet | Private Subnet | Database Subnet |
|---|---|---|---|
| **Route to Internet** | Yes (via IGW) | No direct route | No route |
| **Outbound internet** | Direct | Via NAT Gateway | None (usually) |
| **Inbound from internet** | Possible (via SG) | Not possible | Not possible |
| **Auto-assign public IP** | Enabled | Disabled | Disabled |
| **Typical resources** | ALB, NAT GW, Bastion | App servers, ECS, Lambda | RDS, ElastiCache, Redshift |
| **Security exposure** | Highest | Medium | Lowest |
| **Route table** | Custom with IGW route | Custom, no IGW | Custom, no IGW, no NAT |

---

## Interview Flags for This Topic

> ⚠️ **"Can a subnet span multiple AZs?"** No. One subnet = one AZ. Always. This is a hard constraint.

> ⚠️ **"What makes a subnet public?"** Not a setting on the subnet itself — it's the route table. If the associated route table has `0.0.0.0/0 → Internet Gateway`, it's public.

> ⚠️ **"How many IPs does AWS reserve in each subnet?"** 5. Network address, VPC router, DNS, future use, broadcast. Usable = total - 5.

> ⚠️ **"Can you move a subnet from one AZ to another?"** No. AZ is set at creation and cannot be changed. To "move" a subnet to a different AZ, you'd have to create a new subnet and migrate resources.

> ⚠️ **"Can you resize a subnet after creation?"** No. Once created, a subnet's CIDR block cannot be modified. Plan carefully upfront.

> ⚠️ **"What's the maximum number of subnets per VPC?"** Default limit is 200 subnets per VPC (can be increased via service quota request).

> ⚠️ **"If two instances are in different subnets of the same VPC, can they communicate?"** Yes — via the `local` route that exists in every route table by default, subject to security group rules. Subnets within the same VPC are not isolated from each other by default (NACLs can add isolation).

---

The next natural topics are the **Internet Gateway** (how public subnets actually reach the internet) and the **Route Table** in depth (the brain behind all traffic decisions in your VPC). Drop whichever comes next in your series.
