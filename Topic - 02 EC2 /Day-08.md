# AWS EC2 Security Groups

---

## What Problem Do Security Groups Solve?

When your EC2 instance is running and connected to the internet, it is technically reachable on all 65,535 ports simultaneously. Without any filtering, anyone on the internet can attempt to connect to any port on your server — sending malicious traffic, attempting exploits, or consuming your server's CPU processing requests it should never receive.

A Security Group is AWS's answer to this problem. It sits in front of your instance and acts as a **virtual firewall** — inspecting every packet trying to reach your instance and either allowing it through or silently dropping it, based on rules you define.

> Think of a Security Group as a strict bouncer at a club door. The bouncer has a list of who is allowed in (allowed ports and IPs). Anyone not on the list doesn't get in — they don't even get told the club exists. The traffic is just silently dropped.

---

## Networking Foundation — What Security Groups Actually Filter

To configure Security Groups correctly, you must understand what they're filtering. They work at the **network layer** — filtering based on:

---

### Protocols — The Three You Must Know

**TCP — Transmission Control Protocol**

TCP is a **connection-oriented** protocol. Before any data is sent, TCP performs a **three-way handshake** to establish a reliable connection:

```
Client                    Server
  │── SYN ──────────────→ │   "I want to connect"
  │← SYN-ACK ────────────  │   "OK, I'm ready"
  │── ACK ──────────────→ │   "Great, let's go"
  │                        │
  │══ Data flows both ways ║
  │                        │
  │── FIN ──────────────→ │   "I'm done"
```

TCP guarantees delivery — if a packet is lost, it is automatically retransmitted. Every major application protocol (HTTP, HTTPS, SSH, RDP, MySQL) runs on TCP because reliability matters more than speed.

**UDP — User Datagram Protocol**

UDP is **connectionless** — it just fires packets and doesn't wait for confirmation. There's no handshake, no guaranteed delivery, no retransmission. This makes it much faster.

Used for: DNS lookups, video streaming, gaming, VoIP, NTP (time sync). For these applications, speed matters more than perfect reliability — a dropped video frame is better than a 2-second pause waiting for retransmission.

**ICMP — Internet Control Message Protocol**

ICMP is different from TCP and UDP — it doesn't carry application data. It carries **network diagnostic and control messages**. The most famous use of ICMP is the `ping` command.

```bash
ping 8.8.8.8
# Sends ICMP Echo Request → receives ICMP Echo Reply
# Tells you: is the host alive? How long does a round trip take?
```

ICMP has no ports — you filter it simply by protocol type. Security Groups allow you to block or allow ICMP independently of TCP/UDP rules.

---

### Ports — The 65,535 Doors on Every Server

Every server running TCP or UDP has **65,535 possible ports**. Each port is like a different door on a building. Different services listen on different doors:

**Well-Known Ports (0–1023)** — Assigned by IANA to standardized global protocols:

| Port | Protocol | Service |
|---|---|---|
| 20, 21 | TCP | FTP (File Transfer) |
| 22 | TCP | SSH (Secure Shell — Linux remote access) |
| 23 | TCP | Telnet (unencrypted remote access — never use) |
| 25 | TCP | SMTP (sending email) |
| 53 | TCP/UDP | DNS (domain name resolution) |
| 80 | TCP | HTTP (unencrypted web traffic) |
| 110 | TCP | POP3 (receiving email) |
| 143 | TCP | IMAP (email) |
| 443 | TCP | HTTPS (encrypted web traffic) |
| 3389 | TCP | RDP (Windows Remote Desktop) |

**Registered Ports (1024–49151)** — Assigned by IANA to specific applications. Companies register a port number for their software:

| Port | Protocol | Service |
|---|---|---|
| 1433 | TCP | Microsoft SQL Server |
| 3306 | TCP | MySQL / MariaDB |
| 5432 | TCP | PostgreSQL |
| 5672 | TCP | RabbitMQ (message queue) |
| 6379 | TCP | Redis (in-memory cache) |
| 8080 | TCP | HTTP alternate (common for dev servers, Tomcat) |
| 8443 | TCP | HTTPS alternate |
| 9200 | TCP | Elasticsearch |
| 27017 | TCP | MongoDB |

