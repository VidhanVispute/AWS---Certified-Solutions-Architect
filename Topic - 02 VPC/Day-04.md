# AWS VPC — Part 3: Internet Gateway and Route Tables

---

## The Gap Between "Has a Public IP" and "Is Internet Accessible"

After Part 2, you have a VPC with subnets and EC2 instances running inside them. Some instances have public IPs assigned. And yet — you cannot SSH into them, cannot ping them, cannot reach them in any way from the internet. The public IP exists but does nothing.

This confuses a lot of people early on, and the confusion comes from a mental model borrowed from home networking. At home, if your laptop has a public IP, it's on the internet. In AWS, a public IP assigned to an EC2 instance is just a number — a label — until the underlying network infrastructure is told what to do with traffic destined for that IP. That instruction comes from two places working together: the **Internet Gateway** and the **Route Table**. Neither one alone is sufficient. Both must be correctly configured, and in the right order.

Understanding why requires understanding what actually happens to a packet traveling from your laptop to an EC2 instance's public IP — and that story starts outside AWS entirely.

---

## How a Packet Reaches Your EC2 Instance — The Full Journey

When you type `ssh ec2-user@13.233.45.67` from your laptop, a packet leaves your home router, travels through your ISP, traverses the internet, and eventually reaches AWS's network edge. At that point, AWS's global network infrastructure — specifically its edge routers — knows that `13.233.45.67` belongs to the Mumbai region (AWS publishes its IP ranges publicly). The packet gets directed to AWS's Mumbai infrastructure.

But now it needs to get from AWS's edge into your specific VPC and to your specific EC2 instance. This is where the Internet Gateway enters.

```
Your Laptop
    │
    ▼
Your ISP → Internet → AWS Global Network Edge (Mumbai)
                              │
                              ▼
                    Internet Gateway (IGW)
                    (Attached to your VPC)
                              │
                              ▼
                    Route Table lookup:
                    "Where does 10.0.1.45 live?"
                    Answer: local → ap-south-1a subnet
                              │
                              ▼
                    EC2 Instance (private IP: 10.0.1.45)
                    (public IP 13.233.45.67 translated here)
```

The Internet Gateway performs **NAT (Network Address Translation)** on the way in and out. Your EC2 instance's operating system only knows its private IP (`10.0.1.45`). It has no idea it has a public IP. The IGW maps the public IP to the private IP transparently. When a response packet leaves the EC2 instance with source IP `10.0.1.45`, the IGW rewrites it to `13.233.45.67` before it hits the internet. This is one-to-one NAT — one public IP maps to exactly one private IP.

> ⚠️ **Interview Trap:** "Does the EC2 instance's OS know its own public IP?" No. If you run `ip addr` or `ifconfig` on an EC2 instance, you'll only see the private IP. The public IP is maintained by the IGW, entirely outside the instance. This is why application code that needs to know its own public IP must query the EC2 instance metadata service (`169.254.169.254/latest/meta-data/public-ipv4`) rather than reading it from the network interface.

---

## What the Internet Gateway Actually Is

The Internet Gateway is a **horizontally scaled, redundant, highly available AWS-managed component** that you attach to your VPC. A few architectural facts that matter:

It is **not a single device**. You don't get a virtual router or VM that acts as a gateway. IGW is a logical construct — a fleet of AWS-managed infrastructure that handles traffic in and out of your VPC at massive scale. AWS guarantees its availability; there's no single point of failure, no bandwidth bottleneck, and no throughput limit you need to configure. This is fundamentally different from on-premises networking where your internet router is a physical device with a specific throughput rating.

It is **attached to the VPC, not to individual subnets**. One IGW serves the entire VPC. All public subnets in all AZs within that VPC use the same IGW. You don't create separate IGWs per AZ for resilience — that's handled internally by AWS.

It has a **one-to-one relationship with a VPC**. You cannot attach one IGW to multiple VPCs. You cannot attach multiple IGWs to one VPC. It's always exactly one IGW per VPC.

