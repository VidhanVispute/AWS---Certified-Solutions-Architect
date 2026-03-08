# AWS VPC — Part 8: VPC Peering — Deep Dive, Architecture, and Limitations

---

## Why VPC Peering Exists — The Business Problem First

You've built a well-architected production VPC. Private subnets, NAT Gateway, bastion hosts — the whole thing. Now your company acquires another startup. They have their own AWS account, their own VPC, their own infrastructure. Your payment service needs to talk to their user authentication service. Both are in private subnets. Neither should be exposed to the public internet. How do they communicate?

Or a different scenario: your own company has grown. You've followed best practices and created separate VPCs for production, staging, and a shared-services VPC that runs internal tools — a private Artifactory server, an internal DNS resolver, a logging aggregator. Your production application servers need to reach the logging aggregator in the shared-services VPC without those services ever being internet-facing.

Or another: your application is deployed in both Mumbai (`ap-south-1`) and Singapore (`ap-southeast-1`) for latency reasons. A Mumbai instance needs to replicate data to Singapore. Both are in private subnets.

In all three scenarios, the answer is **VPC Peering** — a private networking connection between two VPCs that makes them behave as if they're on the same network, without either being exposed to the internet.

---

## What VPC Peering Is — Precisely

A VPC Peering connection is a **one-to-one, private network route** between exactly two VPCs. Once established and configured, instances in either VPC can communicate with instances in the other using **private IP addresses**, as if they were in the same VPC.

The key architectural properties:

**It uses AWS's private backbone network.** Traffic between peered VPCs never traverses the public internet. It stays on AWS's internal network fabric. This means lower latency, no internet bandwidth costs, and no exposure to internet-based threats.

**It is non-transitive.** This is the single most important property of VPC Peering and the source of its primary limitation. If VPC-A is peered with VPC-B, and VPC-B is peered with VPC-C, VPC-A cannot reach VPC-C through VPC-B. Each peering relationship is a direct, point-to-point connection.

**It is symmetric.** Once established, traffic can flow in both directions. VPC-A can initiate connections to VPC-B and vice versa, subject to security group and NACL rules.

**It requires non-overlapping CIDRs.** If VPC-A uses `10.0.0.0/16` and VPC-B also uses `10.0.0.0/16`, peering is impossible. AWS cannot create a route for `10.0.0.0/16` that points to two different places simultaneously.

---

## The Non-Transitivity Property — Why It Matters

This deserves its own section because it is the most commonly misunderstood aspect of VPC Peering and appears in almost every architecture interview involving VPCs.

```
VPC-A (10.0.0.0/16)
    │
    │ Peering connection: A↔B
    │
VPC-B (10.1.0.0/16)
    │
    │ Peering connection: B↔C
    │
VPC-C (10.2.0.0/16)

Question: Can VPC-A communicate with VPC-C?
Answer:   NO.
```

Even though VPC-B has connections to both A and C, it does not act as a router between them. AWS explicitly prohibits transit routing through a peering connection. A packet from VPC-A destined for VPC-C's address space `10.2.0.0/16` has no route in VPC-A's route table pointing to VPC-C. The only way to make A and C communicate is to create a **direct peering connection between A and C**.

```
Full mesh for 3 VPCs (all-to-all communication):
VPC-A ←──────────────→ VPC-B
  │                      │
  │                      │
  └──────→ VPC-C ←───────┘

Requires: 3 peering connections for 3 VPCs
          6 peering connections for 4 VPCs
          10 peering connections for 5 VPCs
          Formula: n(n-1)/2
```

As your VPC count grows, the number of peering connections required for full mesh connectivity grows quadratically. At 10 VPCs, you need 45 peering connections. This is the fundamental scalability problem with VPC Peering at enterprise scale, and it's why AWS Transit Gateway exists (covered next in your series). For 2-5 VPCs, peering is clean and simple. For larger architectures, it becomes a management nightmare.

> ⚠️ **Interview Trap:** "If VPC-A is peered with VPC-B and VPC-B is peered with VPC-C, can VPC-A access VPC-C?" Definitively no. Non-transitivity is a hard technical constraint, not a configuration option. The only solution is a direct A↔C peering connection or Transit Gateway.

