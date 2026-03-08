# AWS VPC — Part 6: Bastion Host and Private Instance Access Patterns

---

## The Problem — Precisely Stated

You've built a secure two-tier VPC. Your database server and application backend sit in private subnets — no public IP, no IGW route, completely unreachable from the internet. The architecture is correct. But now you need to SSH into that private instance to install a database engine, debug an application, check logs, or run a migration script. How?

The naive answer — "just SSH to its private IP" — fails immediately. Your laptop is on the internet. The private IP `10.0.11.25` is only meaningful within the VPC. From outside the VPC, that address is unreachable. There is no route from the public internet to a private subnet IP, by design.

This is not a bug. This is the security model working exactly as intended. The solution isn't to poke a hole in the private subnet — it's to create a **controlled, hardened entry point** in the public subnet that acts as a stepping stone. That entry point is the **Bastion Host**.

---

## What a Bastion Host Is — Historically and Architecturally

The term "bastion" comes from military fortification — a bastion is a projecting part of a wall designed to allow defenders to cover adjacent sections. In network security, a bastion host is a server that is intentionally exposed to the internet but hardened to an extreme degree, because it is the single entry point into an otherwise private network.

In AWS, the bastion host pattern is simple in concept:

```
Your Laptop (internet)
        │
        │  SSH on port 22, using PEM key
        ▼
Bastion Host (EC2 in PUBLIC subnet)
  - Has a public IP or Elastic IP
  - Security Group: allows SSH only from your IP (or corporate IP range)
  - Minimal software installed — just an SSH server, nothing else
        │
        │  SSH on port 22, using PEM key
        │  (using PRIVATE IP of target instance)
        ▼
Private EC2 Instance (in PRIVATE subnet)
  - No public IP
  - Security Group: allows SSH ONLY from Bastion Host's Security Group
  - No route to internet
```

The bastion is the only server that has a public-facing SSH port. Every other server in your infrastructure is reachable only through it. This dramatically reduces your attack surface — instead of every server being a potential SSH target, only one hardened box is.

---

## The Network Path — What Actually Happens

When you SSH from your laptop to the private instance via the bastion, two entirely separate TCP connections are established:

```
Connection 1: Your Laptop → Bastion Host
─────────────────────────────────────────
Source:      Your laptop's public IP (e.g., 103.x.x.x)
Destination: Bastion's public IP (e.g., 13.233.45.67) port 22
Path:        Internet → IGW → public subnet → Bastion EC2
Auth:        PEM key pair

Connection 2: Bastion Host → Private EC2
─────────────────────────────────────────
Source:      Bastion's private IP (e.g., 10.0.1.15)
Destination: Private EC2's private IP (e.g., 10.0.11.25) port 22
Path:        Bastion → VPC local router → private subnet → Private EC2
Auth:        PEM key pair (same or different key)
```

The bastion is not a transparent proxy — it's a full SSH endpoint. You land on the bastion's shell, and from there you initiate a second SSH session to the private instance. The private instance never sees your laptop's IP — it only sees the bastion's private IP as the source of the connection.

---

## Security Group Rules — The Precise Configuration

This is where most implementations are done incorrectly. The security group rules must enforce the access chain at the network layer, not just by convention.

### Bastion Host Security Group

```
Bastion-SG: Inbound Rules
┌──────────┬───────┬──────────────────────────────────────────────┐
│ Protocol │ Port  │ Source                                       │
├──────────┼───────┼──────────────────────────────────────────────┤
│ TCP      │ 22    │ YOUR_OFFICE_IP/32  (e.g., 103.x.x.x/32)     │
│          │       │ NOT 0.0.0.0/0 — never open SSH to the world  │
└──────────┴───────┴──────────────────────────────────────────────┘

Bastion-SG: Outbound Rules
┌──────────┬───────┬──────────────────────────────────────────────┐
│ Protocol │ Port  │ Destination                                  │
├──────────┼───────┼──────────────────────────────────────────────┤
│ TCP      │ 22    │ Private-Instance-SG (security group ID)      │
└──────────┴───────┴──────────────────────────────────────────────┘
```