It is **free**. The IGW itself carries no hourly charge and no per-GB charge for inbound traffic. You pay for outbound data transfer from EC2 instances — but that billing shows up on your EC2 bill, not on an IGW line item.

---

## Route Tables — The Brain of VPC Networking

If the Internet Gateway is the door, the route table is the directory that tells traffic which door to use. Every packet that travels within or out of your VPC has its destination IP looked up against a route table to determine where it should go next.

### How Route Table Lookup Works

Route tables contain **routes**, and each route has two parts: a **destination** (a CIDR block) and a **target** (where to send traffic matching that destination). When a packet needs routing, AWS performs a **longest prefix match** — the most specific matching route wins.

```
Route Table for Public Subnet:
┌──────────────────┬────────────────────────┬────────────────────────────┐
│ Destination      │ Target                 │ Meaning                    │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ 10.0.0.0/16      │ local                  │ All VPC-internal traffic   │
│ 0.0.0.0/0        │ igw-0a1b2c3d4e5f6a7b  │ Everything else → internet │
└──────────────────┴────────────────────────┴────────────────────────────┘
```

If an EC2 instance sends a packet to `10.0.11.25` (another instance in a private subnet), it matches `10.0.0.0/16` (the more specific route) — target is `local`, so it stays within the VPC. If it sends a packet to `8.8.8.8` (Google's DNS), it doesn't match `10.0.0.0/16`, so it falls through to `0.0.0.0/0` — target is the IGW, so it exits through the internet gateway.

This longest prefix match behavior is the same as how routing works throughout the internet — the most specific route always wins.

### The `local` Route — Immovable Foundation

Every route table in every VPC always contains a route like `10.0.0.0/16 → local` (where `10.0.0.0/16` is your VPC's CIDR). This route is **automatically created, cannot be deleted, and cannot be modified**. It is AWS's guarantee that resources within the same VPC can always reach each other, regardless of what else you configure.

The `local` target means "route this within the VPC using the VPC's internal router." There's no device you manage — AWS handles intra-VPC routing completely. This is why an EC2 instance in a public subnet can talk to an RDS instance in a private subnet with no additional configuration — they're both in the same VPC, the `local` route handles it.

### Main Route Table vs Custom Route Tables

When you create a VPC, AWS automatically creates a **main route table** for that VPC. Every subnet you create is **implicitly associated** with the main route table unless you explicitly associate it with a different one.

This default behavior creates a significant security risk that most beginners fall into. If you add an IGW route to the main route table to make your public subnet work, every subnet in your VPC — including your private subnets and database subnets — now has an internet route. Everything becomes public.

The correct approach:

```
Main Route Table (keep it restrictive — it's the fallback):
┌──────────────────┬────────┐
│ 10.0.0.0/16      │ local  │
└──────────────────┴────────┘

Custom Route Table: "public-rt"
┌──────────────────┬───────────────────┐
│ 10.0.0.0/16      │ local             │
│ 0.0.0.0/0        │ igw-xxxxxxxxx     │
└──────────────────┴───────────────────┘
(Explicitly associated with: public-subnet-1a, public-subnet-1b)

Custom Route Table: "private-rt"
┌──────────────────┬───────────────────┐
│ 10.0.0.0/16      │ local             │
│ 0.0.0.0/0        │ nat-xxxxxxxxx     │  ← NAT Gateway (covered next)
└──────────────────┴───────────────────┘
(Explicitly associated with: private-subnet-1a, private-subnet-1b)

Custom Route Table: "db-rt"
┌──────────────────┬────────┐
│ 10.0.0.0/16      │ local  │
└──────────────────┴────────┘
(Explicitly associated with: db-subnet-1a, db-subnet-1b)
```

This architecture means your database subnets have zero internet route — no outbound internet, no inbound internet, full stop. Private subnets have outbound-only internet via NAT Gateway (covered in the next part). Only public subnets have a full IGW route.

> ⚠️ **Interview Trap:** "What happens if you don't explicitly associate a subnet with a route table?" It automatically uses the main route table. This is why best practice is to keep the main route table minimal (just the `local` route) and create explicit custom route tables for each tier. Never put internet routes in the main route table.

---

## The Complete Internet Access Checklist

This is the definitive checklist. All of the following must be true for an EC2 instance to be reachable from the internet. Missing any single item and access fails — this is a very common troubleshooting scenario:

```
Internet Access Checklist for EC2 Instance
─────────────────────────────────────────────────────────
□ 1. VPC has an Internet Gateway attached
□ 2. Subnet's route table has 0.0.0.0/0 → IGW route
□ 3. EC2 instance has a public IP or Elastic IP assigned
□ 4. Security Group allows inbound on required port (e.g., 22 for SSH)
□ 5. NACL (if customized) allows inbound and outbound on required port
□ 6. OS-level firewall (iptables/firewalld) is not blocking the port
─────────────────────────────────────────────────────────
```

In a troubleshooting scenario, go through this list top to bottom. The most common failures are items 1, 2, and 3 for custom VPCs (people forget to attach the IGW or forget to update the route table) and item 4 for default VPCs (security groups blocking access).

---

## Intra-VPC Communication — No IGW Needed

An important thing the video demonstrates but doesn't fully explain mechanistically: two EC2 instances in different subnets of the same VPC can talk to each other using private IPs, even if neither subnet has an IGW route and neither instance has a public IP.

This works because of the `local` route. All traffic between `10.0.x.x` addresses stays within the VPC router — it never touches the IGW. The only gate controlling this communication is the security group on each instance.

```
EC2 in public-subnet-1a (10.0.1.15)
         │
         │  ping 10.0.11.25  (private IP)
         │
         ▼
Route Table lookup: 10.0.11.25 matches 10.0.0.0/16 → local
         │
         ▼
VPC internal router delivers packet to:
EC2 in private-subnet-1a (10.0.11.25)
         │
         ▼
Security Group on 10.0.11.25: Does it allow ICMP from 10.0.1.15?
         │
    Yes → ping succeeds
    No  → ping fails (security group drops silently)
```

This is also why security groups are so important within a VPC. VPC isolation protects you from other AWS customers. Security groups protect you from other resources within your own VPC.

---

## IGW Pricing and Data Transfer — The Real Costs

The IGW component itself is free. But understanding the actual data transfer costs around it matters for production cost management:

**Inbound data transfer (internet → EC2 via IGW):** Free. Data flowing into AWS from the internet carries no charge.

**Outbound data transfer (EC2 → internet via IGW):** Charged. Rates vary by region. In Mumbai (`ap-south-1`), outbound data transfer is charged per GB after the first 100GB free tier per month. This is where video streaming companies, CDN origin servers, and backup services rack up significant AWS bills.

**Inter-AZ traffic within a VPC:** Charged at $0.01/GB in both directions. This matters — if your app server in AZ-a frequently queries a database in AZ-b, you're paying for every GB of that traffic. This is one reason to co-locate tightly coupled services in the same AZ, and use Multi-AZ only for standby/failover rather than active-active cross-AZ queries.

**Intra-AZ traffic (same AZ, same VPC):** Free. Traffic between instances in the same AZ using private IPs carries no data transfer charge.

---

## Hands-On: Full Setup Sequence

Here's the complete CLI sequence to set up a VPC with a working public subnet — doing it via CLI teaches you the dependency chain clearly:

```bash
# Step 1: Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'
# → Returns VpcId: vpc-xxxxxxxx

# Step 2: Create public subnet in AZ-a
aws ec2 create-subnet \
  --vpc-id vpc-xxxxxxxx \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1a}]'
# → Returns SubnetId: subnet-aaaaaaaa

# Step 3: Enable auto-assign public IP on the subnet
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-aaaaaaaa \
  --map-public-ip-on-launch

# Step 4: Create Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]'
# → Returns InternetGatewayId: igw-xxxxxxxx

# Step 5: Attach IGW to VPC  ← this step is forgotten most often
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-xxxxxxxx \
  --vpc-id vpc-xxxxxxxx

# Step 6: Create a custom route table (don't pollute the main one)
aws ec2 create-route-table \
  --vpc-id vpc-xxxxxxxx \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'
# → Returns RouteTableId: rtb-xxxxxxxx

# Step 7: Add internet route to the custom route table
aws ec2 create-route \
  --route-table-id rtb-xxxxxxxx \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-xxxxxxxx

# Step 8: Associate the route table with the public subnet
aws ec2 associate-route-table \
  --route-table-id rtb-xxxxxxxx \
  --subnet-id subnet-aaaaaaaa
```

Notice step 5 — attaching the IGW to the VPC. Creating an IGW is not the same as attaching it. A newly created IGW sits in a "detached" state. This is one of the most common mistakes when building custom VPCs: creating the IGW and then wondering why the route `0.0.0.0/0 → igw-xxx` doesn't work. The IGW must be attached to the VPC before it can route any traffic.

---

## Architecture Summary — What We've Built So Far

```
VPC: 10.0.0.0/16
│
├── Internet Gateway (prod-igw)    ← attached to VPC
│
├── Route Table: public-rt         ← 0.0.0.0/0 → igw, 10.0.0.0/16 → local
│   └── Associated with: public-subnet-1a, public-subnet-1b
│
├── Route Table: main (untouched)  ← only local route
│   └── Implicitly used by: private subnets, db subnets
│
├── public-subnet-1a (10.0.1.0/24, AZ-a)
│   └── EC2: web server (private: 10.0.1.15, public: 13.x.x.x)
│
└── private-subnet-1a (10.0.11.0/24, AZ-a)
    └── EC2: app server (private: 10.0.11.25, NO public IP)
```

The web server is internet-accessible. The app server is not — it has no public IP and its subnet has no IGW route. But the app server can receive connections from the web server via private IP, because the `local` route handles intra-VPC traffic. This is exactly the security boundary you want.

---

## Comparison Table: Route Table Targets

| Target | Used For | Direction |
|---|---|---|
| `local` | Intra-VPC traffic | Both |
| `igw-xxx` | Internet access (public subnets) | Both (inbound + outbound) |
| `nat-xxx` | Outbound-only internet (private subnets) | Outbound only |
| `vgw-xxx` | VPN Gateway (on-prem connectivity) | Both |
| `tgw-xxx` | Transit Gateway (multi-VPC routing) | Both |
| `pcx-xxx` | VPC Peering connection | Both |
| `vpce-xxx` | VPC Endpoint (private AWS service access) | Both |
| `eni-xxx` | Network Interface (e.g., NVA/firewall) | Both |

---

## Interview Flags for This Topic

> ⚠️ **"Can you attach one IGW to multiple VPCs?"** No. One IGW per VPC, always.

> ⚠️ **"Is the IGW a single point of failure?"** No. It's a managed, horizontally scaled AWS service. AWS guarantees its availability — you don't need redundancy planning for the IGW itself.

> ⚠️ **"Does an EC2 instance know its own public IP?"** No. The OS only knows the private IP. Public IP is maintained by the IGW via NAT. Use instance metadata to retrieve the public IP programmatically.

> ⚠️ **"If I create an IGW but don't add a route to the route table, will internet traffic work?"** No. IGW creation + attachment is necessary but not sufficient. The route table must have `0.0.0.0/0 → igw-xxx`.

> ⚠️ **"What's the difference between attaching an IGW to a VPC vs associating a route table with a subnet?"** Attaching IGW makes it available to the VPC. Route table association + route determines which subnets actually use it. Both steps are independently required.

> ⚠️ **"Is outbound traffic from EC2 to the internet free?"** No. Inbound is free. Outbound is charged per GB (after free tier). This is one of AWS's primary data transfer revenue sources.

---

The next topic in your series should be **NAT Gateway** — which answers the critical question: how do private subnet instances (your app servers and databases) reach the internet for outbound traffic (package updates, API calls) without being reachable from the internet themselves? That's the counterpart to this topic and completes the public/private subnet networking model.
