# AWS Public IP vs Elastic IP — Complete In-Depth Guide

---

## The Core Problem — Why IP Addresses Matter So Much

Every service on the internet is reached by an IP address. When you point your domain name (`mywebsite.com`) to a server, you're actually pointing it to an IP address behind the scenes. When your mobile app calls your backend API, it's connecting to an IP address. When your monitoring system checks if your server is alive, it pings an IP address.

Now imagine that IP address changes every time you restart your server.

Every DNS record breaks. Every firewall rule referencing that IP breaks. Every hardcoded connection string in a client application breaks. Every partner system that whitelists your IP to accept connections from you — breaks.

This is the exact problem AWS's IP address model creates by default, and Elastic IP is the solution.

---

## Public IP in AWS — The Default, Ephemeral Address

### How It Works

When you launch an EC2 instance with "Auto-assign public IP" enabled, AWS assigns a public IPv4 address from its pool of available addresses. This happens automatically, costs nothing extra, and makes your instance immediately reachable from the internet.

```
Instance Launches
      ↓
AWS picks an available IP from its public pool
      ↓
Assigns: 54.123.45.67 to your instance
      ↓
Instance is reachable at 54.123.45.67
```

This seems perfect — until you stop the instance.

### The Stop/Start Problem

```
Day 1:
  Instance running → Public IP: 54.123.45.67
  Your DNS: mywebsite.com → 54.123.45.67  ✓
  Your app works perfectly

You stop the instance to save costs overnight:
  Instance stopped → IP 54.123.45.67 RELEASED back to AWS pool
  AWS may assign 54.123.45.67 to someone else's instance

Day 2:
  Instance started → New IP assigned: 52.198.76.21
  Your DNS still points to: 54.123.45.67 (old IP)
  54.123.45.67 now belongs to someone else
  Your website: ❌ DOWN
  Your app: ❌ Cannot connect
```

This is not a bug — it is intentional AWS behavior. Public IPv4 addresses are a globally scarce, finite resource. AWS cannot afford to reserve one for every stopped instance. When your instance stops, AWS reclaims the IP and makes it available for other running instances.

### What "Stops" an Instance's Public IP

| Action | Public IP Lost? |
|---|---|
| **Stop** instance | ✅ Yes — IP released when stopped |
| **Terminate** instance | ✅ Yes — IP released permanently |
| **Reboot** instance | ❌ No — IP preserved during reboot |
| Instance **crashes and restarts** | ❌ No — IP preserved |
| **Stop + Start** (even 1 second apart) | ✅ Yes — IP changes |

The critical distinction: **Reboot = same IP. Stop then Start = new IP.**

### Cost of Public IP

As of **February 1, 2024**, AWS charges **$0.005 per hour** for all public IPv4 addresses — including the default public IP automatically assigned to your instances. This is approximately **$3.65/month** per public IPv4 address.

Previously, the default public IP was free. AWS changed this due to global IPv4 scarcity. This cost applies whether the instance is running or stopped (as long as a public IP is attached to a running instance).

---

## Elastic IP — The Static, Persistent Public Address

### What is an Elastic IP?

An **Elastic IP (EIP)** is a **static public IPv4 address** that you allocate to your AWS account and fully control. Unlike a default public IP that belongs to the instance and disappears when the instance stops, an Elastic IP belongs to **your account** until you explicitly release it.

```
Elastic IP lifecycle:

You allocate EIP: 54.200.100.50
      ↓
EIP belongs to YOUR ACCOUNT (not any specific instance)
      ↓
You associate it with Instance A
      ↓
Instance A is reachable at 54.200.100.50
      ↓
You stop Instance A
      ↓
EIP 54.200.100.50 is still YOURS — not released
      ↓
You start Instance A
      ↓
Instance A is still at 54.200.100.50 — UNCHANGED
```

The IP address is decoupled from the instance's lifecycle. It survives stop, start, reboot, and even termination of the associated instance (it stays in your account, just unassociated).

### How to Allocate an Elastic IP

**AWS Console:**
1. EC2 Console → **Network & Security** → **Elastic IPs**
2. Click **"Allocate Elastic IP address"**
3. Choose:
   - **Amazon's pool of IPv4 addresses** — AWS assigns you one from its pool (most common)
   - **Bring your own IP addresses (BYOIP)** — use your company's own IP block (for organizations that own their IP ranges)
4. Optionally add tags
5. Click **"Allocate"**

You now own an IP address (e.g., `54.200.100.50`) until you release it. It is listed in your Elastic IPs dashboard with status "Not associated."

**AWS CLI:**
```bash
# Allocate an Elastic IP
aws ec2 allocate-address --domain vpc

# Output:
# {
#   "PublicIp": "54.200.100.50",
#   "AllocationId": "eipalloc-0abc123def456789",
#   "Domain": "vpc"
# }
```

