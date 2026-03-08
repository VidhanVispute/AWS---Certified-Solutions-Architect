# AWS VPC — Part 7: NAT Gateway — Deep Dive, Setup, and Production Architecture

---

## The Exact Problem NAT Gateway Solves

At the end of the bastion host topic, you could SSH into your private instance. The first thing you'd naturally do is run `sudo yum update -y` or `sudo apt-get update` — and it hangs. Times out. Nothing happens. The instance has no way to reach the Ubuntu or Amazon Linux package repositories because those are on the internet, and the private subnet has no outbound internet path.

This is not an edge case. It's a fundamental operational requirement. Every production server needs to:

- Pull OS security patches (critical — unpatched servers are the most common breach vector)
- Download application dependencies at deploy time (`pip install`, `npm install`, `mvn package`)
- Pull container images from Docker Hub or ECR public
- Call third-party APIs (payment gateways, SMS providers, authentication services)
- Send logs to external monitoring systems (Datadog, Splunk, etc.)
- Sync time with NTP servers

All of this is outbound-initiated traffic. None of it requires the internet to initiate a connection inward. This asymmetry — outbound yes, inbound no — is the precise definition of what NAT Gateway provides, and why it exists as a distinct component from the Internet Gateway.

---

## NAT: The Concept Before the AWS Service

NAT (Network Address Translation) is not an AWS invention. It's a decades-old networking concept that became widespread when IPv4 address exhaustion made it impossible for every device to have its own public IP. Your home router does NAT right now — your laptop, phone, and TV all have private IPs (`192.168.1.x`), and when they access the internet, your router translates all of them to your single ISP-assigned public IP.

AWS's NAT Gateway is the cloud-native, managed implementation of this concept, built specifically for the VPC use case. Understanding the underlying mechanism helps you reason about behavior, limitations, and troubleshooting.

### How NAT Translation Actually Works

When a private instance sends a packet outbound:

```
Private instance: src=10.0.11.25:54231  dst=140.82.112.4:443  (github.com)
                         │
                         ▼ (routed to NAT GW via route table)
NAT Gateway:      Replaces src with its own Elastic IP
                  src=52.66.100.50:54231  dst=140.82.112.4:443
                         │
                         ▼ (records translation in connection table)
                  {52.66.100.50:54231 ↔ 10.0.11.25:54231}
                         │
                         ▼ (exits via IGW)
GitHub's servers see:    src=52.66.100.50:54231
```

When the response arrives:

```
GitHub responds:  src=140.82.112.4:443  dst=52.66.100.50:54231
                         │
                         ▼ (arrives at NAT GW's Elastic IP)
NAT Gateway:      Looks up connection table
                  Translates back: dst=10.0.11.25:54231
                         │
                         ▼
Private instance receives response at 10.0.11.25:54231
```

The private IP `10.0.11.25` is never visible to the internet. GitHub sees only `52.66.100.50` — the NAT Gateway's Elastic IP. Critically, if GitHub tried to initiate a brand new connection to `52.66.100.50`, the NAT Gateway would receive it and find no matching entry in its connection table. It would drop the packet. **This is not a firewall rule — it is the fundamental nature of NAT.** Unsolicited inbound connections simply have nowhere to go.

---

## NAT Gateway Architecture — Every Detail

### Physical Placement

NAT Gateway is provisioned in a **specific public subnet in a specific AZ**. This is the most architecturally important constraint. It means:

```
NAT Gateway in ap-south-1a ──── serves ────▶ Private subnet in ap-south-1a
NAT Gateway in ap-south-1b ──── serves ────▶ Private subnet in ap-south-1b
```

If you have a single NAT Gateway in `ap-south-1a` and a private subnet in `ap-south-1b`, the `ap-south-1b` private instances can still route through it — but every packet crosses the AZ boundary. This has two consequences: cross-AZ data transfer charges ($0.01/GB each direction), and if `ap-south-1a` has an outage, your `ap-south-1b` private instances also lose outbound internet. Both cost and resilience arguments point to the same solution: **one NAT Gateway per AZ**.

```
SINGLE NAT GW (cheaper, fragile):
─────────────────────────────────────────────────────────
  AZ-a public subnet:   [NAT GW] ←── ap-south-1a private instances
                           ↑
                    crosses AZ boundary  ← costs money + single point of failure
                           │
                    ap-south-1b private instances

NAT GW PER AZ (production standard):
─────────────────────────────────────────────────────────
  AZ-a public subnet:   [NAT GW A] ←── ap-south-1a private instances
  AZ-b public subnet:   [NAT GW B] ←── ap-south-1b private instances
  (each AZ self-contained, AZ failure is isolated)
```

