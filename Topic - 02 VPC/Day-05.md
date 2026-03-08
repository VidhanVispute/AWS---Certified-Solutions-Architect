# AWS VPC — Part 4: Two-Tier Architecture, Private Subnets, and NAT Gateway

---

## Why "Everything Public" is an Architecture Failure

At the end of Part 3, you had a working VPC — instances reachable from the internet, subnets routing through an IGW, security groups controlling access. For a personal project or a quick demo, that works. For anything resembling a real application, it's a serious security failure waiting to happen.

Here's the specific problem: your database server is sitting in a public subnet with a public IP. Its security group might block all inbound traffic today — but that's one misconfiguration away from being wide open. Defense in depth is a foundational security principle: you never want a single control (security group rules) to be the only thing standing between the public internet and your most sensitive data. The network layer should be the first line of defense, and the network layer should make it structurally impossible for the internet to reach your database — not just policy-impossible.

This is the core motivation behind the two-tier architecture. You architect the network so that certain resources are **physically unreachable from the internet**, not just protected by rules. Even if every security group rule is deleted, a database in a true private subnet still cannot receive traffic from the internet because there is no route by which that traffic could arrive.

But here's the complication that makes this non-trivial: your private resources still need to reach the internet themselves. Your app server needs to pull OS updates from Ubuntu's package servers. Your database instance needs to download security patches. Your application code needs to call third-party APIs. Outbound internet access from private resources is a legitimate operational need — but you want it to be **one-way**: private resources can initiate connections outward, but no one on the internet can initiate a connection inward.

This asymmetry — outbound yes, inbound no — is exactly what NAT Gateway provides.

---

## The Two-Tier Architecture — Conceptually Precise

"Two-tier" refers to two network tiers, not necessarily two application tiers. The tiers are defined by their relationship to the internet:

**Tier 1 — Public:** Resources that must accept inbound connections from the internet. This includes load balancers, bastion hosts, and NAT Gateways. These live in public subnets with IGW routes.

**Tier 2 — Private:** Resources that should never accept inbound connections from the internet. This includes application servers, databases, cache clusters, and internal services. These live in private subnets with no IGW route.

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  PUBLIC TIER                                                │
│                                                             │
│  ┌────────────────────┐    ┌────────────────────┐           │
│  │  public-subnet-1a  │    │  public-subnet-1b  │           │
│  │  (ap-south-1a)     │    │  (ap-south-1b)     │           │
│  │                    │    │                    │           │
│  │  [Load Balancer]   │    │  [Load Balancer]   │           │
│  │  [NAT Gateway]     │    │                    │           │
│  │  [Bastion Host]    │    │                    │           │
│  └────────┬───────────┘    └────────────────────┘           │
│           │ (inbound from internet, via ALB)                │
└───────────┼─────────────────────────────────────────────────┘
            │
            ▼ (private IP only)
┌─────────────────────────────────────────────────────────────┐
│  PRIVATE TIER                                               │
│                                                             │
│  ┌────────────────────┐    ┌────────────────────┐           │
│  │  private-subnet-1a │    │  private-subnet-1b │           │
│  │  (ap-south-1a)     │    │  (ap-south-1b)     │           │
│  │                    │    │                    │           │
│  │  [App Server]      │    │  [App Server]      │           │
│  │  [RDS Primary]     │    │  [RDS Standby]     │           │
│  └────────┬───────────┘    └────────────────────┘           │
│           │ (outbound only, via NAT GW)                     │
└───────────┼─────────────────────────────────────────────────┘
            │
            ▼
       NAT Gateway (in public subnet)
            │
            ▼
       Internet Gateway
            │
            ▼
         Internet
