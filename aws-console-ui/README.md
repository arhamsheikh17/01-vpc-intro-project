# VPC Intro Project — AWS Management Console (UI) Guide

> **Audience:** Anyone who prefers point-and-click in the browser over the terminal.  
> If you prefer using the command line, see the [AWS CLI guide](../aws-cli/README.md).

---

## Architecture We Are Building

```
AWS Region (e.g. us-east-1)
└── VPC  10.0.0.0/16  (vpc-intro-project)
    ├── Public Subnet   10.0.1.0/24  (us-east-1a)
    ├── Private Subnet  10.0.2.0/24  (us-east-1a)
    ├── Internet Gateway (igw-intro)
    │     └── attached to the VPC
    ├── Public Route Table
    │     ├── local  → 10.0.0.0/16
    │     └── 0.0.0.0/0  → Internet Gateway
    ├── Private Route Table
    │     └── local  → 10.0.0.0/16
    ├── Public Security Group  (sg-public)
    │     ├── Inbound: SSH (22) from 0.0.0.0/0
    │     ├── Inbound: HTTP (80) from 0.0.0.0/0
    │     └── Outbound: All traffic
    └── Private Security Group  (sg-private)
          ├── Inbound: SSH (22) from sg-public only
          └── Outbound: All traffic
```

### What each resource does

| Resource | Purpose |
|---|---|
| **VPC** | Isolated private network for all your AWS resources |
| **Public Subnet** | Subnet connected to the Internet Gateway — resources here can reach the internet |
| **Private Subnet** | Subnet with no direct internet route — ideal for databases, app servers |
| **Internet Gateway (IGW)** | Allows two-way traffic between the VPC and the public internet |
| **Public Route Table** | Routes `0.0.0.0/0` traffic to the IGW for public subnet resources |
| **Private Route Table** | Contains only local VPC routes; no internet access |
| **Security Groups** | Stateful firewalls controlling inbound/outbound traffic per resource |

---

## Prerequisites

