# AWS EC2 Placement Groups — Complete In-Depth Guide

---

## The Problem Placement Groups Solve

When you launch EC2 instances, AWS decides where to physically place them across its data center infrastructure. By default, AWS spreads your instances across different physical hardware automatically — this is good for general availability but it means AWS optimizes for its own infrastructure management, not necessarily for your application's specific needs.

Some applications need something different:

- A **high-performance computing cluster** needs instances physically inches apart with the fastest possible network between them — AWS's default spreading hurts performance here.
- A **mission-critical application** needs instances so isolated from each other that even a physical rack power failure can't take down more than one — default placement doesn't guarantee this level of isolation.
- A **large distributed database** like Cassandra or Hadoop needs to know exactly which "failure domain" each node belongs to, so it can replicate data across domains intelligently.

**Placement Groups** give you control over how AWS physically places your EC2 instances on hardware. Instead of letting AWS decide everything, you declare a strategy — and AWS honors it.

---

## Understanding the Physical Infrastructure

To understand placement groups deeply, you need a mental model of the physical layers inside an AWS data center:

```
AWS Region (e.g., ap-south-1 Mumbai)
    └── Availability Zone 1 (ap-south-1a)  ←── Physical data center building
    └── Availability Zone 2 (ap-south-1b)  ←── Different building
    └── Availability Zone 3 (ap-south-1c)  ←── Yet another building
    
Inside one Availability Zone:
    ├── Row A
    │     ├── Rack 1  ─── [Server] [Server] [Server] [Server]
    │     │                  ↑         ↑
    │     │           These share the rack's power supply,
    │     │           Top-of-Rack (ToR) network switch, and cooling
    │     ├── Rack 2  ─── [Server] [Server] [Server] [Server]
    │     └── Rack 3  ─── [Server] [Server] [Server] [Server]
    ├── Row B
    │     ├── Rack 4  ─── [Server] [Server] [Server] [Server]
    │     └── Rack 5  ─── [Server] [Server] [Server] [Server]
    └── ...
```

**The key insight:** Servers in the same rack share physical infrastructure — the same power distribution unit (PDU), the same Top-of-Rack network switch, and sometimes the same cooling units. If any of these shared components fails, every server in that rack is affected simultaneously.

Servers in different racks are independently powered and networked. A rack failure in Rack 1 has zero effect on Rack 2.

Servers in different AZs are in entirely separate buildings with separate power grids, separate network uplinks, and separate physical security.

**Network latency also depends on physical distance:**
- Same server → sub-microsecond (loopback)
- Same rack → ~10–50 microseconds
- Same AZ, different rack → ~100–500 microseconds
- Different AZ → ~1–3 milliseconds
- Different Region → ~50–200+ milliseconds

Placement Groups allow you to optimize across this physical topology.

---

## What is a Placement Group?

A **Placement Group** is a logical grouping in AWS that tells the EC2 service where and how to physically place the instances you launch into it, relative to each other.

Key facts:
- **Free to create and use** — no additional charge for placement groups
- **Region and account specific** — a placement group in `ap-south-1` cannot be used in `us-east-1`
- **Name must be unique** per AWS account per region
- **Cannot be merged** — two separate placement groups cannot be combined into one
- **One placement group per instance** — an instance can be in only one placement group at a time
- **Cannot move running instances** — to change an instance's placement group, you must stop it, move it, and start it again
- **Dedicated Hosts cannot use placement groups** — physical dedicated servers are managed differently

There are exactly **three types** of placement groups, each designed for a completely different use case.

---

## Type 1 — Cluster Placement Group

### The Philosophy — Sacrifice Resilience for Speed

A Cluster Placement Group packs all your instances as physically close together as possible — ideally in the same rack, in the same AZ. The entire goal is to minimize the physical distance between instances, which directly translates to lower network latency and higher network throughput between them.

```
Cluster Placement Group
         │
         ▼
┌─────────────────────────┐
│   Single Rack (or few   │
│   adjacent racks in     │
│   same AZ)              │
│                         │
│  [Instance A]           │
│  [Instance B]           │
│  [Instance C]           │
│  [Instance D]           │
│  [Instance E]           │
│                         │
│  All share:             │
│  • Same ToR switch      │
│  • High-speed backplane │
│  • Enhanced networking  │
└─────────────────────────┘
         │
  Network speed between instances:
  Up to 10 Gbps (standard)
  Up to 25 Gbps (with enhanced networking)
  Up to 100 Gbps (with Elastic Fabric Adapter)
```

