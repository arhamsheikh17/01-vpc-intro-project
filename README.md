# VPC Intro Project

## Architecture Overview

This project demonstrates how to build a basic, production-style **Virtual Private Cloud (VPC)** on AWS from scratch. By the end you will have:

```
AWS Region (e.g. us-east-1)
└── VPC  10.0.0.0/16
    ├── Public Subnet   10.0.1.0/24  (AZ-a)
    ├── Private Subnet  10.0.2.0/24  (AZ-a)
    ├── Internet Gateway  ──► Public Subnet (outbound internet access)
    ├── Public Route Table   0.0.0.0/0 → IGW
    ├── Private Route Table  (local only, no internet)
    ├── Public Security Group   (inbound SSH 22, HTTP 80 / outbound all)
    └── Private Security Group  (inbound SSH 22 from public SG only)
```

### What each resource does

| Resource | Purpose |
|---|---|
| **VPC** | Isolated private network for all your AWS resources |
| **Public Subnet** | Subnet whose route table points to the Internet Gateway – resources here can reach the internet |
| **Private Subnet** | Subnet with no direct internet route – ideal for databases, app servers |
| **Internet Gateway (IGW)** | Allows two-way traffic between the VPC and the public internet |
| **Public Route Table** | Routes `0.0.0.0/0` traffic to the IGW for public subnet resources |
| **Private Route Table** | Contains only local VPC routes; no internet access |
| **Security Groups** | Stateful firewalls controlling inbound/outbound traffic per resource |

---

## Repository Structure

Choose the guide that matches the way you prefer to work:

```
01-vpc-intro-project/
├── README.md              ← You are here (architecture overview)
├── aws-cli/
│   └── README.md          ← Step-by-step guide using the AWS CLI
└── aws-console-ui/
    └── README.md          ← Step-by-step guide using the AWS Management Console (UI)
```

| Folder | Who should use it |
|---|---|
| [`aws-cli/`](./aws-cli/README.md) | Developers / DevOps engineers comfortable with the terminal |
| [`aws-console-ui/`](./aws-console-ui/README.md) | Beginners or anyone who prefers point-and-click in the browser |

Both guides build **exactly the same architecture** – just through different tools.

---

## Prerequisites

- An AWS account (free tier is sufficient)
- An IAM user / role with at least `AmazonVPCFullAccess` and `AmazonEC2FullAccess`
- For the CLI guide: AWS CLI v2 installed and configured (`aws configure`)
- For the UI guide: a modern web browser and access to the [AWS Console](https://console.aws.amazon.com)
