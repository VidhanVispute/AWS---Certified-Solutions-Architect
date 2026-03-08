# AWS VPC — Part 5: Internet Gateway — Deep Dive

---

## Reframing the IGW — Beyond "It Connects You to the Internet"

Most explanations of the Internet Gateway stop at "it's the door to the internet." That's technically true but architecturally shallow. To use the IGW correctly, design around it intelligently, and answer interview questions about it precisely, you need to understand it at a deeper level — what it actually does to packets, where it sits in the AWS network stack, what its guarantees are, and crucially, what it doesn't do.

The Internet Gateway sits at the boundary between two fundamentally different networking worlds: **AWS's internal SDN fabric** (where your VPC's private IP addresses live) and **the public internet** (where only globally routable public IPs exist). Its job is to bridge these two worlds, and the mechanism it uses to do that is **static one-to-one NAT** — but only half the time. Let's unpack that precisely.

---

## What IGW Does to Packets — The NAT Mechanics

When an EC2 instance in a public subnet sends a packet outbound to the internet, the packet leaves the instance with:
- Source IP: the instance's **private IP** (e.g., `10.0.1.15`)
- Destination IP: the internet host (e.g., `8.8.8.8`)

The private IP `10.0.1.15` is meaningless on the public internet — no router outside AWS knows how to route back to it. So before the packet exits AWS's network, the IGW rewrites the source IP from `10.0.1.15` to the instance's **public IP or Elastic IP** (e.g., `13.233.45.67`). This rewrite is **stateful** — the IGW records the mapping so it can reverse it when the response arrives.

```
Outbound (EC2 → Internet):
  EC2 sends:    src=10.0.1.15,    dst=8.8.8.8
  IGW rewrites: src=13.233.45.67, dst=8.8.8.8  ← public IP replaces private
  Exits to internet

Inbound (Internet → EC2):
  Arrives at:   dst=13.233.45.67, src=8.8.8.8
  IGW rewrites: dst=10.0.1.15,    src=8.8.8.8  ← private IP restored
  Delivered to EC2 instance
```

This is **one-to-one NAT**: one public IP maps to exactly one private IP. This is different from NAT Gateway (many-to-one NAT) — with IGW, each instance that needs internet access needs its own public IP or Elastic IP. Two instances cannot share the same public IP through an IGW.

### The Critical Asymmetry — Why EC2 Doesn't See Its Own Public IP

The NAT translation happens at the IGW, which is entirely outside the EC2 instance's network stack. The instance's OS, the application running on it, and the kernel network interfaces all only ever see the private IP. The public IP doesn't appear on any network interface inside the VM.

This has real practical consequences that trip people up regularly:

```bash
# On the EC2 instance — this shows ONLY the private IP
ip addr show eth0
# Output: inet 10.0.1.15/24

# To get the public IP, you must query instance metadata
curl http://169.254.169.254/latest/meta-data/public-ipv4
# Output: 13.233.45.67
```

If your application needs to know its own public IP — for example, a game server registering itself with a matchmaking service, or a web service generating callback URLs — it must fetch this from the metadata service. Hardcoding the public IP in config files is also wrong because public IPs (non-Elastic) change every time the instance stops and starts.

> ⚠️ **Interview Trap:** "An application on EC2 is trying to bind to its public IP and failing. Why?" Because the public IP doesn't exist on any network interface inside the instance. The IGW holds the public IP, not the instance. Applications must bind to the private IP (or `0.0.0.0`).

---

## IGW as a Horizontally Scaled AWS Service — Architectural Guarantees

The IGW is not a virtual machine or an EC2 instance that AWS manages behind the scenes. It is a **distributed, software-defined networking component** that is part of AWS's underlying network fabric. This has specific architectural implications:

**There is no bandwidth limit to configure or manage.** Unlike a NAT Gateway where you might worry about exceeding throughput, an IGW scales to whatever traffic your instances generate. AWS absorbs this at the infrastructure level. This is why IGW is free — it's not a dedicated resource; it's woven into AWS's existing infrastructure.

**There is no availability concern for the IGW itself.** You don't need two IGWs for redundancy. The IGW is an AWS-managed construct — its availability is tied to the VPC's availability, not to any single underlying device. An AZ outage does not affect the IGW's ability to serve traffic for instances in surviving AZs.

**The one-to-one VPC constraint is a policy constraint, not a technical one.** AWS enforces that one IGW can only attach to one VPC and one VPC can only have one IGW. This is by design — the IGW's NAT table is scoped to the VPC's address space. Allowing one IGW to serve multiple VPCs would create ambiguity in routing since different VPCs can have overlapping CIDR ranges.

---

## IGW State Machine — Detached vs Attached

The IGW has two states, and this is a very common source of confusion when building custom VPCs:

```
IGW Lifecycle:

Created → State: DETACHED
              │
              │  attach-internet-gateway --vpc-id
              ▼
         State: ATTACHED (to a specific VPC)
              │
              │  detach-internet-gateway
              ▼
         State: DETACHED (can be reattached to same or different VPC)
              │
              │  delete-internet-gateway
              ▼
         Deleted
```