**Dynamic / Ephemeral Ports (49152–65535)** — Not assigned to any service. Used temporarily by the operating system for the **return traffic** of outbound connections.

When your server sends a request outward (e.g., calls an external API), the OS picks a random ephemeral port for the return traffic. This is why outbound security group rules need to allow these ports for return traffic.

---

### How a Complete Request Works — Port-by-Port

```
Browser (your laptop)                    Web Server (EC2)
Port: randomly assigned (e.g., 52841)   Port: 80 (HTTP)

1. Browser → SYN packet → Server IP:80
2. Security Group checks: is port 80 allowed inbound? YES → allow through
3. Server receives request, processes it
4. Server → response → Client IP:52841
5. Security Group checks outbound: is port 52841 allowed? (ephemeral range)

Result: Web page loads successfully
```

Now with a blocked port:
```
Attacker                                  Web Server (EC2)
Attempts connection to Server IP:3306    (MySQL port)

1. Attacker → SYN packet → Server IP:3306
2. Security Group checks: is port 3306 allowed inbound? NO
3. Packet is SILENTLY DROPPED — not rejected, not replied to
4. Attacker receives nothing — the port appears to not exist

Result: Attacker cannot reach MySQL. Server CPU never even sees the request.
```

This silent drop (rather than sending back a "connection refused") is important — it gives attackers no information about what's running on your server.

---

## What is an AWS Security Group — Precise Definition

A **Security Group** is a **stateful, virtual firewall** attached to EC2 instances (and other AWS resources) that controls inbound and outbound traffic at the instance level using allow rules only.

Five key properties define how Security Groups behave:

---

### Property 1 — Stateful

This is the most important behavioral property. **Stateful** means the Security Group tracks the state of connections.

If you allow inbound traffic on port 80, the **return traffic is automatically allowed** — you don't need a separate outbound rule to let the response back out.

```
Inbound rule: Allow TCP port 80 from 0.0.0.0/0

A user sends HTTP request to port 80 → ALLOWED IN (matches rule)
Server sends HTTP response back → AUTOMATICALLY ALLOWED OUT
(No outbound rule needed — Security Group remembers the connection)
```

Compare this to **Network ACLs (stateless)** — covered later — where you must explicitly allow both directions.

Stateful tracking happens at the connection level using a state table. Every established connection is tracked, and return traffic is matched against that table rather than evaluated against rules again.

---

### Property 2 — Allow Rules Only (No Deny Rules)

Security Groups only have **allow rules**. You cannot create a rule that says "deny traffic from this IP." Everything not explicitly allowed is **implicitly denied** — blocked by default.

This is actually a security strength. There's no way to accidentally create an allow rule that conflicts with a deny rule. The logic is simple: if no allow rule matches, the traffic is dropped. Period.

If you need to **explicitly deny** specific IPs (e.g., block a known malicious IP while allowing everything else), you must use **Network ACLs** instead, which support both allow and deny rules.

---

### Property 3 — Default Behavior

When you create a brand new Security Group (not using the default one AWS creates for you), it starts with:
- **Inbound:** All traffic BLOCKED (zero allow rules)
- **Outbound:** All traffic ALLOWED (one rule: allow all outbound)

This means a new EC2 instance with a new empty Security Group can reach the internet outbound (to download packages, call APIs) but nothing from the internet can reach it inbound until you add rules.

The default AWS Security Group (created automatically in the default VPC) has slightly different defaults — it allows all inbound traffic from other instances that share the same Security Group. This allows instances in the same security group to communicate with each other freely.

---

### Property 4 — Attached to Instances (Not Subnets)

Security Groups operate at the **instance level** — they are attached to the elastic network interface (ENI) of each EC2 instance individually. This is different from Network ACLs which operate at the subnet level.

Practical implication: two EC2 instances in the same subnet can have completely different Security Groups with completely different rules. The subnet doesn't matter — the Security Group attached to each instance controls its traffic individually.

One instance can have **up to 5 Security Groups** attached simultaneously. The rules of all attached Security Groups are **combined** — if any Security Group allows the traffic, it gets through. There is no conflict resolution needed because there are only allow rules.