### The Elastic IP Requirement

Every NAT Gateway must have exactly one Elastic IP associated with it. This EIP is the public IP that the internet sees as the source of all traffic from your private instances. The EIP must be:

- Allocated to your account before creating the NAT Gateway
- In the same region as the NAT Gateway
- Not currently associated with any other resource

The EIP is not optional and cannot be changed after the NAT Gateway is created. If you need to change the public IP that your private instances appear to come from (for IP allowlisting with a third-party service, for example), you must create a new NAT Gateway with a new EIP and update your route tables.

### Capacity and Performance

NAT Gateway automatically scales from **5 Gbps up to 100 Gbps** without any configuration. There's no instance type to choose, no capacity planning required. AWS handles this entirely. If your private instances collectively need 60 Gbps of outbound throughput, the NAT Gateway handles it. This is the primary operational advantage over a NAT Instance — you never think about capacity.

Each NAT Gateway supports up to **55,000 simultaneous connections** to a single destination (IP + port combination). For most workloads this is irrelevant, but for high-connection-count workloads (Lambda at scale, containers making many concurrent external calls), it's worth knowing.

---

## Step-by-Step Setup — Console and CLI

### The Dependency Order

Before creating the NAT Gateway, the following must already exist:

```
1. VPC with CIDR
2. Public subnet in the target AZ (NAT GW goes HERE)
3. Private subnet in the same AZ (traffic comes FROM here)
4. Internet Gateway attached to the VPC (NAT GW routes through this)
5. Public subnet's route table has 0.0.0.0/0 → IGW
6. Elastic IP allocated
   ↓
7. Create NAT Gateway (in public subnet, with EIP)
   ↓
8. Update private subnet's route table: 0.0.0.0/0 → NAT GW
```

Steps 1-5 are the foundation from Parts 2-5. Steps 6-8 are the NAT Gateway setup.

### Console Walkthrough

**Allocate Elastic IP:**
EC2 → Elastic IPs → Allocate Elastic IP address → Amazon's pool of IPv4 addresses → Allocate

Note the Allocation ID (e.g., `eipalloc-0a1b2c3d4e5f67890`).

**Create NAT Gateway:**
VPC → NAT Gateways → Create NAT Gateway:

```
Name:              nat-gw-1a
Subnet:            public-subnet-1a          ← MUST be public subnet
Connectivity type: Public                    ← Public = internet-facing
Elastic IP:        [select the EIP you just allocated]
```

The "Private" connectivity type creates a NAT Gateway for routing between VPCs without internet access — that's a different, less common use case. For standard internet access, always choose "Public".

After clicking Create, the NAT Gateway enters a **Pending** state. It takes approximately 60 seconds to become **Available**. Do not update route tables until it reaches Available — routes pointing to a pending NAT GW will fail silently.

**Update the Private Subnet Route Table:**

VPC → Route Tables → select the private subnet's route table → Routes → Edit routes → Add route:

```
Destination: 0.0.0.0/0
Target:      nat-gw-1a (select from NAT Gateway dropdown)
```

Save. That's the complete setup.

### CLI Equivalent

```bash
# Step 1: Allocate EIP
aws ec2 allocate-address --domain vpc
# Returns: AllocationId: eipalloc-xxxxxxxxx
#          PublicIp: 52.66.100.50

# Step 2: Create NAT Gateway in public subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a \
  --allocation-id eipalloc-xxxxxxxxx \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gw-1a}]'
# Returns: NatGatewayId: nat-xxxxxxxxxxxxxxxxx
# State: pending → wait for available

# Step 3: Wait for available state
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids nat-xxxxxxxxxxxxxxxxx

# Step 4: Add route in private subnet's route table
aws ec2 create-route \
  --route-table-id rtb-private-1a \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-xxxxxxxxxxxxxxxxx

# Verify
aws ec2 describe-route-tables --route-table-ids rtb-private-1a \
  --query 'RouteTables[0].Routes'
```

---

## Verification — What to Test and How

After setup, verify from inside the private instance (accessed via bastion or EIC endpoint):

