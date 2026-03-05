# AWS EC2 Security Groups — Deep Dive (Part 2)

---

## Quick Recap of Part 1

Before going deeper, here's the foundation from Part 1 in one block:

- Security Group = virtual firewall attached to an EC2 instance
- Allow rules only — no deny rules exist
- Default: all inbound blocked, all outbound allowed
- Filters based on port numbers, protocols, and IP addresses
- Used by every EC2 instance — mandatory

Now Part 2 goes deeper into **how** it actually works at the network level.

---

## The NIC — Where Security Groups Actually Live

To truly understand Security Groups, you need to understand where they attach. A Security Group does not attach to an EC2 instance directly in the technical sense. It attaches to the instance's **Network Interface Card (NIC)** — called an **ENI (Elastic Network Interface)** in AWS.

### What is an ENI?

An **Elastic Network Interface** is a virtual network card. Every EC2 instance has at least one ENI — the primary network interface — and it can have additional ENIs attached.

The ENI is what gives your instance its identity on the network:
- It holds the **private IP address**
- It holds the **public IP address** (if assigned)
- It holds the **MAC address**
- It is where the **Security Group is attached**
- It is the physical point where all traffic enters and exits

```
Internet / AWS Network
         ↓
    ┌─────────────┐
    │    ENI      │  ← Security Group lives HERE
    │  (NIC)      │  ← Private IP: 172.31.10.5
    │             │  ← Public IP:  54.123.45.67
    └─────────────┘
         ↓
    ┌─────────────┐
    │  EC2        │
    │  Instance   │
    └─────────────┘
```

Every single packet of traffic — whether coming in or going out — must pass through the ENI. Because the Security Group is attached to the ENI, it intercepts and evaluates **every packet** before it reaches the instance's operating system.

### Why This Matters

Because the Security Group sits at the ENI level:

**Blocked traffic never reaches the OS.** If a packet is blocked by the Security Group, the Linux or Windows kernel inside your EC2 instance never even sees it. The EC2 instance has no idea the packet was ever sent. This is why blocked traffic doesn't consume CPU or generate error logs inside your server — it's dropped before it gets that far.

**ENIs can be detached and moved.** Because the Security Group belongs to the ENI — not the instance — if you detach an ENI from one instance and attach it to another, the Security Group rules travel with the ENI. The new instance immediately inherits the same network identity and security rules.

**Multiple ENIs = multiple Security Groups.** If an instance has two ENIs, each ENI can have its own Security Group. Traffic coming through ENI-1 is filtered by SG-1. Traffic coming through ENI-2 is filtered by SG-2. This allows very granular traffic control for instances that serve multiple roles.

---

## Stateful — The Most Important Security Group Property

This is the concept the video builds toward, and it is critical to understand correctly.

### What Does Stateful Mean?

A **stateful** firewall keeps track of active network connections. When it allows a packet in one direction, it remembers that connection and automatically allows the reply in the opposite direction — without requiring an explicit rule for the return traffic.

A **stateless** firewall treats every packet independently. It evaluates every single packet against the rules, with no memory of previous packets. Return traffic must be explicitly allowed by a separate rule.

Security Groups are **stateful**. Network ACLs are **stateless**. This is the most important difference between the two.

### How Stateful Works — Step by Step

**Scenario: A user visits your website (HTTP on port 80)**

```
User's Browser                          Your EC2 Web Server
IP: 203.0.113.50                        IP: 54.123.45.67
Ephemeral Port: 52841                   Listening Port: 80
```

**Step 1 — Inbound request arrives at ENI:**
```
Packet: FROM 203.0.113.50:52841  →  TO 54.123.45.67:80

Security Group checks inbound rules:
  Rule: Allow TCP port 80 from 0.0.0.0/0  ✓ MATCH
  Decision: ALLOW
  Action: Packet passes through ENI to the EC2 instance
  
  State table entry created:
  203.0.113.50:52841 ↔ 54.123.45.67:80 = ESTABLISHED
```

