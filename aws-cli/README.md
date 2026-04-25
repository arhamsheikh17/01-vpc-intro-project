# VPC Intro Project — AWS CLI Guide

> **Audience:** Developers / DevOps engineers who prefer working in the terminal.  
> If you prefer clicking through a browser, see the [AWS Console UI guide](../aws-console-ui/README.md).

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

---

## Prerequisites

- AWS CLI v2 installed → [Install guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- Configured with credentials: `aws configure`
- A default region set (e.g. `us-east-1`)

Verify your setup:
```bash
aws sts get-caller-identity
```

---

## Part 1 — Create the VPC Infrastructure

### Step 1 — Create the VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=vpc-intro-project}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC ID: $VPC_ID"
```

Enable DNS hostnames on the VPC (required for public instances to receive a public DNS name):

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-hostnames '{"Value":true}'
```

---

### Step 2 — Create the Public Subnet

```bash
PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-public}]' \
  --query 'Subnet.SubnetId' \
  --output text)

echo "Public Subnet ID: $PUBLIC_SUBNET_ID"
```

Enable auto-assign public IPv4 on the public subnet so instances launched here get a public IP automatically:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --map-public-ip-on-launch
```

---

### Step 3 — Create the Private Subnet

```bash
PRIVATE_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-private}]' \
  --query 'Subnet.SubnetId' \
  --output text)

echo "Private Subnet ID: $PRIVATE_SUBNET_ID"
```

---

### Step 4 — Create and Attach an Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=igw-intro}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo "IGW ID: $IGW_ID"

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"
```

---

### Step 5 — Create the Public Route Table

```bash
PUBLIC_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-public}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "Public Route Table ID: $PUBLIC_RT_ID"
```

Add the default route to the Internet Gateway:

```bash
aws ec2 create-route \
  --route-table-id "$PUBLIC_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"
```

Associate the public route table with the public subnet:

```bash
aws ec2 associate-route-table \
  --route-table-id "$PUBLIC_RT_ID" \
  --subnet-id "$PUBLIC_SUBNET_ID"
```

---

### Step 6 — Create the Private Route Table

```bash
PRIVATE_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "Private Route Table ID: $PRIVATE_RT_ID"
```

Associate the private route table with the private subnet:

```bash
aws ec2 associate-route-table \
  --route-table-id "$PRIVATE_RT_ID" \
  --subnet-id "$PRIVATE_SUBNET_ID"
```

---

### Step 7 — Create the Public Security Group

```bash
PUBLIC_SG_ID=$(aws ec2 create-security-group \
  --group-name "sg-public" \
  --description "Public security group – SSH and HTTP from internet" \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-public}]' \
  --query 'GroupId' \
  --output text)

echo "Public SG ID: $PUBLIC_SG_ID"
```

Allow inbound SSH (port 22) and HTTP (port 80):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$PUBLIC_SG_ID" \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id "$PUBLIC_SG_ID" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

> Outbound "allow all" is the default for new security groups; no extra command needed.

---

### Step 8 — Create the Private Security Group

```bash
PRIVATE_SG_ID=$(aws ec2 create-security-group \
  --group-name "sg-private" \
  --description "Private security group – SSH from public SG only" \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-private}]' \
  --query 'GroupId' \
  --output text)

echo "Private SG ID: $PRIVATE_SG_ID"
```

Allow SSH only from the public security group (not the entire internet):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$PRIVATE_SG_ID" \
  --protocol tcp --port 22 \
  --source-group "$PUBLIC_SG_ID"
```

---

### Step 9 — Verify Everything Was Created

```bash
echo "=== VPC ==="
aws ec2 describe-vpcs --vpc-ids "$VPC_ID" \
  --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,State:State}' --output table

echo "=== Subnets ==="
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table

echo "=== Internet Gateway ==="
aws ec2 describe-internet-gateways --internet-gateway-ids "$IGW_ID" \
  --query 'InternetGateways[*].{ID:InternetGatewayId,Attachments:Attachments}' --output table

echo "=== Route Tables ==="
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes[*].{Dest:DestinationCidrBlock,Target:GatewayId}}' \
  --output json

echo "=== Security Groups ==="
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'SecurityGroups[*].{ID:GroupId,Name:GroupName}' --output table
```

---

## Part 2 — Clean Up (Delete Everything)

> **Delete resources in reverse order** to avoid dependency errors.

### Step 1 — Delete Security Groups

```bash
aws ec2 delete-security-group --group-id "$PRIVATE_SG_ID"
aws ec2 delete-security-group --group-id "$PUBLIC_SG_ID"
```

### Step 2 — Disassociate and Delete Route Tables

Find and delete route table associations first:

```bash
# Public RT association
ASSOC_ID=$(aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RT_ID" \
  --query 'RouteTables[0].Associations[?Main==`false`].RouteTableAssociationId' \
  --output text)
aws ec2 disassociate-route-table --association-id "$ASSOC_ID"

# Private RT association
ASSOC_ID=$(aws ec2 describe-route-tables \
  --route-table-ids "$PRIVATE_RT_ID" \
  --query 'RouteTables[0].Associations[?Main==`false`].RouteTableAssociationId' \
  --output text)
aws ec2 disassociate-route-table --association-id "$ASSOC_ID"
```

Delete the route tables:

```bash
aws ec2 delete-route-table --route-table-id "$PUBLIC_RT_ID"
aws ec2 delete-route-table --route-table-id "$PRIVATE_RT_ID"
```

### Step 3 — Detach and Delete the Internet Gateway

```bash
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"

aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID"
```

### Step 4 — Delete the Subnets

```bash
aws ec2 delete-subnet --subnet-id "$PUBLIC_SUBNET_ID"
aws ec2 delete-subnet --subnet-id "$PRIVATE_SUBNET_ID"
```

### Step 5 — Delete the VPC

```bash
aws ec2 delete-vpc --vpc-id "$VPC_ID"
echo "VPC $VPC_ID deleted."
```

### Step 6 — Confirm Everything Is Gone

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=vpc-intro-project" \
  --query 'Vpcs' --output text
# Expected output: (empty)
```

---

## Quick Reference — All Resource IDs

After running Part 1, save these values for later use or cleanup:

```
VPC_ID          = <printed by Step 1>
PUBLIC_SUBNET_ID = <printed by Step 2>
PRIVATE_SUBNET_ID= <printed by Step 3>
IGW_ID          = <printed by Step 4>
PUBLIC_RT_ID    = <printed by Step 5>
PRIVATE_RT_ID   = <printed by Step 6>
PUBLIC_SG_ID    = <printed by Step 7>
PRIVATE_SG_ID   = <printed by Step 8>
```

> If you opened a new terminal and lost the shell variables, retrieve them with:
> ```bash
> aws ec2 describe-vpcs --filters "Name=tag:Name,Values=vpc-intro-project" --query 'Vpcs[0].VpcId' --output text
> ```
