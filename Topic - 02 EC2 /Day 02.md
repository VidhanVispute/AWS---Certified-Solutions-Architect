# How to Launch a Windows EC2 Instance on AWS (Full UI Guide)

---

## Step 1 — Sign In & Navigate to EC2

Go to **console.aws.amazon.com**, sign in, and in the top search bar type **EC2**. Click on **EC2** under Services. You'll land on the EC2 Dashboard.

On the dashboard you'll see a summary of your running instances, limits, and resources. Click the orange **"Launch Instance"** button to begin.

---

## Step 2 — Name Your Instance

**Name and Tags**
The first field asks for a **Name**. This is just a label for your reference — it becomes an AWS "tag" with the key `Name`. It doesn't affect anything technically, but good naming matters (e.g., `windows-web-server-prod`).

You can also click **"Add additional tags"** to add key-value pairs like `Environment: Production` or `Owner: DevTeam`. Tags are useful for billing tracking and organizing resources.

---

## Step 3 — Choose an Amazon Machine Image (AMI)

An **AMI** is a pre-built template that contains the OS and sometimes pre-installed software. It's the "blueprint" for your instance.

**For Windows, search:** `Windows Server`

You'll see options like:
- **Windows Server 2022 Base** — Latest, most modern. Recommended for new workloads.
- **Windows Server 2019 Base** — Very stable, widely used in enterprise environments.
- **Windows Server 2016 Base** — Older, only use if your app specifically requires it.
- **Windows Server 2022 with SQL Server** — Pre-installs SQL Server (costs extra licensing fee).
- **Windows Server Core** — No GUI, command-line only, lighter and cheaper.

Each AMI shows a label:
- **Free tier eligible** — No extra AMI charge (only compute cost applies).
- **64-bit (x86)** vs **64-bit (Arm)** — x86 is standard; Arm (Graviton) is cheaper but software compatibility may vary. For Windows, always choose **x86**.

> Select **Windows Server 2022 Base** (Free tier eligible) for most use cases.

---

## Step 4 — Choose an Instance Type

The **instance type** determines the hardware: how many CPUs, how much RAM, and network speed. AWS has a naming convention:

`[Family][Generation][Size]` → e.g., `t3.micro`

**Families you need to know:**
- **t3 / t3a** — Burstable, general purpose. Cheap. Good for dev/test servers that don't need constant CPU.
- **m6i / m5** — Balanced CPU + RAM. Good for production workloads.
- **c6i / c5** — Compute-optimized. Good for CPU-heavy apps.
- **r6i / r5** — Memory-optimized. Good for databases, caching.

**Sizes:** nano → micro → small → medium → large → xlarge → 2xlarge → ... (each step roughly doubles resources)

**Free Tier:** `t2.micro` or `t3.micro` (1 vCPU, 1 GB RAM) is free for 750 hours/month.

**For Windows especially**, Microsoft licensing is baked into the per-hour cost of Windows AMIs, so larger instances cost more for both compute AND licensing.

> For learning/testing: pick **t2.micro** (free tier). For a real web server: **t3.medium** (2 vCPU, 4 GB RAM) is a comfortable starting point.

---

## Step 5 — Key Pair (Login)

A **Key Pair** is used to securely retrieve the Windows Administrator password the first time you connect.

Options:
- **Create a new key pair** — Give it a name (e.g., `my-windows-key`), choose:
  - **RSA** (standard, works everywhere) or **ED25519** (not supported for Windows — avoid this for Windows instances)
  - **.pem** format — for use with OpenSSH or AWS CLI
  - **.ppk** format — for use with PuTTY on Windows
- **Use an existing key pair** — If you already created one, select it from the dropdown.
- **Proceed without a key pair** — Only safe if you have another way to access the instance (like Systems Manager Session Manager). Not recommended for beginners.

> Download and save the `.pem` file immediately — AWS will never let you download it again.

**How it's used:** After the instance launches, you go to EC2 → Instances → select your instance → click "Connect" → "Get Windows password" → upload your `.pem` file → AWS decrypts and shows you the Administrator password.

---

## Step 6 — Network Settings

This is one of the most important sections.

**VPC (Virtual Private Cloud)**
A VPC is your isolated private network inside AWS. By default you have a **Default VPC** already set up. For beginners, leave this as default. In production, you'd create a custom VPC.

**Subnet**
A subnet is a segment within your VPC, tied to a specific **Availability Zone** (a physical data center location). Options:
- **No preference** — AWS picks for you automatically.
- Specific subnets like `us-east-1a`, `us-east-1b` — Choose if you need the instance in a specific zone for latency or high availability reasons.

**Auto-assign Public IP**
- **Enable** — Your instance gets a public IP address so it's reachable from the internet. Enable this if you want to RDP (Remote Desktop) into it from your home/office.
- **Disable** — Instance only has a private IP. Only reachable from within the VPC (or via VPN). Used in production where servers sit behind a load balancer.

**Firewall (Security Group)**
A Security Group is a virtual firewall that controls inbound and outbound traffic with rules.