```

The directionality arrows tell the whole story. Public tier: internet can initiate inbound. Private tier: only outbound initiated by the private resource itself, never inbound from internet.

---

## NAT Gateway — What It Is and How It Works

NAT stands for **Network Address Translation**. The NAT Gateway sits in a public subnet (it needs internet access itself to function) and acts as an intermediary for private subnet resources that need outbound internet access.

Here is the precise mechanism:

Your app server (`10.0.11.25`) wants to reach `apt.ubuntu.com` at `91.189.91.38` to run `apt-get update`. Its route table says `0.0.0.0/0 → nat-xxxxxxxx`. The packet goes to the NAT Gateway. The NAT Gateway has an **Elastic IP** (a static public IP) attached to it. It replaces the source IP of the packet from `10.0.11.25` with its own Elastic IP, and records this translation in its connection tracking table. The packet exits to the internet via the IGW as if it came from the NAT Gateway's Elastic IP.

When the response comes back from `91.189.91.38`, it arrives at the NAT Gateway's Elastic IP. The NAT Gateway looks up its connection table, sees this is a response to a session initiated by `10.0.11.25`, and forwards it back to the app server.

The internet never saw `10.0.11.25`. It only ever saw the NAT Gateway's Elastic IP. And crucially, **no one on the internet can initiate a new connection to `10.0.11.25`**, because the NAT Gateway only translates responses to outbound connections — it does not forward unsolicited inbound traffic.

```
App Server (10.0.11.25)
    │
    │  Packet: src=10.0.11.25, dst=91.189.91.38
    ▼
NAT Gateway (private IP: 10.0.1.8, Elastic IP: 52.66.x.x)
    │
    │  Translates: src=52.66.x.x, dst=91.189.91.38
    │  Records: {52.66.x.x:port → 10.0.11.25:port}
    ▼
Internet Gateway → Internet → apt.ubuntu.com

Response arrives:
apt.ubuntu.com → 52.66.x.x
    │
NAT Gateway looks up connection table
    │  Translates back: dst=10.0.11.25
    ▼
App Server receives response
```

This is **many-to-one NAT** (also called NAPT or PAT — Port Address Translation). Multiple private instances can share a single NAT Gateway Elastic IP because the NAT Gateway tracks connections using port numbers in addition to IP addresses, allowing it to distinguish which instance each response belongs to.

---

## NAT Gateway vs NAT Instance — A Critical Distinction

AWS offers two ways to provide NAT for private subnets: the managed NAT Gateway service and a DIY NAT Instance (an EC2 instance configured to do NAT). This comparison comes up frequently in interviews and in real architecture decisions.

**NAT Gateway** is AWS's managed NAT service. You provision it, AWS manages the underlying infrastructure. It scales automatically, is highly available within an AZ, and requires zero ongoing administration.

**NAT Instance** is an EC2 instance running a special AMI where you manually configure IP forwarding and iptables rules. It was the only option before NAT Gateway existed and is now mostly legacy.

| Dimension | NAT Gateway | NAT Instance |
|---|---|---|
| **Management** | Fully managed by AWS | You manage OS, patches, config |
| **Availability** | Highly available within AZ (AWS-managed) | Single EC2 = single point of failure |
| **Scaling** | Automatic, up to 100 Gbps | Limited by instance type |
| **Bandwidth** | Up to 100 Gbps | Depends on instance size |
| **Cost** | Higher (hourly + per-GB) | Lower (just EC2 cost) |
| **Port forwarding** | Not supported | Supported (can act as bastion) |
| **Security Groups** | Cannot attach SGs directly | Can attach SGs |
| **Use case** | All production workloads | Cost-sensitive dev/test only |
| **Elastic IP** | Required (1 per NAT GW) | Required |
| **Disable source/dest check** | Not needed | Must be disabled on the instance |

> ⚠️ **Interview Trap:** "Can you attach a Security Group to a NAT Gateway?" No. NAT Gateway is a managed service — you control access through the route tables and the Security Groups on the instances using the NAT Gateway, not on the NAT Gateway itself.

> ⚠️ **Interview Trap:** "Why must you disable source/destination check on a NAT Instance?" By default, EC2 instances drop packets where they are not the source or destination. A NAT instance by definition forwards packets for other instances, so it must be allowed to handle traffic where it is neither the source nor destination. This check must be explicitly disabled on the EC2 instance acting as NAT.

---

## NAT Gateway Placement — The AZ Design Decision

NAT Gateway is an AZ-scoped resource. When you create one, it lives in a specific AZ. This creates an important architectural decision.

**Option A: Single NAT Gateway (cheaper, single point of AZ failure)**

```
public-subnet-1a: [NAT Gateway]  ← only one
private-subnet-1a → routes to NAT GW in 1a  ✓
private-subnet-1b → routes to NAT GW in 1a  ← crosses AZ boundary
```

If `ap-south-1a` has an outage, your NAT Gateway goes with it, and private subnet resources in `ap-south-1b` lose internet access too. Also, all cross-AZ traffic from `1b` to NAT GW in `1a` incurs inter-AZ data transfer charges.

**Option B: NAT Gateway per AZ (resilient, slightly more expensive)**

```
public-subnet-1a: [NAT Gateway A]
public-subnet-1b: [NAT Gateway B]

