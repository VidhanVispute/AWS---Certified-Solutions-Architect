# AWS EC2 Tenancy Options — Complete In-Depth Guide

---

## The Concept of Tenancy — What Does It Actually Mean?

In the physical world, "tenancy" describes who lives in a building and how they share it. AWS borrows this exact concept for its data center infrastructure.

When you launch an EC2 instance, it runs on a **physical server** — a real machine sitting in a real AWS data center. The question of **tenancy** is: who else's virtual machines are running on that same physical server, and how much control do you have over that physical machine?

By default, AWS runs many customers' instances on the same physical host simultaneously. Your virtual machine is completely isolated from others at the software level — you cannot see their data, they cannot see yours. But you are sharing the same physical CPU, RAM, and potentially the same physical network card.

For most companies, this is perfectly fine. For others — particularly in government, defense, healthcare, and finance — regulations, compliance standards, or security policies explicitly prohibit sharing physical hardware with unknown third parties. For these organizations, AWS provides dedicated hardware options.

> Think of it like housing. Shared tenancy is an apartment building — you have your own locked unit, nobody else can enter, but you share the building's physical walls, plumbing, and electrical infrastructure with neighbors. Dedicated Instance is renting an entire house — nobody else lives there, but the landlord (AWS) still maintains it and decides the layout. Dedicated Host is buying the house outright — it's yours completely, you control every room, and you can renovate however you want.

---

## The Three Tenancy Options

When launching an EC2 instance in the AWS console, navigate to **Advanced details** and find the **Tenancy** dropdown. You will see exactly three options:

1. **Shared** (default)
2. **Dedicated Instance**
3. **Dedicated Host**

Each represents a fundamentally different relationship between you, your EC2 instances, and the physical hardware they run on.

---

## Option 1 — Shared Tenancy (Default)

### How It Works

In Shared tenancy, AWS places your EC2 instances on physical servers alongside instances belonging to other AWS customers. The hypervisor (AWS Nitro) creates complete isolation between all VMs — Customer A cannot access Customer B's data, memory, or processes. But the physical CPU cores, RAM chips, and network hardware are shared resources allocated dynamically by the hypervisor.

```
Physical Host Server (managed entirely by AWS)
┌─────────────────────────────────────────────────────────┐
│                  AWS Nitro Hypervisor                   │
├──────────────┬──────────────┬──────────────┬────────────┤
│ Customer A   │ Customer B   │ Customer C   │  You       │
│ VM instance  │ VM instance  │ VM instance  │ VM instance│
│              │              │              │            │
│ (cannot see  │ (cannot see  │ (cannot see  │            │
│  others)     │  others)     │  others)     │            │
└──────────────┴──────────────┴──────────────┴────────────┘
                  Same physical CPU, RAM, NIC
```

### Key Properties

**Isolation:** Software isolation via the Nitro hypervisor — complete and proven effective. Your data is cryptographically isolated from other tenants. AWS's hypervisor ensures zero cross-tenant data leakage under normal operation.

**Hardware visibility:** Zero. You have no idea which physical server you're on, what other instances share it, how many physical CPU cores are present, or anything about the underlying hardware. AWS abstracts all of this away entirely.

**AWS management:** AWS handles everything — hardware procurement, rack installation, firmware updates, hardware replacement when components fail, CPU microcode updates, hypervisor patches. You never think about any of this.

**Instance placement:** AWS's scheduler places and moves your instances based on its own optimization — balancing load, replacing failed hardware, maintenance windows. You have no control over which physical server you land on.

**Noisy neighbor effect:** In theory, other tenants running very CPU or I/O intensive workloads could marginally affect your instance's performance. In practice, AWS's Nitro hypervisor and resource allocation algorithms minimize this significantly. For most workloads it is completely imperceptible.

**Cost:** Cheapest option — the baseline price you see on AWS's EC2 pricing page. No premium.

### When to Use Shared Tenancy

The vast majority of AWS workloads — probably 95%+ — run on shared tenancy. It is appropriate for:
- Web applications and API servers
- Development, testing, staging environments
- Data processing pipelines
- Applications without explicit hardware isolation compliance requirements
- Any workload where software-level isolation is sufficient

---

## Option 2 — Dedicated Instance

### How It Works

With Dedicated Instance tenancy, AWS guarantees that your EC2 instances run on **physical hardware that is used only by your AWS account**. No other customer's instances will ever run on the same physical server as yours.

