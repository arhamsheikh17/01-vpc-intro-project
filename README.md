# AWS VPC with EC2 Instances — Intro Project

This project documents how to set up a Virtual Private Cloud (VPC) on AWS with two EC2 instances placed in separate subnets, connected to the internet via an Internet Gateway, and secured with a shared Security Group.

---

## Architecture Overview

```
Internet
    │
    ▼
Internet Gateway (IGW)
    │
    ▼
VPC: 172.16.0.0/16
├── Route Table (0.0.0.0/0 → IGW)
│
├── Main Subnet  172.16.0.0/24  (us-east-1a)
│
├── Subnet 1     172.16.1.0/24  (us-east-1b)
│       └── EC2 Instance 1  (t2.micro)
│
└── Subnet 2     172.16.2.0/24  (us-east-1c)
        └── EC2 Instance 2  (t2.micro)

Security Group (vpc-intro-project-SG)
  • Inbound  – HTTP  TCP 80   0.0.0.0/0
  • Inbound  – SSH   TCP 22   <your-ip>/32
  • Outbound – All traffic    0.0.0.0/0
```

**Components**

| Component | Name / Value |
|---|---|
| VPC | `vpc-intro-project` — `172.16.0.0/16` |
| Main Subnet | `Main-Subnet` — `172.16.0.0/24` |
| Subnet 1 | `Subnet-1` — `172.16.1.0/24` |
| Subnet 2 | `Subnet-2` — `172.16.2.0/24` |
| Internet Gateway | `vpc-intro-project-IGW` |
| Route Table | `Main-RouteTable` |
| Security Group | `vpc-intro-project-SG` |
| EC2 Instance 1 | `EC2-Instance-1` — `t2.micro` in Subnet 1 |
| EC2 Instance 2 | `EC2-Instance-2` — `t2.micro` in Subnet 2 |

---

## Prerequisites

- An active **AWS account**.
- Sufficient IAM permissions to create VPCs, subnets, EC2 instances, and related resources.
- (Optional) **AWS CLI v2** installed and configured (`aws configure`) if you prefer CLI commands over the Console.

---

## Step-by-Step: Creating the Architecture

### Step 1 — Sign In to AWS Management Console