---

## VPC Peering Scope — Same Account, Cross-Account, Cross-Region

VPC Peering works across all three boundaries, but the setup process differs slightly for each:

```
┌─────────────────────────────────────────────────────────────────┐
│  Same Account, Same Region                                      │
│  ─────────────────────────                                      │
│  Simplest case. Requester creates peering, accepter accepts.    │
│  Both VPCs visible in same console. No DNS complications.       │
├─────────────────────────────────────────────────────────────────┤
│  Same Account, Different Regions (Inter-Region Peering)         │
│  ──────────────────────────────────────────────────────         │
│  Traffic crosses AWS's backbone between regions.                │
│  Incurs inter-region data transfer charges.                     │
│  Higher latency than intra-region (still low — AWS backbone).   │
│  Must enable DNS resolution separately in each VPC.             │
├─────────────────────────────────────────────────────────────────┤
│  Different Accounts (Cross-Account Peering)                     │
│  ──────────────────────────────────────────                     │
│  Requester sends peering request specifying target account ID   │
│  and VPC ID. Accepter logs in to their account to accept.       │
│  Most common in enterprise multi-account AWS Organizations.     │
│  Each account manages its own route tables and SGs.             │
└─────────────────────────────────────────────────────────────────┘
```

For cross-account peering, the requester needs to know the target account's **AWS Account ID** and **VPC ID**. The target account owner must explicitly accept — there's no way to force a peering connection on another account.

---

## The Three-Step Rule — What Makes Peering Actually Work

The video demonstrates this clearly: creating a peering connection and having it show "Active" status is not enough. Traffic still won't flow. There are three independent things that must all be configured correctly:

```
Step 1: Peering Connection Created and Accepted
─────────────────────────────────────────────────
Both VPC owners agree. Status = Active.
Result: A logical connection exists. No traffic flows yet.

Step 2: Route Tables Updated in BOTH VPCs
─────────────────────────────────────────────────
VPC-A's route table: 10.1.0.0/16 → pcx-xxxxxxxx
VPC-B's route table: 10.0.0.0/16 → pcx-xxxxxxxx
Result: AWS now knows how to forward packets between VPCs.
        Still no traffic flows yet — security layer not open.

Step 3: Security Groups Allow the Traffic
─────────────────────────────────────────────────
VPC-B's instance SG: allow inbound on port X from 10.0.0.0/16
Result: Traffic can now flow end-to-end.
```

Forgetting Step 2 is the most common operational mistake. Forgetting Step 3 is the second most common. Both present as "connection appears Active but traffic doesn't work."

> ⚠️ **Interview Trap:** "You've created a VPC peering connection between VPC-A and VPC-B. The status shows Active. Your EC2 instance in VPC-A cannot ping an instance in VPC-B. What are the possible causes?" Two independent causes: (1) route tables in one or both VPCs not updated with routes pointing to the peering connection, (2) security groups not allowing ICMP from the source VPC's CIDR.

---

## Route Table Configuration — The Exact Entries Required

For a peering connection `pcx-xxxxxxxxx` between VPC-A (`10.0.0.0/16`) and VPC-B (`10.1.0.0/16`):

```
VPC-A Route Table (must add):
┌──────────────┬──────────────────────────┐
│ Destination  │ Target                   │
├──────────────┼──────────────────────────┤
│ 10.0.0.0/16  │ local                    │
│ 10.1.0.0/16  │ pcx-xxxxxxxxx            │  ← NEW: traffic to VPC-B goes via peering
│ 0.0.0.0/0    │ igw-xxxxxxxxx (if public)│
└──────────────┴──────────────────────────┘

VPC-B Route Table (must add):
┌──────────────┬──────────────────────────┐
│ Destination  │ Target                   │
├──────────────┼──────────────────────────┤
│ 10.1.0.0/16  │ local                    │
│ 10.0.0.0/16  │ pcx-xxxxxxxxx            │  ← NEW: traffic to VPC-A goes via peering
│ 0.0.0.0/0    │ nat-xxxxxxxxx (if private)│
└──────────────┴──────────────────────────┘
```