- An AWS account (free tier is sufficient)
- An IAM user with at least **AmazonVPCFullAccess** and **AmazonEC2FullAccess** permissions
- A browser logged in to the [AWS Management Console](https://console.aws.amazon.com)
- Make sure your region is set to **US East (N. Virginia) — us-east-1** (top-right corner of the console). You may use any region, but the steps below reference us-east-1.

---

## Part 1 — Create the VPC Infrastructure

### Step 1 — Create the VPC

1. In the AWS Console search bar, type **VPC** and click **VPC** under *Services*.
2. In the left sidebar, click **Your VPCs**.
3. Click the orange **Create VPC** button (top-right).
4. Fill in the form:

   | Field | Value |
   |---|---|
   | Resources to create | **VPC only** |
   | Name tag | `vpc-intro-project` |
   | IPv4 CIDR block | `10.0.0.0/16` |
   | IPv6 CIDR block | No IPv6 CIDR block |
   | Tenancy | Default |

5. Click **Create VPC**.
6. After creation, select your new VPC → click **Actions** → **Edit VPC settings**.
7. Check **Enable DNS hostnames** → click **Save**.

> **Why?** DNS hostnames allow instances in the VPC to receive a human-readable public DNS name.

---

### Step 2 — Create the Public Subnet

1. In the left sidebar, click **Subnets**.
2. Click **Create subnet**.
3. Fill in the form:

   | Field | Value |
   |---|---|
   | VPC ID | Select `vpc-intro-project` |
   | Subnet name | `subnet-public` |
   | Availability Zone | `us-east-1a` |
   | IPv4 subnet CIDR block | `10.0.1.0/24` |

4. Click **Create subnet**.
5. After creation, select `subnet-public` → click **Actions** → **Edit subnet settings**.
6. Check **Enable auto-assign public IPv4 address** → click **Save**.

> **Why?** This ensures EC2 instances launched in the public subnet automatically receive a public IP.

---

### Step 3 — Create the Private Subnet

1. Click **Create subnet**.
2. Fill in the form:

   | Field | Value |
   |---|---|
   | VPC ID | Select `vpc-intro-project` |
   | Subnet name | `subnet-private` |
   | Availability Zone | `us-east-1a` |
   | IPv4 subnet CIDR block | `10.0.2.0/24` |

3. Click **Create subnet**.

> Do **not** enable auto-assign public IPv4 for this subnet — that's what keeps it private.

---

### Step 4 — Create and Attach an Internet Gateway

1. In the left sidebar, click **Internet gateways**.
2. Click **Create internet gateway**.
3. Fill in the form:

   | Field | Value |
   |---|---|
   | Name tag | `igw-intro` |

4. Click **Create internet gateway**.
5. You will see a banner saying *"Internet gateway created. Attach to a VPC"* — click **Attach to a VPC** in that banner (or select the IGW → **Actions** → **Attach to VPC**).
6. Select `vpc-intro-project` from the dropdown → click **Attach internet gateway**.

> **Why?** Without an Internet Gateway attached, no traffic can flow between your VPC and the internet, even from the public subnet.

---

### Step 5 — Create the Public Route Table

1. In the left sidebar, click **Route tables**.
2. Click **Create route table**.
3. Fill in the form:

   | Field | Value |
   |---|---|
   | Name | `rt-public` |
   | VPC | `vpc-intro-project` |

4. Click **Create route table**.
5. Select `rt-public` → click the **Routes** tab → click **Edit routes**.
6. Click **Add route**:

   | Destination | Target |
   |---|---|
   | `0.0.0.0/0` | Internet Gateway → `igw-intro` |

7. Click **Save changes**.
8. Now click the **Subnet associations** tab → click **Edit subnet associations**.
9. Check `subnet-public` → click **Save associations**.

> **Why?** Associating `subnet-public` with this route table means traffic destined for the internet (`0.0.0.0/0`) from that subnet will be forwarded to the IGW.

---

### Step 6 — Create the Private Route Table

1. Click **Create route table**.
2. Fill in the form:

   | Field | Value |
   |---|---|
   | Name | `rt-private` |
   | VPC | `vpc-intro-project` |

3. Click **Create route table**.
4. Select `rt-private` → click the **Subnet associations** tab → click **Edit subnet associations**.
5. Check `subnet-private` → click **Save associations**.

> No extra route is added here. The default local route (`10.0.0.0/16`) is sufficient for a purely private subnet.

---

### Step 7 — Create the Public Security Group

1. In the left sidebar, click **Security groups**.
2. Click **Create security group**.
3. Fill in the form:

   | Field | Value |
   |---|---|
   | Security group name | `sg-public` |
   | Description | `Public SG – SSH and HTTP from internet` |
   | VPC | `vpc-intro-project` |

4. Under **Inbound rules**, click **Add rule** twice:

   | Type | Protocol | Port range | Source | Description |
   |---|---|---|---|---|
   | SSH | TCP | 22 | `0.0.0.0/0` | Allow SSH from internet |
   | HTTP | TCP | 80 | `0.0.0.0/0` | Allow HTTP from internet |

5. Leave **Outbound rules** as the default (allow all outbound traffic).
6. Click **Create security group**.

---

### Step 8 — Create the Private Security Group

1. Click **Create security group**.
2. Fill in the form:

   | Field | Value |
   |---|---|
   | Security group name | `sg-private` |
   | Description | `Private SG – SSH from public SG only` |
   | VPC | `vpc-intro-project` |

3. Under **Inbound rules**, click **Add rule**:

   | Type | Protocol | Port range | Source | Description |
   |---|---|---|---|---|
   | SSH | TCP | 22 | Custom → `sg-public` | Allow SSH from public instances only |

   > In the **Source** column, choose **Custom** and start typing `sg-public` to select the security group by ID.

4. Leave **Outbound rules** as the default (allow all outbound traffic).
5. Click **Create security group**.

---

### Step 9 — Verify Everything Was Created

Navigate to the following pages and confirm the resources exist:

| Console Page | What to check |
|---|---|
| **VPC → Your VPCs** | `vpc-intro-project` with CIDR `10.0.0.0/16`, State = *available* |
| **VPC → Subnets** | `subnet-public` (`10.0.1.0/24`) and `subnet-private` (`10.0.2.0/24`) |
| **VPC → Internet gateways** | `igw-intro`, State = *attached* to `vpc-intro-project` |
| **VPC → Route tables** | `rt-public` with route `0.0.0.0/0 → igw-intro`; `rt-private` with only local route |
| **EC2 → Security groups** | `sg-public` and `sg-private` associated with `vpc-intro-project` |

---

## Part 2 — Clean Up (Delete Everything)

> **Delete resources in reverse order** to avoid dependency errors. AWS will not let you delete a resource that is still in use by another resource.

---

### Step 1 — Delete Security Groups

1. Go to **EC2 → Security Groups** (or **VPC → Security groups**).
2. Select `sg-private` → **Actions** → **Delete security groups** → confirm.
3. Select `sg-public` → **Actions** → **Delete security groups** → confirm.

> Delete `sg-private` first because `sg-public` is referenced by it.

---

### Step 2 — Delete the Custom Route Tables

1. Go to **VPC → Route tables**.
2. Select `rt-public`:
   - Click **Subnet associations** tab → **Edit subnet associations** → uncheck `subnet-public` → **Save associations**.
   - Click **Routes** tab → **Edit routes** → delete the `0.0.0.0/0` route → **Save changes**.
   - Click **Actions** → **Delete route table** → confirm.
3. Select `rt-private`:
   - Click **Subnet associations** tab → **Edit subnet associations** → uncheck `subnet-private` → **Save associations**.
   - Click **Actions** → **Delete route table** → confirm.

> You cannot delete the **main** route table of a VPC — that is deleted automatically when the VPC is deleted.

---

### Step 3 — Detach and Delete the Internet Gateway

1. Go to **VPC → Internet gateways**.
2. Select `igw-intro` → **Actions** → **Detach from VPC** → confirm.
3. Select `igw-intro` again → **Actions** → **Delete internet gateway** → confirm.

---

### Step 4 — Delete the Subnets

1. Go to **VPC → Subnets**.
2. Select `subnet-public` → **Actions** → **Delete subnet** → confirm.
3. Select `subnet-private` → **Actions** → **Delete subnet** → confirm.

---

### Step 5 — Delete the VPC

1. Go to **VPC → Your VPCs**.
2. Select `vpc-intro-project` → **Actions** → **Delete VPC** → type `delete` to confirm → click **Delete**.

> AWS will warn you if there are still resources inside the VPC. If so, delete those first and retry.

---

### Step 6 — Confirm Everything Is Gone

Refresh each of these pages and confirm no project resources remain:

- **VPC → Your VPCs** — `vpc-intro-project` should be gone
- **VPC → Subnets** — `subnet-public` and `subnet-private` should be gone
- **VPC → Internet gateways** — `igw-intro` should be gone
- **VPC → Route tables** — `rt-public` and `rt-private` should be gone
- **EC2 → Security groups** — `sg-public` and `sg-private` should be gone

---

## Summary of Resources Created

| # | Resource | Name | Value |
|---|---|---|---|
| 1 | VPC | `vpc-intro-project` | `10.0.0.0/16` |
| 2 | Public Subnet | `subnet-public` | `10.0.1.0/24` in `us-east-1a` |
| 3 | Private Subnet | `subnet-private` | `10.0.2.0/24` in `us-east-1a` |
| 4 | Internet Gateway | `igw-intro` | Attached to `vpc-intro-project` |
| 5 | Public Route Table | `rt-public` | `0.0.0.0/0 → igw-intro`, associated with `subnet-public` |
| 6 | Private Route Table | `rt-private` | Local route only, associated with `subnet-private` |
| 7 | Public Security Group | `sg-public` | Inbound SSH+HTTP from anywhere |
| 8 | Private Security Group | `sg-private` | Inbound SSH from `sg-public` only |