```
EC2 Instance with 2 Security Groups:
  SG-Web: Allow port 80, 443 inbound
  SG-SSH: Allow port 22 from office IP

Combined effect: ports 80, 443 (from anywhere) AND port 22 (from office IP) are allowed
All other ports blocked
```

---

### Property 5 — Regional Resource, VPC-Specific

A Security Group belongs to a specific **VPC in a specific region**. You cannot use a Security Group from us-east-1 on an instance in ap-south-1. You cannot use a Security Group from one VPC on an instance in a different VPC, even in the same region.

---

## Anatomy of a Security Group Rule

Every rule in a Security Group has five components:

```
Type | Protocol | Port Range | Source/Destination | Description
```

**Type** — A friendly name AWS provides for common services. When you select "SSH", AWS automatically fills in Protocol=TCP and Port=22. When you select "Custom TCP", you fill in the port yourself.

**Protocol** — TCP, UDP, ICMP, or All.

**Port Range** — A single port (e.g., `443`), a range (e.g., `8000-9000`), or all ports (`0-65535`).

**Source (inbound) / Destination (outbound)** — Who is allowed. Can be:
- **CIDR block** — An IP range in CIDR notation (e.g., `203.0.113.0/24` or `0.0.0.0/0` for anywhere)
- **Another Security Group ID** — Allows traffic from any instance that has that Security Group attached (covered in detail below)
- **My IP** — AWS auto-fills your current public IP
- **Anywhere IPv4** — `0.0.0.0/0` — all IPv4 addresses
- **Anywhere IPv6** — `::/0` — all IPv6 addresses

**Description** — Optional but extremely important for documentation. Always write what the rule is for (e.g., "Allow web traffic from internet" or "Allow SSH from DevOps team office").

---

## CIDR Notation — Understanding the Source Field

CIDR (Classless Inter-Domain Routing) notation specifies IP ranges using a base address and a prefix length:

`IP Address / Prefix Length`

The prefix length tells you how many bits are fixed (the network part) and how many bits are variable (the host part):

| CIDR | Range | Number of IPs | Use Case |
|---|---|---|---|
| `203.0.113.5/32` | Just `203.0.113.5` | 1 IP exactly | Allow a single specific IP |
| `203.0.113.0/24` | `203.0.113.0` to `203.0.113.255` | 256 IPs | Allow an office's IP block |
| `203.0.0.0/16` | `203.0.0.0` to `203.0.255.255` | 65,536 IPs | Allow a large network range |
| `0.0.0.0/0` | All IPv4 addresses | 4.3 billion | Allow anyone (internet-facing services) |
| `10.0.0.0/8` | `10.0.0.0` to `10.255.255.255` | 16.7 million | Allow entire private network |

`/32` means only that one exact IP. Each step down in prefix (from /32 to /31 to /30...) doubles the number of IPs in the range. `/0` means everything.

---

## Security Group as Source — The Powerful Pattern

One of the most useful Security Group features is using **another Security Group as the source** in a rule, instead of an IP address.

**Scenario:** You have an application server that should only accept database connections from your web servers — nothing else.

Instead of trying to list all web server IP addresses (which change when instances scale up or down), you do this:

```
Database Server Security Group (SG-Database):
  Inbound Rule:
    Protocol: TCP
    Port: 3306
    Source: SG-WebServer   ← Security Group reference, not an IP

Web Server 1 (172.31.10.5) — has SG-WebServer attached → CAN reach MySQL
Web Server 2 (172.31.10.8) — has SG-WebServer attached → CAN reach MySQL
New Web Server (172.31.10.12) — auto-scaled, has SG-WebServer → CAN reach MySQL
Attacker (54.123.x.x) — no SG-WebServer → BLOCKED
Your laptop — no SG-WebServer → BLOCKED
```

When you reference a Security Group as a source, AWS dynamically evaluates "does the source instance have this Security Group attached?" — it's not about IP addresses at all. This makes security rules work seamlessly with Auto Scaling where IPs are unpredictable.

---

## Inbound vs Outbound Rules

**Inbound Rules** — Control traffic coming INTO your instance. These are what you configure most carefully.

**Outbound Rules** — Control traffic going OUT from your instance. By default, all outbound traffic is allowed. In high-security environments, you may restrict outbound too (e.g., prevent your server from calling random internet addresses, only allow it to reach specific services).