### How to Associate an Elastic IP with an Instance

**AWS Console:**
1. Select the Elastic IP → **Actions** → **Associate Elastic IP address**
2. Choose:
   - **Resource type:** Instance
   - **Instance:** Select your EC2 instance from the dropdown
   - **Private IP:** If the instance has multiple network interfaces, select which private IP to associate with
3. Click **"Associate"**

**AWS CLI:**
```bash
aws ec2 associate-address \
  --instance-id i-0abc123def456789 \
  --allocation-id eipalloc-0abc123def456789
```

**What happens to the old public IP when you associate an EIP?**

When you associate an Elastic IP with an instance that already has a default public IP, the default public IP is **immediately released back to AWS**. The instance now only has the Elastic IP as its public address. This is permanent for that instance launch — if you later disassociate the EIP, the instance will not get its original public IP back (it would get a new random one if you re-enable auto-assign).

### How to Disassociate an Elastic IP

Disassociating removes the EIP from the instance but keeps it in your account:

**Console:** Select EIP → Actions → Disassociate Elastic IP address

**CLI:**
```bash
aws ec2 disassociate-address \
  --association-id eipassoc-0abc123def456789
```

After disassociation, the EIP is "Not associated" — you still own it, and you're still being charged for it.

### How to Release an Elastic IP

Releasing permanently removes the EIP from your account and returns it to AWS's pool. You can never get that specific IP back:

**Console:** Select EIP → Actions → Release Elastic IP addresses

**CLI:**
```bash
aws ec2 release-address \
  --allocation-id eipalloc-0abc123def456789
```

**After releasing:** Billing stops immediately. The IP address may be assigned to any other AWS customer.

---

## Elastic IP Pricing — The Most Misunderstood Part

The EIP pricing model has a deliberate design that punishes waste:

### Current Pricing (from Feb 2024)

| Scenario | Cost |
|---|---|
| EIP **associated with a running instance** | $0.005/hour (~$3.65/month) |
| EIP **associated with a stopped instance** | $0.005/hour (~$3.65/month) |
| EIP **allocated but NOT associated** with any instance | $0.005/hour (~$3.65/month) |
| EIP **remapped** (re-associated) more than 100 times/month | $0.10 per remap after 100 |

**The key insight: you're charged regardless of whether it's in use.** An EIP sitting in your account doing nothing costs the same as one actively serving traffic. This is AWS's intentional mechanism to prevent hoarding of public IPv4 addresses (which are globally scarce).

### The Practical Implication

Never allocate an EIP and leave it unassociated. Either:
1. Associate it with an instance immediately, or
2. Release it if you don't need it

Many AWS bills have unexpected Elastic IP charges from engineers who allocated EIPs during testing, forgot about them, and paid months of charges for addresses that were never used.

---

## Elastic IP Limits

By default, AWS limits each account to **5 Elastic IPs per region**. This is a soft limit — you can request an increase through AWS Service Quotas if your architecture requires more.

The limit exists for the same reason as the pricing — IPv4 addresses are a finite global resource, and AWS wants to discourage accumulation of unused addresses.

---

## Remapping — Elastic IP for High Availability

One of the most powerful uses of Elastic IPs is **instant failover** by remapping the IP from a failed instance to a healthy one.

### Failover Scenario

```
Normal operation:
  EIP 54.200.100.50 → Instance A (primary)
  DNS: myapp.com → 54.200.100.50
  All users hit Instance A

Instance A fails:
  Problem detected (manually or via CloudWatch alarm)
  
Action: Remap EIP to Instance B:
  aws ec2 associate-address \
    --instance-id i-instanceB \
    --allocation-id eipalloc-0abc123
  
Result:
  EIP 54.200.100.50 → Instance B (backup)
  DNS still: myapp.com → 54.200.100.50 (unchanged)
  Users now hit Instance B
  
Remap time: ~30 seconds
DNS TTL change: none required
User experience: brief interruption, then recovered
```

This pattern is used for:
- **Database failover** — primary fails, EIP remapped to replica (now promoted to primary)
- **Manual blue/green deployment** — switch traffic between old and new version instantly
- **Emergency replacement** — existing instance is compromised, spin up clean instance and remap EIP

### Important Note About Modern Alternatives

For most production architectures today, **Load Balancers** with a DNS name are preferred over EIP-based failover:

```
EIP approach:           DNS → EIP → single instance
                        Failover: manual remap, ~30s downtime

Load Balancer approach: DNS → ALB DNS name → multiple instances
                        Failover: automatic, ~5-10s detection, seamless
```

Load Balancers distribute traffic across multiple instances and automatically route around failed instances. EIP remapping is a manual process. However, EIPs are simpler and useful for:
- Single-instance workloads that don't need scaling
- Non-HTTP protocols where ALB/NLB isn't appropriate
- Architectures where a fixed IP is specifically required (e.g., a partner that whitelists your IP)