### Network Performance — Why Proximity Matters So Much

Normal network communication between EC2 instances in the same AZ traverses multiple network switches, routers, and potentially the AZ's spine network. Each hop adds microseconds of latency and potential congestion points.

With a Cluster Placement Group, AWS places instances on the same physical network segment — often the same Top-of-Rack switch. Traffic between instances may never leave the rack's internal network fabric. This is why:

- **Latency** drops to single-digit microseconds between instances (vs. hundreds of microseconds normally)
- **Throughput** reaches maximum possible speeds — 10 Gbps standard, 25 Gbps with enhanced networking, 100 Gbps with **Elastic Fabric Adapter (EFA)**
- **Packet loss** and **jitter** are minimized

### Elastic Fabric Adapter (EFA)

EFA is AWS's specialized network interface card that bypasses the operating system's network stack entirely for certain traffic types. Combined with a Cluster Placement Group, EFA provides **RDMA (Remote Direct Memory Access)** — one instance can directly read and write to another instance's memory without involving either instance's CPU. This is what makes extremely tight HPC clusters possible on AWS.

### The Fatal Flaw — Correlated Failure

Everything in a Cluster Placement Group shares physical infrastructure:

```
Power Distribution Unit fails
         ↓
Entire rack loses power
         ↓
ALL instances in the Cluster Placement Group go down simultaneously
         ↓
Your entire application is offline
```

This is not a theoretical risk — rack-level power failures, network switch failures, and cooling failures happen in data centers. In a Cluster Placement Group, a single hardware failure can simultaneously kill every instance you have.

**This is an acceptable trade-off for the right workload** — HPC jobs, financial calculations, and scientific simulations that need maximum performance and can restart if something fails. It is completely unacceptable for anything customer-facing that must stay online.

### Ideal Use Cases for Cluster Placement Group

**High Performance Computing (HPC):** Scientific simulations, computational fluid dynamics, finite element analysis, weather modeling — workloads that divide a problem across hundreds of tightly-coupled compute nodes that communicate constantly. The MPI (Message Passing Interface) protocol used by most HPC applications is extremely sensitive to inter-node latency. Cluster Placement makes these jobs feasible.

**Financial risk modeling:** Monte Carlo simulations, options pricing, risk calculations — these divide massive calculations across many instances that share intermediate results. Speed of communication is the bottleneck.

**Machine learning training (distributed):** Training large neural networks across multiple GPU instances using NCCL (NVIDIA Collective Communications Library) requires extremely fast GPU-to-GPU communication. With EFA and Cluster Placement, AWS's p4d and p5 instances can achieve near-theoretical maximum inter-node GPU communication speeds.

**Genomics sequencing:** Processing a whole human genome involves many parallel computation steps that exchange data constantly. Cluster Placement dramatically reduces total analysis time.

**Video rendering farms:** Individual frames of a complex 3D scene are rendered in parallel across many instances, then combined. Tight networking reduces the time to distribute work and collect results.

### Cluster Placement Group — Key Specs

| Property | Value |
|---|---|
| Physical location | Same rack / adjacent racks, single AZ |
| Network speed | Up to 100 Gbps (with EFA) |
| Latency | Single-digit microseconds |
| Fault tolerance | Very low — single rack failure takes everything |
| Spans multiple AZs? | No — single AZ only |
| Instance limit | No explicit limit (hardware availability dependent) |
| Best for | HPC, ML training, financial modeling |

---

## Type 2 — Spread Placement Group

### The Philosophy — Sacrifice Density for Resilience

A Spread Placement Group is the philosophical opposite of Cluster. Instead of packing instances together, it spreads each instance onto **distinct, isolated physical hardware** — different racks with different power supplies and different network switches.

The goal is to ensure that no single hardware failure can affect more than one instance simultaneously.

```
Spread Placement Group across 3 AZs

AZ-1 (ap-south-1a)          AZ-2 (ap-south-1b)          AZ-3 (ap-south-1c)
┌────────────────┐           ┌────────────────┐           ┌────────────────┐
│ Rack 1         │           │ Rack 4         │           │ Rack 7         │
│ [Instance A]   │           │ [Instance C]   │           │ [Instance E]   │
│                │           │                │           │                │
│ Rack 2         │           │ Rack 5         │           │ Rack 8         │
│ [Instance B]   │           │ [Instance D]   │           │ [Instance F]   │
└────────────────┘           └────────────────┘           └────────────────┘

Every instance is on a different rack.
Every rack has independent power and networking.
Instances are also in different AZs for additional isolation.

If Rack 1 loses power → Only Instance A is affected
If AZ-1 goes down entirely → Only Instances A and B are affected
```