```bash
# Test 1: Basic outbound connectivity
ping -c 4 8.8.8.8
# Should succeed — proves IP routing works

ping -c 4 google.com
# Should succeed — proves DNS also works (DNS goes through NAT GW too)

# Test 2: HTTP/S outbound
curl -I https://google.com
# Should return HTTP 301 — proves HTTPS outbound works

# Test 3: Package manager (the real-world test)
sudo apt-get update        # Ubuntu/Debian
sudo yum update -y         # Amazon Linux/CentOS

# Test 4: Verify your public IP (should show NAT GW's EIP, not private IP)
curl https://checkip.amazonaws.com
# Returns: 52.66.100.50  (NAT Gateway's Elastic IP)
# NOT the private IP 10.0.11.25
```

Test 4 is particularly useful — it confirms the NAT translation is working correctly and shows you exactly what IP external services see for your private instances.

**What should NOT work (confirming security boundary held):**

```bash
# From your laptop — this should fail/timeout
ssh ec2-user@10.0.11.25
# Cannot reach private IP from internet — correct

# From internet, try to reach private instance by NAT GW's EIP
# There's no port forwarding, no way to reach private instances
# through the NAT GW from outside — correct
```

---

## The Complete Traffic Flow — Annotated End to End

```
Private EC2 (10.0.11.25) runs: apt-get update
        │
        │ Packet: src=10.0.11.25, dst=91.189.91.38 (ubuntu repos)
        ▼
Route Table lookup (private-rt-1a):
  91.189.91.38 matches 0.0.0.0/0 → nat-gw-1a
        │
        ▼
NAT Gateway (in public-subnet-1a)
  Private IP: 10.0.1.8
  Elastic IP: 52.66.100.50
  Action: src rewrite → 52.66.100.50, records mapping
        │
        │ Packet: src=52.66.100.50, dst=91.189.91.38
        ▼
Route Table lookup (public-rt):
  91.189.91.38 matches 0.0.0.0/0 → igw-xxxxxxxx
        │
        ▼
Internet Gateway
  NAT: 52.66.100.50 is an EIP — no further translation needed
  (IGW only NATs for EC2 instances' private↔public IPs, not for EIPs
   which are already public)
        │
        ▼
Ubuntu package server (91.189.91.38)
  Sees: connection from 52.66.100.50
  Responds to: 52.66.100.50
        │
        ▼
IGW → NAT Gateway receives response
  Looks up connection table: 52.66.100.50 → 10.0.11.25
  Rewrites dst back to 10.0.11.25
        │
        ▼
Private EC2 (10.0.11.25) receives package data
```

---

## NAT Gateway Pricing — Where Surprises Happen

NAT Gateway pricing has two independent components, and both matter:

**Hourly charge:** ~$0.045/hour per NAT Gateway in Mumbai (`ap-south-1`). This means:
- 1 NAT GW = ~$32/month
- 2 NAT GWs (one per AZ, recommended) = ~$65/month
- This charge runs 24/7 regardless of traffic volume

**Data processing charge:** ~$0.045/GB of data passing through the NAT Gateway, in both directions (inbound responses count too, not just outbound requests).

**The S3/DynamoDB optimization:** If your private instances access S3 or DynamoDB heavily, routing that traffic through a NAT Gateway is expensive and unnecessary. AWS provides **VPC Gateway Endpoints** for S3 and DynamoDB — these allow private instances to reach S3/DynamoDB via AWS's private network, completely bypassing the NAT Gateway. The endpoint itself is free. This single optimization can reduce NAT Gateway data processing costs by 60-80% for data-heavy applications.

```
Without VPC Endpoint:
Private EC2 → NAT GW ($0.045/GB) → IGW → S3

With VPC Gateway Endpoint:
Private EC2 → VPC Endpoint (FREE) → S3 private network
```

**Cleanup sequence (important for lab environments):**

```bash
# Order matters — delete NAT GW first, then release EIP

# Step 1: Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id nat-xxxxxxxxxxxxxxxxx
# Wait for state: deleted (~60 seconds)

# Step 2: Remove the route from private route table
aws ec2 delete-route \
  --route-table-id rtb-private-1a \
  --destination-cidr-block 0.0.0.0/0

# Step 3: Release the Elastic IP (ONLY after NAT GW is fully deleted)
aws ec2 release-address --allocation-id eipalloc-xxxxxxxxx
```

If you release the EIP before the NAT GW is deleted, you'll get an error. If you forget to release the EIP after deleting the NAT GW, you'll pay ~$0.005/hour for an idle EIP indefinitely.