You need to add the route on **both sides**. Adding it only in VPC-A means packets from VPC-A can reach VPC-B, but the response from VPC-B has no route back to VPC-A and gets dropped. This is a very common mistake that produces asymmetric connectivity — one-way pings that never get replies.

If you have multiple subnets in each VPC with separate route tables (as recommended in a proper multi-tier architecture), you need to add the peering route to **each route table** in each VPC that needs to participate. A route table that doesn't have the peering route means instances in that subnet can't use the peering connection.

---

## CIDR Planning for VPC Peering — Why It Must Be Done Upfront

The non-overlapping CIDR requirement is not just a VPC Peering constraint — it's a fundamental constraint of how IP routing works. You cannot have two different destinations share the same address range without ambiguity.

This makes **upfront CIDR planning critical** in any organization that will eventually need VPC peering:

```
BAD planning (overlapping CIDRs — peering is impossible):
  Production VPC:   10.0.0.0/16
  Staging VPC:      10.0.0.0/16   ← SAME as prod — cannot peer
  Dev VPC:          10.0.0.0/16   ← SAME as prod — cannot peer

GOOD planning (non-overlapping — peering works freely):
  Production VPC:   10.0.0.0/16   (10.0.0.0 – 10.0.255.255)
  Staging VPC:      10.1.0.0/16   (10.1.0.0 – 10.1.255.255)
  Dev VPC:          10.2.0.0/16   (10.2.0.0 – 10.2.255.255)
  Shared Services:  10.3.0.0/16   (10.3.0.0 – 10.3.255.255)
  On-premises:      172.16.0.0/12 (reserved for future Direct Connect)
```

Many organizations discover this problem after the fact — they've been using `10.0.0.0/16` in every VPC because it's the default suggestion, and then they need to peer them. The fix (re-IPing VPCs) is extremely painful, involving migrating all resources. This is a genuine enterprise-scale problem that AWS Professional Services gets called in to solve.

> ⚠️ **Interview Trap:** "What is the CIDR block of the AWS Default VPC?" `172.31.0.0/16`. If you're peering custom VPCs, avoid this range — if someone tries to peer with a default VPC, they'll have a conflict.

---

## Hands-On: Complete Peering Setup via CLI

```bash
# Scenario: Peer VPC-A (ap-south-1) with VPC-B (us-east-1), same account

# Step 1: Create peering connection request (from VPC-A's region)
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaaaaaa \
  --peer-vpc-id vpc-bbbbbbbb \
  --peer-region us-east-1 \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=mumbai-virginia-peer}]'
# Returns: VpcPeeringConnectionId: pcx-xxxxxxxxxxxxxxxxx
# Status: pending-acceptance

# Step 2: Accept the peering request (run in us-east-1 region)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxxxxxxxxxxxxxxx \
  --region us-east-1
# Status: active

# Step 3: Add route in VPC-A's route table (ap-south-1)
aws ec2 create-route \
  --route-table-id rtb-aaaaaaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-xxxxxxxxxxxxxxxxx

# Step 4: Add route in VPC-B's route table (us-east-1)
aws ec2 create-route \
  --route-table-id rtb-bbbbbbbb \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-xxxxxxxxxxxxxxxxx \
  --region us-east-1

# Step 5: Update Security Groups
# On VPC-B instance's SG — allow SSH from VPC-A's CIDR
aws ec2 authorize-security-group-ingress \
  --group-id sg-bbbbbbbb \
  --protocol tcp \
  --port 22 \
  --cidr 10.0.0.0/16 \
  --region us-east-1

# Step 6: Test
# From VPC-A instance:
ssh ec2-user@10.1.x.x  # VPC-B private IP — should work now
```

---

## DNS Resolution Across Peered VPCs