**Step 2 — Server processes the request and sends a response:**
```
Packet: FROM 54.123.45.67:80  →  TO 203.0.113.50:52841

Security Group checks the state table first:
  Found: 203.0.113.50:52841 ↔ 54.123.45.67:80 = ESTABLISHED
  This is return traffic for an allowed connection
  Decision: AUTOMATICALLY ALLOW (no outbound rule checked)
  
The user's browser receives the webpage
```

The outbound security group rules are **not checked** for return traffic of established inbound connections. The state table entry is enough.

### Why This Is Powerful

Without stateful behavior, you would need two rules for every service:

```
WITHOUT stateful (hypothetical):
  Inbound:  Allow TCP port 80 from 0.0.0.0/0
  Outbound: Allow TCP ports 49152-65535 to 0.0.0.0/0  ← would need this too

WITH stateful (actual Security Groups):
  Inbound:  Allow TCP port 80 from 0.0.0.0/0
  (Nothing else needed — return traffic automatic)
```

The ephemeral port range (49152–65535) is unpredictable — the OS picks a random port. Without stateful tracking, you'd have to open a huge port range outbound, which weakens security. Stateful tracking solves this elegantly.

### Stateful Works Both Directions

The statefulness works for both inbound-initiated and outbound-initiated connections:

**Outbound-initiated (e.g., your server calling an external API):**

```
Your EC2 Server                         External API
IP: 172.31.10.5                         IP: api.example.com
Ephemeral Port: 54321                   Port: 443 (HTTPS)

Outbound packet: FROM 172.31.10.5:54321 → TO api.example.com:443

Security Group checks outbound rules:
  Default: Allow all outbound  ✓ MATCH
  State table entry created

API response: FROM api.example.com:443 → TO 172.31.10.5:54321

Security Group checks state table:
  Found established connection
  Inbound return packet AUTOMATICALLY ALLOWED
  (Even though no inbound rule for port 443 exists)
```

Your server can call any external service and receive the response — even with zero inbound rules — because responses to outbound connections are automatically allowed by stateful tracking.

---

## The Default Security Group — Do Not Delete or Modify

Every AWS account comes with a **Default Security Group** in each VPC. It exists from the moment you create your account and cannot be deleted (AWS prevents this).

### Default Security Group Rules

**Inbound rules (default):**
```
Type        Protocol  Port  Source
─────────────────────────────────────────────────────
All traffic  All      All   sg-xxxxxxxx (itself)
```

This single inbound rule says: allow all traffic from any instance that also has this Default Security Group attached. Instances sharing the Default Security Group can communicate freely with each other on any port.

**Outbound rules (default):**
```
Type        Protocol  Port  Destination
─────────────────────────────────────────
All traffic  All      All   0.0.0.0/0
```

All outbound traffic to anywhere is allowed.

### What This Means in Practice

If you launch two EC2 instances and both use the Default Security Group:
- They can talk to each other on any port (covered by the self-referencing inbound rule)
- Both can reach the internet outbound
- Nothing from the internet can reach them inbound (no rule allows external traffic)

### Why You Should Not Modify the Default Security Group

The Default Security Group is AWS's baseline safety net. If you start removing or changing its rules without understanding the consequences:
- Services that depend on instance-to-instance communication within the VPC may break
- Other resources (RDS, Lambda, ElastiCache) that AWS attaches the Default SG to by default may lose connectivity
- You may lock yourself out of instances

Best practice: **leave the Default Security Group alone.** Create dedicated, purpose-built Security Groups for your resources with precisely the rules they need.

---

## Security Group State Table — How Connection Tracking Works Internally

The state table is an in-memory table maintained by the AWS networking layer at the ENI level. Each entry records:

```
Source IP : Source Port  ↔  Dest IP : Dest Port  |  Protocol  |  State  |  Timeout
──────────────────────────────────────────────────────────────────────────────────────
203.0.113.50:52841  ↔  172.31.10.5:80    TCP    ESTABLISHED   300s
10.0.1.5:43210      ↔  172.31.10.5:3306  TCP    ESTABLISHED   3600s
8.8.8.8:53          ↔  172.31.10.5:54321 UDP    TRACKED       30s
```

**TCP connections** are tracked through their full lifecycle — SYN, ESTABLISHED, FIN/RST, CLOSED. When the connection closes, the entry is removed.

**UDP** has no connection concept, so the state table tracks UDP flows by the 5-tuple (source IP, source port, dest IP, dest port, protocol) and uses a timeout — if no matching packet arrives within the timeout window, the entry expires.

**ICMP** is tracked by type and code — an ICMP Echo Reply is automatically allowed back for an ICMP Echo Request.

---

## Port Filtering — How Rules Are Evaluated

When a packet arrives at the ENI, here is the exact evaluation sequence:

```
Packet arrives at ENI
         ↓
Is this return traffic for an existing state table entry?
  YES → ALLOW immediately (stateful bypass)
  NO  → Continue to rule evaluation
         ↓
Check all inbound rules simultaneously (not in order)
         ↓
Does ANY rule match this packet's protocol, port, and source IP?
  YES → ALLOW — packet passes to EC2 instance
  NO  → DENY — packet silently dropped, EC2 instance never sees it
```

**Key point:** Security Group rules are evaluated **all at once**, not in order. If you have 10 inbound rules, all 10 are checked simultaneously and if any one of them matches, the traffic is allowed. This is different from Network ACLs where rules are evaluated in numerical order and the first match wins.

This means there are no rule conflicts in Security Groups. More rules = more things allowed. You cannot accidentally block something with a later rule.

---

## Traffic Scenarios — Blocked vs Allowed

### Scenario 1 — Web Server (HTTP/HTTPS only)

```
Security Group: SG-WebServer
Inbound Rules:
  TCP  80   0.0.0.0/0    (HTTP)
  TCP  443  0.0.0.0/0    (HTTPS)

Traffic test:
  Request on port 80   → ALLOWED ✓
  Request on port 443  → ALLOWED ✓
  Request on port 22   → BLOCKED ✗ (no rule for SSH)
  Request on port 3306 → BLOCKED ✗ (no rule for MySQL)
  Request on port 8080 → BLOCKED ✗ (no rule for alternate HTTP)
  Ping (ICMP)          → BLOCKED ✗ (no ICMP rule)
```

### Scenario 2 — Database Server

```
Security Group: SG-Database
Inbound Rules:
  TCP  3306  SG-AppServer    (MySQL from app tier only)

Traffic test:
  Request from app server on 3306  → ALLOWED ✓
  Request from your laptop on 3306 → BLOCKED ✗ (not in SG-AppServer)
  Request from internet on 3306    → BLOCKED ✗ (not in SG-AppServer)
  Request on port 22 from anywhere → BLOCKED ✗ (no SSH rule)
```

### Scenario 3 — SSH Bastion Host

```
Security Group: SG-Bastion
Inbound Rules:
  TCP  22   203.0.113.5/32   (SSH from office IP only)

Traffic test:
  SSH from office IP (203.0.113.5) → ALLOWED ✓
  SSH from home IP (different)     → BLOCKED ✗
  SSH from internet (attacker)     → BLOCKED ✗
  HTTP on port 80 from anywhere    → BLOCKED ✗
```

### Scenario 4 — Port Not in Any Rule (Key Concept)

```
Security Group has rules for ports 80 and 443 only.

Someone sends a packet to port 5000.
  → No rule matches
  → Packet is SILENTLY DROPPED
  → The sender receives NO response (not even "connection refused")
  → The EC2 instance never sees the packet
  → No log entry on the server for this attempt
  → From the attacker's perspective: the port doesn't exist
```

This silent drop (called "blackholing") is intentional and is more secure than sending back a TCP RST (reset) or ICMP "port unreachable" — because those responses confirm to an attacker that a host exists at that IP.