### The Hard Limit — 7 Instances Per AZ

This is the most important constraint to memorize for exams and architecture design.

**A Spread Placement Group can only have 7 instances per Availability Zone.**

Why 7? AWS guarantees that each instance in a Spread Placement Group is on completely distinct hardware. To make this guarantee, AWS needs to reserve isolated hardware for each slot. The limit of 7 represents AWS's commitment to ensure true hardware isolation for each instance.

```
Mumbai region (ap-south-1) has 3 AZs:

ap-south-1a: maximum 7 instances
ap-south-1b: maximum 7 instances
ap-south-1c: maximum 7 instances
─────────────────────────────────
Total maximum: 21 instances per Spread Placement Group
```

This hard limit makes Spread Placement Groups unsuitable for large fleets. You cannot run 100 web servers in a Spread Placement Group. It is designed for a small number of highly critical instances that absolutely must survive any single hardware failure.

### Failure Isolation Analysis

```
Scenario: You have 6 instances in a Spread Placement Group across 2 AZs

Without Spread Placement:
  Rack failure → could take down 3+ instances simultaneously
  AZ failure → takes down all instances in that AZ

With Spread Placement:
  Rack failure → takes down exactly 1 instance (guaranteed)
  AZ failure → takes down exactly 3 instances (the ones in that AZ)
  Remaining instances in other AZ: 3 (still serving traffic)
```

### Ideal Use Cases for Spread Placement Group

**Domain Controllers (Active Directory):** If you have 3 DC instances and they all end up on the same rack, a rack power failure takes out all 3 simultaneously — exactly the catastrophic scenario that makes Spread Placement essential here.

**Zookeeper / etcd clusters:** These distributed coordination services rely on quorum — a majority of nodes must be alive. With 3 nodes in a Spread Placement Group, even a hardware failure can only affect 1 node, leaving quorum intact.

**Kafka brokers:** Message queue brokers need to replicate data across multiple nodes. Spread placement ensures broker failures are independent.

**Primary + Standby database pairs:** Put the primary database and its hot standby in a Spread Placement Group across different AZs — guarantees they're never on the same hardware.

**Critical microservices:** Specific services (authentication, payment processing, session management) that must never all go down simultaneously.

### Spread Placement Group — Key Specs

| Property | Value |
|---|---|
| Physical location | Distinct racks, can span multiple AZs |
| Network speed | Standard (no special proximity) |
| Latency | Standard inter-instance latency |
| Fault tolerance | Maximum — each instance isolated |
| Spans multiple AZs? | Yes — strongly recommended |
| Instance limit | **7 per AZ (hard limit)** |
| Best for | Small number of critical instances |

---

## Type 3 — Partition Placement Group

### The Philosophy — Structured Isolation for Large Distributed Systems

Partition Placement Group is the most sophisticated of the three types. It was designed specifically for large-scale distributed systems — applications that consist of hundreds or thousands of nodes that need to be aware of their own failure domain membership.

A Partition Placement Group divides your instances into **logical partitions**, where each partition runs on a separate set of physical racks with independent power and networking. Instances within the same partition share hardware, but instances in different partitions never share hardware.

```
Partition Placement Group with 3 Partitions in one AZ

Partition 1          Partition 2          Partition 3
(Rack Set A)         (Rack Set B)         (Rack Set C)
Independent power    Independent power    Independent power
Independent network  Independent network  Independent network

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ [Instance A] │    │ [Instance D] │    │ [Instance G] │
│ [Instance B] │    │ [Instance E] │    │ [Instance H] │
│ [Instance C] │    │ [Instance F] │    │ [Instance I] │
│              │    │              │    │              │
│ No limit on  │    │ No limit on  │    │ No limit on  │
│ instances    │    │ instances    │    │ instances    │
└──────────────┘    └──────────────┘    └──────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
       Partition 1 failure has ZERO effect on 2 or 3
```

### How Many Partitions?

