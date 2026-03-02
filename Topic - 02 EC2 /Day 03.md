# How to Launch a Ubuntu EC2 Instance on AWS (Full UI Guide)

---

## Step 1 — Sign In & Navigate to EC2

Go to **console.aws.amazon.com**, sign in, and search **EC2** in the top bar. Click **EC2** under Services to reach the EC2 Dashboard.

Click the orange **"Launch Instance"** button to begin.

---

## Step 2 — Name Your Instance

**Name and Tags**
Enter a name for your instance (e.g., `ubuntu-web-server-prod`). This becomes a tag with the key `Name` — purely for your organization, no technical effect.

Click **"Add additional tags"** to add extra metadata like:
- `Environment: Production`
- `Owner: DevTeam`
- `Project: MyApp`

Tags are extremely useful for filtering resources and tracking costs per project in your AWS bill.

---

## Step 3 — Choose an Amazon Machine Image (AMI)

An **AMI** is the OS blueprint for your instance. In the search bar type **Ubuntu**.

You'll see options like:

- **Ubuntu Server 24.04 LTS (Noble Numbat)** — Latest LTS. Best for new projects. LTS = Long Term Support (5 years of security updates).
- **Ubuntu Server 22.04 LTS (Jammy Jellyfish)** — Very stable, most widely used right now. Excellent choice for production.
- **Ubuntu Server 20.04 LTS (Focal Fossa)** — Older, approaching end of standard support. Avoid for new setups.
- **Ubuntu Server 24.04 LTS (Minimal)** — Stripped-down version with fewer pre-installed packages. Smaller attack surface, faster boot. Good for containerized workloads.
- **Ubuntu Pro 22.04 LTS** — Paid version with extended 10-year security patching and compliance tools (FIPS, CIS hardening). For enterprise/regulated environments.

**Architecture options:**
- **64-bit (x86)** — Standard Intel/AMD. Compatible with everything.
- **64-bit (Arm)** — Runs on AWS Graviton processors. **20–40% cheaper** than x86 for the same performance. Ubuntu supports Arm very well — a great choice if your software is compatible.

> For most use cases, select **Ubuntu Server 22.04 LTS, 64-bit (x86)**. For cost savings on compatible workloads, try **Arm**.

---

## Step 4 — Choose an Instance Type

Same naming convention as Windows: `[Family][Generation][Size]` → e.g., `t3.micro`

**Families:**
- **t3 / t3a** — Burstable general purpose. Cheapest option. Great for dev servers, small websites, learning.
- **t4g** — Burstable but Arm-based (Graviton). Cheapest of all — excellent for Ubuntu since Linux has full Graviton support.
- **m6i / m5** — Balanced CPU + RAM. Good for steady production workloads.
- **c6i / c5** — Compute-optimized. For CPU-heavy tasks like video processing or scientific computing.
- **r6i / r5** — Memory-optimized. For databases, in-memory caching (Redis, Elasticsearch).

**Key difference from Windows:** Linux instance types are **cheaper** because there is no OS licensing fee. You only pay for the hardware. A `t3.micro` running Ubuntu costs roughly **half** of what the same instance costs running Windows Server.

**Free Tier:** `t2.micro` (1 vCPU, 1 GB RAM) — free for 750 hours/month.

> For learning: **t2.micro** (free). For a real app server: **t3.small** or **t3.medium** is a solid starting point.

---

## Step 5 — Key Pair (Login)

This is where Linux differs significantly from Windows. On Linux, the key pair is used **directly to SSH into the instance** — not just to retrieve a password.

**Create a new key pair:**
- **Key pair name** — e.g., `ubuntu-server-key`
- **Key pair type:**
  - **RSA** — The classic standard. Works with all SSH clients everywhere.
  - **ED25519** — Modern, faster, more secure. Fully supported on Linux. Use this if your SSH client is up to date (recommended).
- **Private key file format:**
  - **.pem** — Used with OpenSSH (`ssh` command on Mac/Linux/Windows Terminal). Most common.
  - **.ppk** — Used with PuTTY (older Windows SSH client). Only choose this if you specifically use PuTTY.

**Use an existing key pair** — Select from the dropdown if you already have one.

**Proceed without a key pair** — Only viable if you plan to use **EC2 Instance Connect** (browser-based SSH) or **AWS Systems Manager Session Manager**. Not recommended for regular use.

> Download the `.pem` file immediately — you can never download it again from AWS. Store it safely (e.g., `~/.ssh/` on your machine). On Mac/Linux, run `chmod 400 ubuntu-server-key.pem` to set correct permissions or SSH will refuse to use it.