**Because Security Groups are stateful**, in most configurations you can leave outbound rules as "allow all" and only focus on inbound. Return traffic for allowed inbound connections is automatically permitted.

---

## Real-World Security Group Architecture

### Three-Tier Web Application

```
┌──────────────────────────────────────────────────────────────┐
│ SG-LoadBalancer                                              │
│   Inbound:  TCP 80   from 0.0.0.0/0  (HTTP from internet)  │
│   Inbound:  TCP 443  from 0.0.0.0/0  (HTTPS from internet) │
│   Outbound: All traffic allowed                              │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│ SG-AppServer                                                 │
│   Inbound:  TCP 8080 from SG-LoadBalancer  (app traffic)    │
│   Inbound:  TCP 22   from SG-Bastion       (SSH management) │
│   Outbound: All traffic allowed                              │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│ SG-Database                                                  │
│   Inbound:  TCP 3306 from SG-AppServer  (MySQL from app)    │
│   Outbound: All traffic allowed                              │
└──────────────────────────────────────────────────────────────┘
```

Notice: the database Security Group has no public IP and the Security Group only allows traffic from `SG-AppServer`. Even if someone had your DB's private IP, they couldn't connect from anywhere that isn't an app server.

---

## Security Group vs Network ACL — Critical Difference

Both are firewalls in AWS but at different levels:

| Feature | Security Group | Network ACL |
|---|---|---|
| Operates at | Instance level (ENI) | Subnet level |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules | Allow only | Allow AND Deny |
| Rule evaluation | All rules evaluated together | Rules evaluated in number order, first match wins |
| Default | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| Applies to | Only instances you attach it to | All instances in the subnet automatically |
| Use case | Primary instance-level firewall | Additional subnet-level protection, explicit IP blocking |

Security Groups are your **primary and most important** firewall. Network ACLs are a secondary layer, most useful when you need to **explicitly block** a specific IP (something Security Groups can't do).

---

## Common Security Group Mistakes to Avoid

**Opening SSH to 0.0.0.0/0**
Port 22 open to the entire internet is the single most common EC2 security mistake. Bots constantly scan the internet for open port 22 and attempt brute-force logins. Always restrict SSH to your specific IP (`/32`) or your company's IP range.

**Opening RDP to 0.0.0.0/0**
Same issue for Windows. Port 3389 is constantly under attack. Restrict to known IPs only.

**Putting database ports on public-facing rules**
MySQL (3306), PostgreSQL (5432), MongoDB (27017) should never be reachable from the internet. Only allow these from specific Security Groups of your app servers.

**Using overly broad CIDR ranges**
`10.0.0.0/8` allows your entire VPC. Sometimes that's fine, but usually you want `10.0.1.0/24` — only the specific subnet your app servers live in.

**Never reviewing or cleaning up old rules**
Security Groups accumulate rules over time. Rules added for temporary debugging, old IP ranges that no longer exist, ports opened for one-time tasks. Audit your Security Groups regularly.

**Not using descriptions**
Six months later nobody knows why port 8443 is open to `52.33.12.0/24`. Always add a description to every rule.

---

## All Standard Ports Reference — Quick Guide

| Service | Port | Protocol | Who Needs It |
|---|---|---|---|
| SSH | 22 | TCP | Linux management — restrict to known IPs |
| SMTP | 25 | TCP | Email sending |
| DNS | 53 | TCP/UDP | Name resolution |
| HTTP | 80 | TCP | Web traffic inbound |
| HTTPS | 443 | TCP | Secure web traffic |
| RDP | 3389 | TCP | Windows management — restrict to known IPs |
| MySQL | 3306 | TCP | From app servers only |
| PostgreSQL | 5432 | TCP | From app servers only |
| SQL Server | 1433 | TCP | From app servers only |
| Redis | 6379 | TCP | From app servers only |
| MongoDB | 27017 | TCP | From app servers only |
| Elasticsearch | 9200 | TCP | From app servers only |
| HTTP alternate | 8080 | TCP | Dev servers, Tomcat, proxies |
| ICMP (ping) | N/A | ICMP | Network diagnostics — allow within VPC |
| NFS | 2049 | TCP | EFS (Elastic File System) mounting |
| Custom app | Any | TCP | Your application's specific port |
