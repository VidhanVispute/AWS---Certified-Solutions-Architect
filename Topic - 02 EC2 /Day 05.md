# Types of AWS EC2 Instances — In Depth

---

## What is an Instance Type?

When you launch an EC2 instance, you are essentially renting a virtual computer. The **instance type** defines exactly what that computer looks like — how many CPU cores it has, how much RAM, what kind of storage, and how fast its network is.

Think of it like buying a laptop. You wouldn't buy a gaming laptop to write documents, and you wouldn't buy a basic laptop to render 4K videos. The same logic applies to EC2 — you match the instance to the job.

Every instance type follows a naming pattern:

```
[Family] [Generation] [Attributes] . [Size]

Example:  m  6  i  .  2xlarge
          ↓   ↓  ↓       ↓
       Memory  6th  Intel  Double XL
       Family  Gen  chip    size
```

---

## Instance Type Naming Convention

**Family letter** — tells you the purpose (t, m, c, r, etc.)

**Generation number** — higher = newer hardware, better performance, usually cheaper per unit of compute. Always prefer the latest generation (e.g., m7 over m5).

**Attribute letters** (optional, appear after generation):
- **a** — AMD processor (cheaper than Intel, same performance)
- **g** — AWS Graviton (Arm-based, 20–40% cheaper)
- **i** — Intel processor
- **n** — Higher network bandwidth
- **d** — Locally attached NVMe SSD storage
- **e** — Extra memory or storage capacity
- **z** — High frequency (fast single-core clock speed)

**Size** — determines how much of everything you get:
```
nano → micro → small → medium → large → xlarge → 2xlarge → 4xlarge → 8xlarge → 12xlarge → 16xlarge → 24xlarge → 48xlarge → metal
```
Each step up roughly doubles the vCPU and RAM. Metal means bare metal — no hypervisor, direct hardware access.

---

## The Five Main Instance Families

---

### 1. General Purpose — T and M Families

These are the most commonly used instance types. They provide a **balanced ratio of CPU to RAM** — neither extreme, suited to most typical workloads.

**T Family — Burstable Performance**

The T family is unique because it uses a **credit-based CPU model**. When your instance is idle or lightly loaded, it earns CPU credits. When you need a burst of processing (like handling a sudden spike of web traffic), it spends those credits to temporarily use more CPU than its baseline.

- **Baseline CPU** — The guaranteed minimum CPU performance at all times.
- **Burst CPU** — The higher CPU level available when you have credits.
- **Credit modes:**
  - **Standard** — When credits run out, CPU throttles back to baseline. Safe, predictable cost.
  - **Unlimited** — Bursts continue even without credits but you're charged extra for the burst usage.

T instances are the cheapest EC2 instances. Perfect for workloads that don't need constant full CPU — web servers, dev/test environments, small databases, microservices, blogs.

Current T generations: **t2, t3, t3a, t4g**
- t3a = same as t3 but AMD CPU (slightly cheaper)
- t4g = Graviton Arm processor (cheapest T option)

**M Family — Standard General Purpose**

The M family provides **consistent, non-burstable** compute. You always get the full CPU you pay for, every second — no credits, no throttling. M stands for "Most scenarios" — AWS positions it as the default for workloads that need steady, predictable performance.

Ideal for: application servers, backend services, medium-traffic web apps, small-to-medium databases, enterprise applications.

Current M generations: **m5, m5a, m5n, m6i, m6a, m6g, m7i, m7a, m7g**

---

### 2. Compute Optimized — C Family

C instances have a **higher ratio of vCPU to RAM** compared to general purpose. They are built specifically for workloads where raw processing power is the bottleneck — where you need more CPU but don't necessarily need proportionally more memory.

The C family uses the fastest available processors AWS offers, often with higher clock speeds than equivalent M instances.

Ideal for:
- Batch processing jobs (processing thousands of records, images, files)
- High-performance web servers under heavy concurrent load
- Video encoding and transcoding
- Scientific modeling and simulations
- Competitive gaming servers
- Machine learning inference (running a trained model, not training it)
- High-performance computing (HPC)

Current C generations: **c5, c5a, c5n, c6i, c6a, c6g, c6gn, c7i, c7a, c7g**

The **c6gn** has specialized high-bandwidth networking specifically for network-intensive compute workloads.

---

### 3. Memory Optimized — R, X, z, u Families

Memory-optimized instances have a **very high RAM to vCPU ratio**. Use these when your application needs to load and process massive amounts of data entirely in memory.

**R Family — Memory Optimized (Standard)**

