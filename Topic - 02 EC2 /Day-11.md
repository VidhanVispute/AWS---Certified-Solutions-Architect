# AWS Security Groups — Practical Hands-On Guide (Part 4)

---

## What This Section Covers

Parts 1–3 built the theory. This part is entirely practical — walking through every step of creating Security Groups, attaching them to EC2 instances, testing them, and building a real multi-tier secure architecture inside the AWS console. Every action is explained with the reasoning behind it.

---

## Part 1 — Navigating to Security Groups in AWS Console

**Path:** AWS Console → EC2 → Network & Security → Security Groups

When you first land on this page with a fresh account, you will see exactly **one Security Group** — the **Default Security Group**. You cannot delete it. AWS prevents this because other services depend on it as a fallback.

Look at the Default Security Group's inbound rules:
```
Type        Protocol  Port Range  Source
─────────────────────────────────────────────────────
All traffic  All       All         sg-xxxxxxxx (itself)
```

This single self-referencing rule means: any EC2 instance that belongs to this same Default Security Group can freely talk to any other instance in it, on any port, using any protocol. Instances that don't share this Security Group cannot connect inbound at all.

---

## Part 2 — Creating a New Security Group from Scratch

Click **"Create security group"**. You'll see the following fields:

**Security group name** — Give it a meaningful name. Once created, the name cannot be changed, so choose carefully.
- Good: `web-server-sg`, `database-sg`, `bastion-sg`
- Bad: `sg1`, `test`, `my-sg` (tells you nothing 6 months later)

**Description** — Mandatory field. Write what this Security Group does and what it protects.
- Example: `Allows HTTP and HTTPS inbound for public web servers`

**VPC** — Select which VPC this Security Group belongs to. A Security Group is VPC-specific — it only works within the VPC you assign it to here. For most beginners this will be the Default VPC.

After creation, you will immediately see:

```
Inbound rules:   (empty — zero rules)
Outbound rules:  Allow all traffic → 0.0.0.0/0
```

**This is the most important default to memorize:**
- Nothing can reach your instance inbound
- Your instance can reach anything outbound
- This is the safest possible starting state — locked down inbound, open outbound

---

## Part 3 — Attaching a Security Group to an EC2 Instance

You can attach a Security Group to an EC2 instance in two ways:

### Method A — During Instance Launch

When launching an EC2 instance, in the **Network Settings** section:
- Select **"Select existing security group"**
- Choose your Security Group from the dropdown
- The instance launches with your Security Group already attached

### Method B — After Launch (on a Running Instance)

1. Go to EC2 → Instances → select your instance
2. Click **Actions → Security → Change security groups**
3. Add or remove Security Groups
4. Click **Save** — changes take effect **immediately** with no restart required

This is powerful — you can change security rules on a live, running instance without any downtime.

---

## Part 4 — The First Test: No Inbound Rules = No Access

**Scenario:** EC2 instance running a web server (nginx/Apache on port 80). Security Group has zero inbound rules.

Launch the instance, wait for it to reach `2/2 status checks`, copy the public IP, paste it into a browser:

```
Browser: http://54.123.45.67

Result: ⏳ Loading... Loading... Loading... TIMEOUT

The page never loads.
```

**Why?** The HTTP request (TCP port 80) arrives at the ENI. The Security Group checks inbound rules — finds nothing — silently drops the packet. Your web server is running perfectly fine inside the instance, but the Security Group is blocking the door completely.

This is the correct behavior. Your server is invisible to the internet.

---

## Part 5 — Adding the First Inbound Rule (HTTP Port 80)

Go to your Security Group → **Inbound rules** tab → **Edit inbound rules** → **Add rule**:

```
Type:       HTTP
Protocol:   TCP         ← auto-filled when you select HTTP
Port range: 80          ← auto-filled
Source:     Anywhere-IPv4 (0.0.0.0/0)
Description: Allow public HTTP web traffic
```

Click **Save rules**.

Now test again:
```
Browser: http://54.123.45.67

Result: ✅ Web page loads instantly
```

**What changed:** You added one allow rule. The Security Group now lets TCP port 80 traffic through. Everything else is still blocked. The web server was always running — you just opened the door.

**Cleanup test — remove the rule:**
Go back, delete the HTTP rule, save. Refresh the browser:
```
Result: ⏳ Timeout again
```

This proves the Security Group is the only thing controlling access. The rule controls access in real-time, with zero delay.

---

## Part 6 — Testing ICMP (Ping) — Stateful Behavior Proven Live

This is the most educational test in the whole practical. It directly proves stateful behavior.

### Test 1 — Ping From Outside to EC2 (No ICMP Rule)

From your laptop terminal:
```bash
ping 54.123.45.67

Result: Request timeout
         Request timeout
         Request timeout
```

The ping packets are being sent but are silently dropped by the Security Group. No ICMP rule = no response. Your EC2 instance is receiving the packets at the ENI, but they're dropped before reaching the OS.

### Test 2 — Ping From Inside EC2 to Outside (No ICMP Rule)