However — and this is the critical distinction — **AWS still owns, manages, and controls the physical hardware**. You have no visibility into the host, no ability to configure it, and no control over which specific physical server your instances land on. You simply know it's dedicated to you.

```
Physical Host Server (still managed entirely by AWS)
┌─────────────────────────────────────────────────────────┐
│                  AWS Nitro Hypervisor                   │
├──────────────────────────────────────────────────────────┤
│         YOUR ACCOUNT'S INSTANCES ONLY                   │
│                                                         │
│  Your VM 1    Your VM 2    Your VM 3    Your VM 4       │
│  (web server) (app server) (database)  (cache)          │
│                                                         │
│  No other customer's instances on this physical host    │
└─────────────────────────────────────────────────────────┘
              AWS controls and maintains the hardware
```

### The Important Nuance — Same Account, Multiple Instances

Dedicated Instances only guarantee isolation from **other AWS accounts**. Multiple instances within **your own AWS account** may share the same physical host with each other. This is fine — they're all yours anyway.

If you have multiple AWS accounts (common in enterprise environments using AWS Organizations), instances from different accounts within your organization may or may not share hardware depending on your configuration. Linked accounts need specific setup to guarantee cross-account isolation.

### Key Properties

**Hardware isolation:** Physical — no other customer's code runs on your hardware. This satisfies most regulatory requirements around hardware sharing.

**Hardware control:** None — AWS still decides which physical server, still manages firmware and maintenance, still has physical access. You cannot inspect the hardware, choose its specifications beyond the instance type, or configure BIOS settings.

**Compliance use case:** Satisfies regulations that prohibit sharing physical hardware with unknown third parties — common in HIPAA (healthcare), FedRAMP (US government), PCI-DSS (payment processing), and certain EU data regulations.

**License limitations:** Does NOT support Bring Your Own License (BYOL). Software licenses that require knowing the number of physical CPU sockets or cores cannot be managed with Dedicated Instances because you don't have visibility into the underlying hardware topology. This is where Dedicated Host has the advantage.

**Instance movement:** When you stop and start an instance, it may move to a different physical dedicated host (still dedicated to your account, but possibly different hardware). This is unlike Dedicated Host where you control which specific host an instance runs on.