1. Open [https://aws.amazon.com/console/](https://aws.amazon.com/console/) and sign in.
2. Select the AWS **Region** you want to work in (e.g., `us-east-1`). All resources must be created in the **same region**.
3. Navigate to the **VPC** service using the search bar.

---

### Step 2 — Create the VPC

**Console**

1. In the VPC Dashboard, click **Your VPCs** → **Create VPC**.
2. Fill in:
   - **Name tag**: `vpc-intro-project`
   - **IPv4 CIDR block**: `172.16.0.0/16`
   - **Tenancy**: Default
3. Click **Create VPC**.

**AWS CLI**

```bash
aws ec2 create-vpc \
  --cidr-block 172.16.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=vpc-intro-project}]'
```

> Save the returned **VPC ID** (e.g., `vpc-0abc123456789def0`); you will need it in subsequent steps.

---

### Step 3 — Create Subnets

Create **three** subnets inside the VPC, each in a different Availability Zone for resilience.

**Console**

1. In the VPC Dashboard, click **Subnets** → **Create subnet**.
2. Select VPC: `vpc-intro-project`.
3. Add each subnet below (click **Add new subnet** between entries):

| Subnet Name | Availability Zone | IPv4 CIDR Block |
|---|---|---|
| `Main-Subnet` | `us-east-1a` | `172.16.0.0/24` |
| `Subnet-1` | `us-east-1b` | `172.16.1.0/24` |
| `Subnet-2` | `us-east-1c` | `172.16.2.0/24` |

4. Click **Create subnet**.

**AWS CLI** (replace `<vpc-id>` with your actual VPC ID)

```bash
# Main Subnet
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 172.16.0.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Main-Subnet}]'

# Subnet 1
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 172.16.1.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Subnet-1}]'

# Subnet 2
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 172.16.2.0/24 \
  --availability-zone us-east-1c \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Subnet-2}]'
```

> Save each **Subnet ID** returned (e.g., `subnet-0abc…`).

---

### Step 4 — Create and Attach an Internet Gateway

**Console**

1. Click **Internet Gateways** → **Create internet gateway**.
2. **Name tag**: `vpc-intro-project-IGW`.
3. Click **Create internet gateway**.
4. Select the newly created IGW, click **Actions** → **Attach to VPC**.
5. Select `vpc-intro-project` and click **Attach internet gateway**.

**AWS CLI**

```bash
# Create IGW
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=vpc-intro-project-IGW}]'

# Attach IGW to VPC (replace placeholders)
aws ec2 attach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>
```

---

### Step 5 — Create a Route Table and Associate Subnets

**Console**

1. Click **Route Tables** → **Create route table**.
   - **Name**: `Main-RouteTable`
   - **VPC**: `vpc-intro-project`
   - Click **Create route table**.
2. Select `Main-RouteTable`, go to the **Routes** tab → **Edit routes**.
   - Click **Add route**:
     - **Destination**: `0.0.0.0/0`
     - **Target**: select `vpc-intro-project-IGW`
   - Click **Save changes**.
3. Go to the **Subnet associations** tab → **Edit subnet associations**.
   - Select `Main-Subnet`, `Subnet-1`, and `Subnet-2`.
   - Click **Save associations**.

**AWS CLI**

```bash
# Create route table
aws ec2 create-route-table \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Main-RouteTable}]'

# Add default route to IGW
aws ec2 create-route \
  --route-table-id <route-table-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <igw-id>

# Associate each subnet
aws ec2 associate-route-table --route-table-id <route-table-id> --subnet-id <main-subnet-id>
aws ec2 associate-route-table --route-table-id <route-table-id> --subnet-id <subnet-1-id>
aws ec2 associate-route-table --route-table-id <route-table-id> --subnet-id <subnet-2-id>
```

---

### Step 6 — Create a Security Group

**Console**

1. In the VPC Dashboard (or EC2 Dashboard), click **Security Groups** → **Create security group**.
2. Fill in:
   - **Security group name**: `vpc-intro-project-SG`
   - **Description**: `Allow HTTP and SSH access`
   - **VPC**: `vpc-intro-project`
3. Under **Inbound rules**, click **Add rule** twice:

   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | HTTP | TCP | 80 | `0.0.0.0/0` (anywhere) |
   | SSH | TCP | 22 | **My IP** (console fills this in automatically) |

4. Leave **Outbound rules** as the default (all traffic allowed).
5. Click **Create security group**.

**AWS CLI**

```bash
# Create security group
aws ec2 create-security-group \
  --group-name vpc-intro-project-SG \
  --description "Allow HTTP and SSH access" \
  --vpc-id <vpc-id>

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow SSH from your IP only (replace with your actual public IP)
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 22 \
  --cidr <your-ip>/32
```

> **Security tip:** Never open SSH to `0.0.0.0/0`. Always restrict it to your own IP address.

---

### Step 7 — Launch EC2 Instances

You will launch **two** instances. Follow the steps below for each, changing the name and subnet as indicated.

**Console**

1. Navigate to the **EC2** service → **Instances** → **Launch instances**.
2. **Name**: `EC2-Instance-1` (use `EC2-Instance-2` for the second instance).
3. **AMI**: Amazon Linux 2 (free tier eligible). To find the latest AMI, search for "Amazon Linux 2" in the AMI catalog during instance launch.
4. **Instance type**: `t2.micro` (free tier eligible).
5. **Key pair**: Select an existing key pair or create a new one and download the `.pem` file. You need this to SSH into the instance.
6. Under **Network settings** → **Edit**:
   - **VPC**: `vpc-intro-project`
   - **Subnet**: `Subnet-1` (for Instance 1) / `Subnet-2` (for Instance 2)
   - **Auto-assign public IP**: **Enable**
   - **Security group**: Select existing → `vpc-intro-project-SG`
7. Click **Launch instance**.
8. Repeat for the second instance using `Subnet-2`.

**AWS CLI** (replace `<ami-id>` with the current Amazon Linux 2 AMI ID for your region — look it up with the command below or check the EC2 console AMI catalog)

```bash
# Look up the latest Amazon Linux 2 AMI ID for your region
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
```

```bash
# EC2 Instance 1 — in Subnet 1
aws ec2 run-instances \
  --image-id <ami-id> \
  --instance-type t2.micro \
  --key-name <your-key-pair-name> \
  --subnet-id <subnet-1-id> \
  --security-group-ids <sg-id> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EC2-Instance-1}]'

# EC2 Instance 2 — in Subnet 2
aws ec2 run-instances \
  --image-id <ami-id> \
  --instance-type t2.micro \
  --key-name <your-key-pair-name> \
  --subnet-id <subnet-2-id> \
  --security-group-ids <sg-id> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EC2-Instance-2}]'
```

---

### Step 8 — Verify Connectivity

After the instances reach the **running** state:

1. Copy the **Public IPv4 address** of each instance from the EC2 Dashboard.
2. **SSH test** (Linux/macOS terminal):
   ```bash
   chmod 400 <your-key.pem>
   ssh -i <your-key.pem> ec2-user@<instance-public-ip>
   ```
3. **HTTP test**: If you deploy a web server (`sudo yum install -y httpd && sudo systemctl start httpd`), open `http://<instance-public-ip>` in your browser.

---

## Step-by-Step: Cleaning Up (Avoid AWS Charges)

> **Important:** Delete resources in the order below. AWS will prevent deletion if dependent resources still exist.

### Cleanup Step 1 — Terminate EC2 Instances

**Console**

1. Go to **EC2** → **Instances**.
2. Select `EC2-Instance-1` and `EC2-Instance-2`.
3. Click **Instance state** → **Terminate instance** → **Terminate**.
4. Wait until both instances show the **Terminated** state before continuing.

**AWS CLI**

```bash
aws ec2 terminate-instances --instance-ids <instance-1-id> <instance-2-id>

# Wait for termination
aws ec2 wait instance-terminated --instance-ids <instance-1-id> <instance-2-id>
```

---

### Cleanup Step 2 — Delete the Security Group

**Console**

1. Go to **VPC** (or **EC2**) → **Security Groups**.
2. Select `vpc-intro-project-SG`.
3. Click **Actions** → **Delete security group** → **Delete**.

**AWS CLI**

```bash
aws ec2 delete-security-group --group-id <sg-id>
```

---

### Cleanup Step 3 — Disassociate and Delete the Route Table

**Console**

1. Go to **VPC** → **Route Tables**.
2. Select `Main-RouteTable`.
3. Click the **Subnet associations** tab → **Edit subnet associations**.
4. Deselect all subnets → **Save associations**.
5. Click **Actions** → **Delete route table** → **Delete**.

**AWS CLI**

```bash
# Disassociate subnets first
aws ec2 disassociate-route-table --association-id <assoc-id-main>
aws ec2 disassociate-route-table --association-id <assoc-id-subnet1>
aws ec2 disassociate-route-table --association-id <assoc-id-subnet2>

# Delete the route table
aws ec2 delete-route-table --route-table-id <route-table-id>
```

---

### Cleanup Step 4 — Detach and Delete the Internet Gateway

**Console**

1. Go to **VPC** → **Internet Gateways**.
2. Select `vpc-intro-project-IGW`.
3. Click **Actions** → **Detach from VPC** → **Detach internet gateway**.
4. Click **Actions** → **Delete internet gateway** → **Delete**.

**AWS CLI**

```bash
aws ec2 detach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>

aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>
```

---

### Cleanup Step 5 — Delete the Subnets

**Console**

1. Go to **VPC** → **Subnets**.
2. Select `Main-Subnet`, `Subnet-1`, and `Subnet-2` (one at a time or together if the console allows).
3. Click **Actions** → **Delete subnet** → **Delete**.

**AWS CLI**

```bash
aws ec2 delete-subnet --subnet-id <main-subnet-id>
aws ec2 delete-subnet --subnet-id <subnet-1-id>
aws ec2 delete-subnet --subnet-id <subnet-2-id>
```

---

### Cleanup Step 6 — Delete the VPC

**Console**

1. Go to **VPC** → **Your VPCs**.
2. Select `vpc-intro-project`.
3. Click **Actions** → **Delete VPC** → **Delete**.

**AWS CLI**

```bash
aws ec2 delete-vpc --vpc-id <vpc-id>
```

---

### Cleanup Step 7 — (Optional) Delete the Key Pair

If you created a dedicated key pair for this project, delete it to keep things tidy.

**Console**

1. Go to **EC2** → **Key Pairs**.
2. Select your key pair → **Actions** → **Delete** → **Delete**.

**AWS CLI**

```bash
aws ec2 delete-key-pair --key-name <your-key-pair-name>
# Also delete the local .pem file
rm <your-key.pem>
```

---

## Quick Reference — Resource IDs Tracker

Use this table to record IDs as you create resources so that cleanup is easy.

| Resource | ID |
|---|---|
| VPC | `vpc-` |
| Main Subnet | `subnet-` |
| Subnet 1 | `subnet-` |
| Subnet 2 | `subnet-` |
| Internet Gateway | `igw-` |
| Route Table | `rtb-` |
| Security Group | `sg-` |
| EC2 Instance 1 | `i-` |
| EC2 Instance 2 | `i-` |

---

## References

- [Amazon VPC Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [Amazon EC2 Documentation](https://docs.aws.amazon.com/ec2/index.html)
- [AWS CLI EC2 Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html)

---

## Author

[arhamsheikh17](https://github.com/arhamsheikh17)