### Private Instance Security Group

```
Private-Instance-SG: Inbound Rules
┌──────────┬───────┬──────────────────────────────────────────────┐
│ Protocol │ Port  │ Source                                       │
├──────────┼───────┼──────────────────────────────────────────────┤
│ TCP      │ 22    │ Bastion-SG (reference by security group ID)  │
└──────────┴───────┴──────────────────────────────────────────────┘
```

The critical detail here is **referencing a security group ID as the source**, not an IP address. When you specify `Bastion-SG` as the source, the rule means: "allow SSH from any EC2 instance that has `Bastion-SG` attached." This is more robust than using the bastion's private IP because:

- If the bastion is replaced (new instance, same SG), the rule still works
- If the bastion's private IP changes, the rule still works
- It's semantically clear — only instances with the bastion security group can reach this instance

> ⚠️ **Interview Trap:** "Should the private instance's security group reference the bastion's IP address or its security group?" Security group ID. IP-based rules break when instances are replaced. SG-based rules are the AWS best practice and remain valid regardless of the underlying instance's IP.

---

## The PEM Key Problem — Three Approaches

The bastion pattern immediately runs into a key management challenge. To SSH from the bastion to the private instance, the bastion needs the private key for the private instance. How do you get the key there securely?

### Approach 1: SCP the Key to the Bastion (Simple, Least Secure)

The video demonstrates this approach. You copy the PEM file from your laptop to the bastion using SCP, then use it from there.

```bash
# Step 1: Copy PEM from local machine to bastion
scp -i bastion-key.pem private-instance-key.pem \
    ec2-user@<bastion-public-ip>:/home/ec2-user/

# Step 2: SSH into bastion
ssh -i bastion-key.pem ec2-user@<bastion-public-ip>

# Step 3: From bastion, SSH to private instance
chmod 400 private-instance-key.pem
ssh -i private-instance-key.pem ec2-user@<private-instance-private-ip>
```

**The security problem:** Your private key now lives on the bastion host. If the bastion is compromised, the attacker has the key to every private instance. This is why this approach is only acceptable for dev/test environments.

### Approach 2: SSH Agent Forwarding (Better — Key Never Leaves Your Laptop)

SSH agent forwarding allows the SSH authentication to happen on your laptop while using the bastion as a transport. The private key never leaves your machine.

```bash
# Step 1: Add your key to the SSH agent on your laptop
ssh-add private-instance-key.pem

# Step 2: SSH to bastion WITH agent forwarding enabled (-A flag)
ssh -A -i bastion-key.pem ec2-user@<bastion-public-ip>

# Step 3: From bastion, SSH to private instance
# The -A flag means your laptop's SSH agent handles the auth
# No PEM file needed on the bastion
ssh ec2-user@<private-instance-private-ip>
```

With `-A`, when the private instance asks for authentication, the request is tunneled back through the bastion to the SSH agent running on your laptop, which performs the signing operation. The private key never leaves your laptop. This is the correct production approach.

> ⚠️ **Interview Trap:** "What is SSH agent forwarding and why is it used with bastion hosts?" It allows key-based auth to downstream hosts without storing private keys on intermediate servers. The bastion acts as a transport, not an authentication endpoint.

### Approach 3: ProxyJump / SSH ProxyCommand (Most Elegant)

Modern SSH (version 7.3+) supports `ProxyJump`, which creates a single transparent tunnel from your laptop through the bastion to the private instance. To the user, it feels like a direct connection.