private-subnet-1a → routes to NAT Gateway A  (same AZ)
private-subnet-1b → routes to NAT Gateway B  (same AZ)
```

Each AZ's private subnet uses the NAT Gateway in its own AZ. AZ failure only affects that AZ. No cross-AZ data transfer charges. This is the AWS recommended production pattern.

The cost difference is one extra NAT Gateway hourly charge (~$0.045/hour in Mumbai = ~$32/month). For most production workloads, this is trivially small compared to the resilience benefit.

> ⚠️ **Interview Trap:** "If you have private subnets in two AZs, how many NAT Gateways should you have?" For production: one per AZ, each referenced by its own AZ's private route table. One common wrong answer is "one is enough since they're in the same VPC."

---

## Route Table Configuration for Two-Tier Architecture

This is what the complete route table setup looks like for a proper two-tier architecture:

```
Route Table: public-rt
  Routes:  10.0.0.0/16 → local
           0.0.0.0/0   → igw-xxxxxxxx
  Subnets: public-subnet-1a, public-subnet-1b

Route Table: private-rt-1a
  Routes:  10.0.0.0/16 → local
           0.0.0.0/0   → nat-aaaaaaaa  (NAT GW in ap-south-1a)
  Subnets: private-subnet-1a

Route Table: private-rt-1b
  Routes:  10.0.0.0/16 → local
           0.0.0.0/0   → nat-bbbbbbbb  (NAT GW in ap-south-1b)
  Subnets: private-subnet-1b
```

Notice that public subnets share one route table (they both go to the same IGW). Private subnets each have their own route table pointing to their AZ-local NAT Gateway. This is the standard production pattern.

---

## NAT Gateway Costs — Where Bills Accumulate

NAT Gateway is one of the more expensive VPC components when misunderstood. The pricing has two parts:

**Hourly charge:** You pay for every hour a NAT Gateway exists, regardless of whether any traffic flows through it. In `ap-south-1`, this is approximately $0.045/hour, or ~$32/month per NAT Gateway. If you have two NAT Gateways (one per AZ), that's ~$64/month just in existence charges.

**Per-GB data processing charge:** Every GB of data that passes through the NAT Gateway is charged separately. In `ap-south-1`, this is approximately $0.045/GB. This is in addition to any EC2 outbound data transfer charges.

Real-world implication: a private EC2 instance that streams large amounts of data outbound (log shipping, backups, large API payloads) can accumulate significant NAT Gateway data processing charges. This is why S3 VPC Endpoints (covered in a later topic) are important — S3 traffic routed through a VPC Endpoint bypasses the NAT Gateway entirely, saving both the NAT GW data processing charge and the cross-AZ charge.

> ⚠️ **Interview Trap:** "Is NAT Gateway free like Internet Gateway?" No. IGW is free. NAT Gateway has both hourly and per-GB charges. This is a common cost optimization question in architecture interviews.

---

## Hands-On: Complete Two-Tier Setup via CLI

```bash
# Assumes VPC (vpc-xxx), public subnets, IGW, and public-rt already exist from Part 3

# Step 1: Create private subnets
aws ec2 create-subnet \
  --vpc-id vpc-xxxxxxxx \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1a}]'
# → SubnetId: subnet-private-1a

aws ec2 create-subnet \
  --vpc-id vpc-xxxxxxxx \
  --cidr-block 10.0.12.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1b}]'
# → SubnetId: subnet-private-1b

# Step 2: Allocate Elastic IPs for NAT Gateways
aws ec2 allocate-address --domain vpc
# → AllocationId: eipalloc-aaaaaaaa

aws ec2 allocate-address --domain vpc
# → AllocationId: eipalloc-bbbbbbbb

# Step 3: Create NAT Gateways in PUBLIC subnets (critical — not in private subnets)
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a \
  --allocation-id eipalloc-aaaaaaaa \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gw-1a}]'
# → NatGatewayId: nat-aaaaaaaa
# Wait for state: available (~60 seconds)

aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1b \
  --allocation-id eipalloc-bbbbbbbb \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gw-1b}]'
# → NatGatewayId: nat-bbbbbbbb

# Step 4: Create private route tables (one per AZ)
aws ec2 create-route-table \
  --vpc-id vpc-xxxxxxxx \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt-1a}]'
# → RouteTableId: rtb-private-1a

aws ec2 create-route-table \
  --vpc-id vpc-xxxxxxxx \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt-1b}]'
# → RouteTableId: rtb-private-1b

# Step 5: Add NAT routes (each AZ points to its local NAT GW)
aws ec2 create-route \
  --route-table-id rtb-private-1a \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-aaaaaaaa

aws ec2 create-route \
  --route-table-id rtb-private-1b \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-bbbbbbbb

# Step 6: Associate route tables with private subnets
aws ec2 associate-route-table \
  --route-table-id rtb-private-1a \
  --subnet-id subnet-private-1a

aws ec2 associate-route-table \
  --route-table-id rtb-private-1b \
  --subnet-id subnet-private-1b
```

> ⚠️ **Interview Trap — most common mistake:** NAT Gateway must be placed in a **public subnet**, not a private subnet. It needs internet access itself (via the IGW) to forward traffic. Placing it in a private subnet is a very common error that results in it silently not working.

---

## The Security Boundary — What Private Really Means

Let's be precise about what a resource in a private subnet can and cannot do, end-to-end:

```
Can a private EC2 instance...

Receive inbound traffic from internet?        NO  (no IGW route, no public IP)
Initiate outbound connection to internet?     YES (via NAT Gateway)
Receive a response to its outbound request?   YES (NAT Gateway tracks state)
Receive traffic from public subnet in VPC?    YES (local route + SG rules)
Receive traffic from other private subnet?    YES (local route + SG rules)
Be SSHed into directly from internet?         NO  (use Bastion Host instead)
```

The bastion host pattern (a small EC2 in the public subnet used as a jump server to SSH into private instances) follows naturally from this. You SSH into the bastion with its public IP, then from the bastion you SSH into private instances using their private IPs. The private instances never need to be internet-facing.

---

## Comparison Table: IGW vs NAT Gateway

| Dimension | Internet Gateway | NAT Gateway |
|---|---|---|
| **Direction** | Inbound + outbound | Outbound only |
| **Placed in** | Attached to VPC | Specific public subnet |
| **Cost** | Free | Hourly + per-GB |
| **Managed by** | AWS | AWS |
| **Scales to** | Unlimited | Up to 100 Gbps |
| **Public IP** | Uses instance's EIP/public IP | Has its own EIP |
| **Used by** | Public subnet instances | Private subnet instances |
| **AZ-redundant** | Yes (VPC-level) | No (one per AZ needed) |
| **Security Groups** | N/A | Cannot attach |
| **Inbound initiation** | Allowed | Blocked |

---

## Interview Flags for This Topic

> ⚠️ **"Where should you place a NAT Gateway — public or private subnet?"** Always public subnet. It needs IGW access to function. Private subnet placement is a common mistake.

> ⚠️ **"Can a private subnet instance be reached from the internet via NAT Gateway?"** No. NAT Gateway only translates outbound connections from private instances and their responses. It does not forward unsolicited inbound traffic from the internet.

> ⚠️ **"How many NAT Gateways for a production two-AZ setup?"** Two — one per AZ. Single NAT Gateway creates AZ-level single point of failure and adds cross-AZ data transfer costs.

> ⚠️ **"What happens if you forget to attach an Elastic IP to a NAT Gateway?"** You can't create a NAT Gateway without specifying an allocation ID for an Elastic IP — the API requires it. But if the EIP is released while the NAT GW exists, outbound traffic from private instances will fail.

> ⚠️ **"Can you use a NAT Gateway to allow inbound connections to a private instance on a specific port?"** No. For inbound access to private resources, the pattern is: ALB in public subnet (for HTTP/S) or Bastion Host (for SSH). NAT Gateway cannot be used for inbound traffic.

---

The next topic that completes this architecture is either **Bastion Hosts / SSH jump patterns** for admin access into private instances, or **NACLs** for subnet-level stateless traffic control — the second layer of defense beyond security groups. Drop whichever comes next in your series.