R stands for RAM. These are the most commonly used memory-optimized instances. They offer roughly double the RAM of equivalent M instances at a comparable price per GB of memory.

Ideal for:
- In-memory databases (Redis, Memcached, SAP HANA)
- Real-time big data analytics
- High-performance relational databases (MySQL, PostgreSQL with large buffer pools)
- Genome sequencing and bioinformatics
- Caching layers at scale

Current R generations: **r5, r5a, r5b, r5n, r6i, r6a, r6g, r7i, r7a, r7g**

The **r5b** has the highest EBS bandwidth of any R instance — useful for databases that need extremely fast disk I/O alongside large memory.

**X Family — Extreme Memory**

X instances have enormous amounts of RAM — up to **6 TB** in a single instance. Built for the most memory-hungry enterprise workloads on the planet.

Ideal for:
- SAP HANA (in-memory ERP systems)
- Apache Spark in-memory big data processing
- Very large in-memory databases
- High-performance computing that requires terabytes of working memory

Current: **x1, x1e, x2idn, x2iedn, x2iezn**

**z1d Family — High Frequency**

A unique family that combines high memory with extremely high single-core clock speeds (up to 4.0 GHz). Most EC2 instance types run at moderate clock speeds. z1d is for workloads where **per-core performance** matters more than total core count — typically licensing-heavy enterprise software that charges per CPU core.

Ideal for: Electronic Design Automation (EDA), financial simulations, per-core licensed software.

---

### 4. Storage Optimized — I, D, H Families

These instances are built for workloads that need either extremely fast local storage I/O or massive local storage capacity. They come with **physically attached NVMe SSDs or HDDs** directly on the host — no network hop to EBS.

**I Family — High IOPS NVMe SSD**

The I family provides the highest local storage I/O performance AWS offers. The local NVMe SSDs deliver millions of IOPS with sub-millisecond latency — impossible to match with network-attached EBS.