```bash
# Single command — SSH through bastion directly to private instance
ssh -J ec2-user@<bastion-public-ip> ec2-user@<private-instance-private-ip>

# Or configure it permanently in ~/.ssh/config:
Host bastion
    HostName <bastion-public-ip>
    User ec2-user
    IdentityFile ~/.ssh/bastion-key.pem

Host private-instance
    HostName <private-instance-private-ip>
    User ec2-user
    IdentityFile ~/.ssh/private-key.pem
    ProxyJump bastion
```

After the config is set, accessing the private instance is simply:

```bash
ssh private-instance
```

SSH handles the tunneling automatically. This combines the security of agent forwarding with a single-command user experience. For teams managing many private instances, this is the standard approach.

---

## VPC Endpoint (EC2 Instance Connect Endpoint) — The Modern AWS Alternative

The video introduces this as an alternative to the bastion host. It's worth understanding precisely how it works and when to use it.

**EC2 Instance Connect Endpoint (EIC Endpoint)** is an AWS-managed service that creates a private tunnel into your VPC from AWS's control plane. You connect via the AWS Console or CLI, and AWS handles the tunneling — no bastion host, no public IP needed on the target instance.

```
Your Browser / AWS CLI
        │
        │  HTTPS to AWS API
        ▼
AWS EC2 Instance Connect Service
        │
        │  Private tunnel through EIC Endpoint (in your subnet)
        ▼
EIC Endpoint (ENI in your private subnet)
        │
        │  TCP to private instance (port 22 or 3389)
        ▼
Private EC2 Instance
```

```bash
# Connect via CLI (no bastion needed)
aws ec2-instance-connect open-tunnel \
    --instance-id i-xxxxxxxxxxxxxxxxx \
    --remote-port 22 \
    --local-port 2222

# Then SSH through the local port
ssh -p 2222 ec2-user@localhost
```

**When EIC Endpoint is appropriate:**
- Teams where all engineers have AWS IAM credentials and AWS CLI access
- Organizations using AWS SSO — engineers authenticate via their corporate identity
- Environments where you want audit logging of all access via CloudTrail (every EIC connection is logged)
- When you want to eliminate the operational overhead of managing bastion host patching and availability

**When Bastion Host is more appropriate:**
- Third-party vendors or contractors who should not have AWS IAM credentials
- Legacy tooling that doesn't support EIC Endpoint
- Environments where the bastion also performs other functions (VPN termination, jump to non-EC2 resources)
- When you need to tunnel arbitrary TCP ports, not just SSH (EIC Endpoint is SSH/RDP specific)

---

## Hardening the Bastion Host — Production Considerations

A bastion that isn't hardened is just another attack surface. In production, the bastion should be treated as a critical security boundary:

**Minimize the attack surface.** The bastion should run nothing except the SSH daemon. No web servers, no application code, no databases. Every additional service is an additional vulnerability.

**Restrict SSH access by IP.** Never use `0.0.0.0/0` as the source for SSH in the bastion's security group. Lock it to your office's public IP range, your VPN exit IP, or your corporate IP block. For dynamic IPs, some teams use an Elastic IP on a small VPN server as the SSH source.

**Use a small instance type.** The bastion doesn't do compute work. A `t3.nano` or `t3.micro` is sufficient and keeps cost minimal (~$3-4/month).

**Enable detailed logging.** Configure the bastion to log all SSH sessions. Consider tools like `tlog` or `script` to record session output for audit purposes.

**Use Elastic IP on the bastion.** If the bastion is stopped and restarted (for OS updates, for example), it should come back with the same IP — otherwise your Security Group rules and SSH config files break.

**Patch aggressively.** The bastion is internet-facing. OS security patches should be applied immediately, not on a monthly cycle.

---

## The Complete SSH Access Flow — Visualized End to End