An IGW in the DETACHED state does absolutely nothing, even if your route table has a route pointing `0.0.0.0/0 → igw-xxxxxxxx`. The route exists but the target cannot fulfill it. Traffic will be blackholed. This is one of the most common debugging scenarios in custom VPC setups — everything looks correct on paper, routes are right, security groups are right, but nothing works because the IGW was created but never attached.

```bash
# Check IGW attachment state
aws ec2 describe-internet-gateways --internet-gateway-ids igw-xxxxxxxx
# Look for: "Attachments": [{"State": "available", "VpcId": "vpc-xxxxxxxx"}]
# If Attachments is empty → IGW is detached
```

> ⚠️ **Interview Trap:** "You've created an IGW, added a route `0.0.0.0/0 → igw-xxx` to the route table, and assigned a public IP to the instance. The instance is still unreachable. What's the most likely cause?" The IGW is not attached to the VPC.

---

## The Five-Gate Model — All Conditions for Internet Access

This is the complete mental model for internet accessibility in AWS. All five must be satisfied simultaneously. This is the most interview-relevant synthesis of VPC knowledge up to this point:

```
Gate 1: Does the VPC have an Internet Gateway ATTACHED?
        └── If no → no internet possible for any resource in the VPC

Gate 2: Does the subnet's route table have 0.0.0.0/0 → IGW?
        └── If no → traffic from this subnet cannot reach IGW

Gate 3: Does the EC2 instance have a public IP or Elastic IP?
        └── If no → IGW cannot perform NAT (nothing to translate to/from)
        └── Note: inbound packets addressed to what IP?

Gate 4: Does the Security Group allow the relevant inbound/outbound traffic?
        └── Stateful — if inbound is allowed, response is automatically allowed

Gate 5: Does the NACL allow the relevant traffic?
        └── Stateless — must explicitly allow BOTH inbound AND outbound
```

In troubleshooting order, check them top to bottom. Gates 1 and 2 are the most commonly missed in custom VPC setups. Gate 3 is often forgotten because in the default VPC, auto-assign public IP is on by default, so people don't realize it needs to be explicitly enabled in custom VPCs. Gates 4 and 5 are the security layer — Gate 4 is almost always the issue in default VPCs where the IGW is already set up but someone locked down a security group too aggressively.

---

## Public IP vs Elastic IP — The IGW Perspective

From the IGW's perspective, both a regular public IP and an Elastic IP work identically — both are public IPs that get NAT-translated. The difference is in lifecycle and control:

**Regular public IP (auto-assigned):**
- AWS assigns it from AWS's IP pool at instance launch
- Not associated with your account — it belongs to AWS
- Released back to AWS's pool when the instance is stopped or terminated
- A restarted instance gets a completely different public IP
- Free

**Elastic IP (EIP):**
- You explicitly allocate it to your account — it becomes yours until you release it
- Persists across instance stop/start/reboot
- Can be remapped to a different instance in seconds (useful for failover)
- Associated with your account's billing even when not attached to anything
- Charged when NOT associated with a running instance (~$0.005/hour when idle — AWS discourages hoarding unused EIPs)

```
Regular Public IP lifecycle:
Instance Running  → has public IP 13.233.45.67
Instance Stopped  → public IP released, gone
Instance Started  → gets new public IP 52.66.12.34 (different)

Elastic IP lifecycle:
Allocate EIP      → 13.233.100.50 belongs to your account
Associate to EC2  → instance uses 13.233.100.50
Instance Stopped  → EIP stays associated, still yours
Instance Started  → still uses 13.233.100.50
Remap to new EC2  → new instance now uses 13.233.100.50
Release EIP       → 13.233.100.50 returned to AWS pool
```

For any resource that external systems need to reach at a stable IP — a game server, a mail server, an API endpoint not behind a load balancer, a VPN endpoint — you must use an Elastic IP. For dev/test instances or anything behind a load balancer (where the load balancer has the stable IP), regular public IPs are fine.

> ⚠️ **Interview Trap:** "An EC2 instance is running a public-facing service. It gets stopped for maintenance and restarted. Users report they can no longer reach it via IP. Why?" The regular public IP changed on restart. Solution: allocate and associate an Elastic IP.

---

## IGW and IPv6 — A Different Model

IPv6 is worth mentioning here because the IGW's behavior with IPv6 is fundamentally different from IPv4, and this trips people up.

With **IPv4**, the IGW performs NAT — it translates between private internal IPs and public external IPs. This translation is necessary because private IPs are not globally routable.