**Billing:** Charged per instance-hour (same model as Shared), plus a dedicated per-region fee of approximately **$2/hour per region** (charged once per region regardless of how many Dedicated Instances you're running), plus the normal per-instance-hour charge.

### When to Use Dedicated Instance

- Organizations that must comply with regulations requiring hardware-level tenant isolation
- Healthcare companies under HIPAA who need physical isolation assurance
- Government contractors under FedRAMP or ITAR requirements
- Financial institutions with internal security policies requiring dedicated hardware
- Any workload where you need to prove to an auditor that your data was never on shared hardware
- When you need isolation guarantees but don't need control over the physical host

---

## Option 3 — Dedicated Host

### How It Works

A Dedicated Host is a **physical server that you fully allocate to your AWS account and have significant control over**. You are not just guaranteed isolation — you are reserving an entire specific physical machine, and you know its exact hardware specifications.

```
Physical Host Server (YOU allocated this specific machine)
┌─────────────────────────────────────────────────────────┐
│  You know:                                              │
│  • Exact number of physical CPU sockets (e.g., 2)      │
│  • Exact number of physical cores per socket (e.g., 24) │
│  • How many instances it can host                       │
│  • Which specific host ID this is (h-0abc123def)        │
│  • Which AZ it's in                                     │
├──────────────────────────────────────────────────────────┤
│      YOU DECIDE which instances go on this host         │
│                                                         │
│  Your VM 1    Your VM 2    Your VM 3    Your VM 4       │
│  (you placed  (you placed  (you placed  (you placed     │
│   it here)    it here)     it here)     it here)        │
└─────────────────────────────────────────────────────────┘
     AWS physically maintains the hardware,
     but YOU control instance placement on it
```

### The Critical Differentiator — Bring Your Own License (BYOL)

This is why Dedicated Host exists. Many enterprise software vendors — Microsoft, Oracle, IBM, SAP — license their software based on the **number of physical CPU cores or sockets** the software runs on. Their licensing agreements predate cloud computing and assume you know exactly what hardware you're using.

On Shared tenancy and even Dedicated Instances, you have no idea how many physical CPU cores or sockets the underlying server has. A license auditor from Microsoft or Oracle would reject your compliance claim.

With a Dedicated Host, you know precisely:
- The host model (e.g., `r6i.metal` — dual socket Intel Sapphire Rapids)
- The number of physical sockets (e.g., 2)
- The number of physical cores per socket (e.g., 32 cores per socket = 64 total physical cores)
- The number of vCPUs available (e.g., 128 vCPUs from 64 physical cores with hyperthreading)

**Microsoft Windows Server licensing (BYOL example):**
Microsoft licenses Windows Server per physical core. On a Dedicated Host, you know you have exactly 64 physical cores. You purchase 64 core licenses, install Windows on your instances, and your license audit passes. You cannot do this on Shared or Dedicated Instance — the physical core count is unknown.

**Oracle Database licensing (BYOL example):**
Oracle licenses its database per physical processor or per core with a "core factor" table. Oracle's auditors specifically require proof of physical hardware topology. Only Dedicated Host provides this.

**IBM Db2 and WebSphere:** Similarly licensed per physical CPU, requiring hardware visibility.

### How to Allocate a Dedicated Host

Unlike Shared and Dedicated Instance (which are just tenancy settings on regular instance launches), a Dedicated Host requires a separate allocation step before you can launch instances on it:

**Step 1 — Allocate the host:**
1. EC2 Console → **Dedicated Hosts** (under "Instances" in left sidebar)
2. Click **"Allocate Dedicated Host"**
3. Configure:
   - **Instance family** — which family of instances will run on this host (e.g., `r6i`, `c6i`, `m6i`)
   - **Instance type** — the specific type (e.g., `r6i.xlarge`) OR select the metal type for full host flexibility
   - **Availability Zone** — which AZ to place the physical host in
   - **Quantity** — how many physical hosts to allocate
   - **Host tenancy** — Dedicated (default)
   - **Auto placement** — whether instances launched with Dedicated Instance tenancy can auto-land on this host
   - **Host recovery** — whether AWS should automatically replace a failed host
4. Click **"Allocate"**

AWS provisions a specific physical server in your account. You receive a **Host ID** (e.g., `h-0abc123def456789`) — a permanent identifier for that specific physical machine.

**Step 2 — Launch instances targeting the host:**
```bash
# CLI example: launch instance on a specific Dedicated Host
aws ec2 run-instances \
  --image-id ami-051a31ab2f4d498f5 \
  --instance-type r6i.xlarge \
  --placement "Tenancy=host,HostId=h-0abc123def456789" \
  --key-name my-key
```

Or in the console: Launch instance → Advanced details → Tenancy = Dedicated Host → select Host ID.

### Host Recovery

**Host Recovery** is an optional feature on Dedicated Hosts. When enabled, if AWS detects hardware degradation on your Dedicated Host, it automatically allocates a new replacement host and migrates your instances to it — without you doing anything.

```
Original Host h-0abc123 detects hardware issue
       ↓
AWS allocates replacement Host h-0xyz789
       ↓
Instances migrated automatically
       ↓
You receive notification via CloudWatch Events
       ↓
Original host released/replaced
```

Without Host Recovery, a failing Dedicated Host would just fail, and you'd be responsible for detecting it and launching replacement instances manually.

### Key Properties of Dedicated Host

**Hardware control:** You see the exact host model, socket count, core count, and total instance capacity. You decide which instances run on which specific host. You can control NUMA (Non-Uniform Memory Access) topology for latency-sensitive applications.

**License compliance:** Full BYOL support. Provide hardware topology to software license auditors with confidence.

**Instance limit:** Each Dedicated Host has a fixed capacity based on its hardware model. For example, an `r6i` Dedicated Host can run a specific number of `r6i` instances depending on their sizes (the total vCPUs must not exceed the host's capacity). AWS provides a capacity calculator.

**Persistence:** Unlike regular instances that can move between physical hosts when stopped/started, instances on a Dedicated Host stay on **that specific host** when stopped and started. The instance ID changes but the physical host doesn't.

**Placement Groups:** Dedicated Hosts **cannot** be placed in Placement Groups (Cluster, Spread, or Partition). This is a hard limitation — if you need placement groups, you must use Shared or Dedicated Instance tenancy.

**Cost:** Most expensive option. You pay for the **entire physical host** regardless of how many instances run on it. Pricing is per host-hour and varies by host family. An `m6i` Dedicated Host costs approximately $3–4/hour continuously, whether you run 1 instance on it or 40.

---

## The Key Distinctions — Dedicated Instance vs Dedicated Host

This is the most commonly confused comparison. Memorize these differences:

| Distinction | Dedicated Instance | Dedicated Host |
|---|---|---|
| Who controls the physical host? | AWS | You |
| Do you know the Host ID? | No | Yes |
| Do you know physical core/socket count? | No | Yes |
| Can you control instance placement on host? | No | Yes |
| Supports BYOL licenses? | No | Yes |
| Instance stays on same host after stop/start? | Not guaranteed | Yes (same host) |
| Can use Placement Groups? | Yes | No |
| Billing model | Per instance + $2/region/hour | Per physical host/hour |
| Hardware visibility | None | Full |

---

## Compliance Requirements — Which Tenancy They Require

| Regulation / Standard | Shared | Dedicated Instance | Dedicated Host |
|---|---|---|---|
| General commercial use | ✅ | ✅ | ✅ |
| HIPAA (healthcare) | ✅ with BAA | ✅ | ✅ |
| PCI-DSS (payments) | ✅ | ✅ | ✅ |
| FedRAMP (US govt moderate) | ✅ | ✅ | ✅ |
| FedRAMP High / DoD IL4/IL5 | ❌ | ✅ | ✅ |
| ITAR (defense export) | ❌ | ✅ | ✅ |
| Custom security policy requiring hardware isolation | ❌ | ✅ | ✅ |
| BYOL (Oracle, Microsoft, IBM) | ❌ | ❌ | ✅ |
| Need hardware topology for audit | ❌ | ❌ | ✅ |

---

## Cost Comparison

Understanding the pricing model for each tenancy is critical for architectural decisions:

**Shared Tenancy:**
```
Cost = Instance hourly rate
Example: t3.medium = $0.0416/hour
You pay only for running instances.
```

**Dedicated Instance:**
```
Cost = Instance hourly rate (slightly higher than shared)
    + $2.00/hour per region (flat fee, charged once per region per hour)
    
Example: Dedicated t3.medium ≈ $0.0520/hour per instance
       + $2.00/hour region fee (regardless of instance count)
       
With 10 dedicated instances:
  = (10 × $0.0520) + $2.00 = $2.52/hour
```

**Dedicated Host:**
```
Cost = Per host/hour rate (regardless of instances running on it)

Example: m6i Dedicated Host ≈ $3.528/hour
         (you can run up to 64 m6i.xlarge on one m6i host)
         
Running 1 instance:  Pay $3.528/hour
Running 64 instances: Pay $3.528/hour (same cost!)

Break-even: If running enough instances to fill the host,
Dedicated Host becomes cheaper per instance than Dedicated Instance.
```

**Cost optimization with Dedicated Hosts:**

Dedicated Hosts support both **On-Demand** and **Reserved** pricing:
- **On-Demand:** Pay per hour with no commitment
- **1-Year Reserved:** Up to 30% discount vs On-Demand
- **3-Year Reserved:** Up to 60% discount vs On-Demand

For long-running BYOL workloads, a 3-year Reserved Dedicated Host combined with BYOL licenses (vs paying AWS's Windows/SQL Server licensing) can result in dramatic cost savings compared to running licensed instances on Shared tenancy.

---

## Selecting Tenancy — Decision Framework

```
Do you have regulatory/compliance requirements
mandating hardware isolation from other tenants?
        │
        ├── NO → Use SHARED (cheapest, simplest)
        │
        └── YES ──→ Do you need to manage software licenses
                    based on physical CPU/socket counts? (BYOL)
                            │
                            ├── NO → Use DEDICATED INSTANCE
                            │         (isolation without host control)
                            │
                            └── YES → Use DEDICATED HOST
                                      (full hardware visibility + BYOL)
```

---

## Summary — Everything at a Glance

| Property | Shared | Dedicated Instance | Dedicated Host |
|---|---|---|---|
| Physical host shared with other accounts? | Yes | No | No |
| AWS manages physical host? | Yes | Yes | Yes (physically) |
| You control instance placement? | No | No | Yes |
| Physical core/socket visibility? | No | No | Yes |
| BYOL support? | No | No | Yes |
| Placement Groups supported? | Yes | Yes | No |
| Instance same host after stop/start? | Not guaranteed | Not guaranteed | Guaranteed |
| Host Recovery available? | N/A | N/A | Yes |
| Billing model | Per instance | Per instance + region fee | Per host |
| Relative cost | Lowest | Medium | Highest |
| Best for | Everything general | Compliance, isolation | BYOL, full control, audit |
| Interview tip | Default for all EC2 | Isolation without control | Control + BYOL |
