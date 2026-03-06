# AWS EC2 Purchase Options — Complete In-Depth Guide

---

## Why Purchase Options Matter

Running EC2 instances on AWS without thinking about purchase options is like always buying plane tickets at the full walk-up price — technically valid, but massively wasteful. AWS offers multiple ways to pay for compute, and the difference between the most expensive and cheapest option for the same instance can be **90% cost savings**. For a company spending $1 million/year on EC2, that's $900,000 saved annually.

The right purchase option depends entirely on one question: **how predictable and long-lived is this workload?**

---

## The Five Purchase Options — Overview

```
Most Expensive ←─────────────────────────────────→ Cheapest
     │                                                  │
On-Demand    Reserved     Savings Plans     Spot
(no commit)  (1-3 years)  (1-3 years,       (spare capacity,
                           flexible)         interruptible)
```

---

## Option 1 — On-Demand Instances

### How It Works

Pay for compute by the second (Linux) or by the hour (Windows), with zero commitment. Start and stop whenever you want. No contracts, no upfront payments, no discounts.

```
You need a server at 9 AM → Launch it → Pay from 9 AM
You stop it at 5 PM → Billing stops → Paid for 8 hours
You restart it tomorrow → New billing session begins
```

AWS bills you monthly in arrears — you use resources, then pay for what you used.

### Pricing Model

The per-second/per-hour rate is the **highest of all options**. This is the baseline price that all discounts are calculated against. When AWS says "Reserved Instances save 72%", that 72% is compared to On-Demand pricing.

### When to Use On-Demand

**Short-term, unpredictable workloads** — if you genuinely don't know how long you need the instance, On-Demand is the only responsible choice. Committing to a 1-year Reserved Instance and terminating it after 2 months wastes money.

**Development and testing** — dev servers get created, destroyed, resized, and reconfigured constantly. The flexibility of On-Demand matches this workflow perfectly.

**Application spikes** — traffic that spikes unpredictably beyond your baseline (covered by Reserved Instances) can overflow onto On-Demand instances temporarily.

**First-time deployments** — before you understand your usage patterns, start On-Demand. After 2–3 months of CloudWatch metrics, you'll know your baseline and can commit to Reserved Instances or Savings Plans with confidence.

### Key Properties

| Property | Value |
|---|---|
| Commitment | None |
| Pricing model | Pay-as-you-go (per second/hour) |
| Discount | 0% — full price |
| Interruption risk | None — AWS cannot terminate it |
| Flexibility | Maximum — launch, stop, resize, terminate anytime |
| Upfront cost | None |

---

## Option 2 — Spot Instances

### The Core Concept — AWS's Spare Capacity Auction

AWS data centers always have physical servers that are sitting idle — not running any customer instances. Rather than waste this capacity, AWS sells it at a massive discount through the Spot Instance market. The catch: AWS can reclaim it with only 2 minutes warning when paying customers need that capacity.

```
AWS Data Center Physical Host
┌─────────────────────────────────────────────────────┐
│  Slot 1: Customer A (On-Demand) — always protected  │
│  Slot 2: Customer B (Reserved) — always protected   │
│  Slot 3: ██ EMPTY ██ → AWS sells this as Spot       │
│  Slot 4: ██ EMPTY ██ → AWS sells this as Spot       │
└─────────────────────────────────────────────────────┘

You get Slot 3 at 90% off On-Demand price.
When a Reserved/On-Demand customer needs Slot 3:
  → 2-minute interruption notice sent to your instance
  → Instance hibernated, stopped, or terminated
  → You lose the slot
```

### The Bidding Model

You set a **maximum price** you are willing to pay per hour. The actual Spot price fluctuates based on supply and demand for that instance type in that AZ. If the Spot price rises above your maximum bid, your instance is interrupted.

In practice, Spot prices are relatively stable for common instance types — they rarely spike enough to cause frequent interruptions. Niche instance types in specific AZs can be more volatile.

```bash
# AWS CLI example: launch a Spot Instance with maximum price
aws ec2 request-spot-instances \
  --instance-count 1 \
  --type "one-time" \
  --launch-specification '{
    "ImageId": "ami-051a31ab2f4d498f5",
    "InstanceType": "c5.xlarge",
    "SpotPrice": "0.08"
  }'
```