```
DevOps Engineer's Laptop
(Public IP: 103.x.x.x)
        │
        │ ssh -J  or  ssh -A
        │ Port 22
        ▼
┌──────────────────────────────────────────────────┐
│  PUBLIC SUBNET (10.0.1.0/24)                     │
│                                                  │
│  Bastion Host                                    │
│  Private IP:  10.0.1.10                          │
│  Public IP:   13.233.45.67 (Elastic IP)          │
│  SG: Bastion-SG                                  │
│    Inbound: TCP/22 from 103.x.x.x/32 only        │
└──────────────┬───────────────────────────────────┘
               │
               │ TCP/22 (private IP only)
               ▼
┌──────────────────────────────────────────────────┐
│  PRIVATE SUBNET (10.0.11.0/24)                   │
│                                                  │
│  Application/Database Server                     │
│  Private IP: 10.0.11.25                          │
│  NO public IP                                    │
│  SG: Private-Instance-SG                         │
│    Inbound: TCP/22 from Bastion-SG only           │
└──────────────────────────────────────────────────┘
```

---

## Comparison Table: Private Instance Access Methods

| Dimension | Bastion Host | EIC Endpoint | VPN | SSM Session Manager |
|---|---|---|---|---|
| **Requires public IP on target** | No | No | No | No |
| **Requires AWS credentials** | No | Yes (IAM) | No | Yes (IAM) |
| **Key management** | PEM file | AWS handles | Certificate/PSK | No keys needed |
| **Cost** | EC2 instance cost | Free (data charges) | VPN Gateway cost | Free |
| **Audit logging** | Manual setup | CloudTrail | Manual | CloudTrail natively |
| **Patching overhead** | You patch the bastion | None (managed) | Depends | None (managed) |
| **Non-SSH protocols** | Yes (any TCP tunnel) | SSH/RDP only | Yes | Limited |
| **Third-party access** | Easy | Needs IAM user | Possible | Needs IAM user |
| **Production recommendation** | Yes (with hardening) | Yes (modern) | For site-to-site | Yes (preferred by many) |

> ⚠️ **Interview Trap:** "What is the most secure way to access private EC2 instances in 2024?" SSM Session Manager is increasingly the preferred answer for internal teams — it requires no open port 22 at all, no public IP, no key management, and every session is auditable via CloudTrail and S3. The bastion is still valid but requires more operational overhead. EIC Endpoint is a middle ground. Knowing all three and their trade-offs is what separates mid-level from senior answers.

---

## Interview Flags for This Topic

> ⚠️ **"Why shouldn't you open SSH (port 22) to `0.0.0.0/0` on a bastion host?"** Because automated bots on the internet continuously scan for open SSH ports and attempt brute-force or exploit-based attacks. Restricting to known IP ranges eliminates this entire attack vector.

> ⚠️ **"What happens if you SCP the PEM key to the bastion and the bastion is compromised?"** The attacker now has the private key to all instances that key unlocks. Keys should never be stored on intermediate hosts — use SSH agent forwarding instead.

> ⚠️ **"Can you use the same key pair for the bastion and the private instances?"** Technically yes, but it's a security anti-pattern. If the bastion key is compromised, all instances are compromised. Use separate key pairs for the bastion and for private instances.

> ⚠️ **"What is the difference between `ssh -A` and `ssh -J`?"** `-A` enables agent forwarding — your local SSH agent handles authentication for downstream hops. `-J` (ProxyJump) creates an explicit tunnel through a jump host. Both can be combined. `-J` is cleaner for simple jump scenarios; `-A` is needed when the downstream host needs to make further SSH connections of its own.

> ⚠️ **"What AWS service eliminates the need for a bastion host entirely?"** AWS Systems Manager Session Manager. It creates an encrypted tunnel to EC2 instances via the SSM agent — no port 22, no public IP, no key pairs needed. Access is controlled entirely through IAM policies.

---

The next topic completing the private subnet picture is **NAT Gateway in depth** — the outbound internet story for private instances. The video in this part explicitly defers it, noting that private instances cannot install packages without it. Drop the NAT Gateway topic next and we'll cover it with the same depth.