By default, if an instance in VPC-A does a DNS lookup for a hostname in VPC-B (e.g., an RDS endpoint's private DNS name), it gets the public IP back, not the private IP. This means traffic would go to the internet and come back, completely defeating the purpose of peering.

To fix this, you must enable **DNS resolution support** for peering on both sides:

```bash
# Enable DNS resolution from VPC-A to VPC-B's hostnames
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxxxxxxxxxxxxxxx \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

# Enable DNS resolution from VPC-B to VPC-A's hostnames
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxxxxxxxxxxxxxxx \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
```

In the Console: VPC Peering Connections → select connection → Actions → Edit DNS Settings → enable for both requester and accepter.

After this, private DNS names in both VPCs resolve to private IPs across the peering connection. This is almost always required in production peering setups.

> ⚠️ **Interview Trap:** "You've set up VPC peering, updated route tables, and security groups allow traffic. SSH via private IP works. But your application can't connect to the RDS instance in the peered VPC using its DNS endpoint. Why?" DNS resolution across peering is not enabled by default. Enable it on both sides of the peering connection.

---

## VPC Peering vs Transit Gateway — Knowing When to Use Each

This comparison appears constantly in architecture interviews and AWS certification exams:

| Dimension | VPC Peering | Transit Gateway |
|---|---|---|
| **Topology** | Point-to-point (1:1) | Hub-and-spoke (1:many) |
| **Transitivity** | Non-transitive | Transitive |
| **Max VPCs** | Practical limit ~10-15 (mesh becomes unmanageable) | 5,000 attachments |
| **Cross-region** | Yes (inter-region peering) | Yes (via TGW peering) |
| **Cross-account** | Yes | Yes |
| **Cost** | Free (pay data transfer only) | Hourly per attachment + per-GB |
| **Bandwidth** | No limit | Up to 50 Gbps per VPC attachment |
| **Route management** | Manual per VPC route table | Centralized in TGW route tables |
| **On-premises connectivity** | Not supported directly | Yes (VPN/Direct Connect to TGW) |
| **Best for** | 2-5 VPCs, simple topology | 5+ VPCs, enterprise, hub-spoke |

The decision rule is straightforward: if you have fewer than 5 VPCs and the topology is simple, peering is cheaper and simpler. Beyond that, Transit Gateway's centralized management and transitivity justify its cost.

---

## VPC Peering Limitations — The Complete List

These are not edge cases — they come up in production and in interviews:

**No transitive routing.** Already covered in depth. The hard limit.

**No overlapping CIDRs.** Plan upfront or suffer re-IPing later.

**No edge-to-edge routing.** If VPC-A is peered with VPC-B, and VPC-B has a VPN connection to an on-premises network, VPC-A cannot reach the on-premises network through VPC-B. Same logic: no transit.

**One active peering per VPC pair.** You cannot create two peering connections between the same two VPCs.

**Peering is not a VPN.** The traffic is private (stays on AWS backbone) but is not encrypted by default. For compliance requirements that mandate encryption in transit, you'd need additional controls.

**Security Group cross-referencing has limits.** Within the same region, you can reference a security group from a peered VPC as a source/destination in your security group rules (instead of a CIDR). This is more precise than CIDR-based rules. Across regions (inter-region peering), you cannot reference security groups from the peered VPC — you must use CIDR blocks.

---

## Interview Flags for This Topic

> ⚠️ **"How many VPC peering connections are needed for 5 VPCs to all communicate with each other?"** n(n-1)/2 = 5×4/2 = **10 peering connections**. This is the full-mesh formula.

> ⚠️ **"Can you peer VPCs with overlapping CIDRs?"** No. Overlapping CIDRs make it impossible to create unambiguous routes. Planning non-overlapping CIDRs upfront is essential.

> ⚠️ **"VPC peering status is Active but instances can't communicate. What are the two most likely causes?"** Route tables not updated in one or both VPCs, and/or security groups not allowing traffic from the peered VPC's CIDR.

> ⚠️ **"Can an on-premises network reach VPC-A through a peered VPC-B?"** No. Edge-to-edge routing through peering is not supported. On-premises connectivity must be established directly to each VPC.

> ⚠️ **"Is VPC peering traffic encrypted?"** No. It traverses AWS's private backbone and never hits the public internet, but it is not encrypted by default. Application-layer encryption (TLS) or VPN over peering is needed for encryption requirements.

> ⚠️ **"What's the default limit for VPCs per region?"** 5 (soft limit, can be increased via service quota request to 100+). For large enterprises, this limit is routinely increased.

---

The next logical topic is either **AWS Transit Gateway** (the scalable alternative to VPC Peering for complex topologies) or **Security Groups vs NACLs** (the two-layer defense model). Both are important — check your video series and drop whichever comes next.