### Interruption Behaviors

When AWS needs the capacity back, you receive a 2-minute warning. At that point, the instance undergoes one of three actions depending on your configuration:

**Terminate** — Instance is permanently deleted. Default behavior. Data in instance store is lost. EBS volumes persist (if Delete on Termination is disabled).

**Stop** — Instance is stopped. Picks up from where it left off when capacity becomes available again. Useful for jobs that can pause and resume.

**Hibernate** — RAM contents are saved to the EBS root volume. When capacity returns, the instance resumes from exactly the same state — processes continue where they left off.

### Spot Fleet — The Production-Grade Approach

A **Spot Fleet** is a collection of Spot Instances (and optionally On-Demand Instances) managed as a group. You define:
- Target capacity (e.g., 100 vCPUs total)
- A pool of instance types you're willing to use (e.g., c5.xlarge, c5a.xlarge, m5.xlarge)
- Your maximum price
- Allocation strategy

The Spot Fleet automatically picks the cheapest available instances from your pool to meet the target capacity. If one instance type gets interrupted, the fleet can rebalance to other types automatically.

```
Spot Fleet: "Give me 100 vCPUs as cheaply as possible"
    ├── Try c5.xlarge in us-east-1a → available at $0.04 → use 5
    ├── Try c5.xlarge in us-east-1b → too expensive → skip
    ├── Try c5a.xlarge in us-east-1b → available at $0.038 → use 10
    └── Try m5.xlarge in us-east-1c → available at $0.042 → use 10

c5.xlarge in 1a gets interrupted:
    → Fleet automatically launches c5a.xlarge elsewhere to compensate
    → Total capacity maintained at 100 vCPUs
```

### Designing for Spot Interruptions

Any architecture using Spot Instances must be built to handle sudden loss of instances gracefully:

- **Stateless application servers** — if a server dies, the load balancer routes to surviving instances. New Spot instances launch and join automatically.
- **Checkpointing** — batch jobs save progress periodically. If interrupted, resume from the last checkpoint rather than starting over.
- **Auto Scaling Groups** — mix Spot and On-Demand instances. On-Demand provides baseline capacity that is never interrupted. Spot provides cheap burst capacity.

### Ideal Use Cases for Spot Instances

**Batch processing** — overnight jobs processing large datasets, image/video rendering, log analysis. Interruption delays the job but doesn't break it.

**CI/CD pipelines** — build servers can be Spot instances. If a build is interrupted, re-run it. Build time is cheap; build infrastructure costs are significant.

**Machine learning training** — with checkpointing enabled, training jobs can resume after interruption. Spot instances for ML training are extremely popular.

**Big data processing (Spark, Hadoop)** — these frameworks natively handle node failures. Running worker nodes on Spot with a protected On-Demand master node is a standard pattern.

**Web scraping, crawling** — if a request fails due to interruption, retry it. The work is inherently fault-tolerant.

**Dev/test environments** — developers tolerate occasional interruptions. Savings vs On-Demand make this worthwhile.

### Key Properties

| Property | Value |
|---|---|
| Commitment | None |
| Pricing model | Bidding — pay current Spot price up to your max |
| Discount vs On-Demand | Up to 90% |
| Interruption risk | Yes — 2-minute warning, then stopped/terminated/hibernated |
| Flexibility | High (launch/terminate freely) but interrupted unpredictably |
| Upfront cost | None |
| Resellable | No |

---

## Option 3 — Reserved Instances (RI)

### The Core Concept — Pay in Advance, Get a Big Discount

Reserved Instances are a billing commitment, not a capacity reservation in the traditional sense. You commit to using a specific instance configuration for 1 or 3 years, and in exchange AWS gives you a significantly discounted rate. The "reservation" means AWS guarantees you'll have capacity when you want to launch that instance type.

**Important:** A Reserved Instance is not a running instance. It is a discount coupon attached to your account. When you launch an EC2 instance that matches the RI's attributes (instance type, region, OS, tenancy), AWS automatically applies the RI discount to your bill. You don't "use" a Reserved Instance — you launch On-Demand instances that are automatically discounted because of your RI commitment.

### Payment Options — More Upfront = More Discount

For any RI type, you choose one of three payment structures:

```
No Upfront:      Pay nothing now → Pay discounted hourly rate monthly
                 Smallest discount in the RI tier

Partial Upfront: Pay a portion now → Pay reduced hourly rate monthly
                 Middle discount

All Upfront:     Pay everything now → Zero ongoing hourly charge
                 Maximum discount in the RI tier
```

Longer commitment + more upfront = deepest discount.

### The Three RI Types

---

#### 3A — Standard Reserved Instance

The most straightforward RI. You commit to a specific **instance type** (e.g., `m6i.xlarge`), **region**, **OS**, and **tenancy**. In exchange you get up to **72% discount** vs On-Demand.

**What you CAN change:** Instance size within the same family (e.g., from `m6i.large` to `m6i.xlarge`). This is called **Instance Size Flexibility** and applies automatically for Linux RIs in the same instance family.

**What you CANNOT change:**
- Instance family (can't switch from `m6i` to `c6i`)
- Region (can't move from us-east-1 to ap-south-1)
- OS (can't switch from Linux to Windows)
- Tenancy (can't switch from Shared to Dedicated)

**Standard RI Marketplace:** If you commit to a 3-year Standard RI and your business changes after year 1, you can **sell your remaining Standard RI** to another AWS customer through the Reserved Instance Marketplace. This is the only RI type with this exit option. Convertible RIs cannot be sold.

---

#### 3B — Convertible Reserved Instance

Convertible RIs offer slightly less discount (still up to ~66%) but in exchange give you significant flexibility to change the RI's attributes during the commitment period.

**What you CAN change (by "exchanging" the RI):**
- Instance family (e.g., `m6i` → `c6i`)
- Instance size (e.g., `xlarge` → `2xlarge`)
- OS (e.g., Linux → Windows, though the new RI may cost more)
- Tenancy (Shared → Dedicated)

**What you CANNOT do:**
- Sell in the RI Marketplace (Convertible RIs cannot be resold)
- Reduce the total value of the RI when exchanging (you can upgrade but not downgrade to a cheaper config without making up the difference)

**When does Convertible make sense?** When you're uncertain whether your instance needs will change over a 3-year horizon. You lock in the savings now but retain the flexibility to pivot to different instance families as your application evolves.

---

#### 3C — Scheduled Reserved Instance

A niche RI type for workloads that run on a **predictable, recurring schedule** — not 24/7.

Example: A batch job that runs every weekday from 8 PM to 4 AM (8 hours). A standard RI covers 24/7 — you'd pay for 16 hours a day you're not using. A Scheduled RI covers exactly your 8-hour window daily, for 1 year.

**Important constraints:**
- Minimum commitment: 1 year, minimum 1,200 hours per year
- Not available in all regions — notably absent from Mumbai (ap-south-1). Available in specific regions like US East (N. Virginia), EU (Ireland), and Asia Pacific (Singapore)
- Cannot be cancelled once purchased

---

### RI Best Practices

**Right-size before committing** — analyze 2–3 months of CloudWatch utilization metrics. If your instance is consistently at 15% CPU, downsize to a smaller instance type before purchasing the RI for it.

**Layer RIs with On-Demand** — cover your minimum baseline usage with RIs. Use On-Demand or Spot for everything above baseline.

**Use AWS Cost Explorer's RI recommendations** — AWS analyzes your usage history and recommends specific RI purchases that would have saved money. This is based on real data, not guesswork.

### Key Properties

| Property | Standard RI | Convertible RI | Scheduled RI |
|---|---|---|---|
| Max discount vs On-Demand | ~72% | ~66% | ~70% |
| Commitment | 1 or 3 years | 1 or 3 years | 1 year |
| Change size? | Yes (same family) | Yes (any) | Yes |
| Change family? | No | Yes | No |
| Change OS? | No | Yes | No |
| Change region? | No | No | No |
| Resellable? | Yes (Marketplace) | No | No |
| Interruption risk | None | None | None |
| 24/7 or scheduled? | 24/7 | 24/7 | Fixed schedule |

---

## Option 4 — Savings Plans

### How Savings Plans Differ from Reserved Instances

Savings Plans are AWS's newer, more flexible alternative to Reserved Instances. Instead of committing to a **specific instance type**, you commit to a **dollar amount of compute spend per hour**.

```
Reserved Instance commitment:  "I will use m6i.xlarge, Linux, us-east-1 for 3 years"
Savings Plan commitment:       "I will spend at least $10/hour on compute for 3 years"
```

The Savings Plan discount automatically applies to any eligible compute usage up to your committed hourly spend. If you use more than your committed amount, the excess is billed at On-Demand rates.

**Example:**
```
You commit to $5/hour Compute Savings Plan
Today you run:
  - 3x m6i.xlarge (On-Demand would cost $0.192/hr each = $0.576)
  - 2x c6i.2xlarge (On-Demand would cost $0.340/hr each = $0.680)
  Total On-Demand cost: $1.256/hour

Savings Plan applies discount to first $5 worth:
  - Your actual cost with savings plan: ~$0.52/hour (savings plan rate)
  - Since $0.52 < $5 commitment, full savings apply
  
You pay: $0.52/hour (savings plan rate) for everything used
```

### The Three Types of Savings Plans

---

#### 4A — Compute Savings Plans (Most Flexible)

The most flexible savings plan. Up to **66% discount**. Automatically applies to any compute usage regardless of:
- EC2 instance family (t, m, c, r, any family)
- EC2 instance size (micro through 48xlarge)
- Region (any AWS region)
- OS (Linux, Windows, RHEL, SUSE)
- Tenancy (Shared, Dedicated Instance, Dedicated Host)
- **Also applies to AWS Lambda and AWS Fargate** — not just EC2

This is the plan for organizations that are actively evolving their infrastructure — migrating regions, changing instance families, experimenting with serverless, or planning major architectural changes within the commitment window.

---

#### 4B — EC2 Instance Savings Plans (Higher Discount, Less Flexible)

Slightly higher discount than Compute Savings Plans (up to **72%**) but with one constraint: you commit to a specific **EC2 instance family in a specific region** (e.g., `m6i` in `ap-south-1`).

Within that commitment you can still change:
- Instance size (large, xlarge, 2xlarge)
- OS
- Tenancy

But you cannot change:
- Instance family
- Region

Does NOT apply to Lambda or Fargate.

---

#### 4C — SageMaker Savings Plans

Specifically for Amazon SageMaker ML workloads. Not covered in the EC2-focused content.

---

### Savings Plans vs Reserved Instances — The Decision

| Question | Savings Plans | Reserved Instances |
|---|---|---|
| Need to change regions? | Compute SP: Yes | No — locked to region |
| Need to change instance families? | Compute SP: Yes, EC2 SP: No | Convertible RI: Yes, Standard: No |
| Need to cover Lambda/Fargate too? | Compute SP: Yes | No — EC2 only |
| Need to sell unused commitment? | No | Standard RI: Yes |
| Simpler to manage? | Yes — one plan covers everything | No — need separate RIs per instance type |
| Maximum discount? | 72% (EC2 SP) | 72% (Standard RI, all upfront) |

**For most modern AWS architectures, Compute Savings Plans are the recommended starting point** — they cover more services, require less management overhead, and provide full flexibility for infrastructure changes.

---

## The Complete Comparison — From the Video Image

This is the full comparison table shown in the video, filled in completely:

| Feature | On-Demand | Spot | Reserved (Standard) | Reserved (Convertible) | Reserved (Scheduled) | Savings Plans (EC2) | Savings Plans (Compute) |
|---|---|---|---|---|---|---|---|
| **Pricing Model** | Pay-as-you-go | Bidding model | No upfront / Partial / All upfront | No upfront / Partial / All upfront | No upfront / Partial / All upfront | No upfront / Partial / All upfront | No upfront / Partial / All upfront |
| **Commitment Duration** | None | None | 1 or 3 years | 1 or 3 years | 1 year | 1 or 3 years | 1 or 3 years |
| **Max Savings vs On-Demand** | 0% | Up to 90% | Up to 72% | Up to 66% | Up to 70% | Up to 72% | Up to 66% |
| **Interruption Risk** | No | **Yes** | No | No | No | No | No |
| **Change Size** (e.g., small → medium) | — | — | Yes | Yes | Yes | Yes | Yes |
| **Change Family** (e.g., t2 → c5) | — | — | **No** | Yes | **No** | **No** | Yes |
| **Change Compute Type** (EC2 → Lambda) | — | — | **No** | **No** | **No** | **No** | Yes |
| **Change OS** | — | — | **No** | Yes | **No** | Yes | Yes |
| **Change Region** | — | — | **No** | **No** | **No** | **No** | Yes |
| **Change Tenancy** | — | — | **No** | **No** | **No** | **No** | Yes |
| **Resellable?** | N/A | N/A | **Yes** (Marketplace) | No | No | No | No |

---

## The AWS Pricing Calculator

Before committing to any purchase option, you should model the costs using the **AWS Pricing Calculator** at `calculator.aws`.

### How to Use It

1. Go to `calculator.aws` → click **"Create estimate"**
2. Select **"EC2"** from the service list
3. Choose your **Region** (Mumbai = `ap-south-1`)
4. Configure:
   - **Tenancy:** Shared / Dedicated Instance / Dedicated Host
   - **OS:** Linux (free) / Windows (adds Microsoft licensing cost)
   - **Instance type:** Select from the full list
   - **Usage:** Hours per month (730 = 24/7 for a full month)
5. Select **"Show calculations"** to see breakdown by purchase option:

```
Sample output for m6i.xlarge, Linux, ap-south-1, 730 hrs/month:

On-Demand:                    $146.72/month
Reserved Standard 1yr No UF:  $86.28/month  (41% savings)
Reserved Standard 1yr All UF: $74.09/month  (50% savings)
Reserved Standard 3yr All UF: $47.92/month  (67% savings)
EC2 Savings Plan 1yr:         $86.28/month  (41% savings)
Compute Savings Plan 1yr:     $95.27/month  (35% savings)
```

This lets you see exactly what each commitment costs before signing anything.

---

## Cost Optimization Strategy — Layering Purchase Options

The correct approach in production is never to use just one purchase option. You layer them based on usage patterns:

```
Your total compute capacity:

├── Baseline (always-on, predictable) 
│     └── Cover with: Reserved Instances or Savings Plans
│           3-year All Upfront → maximum discount
│
├── Variable (fluctuates but generally steady)
│     └── Cover with: 1-year Reserved Instances or Savings Plans
│           Balance flexibility and savings
│
├── Burst (handles traffic spikes, unpredictable)
│     └── Cover with: On-Demand instances
│           Only pay when actually needed
│
└── Batch / fault-tolerant (can be interrupted)
      └── Cover with: Spot Instances
            Maximum cost savings for tolerant workloads
```

**Real example — e-commerce company:**
```
Baseline 20 app servers (run 24/7 always):
  → 3-year Compute Savings Plan → saves 66% vs On-Demand

Variable 10 servers (weekday business hours):
  → 1-year EC2 Savings Plan → saves 40-50%

Black Friday burst (5x normal traffic for 2 days):
  → On-Demand for the peak period → full price but only for 2 days

Nightly data processing (batch jobs, fault-tolerant):
  → Spot Fleet → saves 80-90%
```

---

## Summary — Which Option for Which Scenario

| Scenario | Best Option | Why |
|---|---|---|
| Unknown duration workload | On-Demand | No lock-in, maximum flexibility |
| Short-term project (< 1 year) | On-Demand | Commitment not worth it |
| Dev/test environment | On-Demand or Spot | Flexibility needed, interruptions OK |
| Steady-state production server | Reserved Standard or Savings Plan | Predictable baseline, maximize savings |
| Production with changing instance needs | Convertible RI or Compute Savings Plan | Flexibility to change instance type |
| Multi-region architecture | Compute Savings Plan | Only option covering all regions |
| Lambda + EC2 together | Compute Savings Plan | Only option covering both services |
| Batch processing, HPC, ML training | Spot Fleet | Fault-tolerant by design, 90% savings |
| Scheduled predictable batch (not Mumbai) | Scheduled RI | Pay only for scheduled hours |
| BYOL software licensing | Dedicated Host (On-Demand or Reserved) | Need physical hardware visibility |
| Maximum savings, 3-year commitment | Standard RI All Upfront or EC2 SP 3yr | Up to 72% savings |
| Need to exit commitment early | Standard RI (sell on Marketplace) | Only resellable option |