SSH into your EC2 instance and run:
```bash
ping 8.8.8.8

Result: 64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=1.23 ms
        64 bytes from 8.8.8.8: icmp_seq=2 ttl=118 time=1.19 ms
        ✅ Works perfectly
```

**Why does this work when Test 1 didn't?**

This is stateful behavior in action, demonstrated live:

```
Test 1 (External → EC2):
  Ping request arrives → NEW inbound connection
  Security Group checks rules → No ICMP rule → BLOCKED

Test 2 (EC2 → External):
  Ping request leaves EC2 → Matches outbound allow-all rule → ALLOWED
  State table entry created: {EC2 ↔ 8.8.8.8 | ICMP | ESTABLISHED}
  Ping reply arrives from 8.8.8.8 → RETURN TRAFFIC
  Security Group checks state table first → FOUND → AUTOMATICALLY ALLOWED
```

The ping reply in Test 2 is technically **inbound traffic** — it's coming from the internet into your EC2. But it's allowed because it's a reply to something your instance started. The state table handles it without needing an explicit inbound rule.

This is the live, hands-on proof of statefulness that all the theory was building toward.

### Test 3 — Add ICMP Inbound Rule, Test From Outside Again

Add this rule to your Security Group:
```
Type:       All ICMP - IPv4
Protocol:   ICMP
Port range: All
Source:     Anywhere-IPv4 (0.0.0.0/0)
Description: Allow ping from anywhere for diagnostics
```

Now from your laptop:
```bash
ping 54.123.45.67

Result: 64 bytes from 54.123.45.67: icmp_seq=1 ttl=64 time=45.2 ms
        64 bytes from 54.123.45.67: icmp_seq=2 ttl=64 time=44.8 ms
        ✅ Works now
```

**Why?** You added an explicit inbound rule for ICMP. Now new inbound ICMP (ping requests from the internet) are allowed. The Security Group now knows: if a new ICMP connection arrives from anywhere, let it through.

---

## Part 7 — The Multi-Tier Secure Architecture (The Real-World Pattern)

This is the practical that matters most for interviews and real AWS work. You're building a two-tier architecture where only web servers can access the database — nobody else.

### The Business Requirement

```
Internet users → Web Servers → Database Server

Rules:
  ✓ Anyone on internet can reach web servers (port 80)
  ✓ Web servers can reach the database (port 3306)
  ✗ Internet CANNOT directly reach the database
  ✗ Other EC2 instances CANNOT reach the database
  ✗ Your laptop CANNOT reach the database directly
```

### Step 1 — Create Web_SG (Web Server Security Group)

```
Security Group Name: Web_SG
Description: Security group for public-facing web servers

Inbound Rules:
  Type    Protocol  Port  Source          Description
  ──────────────────────────────────────────────────────────
  HTTP    TCP       80    0.0.0.0/0       Public web traffic
  HTTPS   TCP       443   0.0.0.0/0       Public secure web traffic
  SSH     TCP       22    YOUR-IP/32      Admin SSH access

Outbound Rules:
  All traffic → 0.0.0.0/0 (default — leave as is)
```

### Step 2 — Create DB_SG (Database Security Group)

```
Security Group Name: DB_SG
Description: Security group for MySQL database - allow only from web tier

Inbound Rules:
  Type         Protocol  Port  Source    Description
  ────────────────────────────────────────────────────────────────
  MySQL/Aurora  TCP      3306  Web_SG    Allow MySQL from web servers ONLY

(Source is the Security Group ID of Web_SG, not an IP address)

Outbound Rules:
  All traffic → 0.0.0.0/0 (default)
```

### Step 3 — Launch Instances with Correct Security Groups

**Web Server EC2:**
- Attach: `Web_SG`
- Has public IP: Yes
- Install nginx/Apache

**Database EC2:**
- Attach: `DB_SG`
- Has public IP: No (private subnet or just no public IP assigned)
- Install MySQL

### Step 4 — Test the Architecture

**Test A — Public web access (should work):**
```
Browser → http://web-server-public-ip:80
Result: ✅ Web page loads
```

**Test B — Web server connects to database (should work):**
```bash
# From inside the web server EC2:
mysql -h 172.31.20.8 -u admin -p
Result: ✅ Connected to MySQL successfully
```

Web server has `Web_SG` attached. DB_SG allows traffic from `Web_SG`. The connection is permitted.

**Test C — Direct internet access to database (should be blocked):**
```bash
# From your laptop:
mysql -h database-private-ip -u admin -p
Result: ❌ Cannot connect (no route — private IP not internet-routable)

# Even if database had a public IP:
mysql -h database-public-ip -u admin -p
Result: ❌ Connection timeout
```

Your laptop doesn't have `Web_SG` attached (it's not an EC2 instance). The DB_SG rule only allows traffic from instances with `Web_SG`. So your laptop — even if it knew the IP — is blocked.

**Test D — Another random EC2 instance tries to reach database (blocked):**
```bash
# From a different EC2 that uses a different Security Group:
mysql -h 172.31.20.8 -u admin -p
Result: ❌ Connection timeout
```