With **IPv6**, there is no NAT. IPv6 addresses are globally unique and globally routable by design. When you enable IPv6 on a VPC and subnet, each EC2 instance gets a globally routable IPv6 address directly on its network interface — the OS sees it, applications can bind to it. The IGW routes IPv6 traffic without any address translation. This is architecturally cleaner but means every IPv6-enabled instance is directly reachable by its IPv6 address from anywhere on the internet, subject only to security groups and NACLs. If you need IPv6 outbound-only (like NAT GW for IPv4), you need an **Egress-Only Internet Gateway** — a separate AWS component specifically for IPv6 outbound-only traffic from private subnets.

---

## What IGW Cannot Do — Important Boundaries

Understanding the limitations of IGW prevents architectural mistakes:

**IGW cannot connect two VPCs.** Traffic between VPCs goes via VPC Peering or Transit Gateway, not through the internet via IGW (that would be insecure and slow).

**IGW cannot connect your VPC to your on-premises network.** That requires a Virtual Private Gateway (VGW) for VPN, or Direct Connect.

**IGW cannot provide outbound-only internet access for private subnets.** That's NAT Gateway's job. If you add an IGW route to a private subnet's route table, it becomes a public subnet — both inbound and outbound internet access are now possible.

**IGW cannot be used as a target for VPC Peering traffic.** You cannot route peering traffic through an IGW.

**IGW is not a firewall.** It performs no traffic inspection, no content filtering, no DDoS mitigation. For those capabilities you need AWS WAF, AWS Shield, or third-party network appliances.

---

## Putting It All Together — The Complete VPC Topology So Far

```
Region: ap-south-1
VPC: 192.168.0.0/16
│
├── Internet Gateway: prod-igw [ATTACHED]
│
├── Route Table: public-rt
│   ├── 192.168.0.0/16 → local
│   └── 0.0.0.0/0      → prod-igw
│   └── Associated: public-subnet-1a, public-subnet-1b
│
├── Route Table: private-rt-1a
│   ├── 192.168.0.0/16 → local
│   └── 0.0.0.0/0      → nat-gw-1a
│   └── Associated: private-subnet-1a
│
├── Route Table: private-rt-1b
│   ├── 192.168.0.0/16 → local
│   └── 0.0.0.0/0      → nat-gw-1b
│   └── Associated: private-subnet-1b
│
├── PUBLIC SUBNETS
│   ├── public-subnet-1a (192.168.1.0/24, AZ-a)
│   │   ├── NAT Gateway 1a (EIP: 13.233.x.x)
│   │   └── Bastion Host / ALB node
│   └── public-subnet-1b (192.168.2.0/24, AZ-b)
│       └── ALB node
│
└── PRIVATE SUBNETS
    ├── private-subnet-1a (192.168.11.0/24, AZ-a)
    │   └── App Server / DB (no public IP)
    └── private-subnet-1b (192.168.12.0/24, AZ-b)
        └── App Server / DB (no public IP)
```

---

## Comparison Table: IGW vs NAT GW vs Egress-Only IGW

| Dimension | Internet Gateway | NAT Gateway | Egress-Only IGW |
|---|---|---|---|
| **IP version** | IPv4 + IPv6 | IPv4 only | IPv6 only |
| **Direction** | Inbound + outbound | Outbound only | Outbound only |
| **NAT performed** | Yes (IPv4), No (IPv6) | Yes (many-to-one) | No (IPv6 is globally routable) |
| **Used by** | Public subnet instances | Private subnet instances | IPv6 private instances |
| **Cost** | Free | Hourly + per-GB | Free |
| **Requires EIP** | On the instance | On the NAT GW itself | No |
| **Attached to** | VPC | Specific AZ subnet | VPC |
| **AZ redundancy** | VPC-level (no concern) | One per AZ needed | VPC-level |

---

## Interview Flags for This Topic

> ⚠️ **"Does the IGW perform NAT for IPv6 traffic?"** No. IPv6 addresses are globally routable — no translation needed. IGW routes IPv6 without NAT. The instance OS sees its IPv6 address directly.

> ⚠️ **"Can you have two IGWs attached to one VPC?"** No. AWS enforces a one-to-one relationship. Attempting to attach a second IGW returns an error.

> ⚠️ **"An instance has a public IP, is in a public subnet, and has permissive security groups, but can't reach the internet. What do you check?"** IGW attachment state first, then route table association and routes, then whether the subnet has auto-assign public IP enabled.

> ⚠️ **"What happens to an EC2 instance's public IP when it's stopped?"** Regular public IPs are released — the instance will get a different IP on next start. Elastic IPs persist. This is a critical operational difference.

> ⚠️ **"Is Elastic IP free?"** Free when associated with a running instance. Charged when allocated but not associated with a running instance.

> ⚠️ **"Can you route traffic between two VPCs through the IGW?"** No. IGW is for VPC-to-internet traffic only. VPC-to-VPC traffic uses Peering or Transit Gateway.

---

The natural next topic in your series is the **Bastion Host** — the solution to the problem raised at the end of the video: "How do you SSH into private subnet instances that have no public IP and no direct internet access?" It's the operational pattern that completes the two-tier architecture. Drop it whenever you're ready.