---

## Multiple Security Groups on One Instance

A single EC2 instance can have up to **5 Security Groups** attached to the same ENI. The rules from all attached Security Groups are **merged** into one combined allow list.

```
Instance has SG-A and SG-B attached:

SG-A Inbound Rules:          SG-B Inbound Rules:
  TCP 80  from 0.0.0.0/0       TCP 22  from 10.0.0.0/8
  TCP 443 from 0.0.0.0/0       TCP 8080 from SG-LoadBalancer

Combined effective rules:
  TCP 80   from 0.0.0.0/0         (from SG-A)
  TCP 443  from 0.0.0.0/0         (from SG-A)
  TCP 22   from 10.0.0.0/8        (from SG-B)
  TCP 8080 from SG-LoadBalancer   (from SG-B)
```

All other traffic is still blocked. There is no "most restrictive wins" logic or conflict resolution — it's purely additive. Each Security Group contributes its allow rules to the combined set.

**Practical use:** You might have a base Security Group that all instances get (e.g., allows monitoring agent traffic and SSH from bastion), and then role-specific Security Groups (web server SG, database SG, etc.) added on top.

---

## Security Group Limits (AWS Default Limits)

| Limit | Default Value |
|---|---|
| Security Groups per VPC | 2,500 |
| Inbound rules per Security Group | 60 |
| Outbound rules per Security Group | 60 |
| Security Groups per ENI (network interface) | 5 |
| Total rules per ENI (SGs × rules) | 300 (60 rules × 5 SGs) |

These limits can be increased by submitting a service quota increase request to AWS, but they are rarely hit in normal architectures.

---

## Common Ports Quick Reference — Security Group Rules

| Rule Name | Port | Protocol | Typical Source | Purpose |
|---|---|---|---|---|
| SSH | 22 | TCP | Your IP only `/32` | Linux terminal access |
| RDP | 3389 | TCP | Your IP only `/32` | Windows desktop access |
| HTTP | 80 | TCP | `0.0.0.0/0` | Public web traffic |
| HTTPS | 443 | TCP | `0.0.0.0/0` | Secure web traffic |
| MySQL | 3306 | TCP | App server SG | Database queries |
| PostgreSQL | 5432 | TCP | App server SG | Database queries |
| Redis | 6379 | TCP | App server SG | Cache access |
| MongoDB | 27017 | TCP | App server SG | NoSQL database |
| HTTP Alt | 8080 | TCP | Load balancer SG | App server traffic |
| HTTPS Alt | 8443 | TCP | Load balancer SG | Encrypted app traffic |
| DNS | 53 | TCP/UDP | VPC CIDR | Name resolution |
| ICMP Ping | N/A | ICMP | VPC CIDR | Internal diagnostics |
| NFS/EFS | 2049 | TCP | VPC CIDR | File system mounting |
| SMTP | 25 | TCP | Specific mail IPs | Email sending |
| All VPC internal | All | All | VPC CIDR block | Internal communication |

---

## Everything at a Glance

| Concept | Detail |
|---|---|
| Attaches to | ENI (Elastic Network Interface) — not the instance directly |
| Traffic blocked | Never reaches the OS — dropped at the ENI level |
| Stateful | Return traffic for allowed connections is auto-allowed |
| State table | Tracks active connections by 5-tuple with timeouts |
| Rule type | Allow only — no deny rules possible |
| Rule evaluation | All rules checked simultaneously — any match = allow |
| Default inbound | All blocked (new SG) |
| Default outbound | All allowed (new SG) |
| Default SG | Self-referencing inbound — instances in same SG talk freely |
| Multiple SGs | Rules from all attached SGs are merged additively |
| Source options | CIDR, specific IP `/32`, another Security Group ID, My IP |
| Max SGs per ENI | 5 |
| Max rules per SG | 60 inbound + 60 outbound |
| Silent drop | Blocked packets dropped with no response to sender |