**How to connect later:**
```
ssh -i "ubuntu-server-key.pem" ubuntu@<your-public-ip>
```
Note: The default username for Ubuntu AMIs is always **`ubuntu`** (not `root`, not `ec2-user`).

---

## Step 6 — Network Settings

**VPC (Virtual Private Cloud)**
Your isolated private network in AWS. Leave as **Default VPC** for learning. In production, use a custom VPC with public/private subnet separation.

**Subnet**
A subdivision of your VPC tied to an Availability Zone (physical data center). Options:
- **No preference** — AWS assigns automatically.
- **Specific AZ (e.g., us-east-1a)** — Choose when you need the instance in a particular zone, for example to co-locate with a database or for high-availability architecture.

**Auto-assign Public IP**
- **Enable** — Instance gets a public IP. Required if you want to SSH into it from your laptop or serve traffic from it directly.
- **Disable** — Instance only has a private IP. Used when instances sit behind a load balancer or are accessed only through a bastion host or VPN.

**Firewall (Security Group)**
A virtual firewall controlling what traffic can reach your instance.

**Inbound rules for Ubuntu — common ports:**

| Port | Protocol | Purpose |
|---|---|---|
| **22** | SSH | Remote terminal access — essential for Linux |
| **80** | HTTP | Web traffic (if hosting a site) |
| **443** | HTTPS | Secure web traffic |
| **8080** | Custom TCP | Common for Java apps, Node.js dev servers |
| **3000** | Custom TCP | Node.js / React dev servers |
| **5432** | PostgreSQL | Database access (restrict heavily) |

**For SSH (Port 22) — Source options:**
- **My IP** — Strongly recommended. Only your current IP can SSH in.
- **Anywhere (0.0.0.0/0)** — Anyone on the internet can attempt to connect. Dangerous — automated bots constantly scan for open port 22 and attempt brute-force logins.
- **Custom CIDR** — Specific IP range, e.g., your company's office network `203.0.113.0/24`.

> Always set SSH source to **My IP**. If your home IP changes later, you can update the security group rule.

**Key Linux vs Windows difference here:** For Windows you open **port 3389 (RDP)**. For Linux you open **port 22 (SSH)**. Never need to open 3389 for Ubuntu.

---

## Step 7 — Configure Storage (EBS Volume)

**Root volume (mounted as `/` on Ubuntu):**

- **Size** — Default for Ubuntu 22.04 is **8 GB**. This is much smaller than Windows' 30 GB minimum because Linux is far more lightweight. For a basic server 8–20 GB is fine. Increase if you'll be storing data or installing large applications.
- **Volume type:**
  - **gp3** — Best default choice. Fast SSD, consistent performance, cost-effective. Allows you to independently configure IOPS and throughput.
  - **gp2** — Older SSD type. IOPS tied to volume size (3 IOPS/GB). No advantage over gp3 today.
  - **io1 / io2** — Provisioned IOPS SSD. For high-performance databases needing guaranteed IOPS. Expensive — only use when you specifically need it.
  - **st1** — Throughput-optimized HDD. Cheap, sequential-read optimized. For log processing, big data. Cannot be root volume.
  - **sc1** — Cold HDD. Cheapest storage. For infrequently accessed data. Cannot be root volume.
- **IOPS** — gp3 default is 3,000 IOPS at no extra charge. Increase only for high-demand database workloads.
- **Throughput** — gp3 default is 125 MB/s free. Increase for workloads that transfer large files frequently.
- **Delete on termination** — Default **Yes**. The disk is deleted when you terminate the instance. Set to **No** in production if you want to preserve data.
- **Encrypted** — Encrypts data at rest using AWS KMS. No performance impact. Recommended for any real-world use.

**Adding extra volumes:**
Click **"Add new volume"** for additional EBS disks (e.g., a separate `/data` partition). On Ubuntu, after launch you'll need to format and mount them manually:
```bash
sudo mkfs -t ext4 /dev/xvdb
sudo mount /dev/xvdb /data
```

---

## Step 8 — Advanced Details (Optional but Important)

**IAM Instance Profile**
Attach an IAM Role so your Ubuntu server can call other AWS services (S3, DynamoDB, Secrets Manager, etc.) without hardcoding credentials. Best practice — always use roles instead of storing AWS access keys on the instance.