Important caveat: local instance storage is **ephemeral** — data is lost when the instance stops. These instances are used when you can afford to lose the local data (because you replicate it, or it's temporary), but need maximum speed.

Ideal for:
- NoSQL databases (Cassandra, MongoDB, DynamoDB local)
- OLTP databases requiring millions of IOPS
- Elasticsearch clusters
- Caching (when Redis needs more than memory)

Current: **i3, i3en, i4i, i4g**

**D Family — Dense HDD Storage**

D instances offer the highest local HDD storage density on EC2 — up to **336 TB of local HDD** in a single instance. Sequential read/write throughput is very high (though IOPS are lower than NVMe SSD). Storage cost per TB is very low.

Ideal for:
- Hadoop distributed file system (HDFS)
- Data warehousing
- Distributed file systems
- Log and data analytics processing massive sequential datasets

Current: **d2, d3, d3en**

**H Family — High Disk Throughput**

A balance between compute and high sequential disk throughput. Less storage than D but better compute.

Ideal for: MapReduce workloads, distributed file systems, scalable data analytics.

---

### 5. Accelerated Computing — P, G, Inf, Trn, F, VT Families

These instances attach specialized hardware accelerators — GPUs, FPGAs, or AWS custom chips — to massively speed up specific types of computation that CPUs handle poorly.

**P Family — GPU for Machine Learning Training**

P instances have high-end NVIDIA GPUs (A100, V100). Used for training large deep learning models. Training a neural network on a GPU is 100x faster than on CPU.

Ideal for:
- Training deep learning models (computer vision, NLP, generative AI)
- Scientific simulation
- Large-scale parallel computation

Current: **p3, p4, p5** (p5 has NVIDIA H100 GPUs — the most powerful available)

**G Family — GPU for Graphics and Inference**

G instances have NVIDIA GPUs but are less powerful (and cheaper) than P instances. Originally designed for graphics workloads, they are now widely used for ML inference (running trained models in production).

Ideal for:
- ML inference at scale
- Video transcoding and streaming
- 3D visualization, virtual desktops
- Game streaming

Current: **g4dn, g4ad, g5, g5g, g6**

**Inf Family — AWS Inferentia Chip**

AWS built its own chip called **Inferentia** specifically for running ML inference workloads. Cheaper than GPU instances for inference but cannot be used for training.

Current: **inf1, inf2**

**Trn Family — AWS Trainium Chip**

AWS built **Trainium** for ML model training — a custom chip designed to compete with NVIDIA A100 at lower cost for training large models.

Current: **trn1, trn1n**

**F Family — FPGA**

FPGA stands for Field Programmable Gate Array. Unlike CPUs or GPUs which are fixed circuits, an FPGA is a chip you can reprogram at the hardware level. You write hardware logic (using HDL languages) and flash it onto the chip to create custom hardware acceleration.

Ideal for:
- Real-time genomics processing
- Financial analytics (custom hardware trading algorithms)
- Video and image processing pipelines
- Network packet processing

Current: **f1**

**VT Family — Video Transcoding**

Purpose-built for real-time video transcoding using Xilinx FPGAs. Extremely high video stream throughput at low cost per stream.

Ideal for: Live streaming platforms (exactly like Hotstar cricket streaming), broadcast video processing.

---

## Hypervisor and Virtualization — How Instance Types Actually Work

Every EC2 instance (except bare metal) is a virtual machine. Here is what happens at the infrastructure level:

```
Your EC2 Instance (VM)
        ↓
   AWS Hypervisor (Nitro)
        ↓
Physical Host Server (CPU, RAM, NVMe, Network)
```

**The AWS Nitro Hypervisor** is AWS's custom-built, extremely lightweight hypervisor. Unlike traditional hypervisors (VMware, Xen) that run in software and consume CPU cycles, Nitro offloads virtualization functions to dedicated hardware cards. This means nearly zero overhead — your instance gets almost 100% of the raw hardware performance.

Nitro also handles:
- **Nitro Cards** — dedicated hardware for EBS I/O, VPC networking, and instance storage, freeing the main CPU entirely
- **Nitro Security Chip** — hardware root of trust, makes the physical host tamper-proof
- **NitroTPM** — Trusted Platform Module for Secure Boot

**Bare Metal instances** (`*.metal`) have no hypervisor at all. Your OS runs directly on the physical hardware. Used for:
- Workloads that need direct hardware access (custom hypervisors, VMware Cloud on AWS)
- License compliance that requires physical core counting
- Maximum performance with zero virtualization overhead

---

## EC2 Instance Types — Comprehensive Reference Table

| Family | Type | vCPU | Architecture | Memory (RAM) | Local Storage | Storage Type | Network Performance | Use Case |
|---|---|---|---|---|---|---|---|---|
| **General Purpose** | t2.micro | 1 | x86 | 1 GB | EBS only | — | Low to Moderate | Free tier, dev/test |
| | t3.medium | 2 | x86 | 4 GB | EBS only | — | Up to 5 Gbps | Small web servers |
| | t3a.medium | 2 | x86 (AMD) | 4 GB | EBS only | — | Up to 5 Gbps | Cost-optimized burst |
| | t4g.medium | 2 | Arm (Graviton2) | 4 GB | EBS only | — | Up to 5 Gbps | Cheapest burst instances |
| | m6i.large | 2 | x86 (Intel) | 8 GB | EBS only | — | Up to 12.5 Gbps | App servers, backends |
| | m6a.large | 2 | x86 (AMD) | 8 GB | EBS only | — | Up to 12.5 Gbps | Cost-optimized steady |
| | m6g.large | 2 | Arm (Graviton2) | 8 GB | EBS only | — | Up to 10 Gbps | Cost-optimized apps |
| | m7i.xlarge | 4 | x86 (Intel Sapphire) | 16 GB | EBS only | — | Up to 12.5 Gbps | Latest gen app server |
| **Compute Optimized** | c5.large | 2 | x86 | 4 GB | EBS only | — | Up to 10 Gbps | Batch jobs |
| | c6i.xlarge | 4 | x86 (Intel) | 8 GB | EBS only | — | Up to 12.5 Gbps | HPC, encoding |
| | c6g.xlarge | 4 | Arm (Graviton2) | 8 GB | EBS only | — | Up to 10 Gbps | Cost-optimized compute |
| | c6gn.xlarge | 4 | Arm (Graviton2) | 8 GB | EBS only | — | Up to 25 Gbps | High network compute |
| | c7g.2xlarge | 8 | Arm (Graviton3) | 16 GB | EBS only | — | Up to 15 Gbps | Latest compute Arm |
| | c7i.2xlarge | 8 | x86 (Intel Sapphire) | 16 GB | EBS only | — | Up to 15 Gbps | Latest compute Intel |
| **Memory Optimized** | r5.large | 2 | x86 | 16 GB | EBS only | — | Up to 10 Gbps | In-memory DB |
| | r6i.xlarge | 4 | x86 (Intel) | 32 GB | EBS only | — | Up to 12.5 Gbps | Redis, PostgreSQL |
| | r6g.xlarge | 4 | Arm (Graviton2) | 32 GB | EBS only | — | Up to 10 Gbps | Cost-optimized memory |
| | r7i.2xlarge | 8 | x86 (Intel Sapphire) | 64 GB | EBS only | — | Up to 15 Gbps | Latest gen memory |
| | r5b.xlarge | 4 | x86 | 32 GB | EBS only | — | Up to 10 Gbps | High EBS bandwidth DB |
| | x1e.xlarge | 4 | x86 | 122 GB | 1 × 120 GB SSD | NVMe SSD | Up to 10 Gbps | SAP HANA, extreme RAM |
| | x2idn.16xlarge | 64 | x86 (Intel Ice Lake) | 1024 GB | 1 × 1900 GB SSD | NVMe SSD | 50 Gbps | Terabyte-scale in-memory |
| | z1d.large | 2 | x86 (4.0 GHz) | 16 GB | 1 × 75 GB SSD | NVMe SSD | Up to 10 Gbps | Per-core licensed software |
| **Storage Optimized** | i3.large | 2 | x86 | 15.25 GB | 1 × 475 GB | NVMe SSD | Up to 10 Gbps | High IOPS NoSQL |
| | i3.2xlarge | 8 | x86 | 61 GB | 1 × 1900 GB | NVMe SSD | Up to 10 Gbps | Cassandra, Elasticsearch |
| | i4i.xlarge | 4 | x86 (Intel Ice Lake) | 32 GB | 1 × 937 GB | NVMe SSD | Up to 10 Gbps | Highest local IOPS |
| | i3en.large | 2 | x86 | 16 GB | 1 × 1250 GB | NVMe SSD | Up to 25 Gbps | Large NVMe + network |
| | d2.xlarge | 4 | x86 | 30.5 GB | 3 × 2000 GB | HDD | Moderate | Hadoop HDFS |
| | d3.xlarge | 4 | x86 | 32 GB | 4 × 2000 GB | HDD | Up to 15 Gbps | Dense big data |
| | d3en.xlarge | 4 | x86 | 16 GB | 2 × 7500 GB | HDD | Up to 25 Gbps | Maximum HDD density |
| **Accelerated Computing** | p3.2xlarge | 8 | x86 | 61 GB | EBS only | — | Up to 10 Gbps | ML training (V100 GPU) |
| | p4d.24xlarge | 96 | x86 | 1152 GB | 8 × 1000 GB SSD | NVMe SSD | 400 Gbps | Large model training |
| | p5.48xlarge | 192 | x86 | 2048 GB | 8 × 3840 GB SSD | NVMe SSD | 3200 Gbps | H100 GPUs, LLM training |
| | g4dn.xlarge | 4 | x86 | 16 GB | 1 × 125 GB | NVMe SSD | Up to 25 Gbps | ML inference, video |
| | g5.xlarge | 4 | x86 | 16 GB | 1 × 250 GB | NVMe SSD | Up to 10 Gbps | NVIDIA A10G GPU |
| | inf1.xlarge | 4 | x86 | 8 GB | EBS only | — | Up to 25 Gbps | AWS Inferentia chip |
| | inf2.xlarge | 4 | x86 | 16 GB | EBS only | — | Up to 15 Gbps | Inferentia v2 |
| | trn1.2xlarge | 8 | x86 | 32 GB | EBS only | — | Up to 12.5 Gbps | AWS Trainium training |
| | f1.2xlarge | 8 | x86 | 122 GB | 1 × 470 GB SSD | NVMe SSD | Up to 10 Gbps | FPGA custom hardware |
| **Bare Metal** | m6i.metal | 128 | x86 (Intel) | 512 GB | EBS only | — | 50 Gbps | VMware, no hypervisor |
| | r6i.metal | 128 | x86 (Intel) | 1024 GB | EBS only | — | 50 Gbps | Direct hardware memory |
| | i3.metal | 72 | x86 | 512 GB | 8 × 1900 GB | NVMe SSD | 25 Gbps | Physical host access |

---

## Quick Selection Guide

| Your workload needs... | Choose |
|---|---|
| Cheapest possible, light traffic | t3.micro / t4g.micro |
| Steady web/app server | m6i or m6g |
| CPU-heavy batch jobs | c6i or c7g |
| Large in-memory database | r6i or r7i |
| SAP HANA / terabyte RAM | x2idn or x1e |
| Millions of local disk IOPS | i4i |
| Huge HDD storage (Hadoop) | d3en |
| ML model training | p4d or trn1 |
| ML inference in production | g4dn or inf2 |
| Custom hardware logic | f1 |
| No hypervisor / VMware | *.metal |