---

## Common Mistakes and Troubleshooting

**Mistake 1: NAT Gateway in private subnet**

The most common beginner error. NAT Gateway must be in a public subnet — it needs its own path to the internet via the IGW to forward traffic. A NAT Gateway in a private subnet has no internet path and is non-functional.

```
Symptom: Private instances still can't reach internet after NAT GW creation
Check:   aws ec2 describe-nat-gateways → look at SubnetId → verify it's a public subnet
```

**Mistake 2: Route table updated on wrong subnet**

You need to update the **private subnet's** route table, not the public subnet's. A very easy mistake when subnets aren't clearly named.

```
Symptom: Same as above
Check:   Verify the route table is associated with the private subnet, not the public one
```

**Mistake 3: NAT Gateway still in Pending state when route was added**

Routes pointing to a pending NAT GW fail silently. The route exists but doesn't work.

```
Symptom: Route exists, NAT GW exists, but traffic doesn't flow
Check:   aws ec2 describe-nat-gateways → State must be "available"
```

**Mistake 4: Trying to use NAT Gateway for inbound access**

NAT Gateway is strictly outbound. No amount of port forwarding configuration will make it work for inbound connections — it doesn't support that model at all.

```
Symptom: Trying to reach private instance through NAT GW's EIP
Reality: Not possible. Use ALB, bastion host, or EIC endpoint for inbound access
```

---

## Comparison Table: NAT Gateway vs NAT Instance vs No NAT

| Dimension | NAT Gateway | NAT Instance | No NAT |
|---|---|---|---|
| **Outbound internet** | Yes | Yes | No |
| **Inbound internet** | No | Configurable (port fwd) | No |
| **Management** | AWS managed | You manage OS/patches | N/A |
| **Availability** | Highly available in AZ | Single EC2 (SPOF) | N/A |
| **Bandwidth** | 5–100 Gbps auto-scale | Instance type limited | N/A |
| **Cost** | ~$32/mo + $0.045/GB | EC2 cost only (~$4/mo) | Free |
| **Security Groups** | Not attachable | Attachable | N/A |
| **Port forwarding** | Not supported | Supported | N/A |
| **Source/dest check** | N/A | Must disable | N/A |
| **Use case** | All production | Dev/test cost saving | Air-gapped environments |
| **Bastion combined** | No | Possible | No |
| **IPv6 support** | No (use Egress-Only IGW) | Yes | N/A |

---

## Interview Flags for This Topic

> ⚠️ **"Can NAT Gateway be used for inbound traffic?"** No. It only supports outbound-initiated connections. For inbound to private instances, you need ALB (HTTP/S), NLB (TCP), or bastion host (SSH).

> ⚠️ **"Where must NAT Gateway be placed?"** In a public subnet. It requires its own outbound internet path via the IGW.

> ⚠️ **"How many NAT Gateways for a 3-AZ production setup?"** Three — one per AZ. Each AZ's private route table points to its local NAT Gateway for resilience and to avoid cross-AZ data transfer charges.

> ⚠️ **"Is NAT Gateway traffic charged twice — once for NAT GW data processing and once for EC2 outbound transfer?"** Yes. Both charges apply independently. The NAT GW charges per-GB for processing, and the EC2 instance's outbound data transfer rate also applies. This is why heavy S3 users should use VPC Gateway Endpoints.

> ⚠️ **"What's the difference between NAT Gateway and Internet Gateway?"** IGW is bidirectional — allows both inbound and outbound. NAT GW is outbound-only — private instances initiate connections out, internet cannot initiate connections in.

> ⚠️ **"What happens to existing connections through a NAT Gateway if it's deleted?"** They drop immediately. The NAT Gateway's connection tracking table is gone, so all active sessions are terminated. New connection attempts from private instances to the internet will fail until a new NAT GW is created and the route table is updated.

> ⚠️ **"Can two VPCs share one NAT Gateway?"** No — a NAT Gateway lives in a specific VPC's subnet and is not shareable across VPCs. Each VPC needs its own NAT Gateway (or uses VPC Peering/Transit Gateway to route through a central VPC's NAT GW, which is a valid centralized egress architecture pattern).

---

The complete picture of your two-tier VPC is now fully built — subnets, IGW, route tables, NAT Gateway, and bastion access. The next natural topic is **Security Groups vs NACLs** — the two-layer defense model for controlling traffic within and across subnets. Drop it whenever you're ready.