Only EC2 instances with `Web_SG` specifically attached can reach the database. Any other instance — regardless of its IP — is blocked.

---

## Part 8 — Why Security Group References Beat IP Whitelisting

The most important architectural decision in this setup is using **`Web_SG` as the source** in the database rule instead of an IP address. Here is why this matters enormously in production:

### The IP Whitelisting Problem

Imagine your web tier has 3 servers today, and you whitelist their IPs:

```
DB_SG Inbound (IP approach):
  MySQL TCP 3306 from 172.31.10.5/32   (Web Server 1)
  MySQL TCP 3306 from 172.31.10.6/32   (Web Server 2)
  MySQL TCP 3306 from 172.31.10.7/32   (Web Server 3)
```

Then your traffic spikes. Auto Scaling launches 5 more web servers with new IPs:
```
New servers: 172.31.10.8, .9, .10, .11, .12
These CANNOT reach the database — not in the whitelist
Your application is broken
Someone must manually add 5 new rules at 2 AM
```

Then traffic drops and Auto Scaling terminates 7 servers:
```
Old rules for dead IPs remain in the Security Group
Clutter grows, rules become outdated, security audits get messy
```

### The Security Group Reference Solution

```
DB_SG Inbound:
  MySQL TCP 3306 from Web_SG
```

This one rule dynamically means: **any EC2 instance that currently has Web_SG attached may connect.**

When Auto Scaling launches a new web server and attaches `Web_SG`:
```
New web server → automatically gets access to database
Zero manual intervention needed
```

When Auto Scaling terminates a web server:
```
Instance is gone → access automatically revoked
No stale rules, no cleanup needed
```

This scales from 1 to 1,000 web servers and back down, with never a single rule change.

---

## Part 9 — Real-World Security Group Architecture Summary

Here is the complete production-ready multi-tier architecture with all Security Groups:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Bastion_SG                                                          │
│   Inbound: SSH (22) from YOUR-OFFICE-IP/32 only                     │
│   Purpose: Admin access jump server                                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ SSH only
┌─────────────────────────────────────────────────────────────────────┐
│ ALB_SG (Application Load Balancer)                                  │
│   Inbound: HTTP (80) from 0.0.0.0/0                                 │
│            HTTPS (443) from 0.0.0.0/0                               │
│   Purpose: Accept all public web traffic                            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ port 8080 only
┌─────────────────────────────────────────────────────────────────────┐
│ Web_SG (Application Servers)                                        │
│   Inbound: TCP 8080 from ALB_SG                                     │
│            SSH (22) from Bastion_SG                                 │
│   Purpose: App servers behind load balancer                         │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ port 3306 only
┌─────────────────────────────────────────────────────────────────────┐
│ DB_SG (Database)                                                    │
│   Inbound: MySQL (3306) from Web_SG                                 │
│            SSH (22) from Bastion_SG                                 │
│   Purpose: Database accessible only from app tier                   │
└─────────────────────────────────────────────────────────────────────┘
```

Each layer can only talk to the layer directly below it. Nothing can jump layers. The internet can only reach the Load Balancer. The database is never reachable from the internet under any circumstances.

---

## Part 10 — Always Clean Up After Practice

After any practical exercise, terminate all EC2 instances you created. AWS charges by the hour even for stopped instances (for EBS storage), and running instances accumulate charges continuously.

**Cleanup checklist:**
1. EC2 → Instances → select all test instances → Instance state → **Terminate**
2. EC2 → Security Groups → delete any custom Security Groups you created (cannot delete Default SG)
3. EC2 → Elastic IPs → release any unattached Elastic IPs (charged even when not in use)

---

## Complete Security Group Rules Reference

| Use Case | Type | Port | Protocol | Source | Notes |
|---|---|---|---|---|---|
| Public website | HTTP | 80 | TCP | 0.0.0.0/0 | Open to internet |
| Secure website | HTTPS | 443 | TCP | 0.0.0.0/0 | Open to internet |
| Linux SSH admin | SSH | 22 | TCP | YOUR-IP/32 | Never open to world |
| Windows RDP admin | RDP | 3389 | TCP | YOUR-IP/32 | Never open to world |
| MySQL database | MySQL | 3306 | TCP | Web_SG | SG reference only |
| PostgreSQL database | PostgreSQL | 5432 | TCP | Web_SG | SG reference only |
| Redis cache | Custom TCP | 6379 | TCP | Web_SG | SG reference only |
| MongoDB | Custom TCP | 27017 | TCP | Web_SG | SG reference only |
| App behind LB | Custom TCP | 8080 | TCP | ALB_SG | SG reference only |
| Internal ping | All ICMP | N/A | ICMP | VPC CIDR | Diagnostics only |
| External ping | All ICMP | N/A | ICMP | 0.0.0.0/0 | Only if needed |
| DNS outbound | DNS | 53 | TCP/UDP | 0.0.0.0/0 | Usually outbound |
| NFS / EFS mount | NFS | 2049 | TCP | VPC CIDR | File system access |