**Create a new security group** options:
- **Security group name** — Give it a clear name like `windows-rdp-sg`.
- **Description** — Describe what it's for.

**Inbound rules** — You'll see a default rule added:
- **RDP (Port 3389)** — This is what you need to connect to Windows via Remote Desktop. 
  - Source: **My IP** (recommended — only allows your current IP to connect, much safer than opening to the world)
  - Source: **Anywhere (0.0.0.0/0)** — Allows anyone to attempt to connect. Risky — bots constantly scan for open RDP ports.
  - Source: **Custom** — You enter a specific IP range in CIDR notation (e.g., `203.0.113.0/24`)

You can also add:
- **HTTP (Port 80)** — If this server will host a website.
- **HTTPS (Port 443)** — For secure website traffic.
- **Custom TCP** — For any application-specific ports.

> Always restrict RDP to **My IP** unless you have a specific reason not to.

---

## Step 7 — Configure Storage (EBS Volume)

EBS (Elastic Block Store) is the hard drive for your instance.

**Root volume (C: drive on Windows):**
- **Size** — Default for Windows Server 2022 is **30 GB**. You can increase this (minimum for Windows is 30 GB). You cannot decrease below the AMI's default.
- **Volume type:**
  - **gp3** (General Purpose SSD v3) — Best choice in almost all cases. Fast, consistent, cost-effective. You can independently set IOPS and throughput.
  - **gp2** (General Purpose SSD v2) — Older version. IOPS scales with size. No reason to choose this over gp3 today.
  - **io1 / io2** (Provisioned IOPS SSD) — For very high-performance databases that need guaranteed IOPS. Expensive.
  - **sc1 / st1** (HDD types) — Slower, very cheap. Only for infrequent large data access. Cannot be used as root volume.
- **IOPS** — Input/Output operations per second. gp3 default is 3000 IOPS (free). You can provision more if needed.
- **Throughput** — Default gp3 throughput is 125 MB/s (free up to 125). Increase for large file transfer workloads.
- **Delete on termination** — Default is **Yes**, meaning the disk is deleted when you terminate the instance. Uncheck this if you want to keep the data even after the instance is terminated (useful for production).
- **Encrypted** — Encrypts the EBS volume using AWS KMS. Strongly recommended for any sensitive data. No performance impact.

You can also click **"Add new volume"** to attach additional drives (they appear as D:, E:, etc. in Windows after you initialize them).

---

## Step 8 — Advanced Details (Optional but Important)

Most beginners skip this, but here are the key options:

**IAM Instance Profile** — Attaches an IAM role to your instance, granting it permissions to access other AWS services (like S3 or DynamoDB) without needing hardcoded credentials. Best practice for production.

**User Data** — A script that runs automatically on first boot. For Windows, this is a PowerShell or batch script. Example use: auto-install IIS, configure settings, download your application on launch.

**Tenancy:**
- **Shared** — Your VM runs on hardware shared with other AWS customers (fully isolated, but shared physical host). Default and cheapest.
- **Dedicated Instance** — Your VM runs on hardware dedicated to your AWS account. Higher cost.
- **Dedicated Host** — You get a full physical server to yourself. Required for certain Windows Server licenses (BYOL — Bring Your Own License).

**Hibernation** — Allows the instance to be paused and resumed, saving RAM to disk. Useful for dev environments you want to stop and resume quickly.

**Monitoring** — Enables **CloudWatch Detailed Monitoring** (1-minute metric intervals instead of the default 5-minute). Has an extra cost.

**Placement Group** — Groups instances physically close together in the data center for low-latency networking between them. Useful for clustered applications.

---

## Step 9 — Review & Launch

You'll see a **Summary panel** on the right showing:
- Number of instances (default: 1 — you can launch multiple identical instances at once)
- AMI name
- Instance type
- Key pair name
- Security group
- Storage

Click **"Launch Instance"**.

AWS will show a success screen with your **Instance ID** (e.g., `i-0a1b2c3d4e5f67890`). Click on the ID to go to your instance.

---

## Step 10 — Connect to Your Windows Instance via RDP

**Wait 2–5 minutes** after launch. Windows needs time to initialize and generate the password.

1. Go to **EC2 → Instances**, select your instance.
2. Check **Instance State** = `Running` and **Status Checks** = `2/2 checks passed`.
3. Click **"Connect"** at the top.
4. Go to the **"RDP client"** tab.
5. Click **"Get Password"** → upload your `.pem` key file → click **"Decrypt Password"**.
6. AWS shows you the **Administrator password** — copy it.
7. Click **"Download Remote Desktop File"** — this downloads a `.rdp` file pre-configured with your instance's public IP.
8. Open the `.rdp` file → enter the password when prompted → you're in.

---

## Quick Reference Summary

| Setting | Recommended for Beginners |
|---|---|
| AMI | Windows Server 2022 Base |
| Instance Type | t2.micro (free tier) |
| Key Pair | RSA, .pem format |
| Public IP | Enabled |
| Security Group | RDP port 3389, source = My IP |
| Storage | 30 GB gp3, delete on termination = Yes |
| User Data | Leave blank |