---

## BYOIP — Bring Your Own IP

Large enterprises and organizations that own their own IP address blocks (IPv4 or IPv6) can bring those IPs to AWS and use them as Elastic IPs. This is called **BYOIP (Bring Your Own IP)**.

**Why this matters:**
- Your company has been using IP range `203.0.113.0/24` for years
- Partners, customers, and regulators have whitelisted this specific range
- When migrating to AWS, changing IPs would require updating all those whitelists
- BYOIP lets you bring `203.0.113.0/24` to AWS and use those addresses as EIPs

**Requirements for BYOIP:**
- You must own and control the IP block (registered with ARIN, RIPE, APNIC, etc.)
- The prefix must be at minimum a /24 (256 IPs) for IPv4, /48 for IPv6
- You must create a Route Origin Authorization (ROA) that AWS signs
- The IP block must have a clean reputation (not listed in spam/abuse blacklists)

---

## IPv6 and Elastic IPs

AWS also supports **IPv6 addresses** for EC2 instances. IPv6 works differently:

- IPv6 addresses assigned to EC2 instances are **persistent by default** — they do not change on stop/start
- This is because IPv6 has effectively unlimited address space — there's no scarcity problem
- There is no equivalent of "Elastic IPv6" because regular IPv6 addresses already behave like Elastic IPs
- IPv6 addresses are **free** — no per-address charge like IPv4

For new architectures, AWS recommends using IPv6 where possible to avoid the IPv4 scarcity and cost issues entirely.

---

## Complete Comparison — Public IP vs Elastic IP

| Property | Default Public IP | Elastic IP |
|---|---|---|
| Assignment | Automatic at launch | Manual — you allocate and associate |
| Persistence | **Lost when instance stops** | **Persists — belongs to your account** |
| Survives stop/start? | ❌ No — new IP assigned | ✅ Yes — same IP always |
| Survives reboot? | ✅ Yes | ✅ Yes |
| Survives termination? | ❌ No — released | ✅ Yes — stays in your account |
| Ownership | Belongs to the instance | Belongs to **your account** |
| Can be remapped? | No | Yes — move between instances instantly |
| Cost | $0.005/hr (from Feb 2024) | $0.005/hr (charged even when unassociated) |
| Charged when unassociated? | N/A | ✅ Yes — still charged |
| Per-account limit | Unlimited (one per instance) | **5 per region** (soft limit, increasable) |
| BYOIP support? | No | Yes |
| Best for | Dev/test, short-lived workloads | Production servers needing fixed IP |
| Release process | Automatic on instance stop/terminate | Manual — you must explicitly release |

---

## Best Practices

**Always release unused Elastic IPs.** An unassociated EIP is $3.65/month of pure waste. Make it a habit to check your EIP dashboard monthly.

**Use Load Balancers instead of EIPs for scalable production workloads.** A Load Balancer gives you a stable DNS endpoint, automatic failover, health checking, and the ability to scale behind it — all things a single EIP cannot provide.

**Use EIPs for fixed-IP requirements.** If a partner, customer, or compliance requirement mandates a specific IP address, EIP is the right tool. Document which EIPs are used for which purpose in tags.

**Tag your EIPs.** Just like any AWS resource, tag EIPs with `Name`, `Environment`, `Owner`, and `Purpose`. Without tags, a list of 5 IP addresses is impossible to manage 6 months later.

**Set up billing alerts.** Create a CloudWatch billing alarm that triggers if your Elastic IP charges exceed a threshold. Unassociated EIPs are a common source of unexpected bills.

**Automate EIP association with scripts.** If an instance needs an EIP but might be rebuilt frequently, use User Data or a startup script to automatically find and associate a specific EIP by its Allocation ID:

```bash
#!/bin/bash
# In User Data — automatically associate the EIP at launch

INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
ALLOCATION_ID="eipalloc-0abc123def456789"

aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $ALLOCATION_ID \
  --region ap-south-1
```

This ensures the instance always gets the same EIP even if it's replaced.

---

## Summary — Everything at a Glance

| Scenario | Use Default Public IP | Use Elastic IP |
|---|---|---|
| Development/test instance | ✅ | Overkill |
| Instance you stop overnight to save cost | ❌ IP changes | ✅ IP stays same |
| Web server with DNS record pointing to it | ❌ DNS breaks on stop | ✅ DNS always valid |
| Partner that has whitelisted your IP | ❌ IP changes | ✅ IP never changes |
| High availability failover | ❌ Can't remap | ✅ Remap in 30s |
| Instance behind a Load Balancer | ✅ (LB handles stability) | Usually unnecessary |
| BYOIP compliance requirement | ❌ | ✅ |
| Temporary compute job | ✅ | Waste money |
| Long-running production server | Consider EIP | ✅ For stability |