- **Up to 7 partitions per Availability Zone** (same number as Spread's instance limit — this is intentional)
- **No limit on instances per partition** (unlike Spread's 7 instance limit)
- **Can span multiple AZs** — you can have partitions in different AZs

This means Partition Placement Groups can theoretically host **thousands of instances** while still providing fault domain isolation — something Spread Placement Groups cannot do.

### The Key Differentiator — Partition Metadata Awareness

Unlike Cluster and Spread, Partition Placement Groups expose **partition membership information** to the instances themselves. Using the EC2 metadata service, each instance can query which partition it belongs to:

```bash
# From inside an EC2 instance in a Partition Placement Group:
curl http://169.254.169.254/latest/meta-data/placement/partition-number

# Returns: 1, 2, or 3 (the partition number this instance is in)
```

This is crucial for distributed systems. Here is why:

**Cassandra topology-aware replication:** Cassandra replicates each piece of data across multiple nodes. If you tell Cassandra about your partition structure, it can ensure that replicas are placed in different partitions. If Partition 1 fails, Cassandra knows which other partitions have the data and can continue serving reads and writes.

**Hadoop HDFS rack awareness:** HDFS (Hadoop Distributed File System) uses "rack awareness" to ensure replicas of each data block are stored on different racks. By mapping Hadoop "racks" to AWS partitions, HDFS can make intelligent replication decisions. Default HDFS replication (3 copies) across 3 partitions means any single partition failure has zero data loss.

**Kafka partition leaders:** Kafka distributes partition leadership across brokers. By spreading brokers across AWS partitions and configuring Kafka to be partition-aware, you ensure that a single hardware failure cannot take down more than a predictable fraction of Kafka leadership.

### Partition vs Spread — The Key Decision

These two types are often confused. The distinction is:

| Question | Spread | Partition |
|---|---|---|
| How many instances? | Small (max 7/AZ) | Large (thousands) |
| Each instance isolated? | Yes — own dedicated rack | No — instances share racks within a partition |
| Application needs fault domain info? | No | Yes |
| Use case scale | A few critical nodes | Hundreds to thousands of nodes |

Choose **Spread** when you have a small number (3–7) of instances that must each be individually isolated.

Choose **Partition** when you have a large distributed system that manages its own replication and needs to know which failure domain each node is in.

### Ideal Use Cases for Partition Placement Group

**Apache Hadoop (HDFS + YARN):** HDFS block replication across partitions, YARN resource allocation with topology awareness.

**Apache Cassandra:** Partition-aware token range assignment and replication strategy.

**Apache Kafka:** Broker distribution across partitions for leader election resilience.

**Apache HBase:** Region server distribution for fault-tolerant data distribution.

**Elasticsearch:** Shard allocation across partitions for resilient search clusters.

All of these are large-scale distributed systems with their own internal replication mechanisms. They all need to know about failure domains to make intelligent data placement decisions — and Partition Placement Groups provide exactly that information through the metadata service.

### Partition Placement Group — Key Specs

| Property | Value |
|---|---|
| Physical location | Per-partition rack sets, partitions are isolated |
| Network speed | High within partition, standard across partitions |
| Latency | Low within partition, standard across partitions |
| Fault tolerance | High — partition-level isolation |
| Spans multiple AZs? | Yes |
| Instance limit | No limit per partition |
| Partition limit | **7 partitions per AZ** |
| Metadata visibility | Yes — instances know their partition number |
| Best for | Large distributed systems (Hadoop, Cassandra, Kafka) |

---

## How to Create a Placement Group

### In the AWS Console

1. EC2 Console → **Network & Security** → **Placement Groups** (left sidebar)
2. Click **"Create placement group"**
3. Fill in:
   - **Name** — unique within your account and region (e.g., `hpc-cluster-prod`, `critical-nodes-spread`)
   - **Placement strategy** — select Cluster, Spread, or Partition
   - **For Partition:** also specify the number of partitions (1–7)
4. Click **"Create group"**

### Using the AWS CLI

```bash
# Create a Cluster Placement Group
aws ec2 create-placement-group \
  --group-name hpc-cluster-prod \
  --strategy cluster

# Create a Spread Placement Group
aws ec2 create-placement-group \
  --group-name critical-nodes-spread \
  --strategy spread

# Create a Partition Placement Group with 3 partitions
aws ec2 create-placement-group \
  --group-name hadoop-cluster-partition \
  --strategy partition \
  --partition-count 3
```

### Launching an Instance Into a Placement Group

**Console:** During EC2 launch → Advanced details → Placement group → select your group

**CLI:**
```bash
aws ec2 run-instances \
  --image-id ami-051a31ab2f4d498f5 \
  --instance-type c6i.2xlarge \
  --count 5 \
  --placement "GroupName=hpc-cluster-prod" \
  --key-name my-key
```

**For Partition Placement Group — specify the partition:**
```bash
aws ec2 run-instances \
  --image-id ami-051a31ab2f4d498f5 \
  --instance-type r6i.xlarge \
  --count 10 \
  --placement "GroupName=hadoop-cluster-partition,PartitionNumber=2" \
  --key-name my-key
```

---

## Moving an Instance Into a Placement Group

You cannot move a running instance directly. The process:

```
1. Stop the instance (not terminate)
2. Modify the instance's placement group:
   CLI: aws ec2 modify-instance-placement \
          --instance-id i-0abc123 \
          --group-name my-placement-group
3. Start the instance
```

Note: After moving, the instance may land on different physical hardware. Its public IP will change (unless Elastic IP). Private IP and EBS data are preserved.

---

## Placement Group Rules and Constraints

**Naming:** Must be unique within the same AWS account and region. You cannot have two placement groups with the same name in ap-south-1, but you can have the same name in us-east-1.

**Cannot be merged:** Once created, two separate placement groups cannot be combined. If you want instances from two groups together, you must recreate them in a single group.

**One group per instance:** An instance can only belong to one placement group at a time.

**Dedicated Hosts excluded:** Physical servers dedicated to your account (Dedicated Hosts) cannot be placed in any placement group. Dedicated Instances (software-isolated but not physically dedicated) can use placement groups.

**Insufficient capacity errors:** For Cluster Placement Groups especially, AWS may return an "InsufficientInstanceCapacity" error because it physically cannot fit all requested instances into adjacent hardware. Solution: stop all instances in the group and restart them together — AWS finds a new physical location that fits everyone.

**Instance type homogeneity:** While not strictly required, Cluster Placement Groups work best with the same instance type. Mixed types may result in sub-optimal physical placement.

**Reserved Instances and Placement Groups:** RIs can be used with placement groups. The RI discount applies normally, and instances benefit from placement group properties.

---

## Choosing the Right Placement Group

```
What is your primary concern?

├── Maximum network performance between instances?
│     └── CLUSTER PLACEMENT GROUP
│           ├── HPC workloads
│           ├── ML training (multi-GPU)
│           ├── Financial simulations
│           └── Accept: low fault tolerance, single AZ only
│
├── Maximum fault isolation for a small number of critical instances?
│     └── SPREAD PLACEMENT GROUP
│           ├── Domain Controllers, ZooKeeper, etcd
│           ├── Primary/Standby database pairs
│           ├── Critical infrastructure nodes
│           └── Accept: hard limit of 7 instances per AZ
│
└── Large distributed system that manages its own replication?
      └── PARTITION PLACEMENT GROUP
            ├── Hadoop, Cassandra, Kafka, HBase, Elasticsearch
            ├── Need fault domain metadata awareness
            ├── Hundreds to thousands of instances
            └── Accept: some complexity in configuration
```

---

## Complete Comparison Table

| Property | Cluster | Spread | Partition |
|---|---|---|---|
| Primary goal | Maximum performance | Maximum isolation | Fault-domain awareness |
| Physical placement | Same rack, close together | Each instance on distinct rack | Groups of racks isolated per partition |
| Network speed | Very High (up to 100 Gbps) | Standard | High within partition, standard across |
| Latency between instances | Microseconds | Milliseconds | Low within partition |
| Fault tolerance | Very Low | Maximum | High |
| Spans multiple AZs? | No — single AZ only | Yes | Yes |
| Instance limit | No explicit limit | 7 per AZ (21 in 3-AZ region) | No limit per partition |
| Partition/group limit | N/A | N/A | 7 partitions per AZ |
| Metadata awareness | No | No | Yes — partition number visible |
| Dedicated Hosts | Not supported | Not supported | Not supported |
| Best instance types | C, P, G (compute/GPU) | Any | Any |
| Use cases | HPC, ML training, rendering | Critical nodes, DCs, ZooKeeper | Hadoop, Cassandra, Kafka |
| Cost | Free (no extra charge) | Free | Free |
| Can span regions? | No | No | No |