**User Data**
A shell script that runs automatically on **first boot**. This is extremely powerful for Ubuntu — you can fully configure your server automatically at launch. Example:

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
```

This script installs and starts Nginx the moment your instance launches — no manual SSH needed.

**Tenancy:**
- **Shared** — Default. Your VM shares a physical host with other AWS customers (fully isolated). Cheapest.
- **Dedicated Instance** — Physical hardware dedicated to your AWS account. Higher cost.
- **Dedicated Host** — Full physical server to yourself. Used for BYOL (Bring Your Own License) software that is licensed per physical core/socket.

**Hibernation**
Pauses the instance and saves RAM contents to the root EBS volume. When resumed, picks up exactly where it left off. Useful for development environments. Requires an encrypted root volume and the instance must be EBS-backed (which it is by default).

**Detailed CloudWatch Monitoring**
Default monitoring sends metrics (CPU, network, disk) every **5 minutes** for free. Enabling detailed monitoring sends them every **1 minute** — costs extra. Only needed if you need fine-grained alerting.

**Elastic Inference** — Attaches a fractional GPU for machine learning inference workloads at lower cost than a full GPU instance.

**Credit specification (for T-type instances)**
- **Standard** — When CPU credits run out, performance is throttled back to baseline. Cheaper.
- **Unlimited** — Bursts beyond credits indefinitely, but you're charged for extra burst usage. Good for unpredictable workloads.

**Placement Group**
Groups multiple instances physically close for ultra-low latency networking between them. Types:
- **Cluster** — All instances in one rack, lowest latency. For HPC and tightly coupled distributed apps.
- **Spread** — Instances on separate hardware, maximizing fault tolerance.
- **Partition** — Instances divided into isolated partitions across racks. For large distributed systems like Hadoop or Kafka.

---

## Step 9 — Review & Launch

Check the **Summary panel** on the right:
- Number of instances — default 1. Increase to launch multiple identical servers simultaneously.
- AMI name
- Instance type
- Key pair
- Security group
- Storage configuration

Click **"Launch Instance"**. You'll get a success screen with your **Instance ID** (e.g., `i-0a1b2c3d4e5f67890`).

---

## Step 10 — Connect to Your Ubuntu Instance via SSH

**Wait 1–2 minutes** (much faster than Windows — Linux boots quickly).

1. Go to **EC2 → Instances**, select your instance.
2. Confirm **Instance State** = `Running` and **Status Checks** = `2/2 checks passed`.
3. Copy the **Public IPv4 address** from the instance details panel.

**Option A — Terminal (Mac/Linux/Windows Terminal):**
```bash
chmod 400 ubuntu-server-key.pem
ssh -i "ubuntu-server-key.pem" ubuntu@<your-public-ip>
```

**Option B — EC2 Instance Connect (Browser-based, no key needed):**
1. Select your instance → click **"Connect"**
2. Choose **"EC2 Instance Connect"** tab
3. Username is pre-filled as `ubuntu`
4. Click **"Connect"** — a terminal opens directly in your browser

**Option C — AWS Systems Manager Session Manager:**
No open ports, no key pair required. Requires an IAM role with SSM permissions attached to the instance. Most secure option for production.

> Once connected you'll see the Ubuntu terminal prompt: `ubuntu@ip-10-0-1-5:~$` — you're in.

---

## Key Differences: Ubuntu vs Windows EC2

| | Ubuntu | Windows Server |
|---|---|---|
| Default AMI size | 8 GB | 30 GB |
| OS licensing cost | Free (open source) | Included in hourly price |
| Connect method | SSH (port 22) | RDP (port 3389) |
| Default username | `ubuntu` | `Administrator` |
| Password retrieval | Not needed | Decrypt with `.pem` key |
| Key pair used for | Direct SSH login | Password decryption only |
| Boot time | ~30–60 seconds | ~3–5 minutes |
| Recommended key type | ED25519 or RSA | RSA only |
| User Data format | Bash shell script | PowerShell script |
| Cost (same hardware) | Lower (no OS fee) | Higher (Windows license) |

---

## Quick Reference Summary

| Setting | Recommended for Beginners |
|---|---|
| AMI | Ubuntu Server 22.04 LTS (x86) |
| Instance Type | t2.micro (free tier) |
| Key Pair | ED25519, .pem format |
| Public IP | Enabled |
| Security Group | SSH port 22, source = My IP |
| Storage | 8–20 GB gp3, delete on termination = Yes |
| User Data | Optional bash script |
| Connect via | `ssh -i key.pem ubuntu@<ip>` |
