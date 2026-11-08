# 🏗️ Terraform-Based Multi-Tier VPC Infrastructure on AWS

**Author:** Het Bhingradiya

Build a custom AWS network (VPC) with a public and a private subnet using Terraform (Infrastructure as Code) instead of clicking through the AWS Console.

✅ Tested on Kali Linux (VirtualBox on Windows 11) with Terraform + AWS CLI v2, region `us-east-1`.

---

## 📑 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#️-architecture)
- [Resources Created](#-resources-created)
- [Prerequisites](#-prerequisites)
- [Step 1: Install Terraform](#-step-1-install-terraform)
- [Step 2: Install AWS CLI v2](#️-step-2-install-aws-cli-v2)
- [Step 3: Create an AWS Access Key](#-step-3-create-an-aws-access-key)
- [Step 4: Configure AWS Credentials](#-step-4-configure-aws-credentials)
- [Step 5: Create the Project Folder](#-step-5-create-the-project-folder)
- [Step 6: Write the Terraform Code](#-step-6-write-the-terraform-code-maintf)
- [Step 7: Initialize Terraform](#-step-7-initialize-terraform)
- [Step 8: Format and Validate](#-step-8-format-and-validate)
- [Step 9: Preview the Plan](#-step-9-preview-the-plan)
- [Step 10: Deploy the Infrastructure](#-step-10-deploy-the-infrastructure)
- [Step 11: Verify the Resources](#-step-11-verify-the-resources)
- [Step 12: Clean Up](#-step-12-clean-up-destroy)
- [Troubleshooting](#️-troubleshooting)
- [Security Best Practices](#-security-best-practices)
- [Key Learnings](#-key-learnings)
- [Author](#-author)

---

## 📌 Project Overview

| Item | Details |
|------|---------|
| **Goal** | Create a multi-tier VPC network on AWS using code |
| **Tool** | Terraform (HashiCorp Configuration Language, HCL) |
| **Cloud** | Amazon Web Services (AWS) |
| **Region** | `us-east-1` (N. Virginia) |
| **Provider** | `hashicorp/aws` (version `~> 5.0`) |
| **Cost** | VPC, subnets, route tables and Internet Gateway are free (no NAT Gateway used) |

### 🤔 What is Terraform?

Terraform is an **Infrastructure as Code (IaC)** tool. You describe the infrastructure you want in a file (`main.tf`), and Terraform talks to AWS and creates it for you. The same code can be reused, version-controlled and destroyed with a single command.

---

## 🗺️ Architecture

```
                         ┌───────────────────────────────────────────┐
                         │          Student-VPC  (10.0.0.0/16)       │
                         │                                           │
   🌐 Internet           │   ┌───────────────────────────────────┐   │
       │                 │   │  Public Subnet  (10.0.1.0/24)     │   │
       ▼                 │   │  Route: 0.0.0.0/0 → Internet GW   │   │
 ┌──────────────┐        │   └───────────────────────────────────┘   │
 │ Internet     │◄───────┤                                           │
 │ Gateway      │        │   ┌───────────────────────────────────┐   │
 │ Student-IGW  │        │   │  Private Subnet (10.0.10.0/24)    │   │
 └──────────────┘        │   │  Route: local only (no internet)  │   │
                         │   └───────────────────────────────────┘   │
                         └───────────────────────────────────────────┘
```

- 🟢 **Public subnet:** can reach the internet through the Internet Gateway.
- 🔒 **Private subnet:** no direct route to the Internet Gateway (safe place for databases and backend servers).

---

## 📦 Resources Created

Terraform creates **8 resources**:

| # | Terraform Resource | Name / Value | Purpose |
|---|--------------------|--------------|---------|
| 1 | `aws_vpc` | `Student-VPC` · `10.0.0.0/16` | Your private network in AWS |
| 2 | `aws_internet_gateway` | `Student-IGW` | Door between the VPC and the internet |
| 3 | `aws_subnet` (public) | `Public-Subnet` · `10.0.1.0/24` | Subnet with internet access |
| 4 | `aws_subnet` (private) | `Private-Subnet` · `10.0.10.0/24` | Subnet without direct internet access |
| 5 | `aws_route_table` (public) | `Public-RT` | Sends `0.0.0.0/0` to the IGW |
| 6 | `aws_route_table` (private) | `Private-RT` | Local traffic only |
| 7 | `aws_route_table_association` | public | Links `Public-RT` to the public subnet |
| 8 | `aws_route_table_association` | private | Links `Private-RT` to the private subnet |

> Also used: `data.aws_availability_zones` (reads the available zones, not counted as a created resource).

---

## ✅ Prerequisites

- 💻 A Linux machine or VM (Kali / Ubuntu / Debian), Windows 11 or macOS
- 🌐 Working internet connection
- ☁️ An AWS account
- 🔑 An IAM user (or permissions) that can create VPC resources, for example the `AmazonVPCFullAccess` policy
- 📝 A text editor (`nano` or VS Code)

---

## 🔧 Step 1: Install Terraform

### 🐧 Kali Linux / Debian / Ubuntu

```bash
# 1. Update the system and install required packages
sudo apt update
sudo apt install -y gnupg curl wget unzip software-properties-common
```

```bash
# 2. Add the HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

```bash
# 3. Add the HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com bookworm main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

> 💡 **Kali note:** Kali is Debian based. The command `lsb_release -cs` returns `kali-rolling`, which the HashiCorp repository does not have, so `bookworm` is written manually. On Ubuntu you can replace `bookworm` with `$(lsb_release -cs)`.

```bash
# 4. Install Terraform
sudo apt update
sudo apt install -y terraform
```

```bash
# 5. Verify
terraform -version
```

### 🪟 Windows 11 (PowerShell)

```powershell
winget install -e --id Hashicorp.Terraform
```

Close and reopen PowerShell, then run `terraform -version`.

### 🍎 macOS

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -version
```

---

## ☁️ Step 2: Install AWS CLI v2

### 🐧 Linux (Kali / Ubuntu / Debian)

First, check the CPU architecture:

```bash
uname -m
```

If the output is `x86_64`:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

If the output is `aarch64` (ARM):

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify:

```bash
aws --version
```

### 🪟 Windows 11 (PowerShell)

```powershell
winget install -e --id Amazon.AWSCLI
```

### 🍎 macOS

```bash
brew install awscli
```

---

## 🔑 Step 3: Create an AWS Access Key

1. Log in to the **AWS Management Console**.
2. Open **IAM → Users → your user → Security credentials**.
3. Under **Access keys**, click **Create access key**.
4. Choose **Command Line Interface (CLI)** and continue.
5. Copy the **Access Key ID** and the **Secret Access Key** and keep them safe.

> ⚠️ The secret key is shown **only once**. Never share it and never upload it to GitHub.

---

## 🔐 Step 4: Configure AWS Credentials

```bash
aws configure
```

Enter the values when asked:

```
AWS Access Key ID [None]: <your-access-key-id>
AWS Secret Access Key [None]: <your-secret-access-key>
Default region name [None]: us-east-1
Default output format [None]: json
```

Verify that the credentials work:

```bash
aws sts get-caller-identity
```

If the command returns your `Account`, `UserId` and `Arn`, the setup is correct. ✅

---

## 📁 Step 5: Create the Project Folder

```bash
cd ~
mkdir terraform-vpc-lab
cd terraform-vpc-lab
nano main.tf
```

> 💡 If you use VS Code: run `code .` inside the folder and create a new file named `main.tf`.

---

## 📝 Step 6: Write the Terraform Code (`main.tf`)

Paste the following code into `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Read the available Availability Zones in the region
data "aws_availability_zones" "available" {
  state = "available"
}

# ---------------------------------------------------------
# VPC
# ---------------------------------------------------------
resource "aws_vpc" "student_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "Student-VPC"
  }
}

# ---------------------------------------------------------
# Internet Gateway
# ---------------------------------------------------------
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.student_vpc.id

  tags = {
    Name = "Student-IGW"
  }
}

# ---------------------------------------------------------
# Subnets
# ---------------------------------------------------------
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.student_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = true

  tags = {
    Name = "Public-Subnet"
  }
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.student_vpc.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "Private-Subnet"
  }
}

# ---------------------------------------------------------
# Route Tables
# ---------------------------------------------------------

# Public route table: internet traffic goes through the IGW
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.student_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "Public-RT"
  }
}

# Private route table: no route to the Internet Gateway
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.student_vpc.id

  tags = {
    Name = "Private-RT"
  }
}

# ---------------------------------------------------------
# Route Table Associations
# ---------------------------------------------------------
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private_rt.id
}

# ---------------------------------------------------------
# Outputs
# ---------------------------------------------------------
output "vpc_id" {
  value = aws_vpc.student_vpc.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}
```

Save the file in nano:

1. Press `Ctrl + O`
2. Press `Enter` to confirm the file name `main.tf`
3. Press `Ctrl + X` to exit

Check that the file exists:

```bash
ls
```

### 🧠 Code Explained

| Block | What it does |
|-------|--------------|
| `terraform { required_providers }` | Tells Terraform to use the AWS provider |
| `provider "aws"` | Sets the AWS region to `us-east-1` |
| `data "aws_availability_zones"` | Reads the available zones so the subnets can use the first one |
| `aws_vpc` | Creates the VPC with CIDR `10.0.0.0/16` and DNS support |
| `aws_internet_gateway` | Creates the IGW and attaches it to the VPC |
| `aws_subnet` | Creates the public (`10.0.1.0/24`) and private (`10.0.10.0/24`) subnets |
| `aws_route_table` | Public table has the `0.0.0.0/0 → IGW` route, private table has none |
| `aws_route_table_association` | Connects each route table to its subnet |
| `output` | Prints the VPC ID and subnet IDs after deployment |

---

## 🚀 Step 7: Initialize Terraform

```bash
terraform init
```

This downloads the AWS provider plugin and creates the `.terraform` folder and the `.terraform.lock.hcl` file.

✅ **Expected result:**

```
Terraform has been successfully initialized!
```

---

## 🧹 Step 8: Format and Validate

```bash
terraform fmt
```

Formats the code in the standard Terraform style.

```bash
terraform validate
```

✅ **Expected result:**

```
Success! The configuration is valid.
```

---

## 🔍 Step 9: Preview the Plan

```bash
terraform plan
```

Terraform shows what it **will** create. Nothing is created yet.

✅ **Expected result at the end:**

```
Plan: 8 to add, 0 to change, 0 to destroy.
```

---

## ⚡ Step 10: Deploy the Infrastructure

```bash
terraform apply
```

Review the changes, then type:

```
yes
```

Wait a few seconds while Terraform builds the resources.

✅ **Expected result:**

```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

private_subnet_id = "subnet-xxxxxxxxxxxxxxxxx"
public_subnet_id  = "subnet-xxxxxxxxxxxxxxxxx"
vpc_id            = "vpc-xxxxxxxxxxxxxxxxx"
```

List everything Terraform manages and show the outputs:

```bash
terraform state list
terraform output
```

---

## 🔎 Step 11: Verify the Resources

### 🖥️ Option A: AWS Console

1. Open the AWS Console and make sure the region is **N. Virginia (us-east-1)**.
2. Go to the **VPC** service.
3. Check each item:

| Where | What to check |
|-------|---------------|
| **Your VPCs** | `Student-VPC` exists with CIDR `10.0.0.0/16` and state **Available** |
| **Subnets** | `Public-Subnet` is `10.0.1.0/24` and `Private-Subnet` is `10.0.10.0/24` |
| **Internet Gateways** | `Student-IGW` is **Attached** to `Student-VPC` |
| **Route Tables** | `Public-RT` has `0.0.0.0/0 → igw-xxxx` |
| **Route Tables** | `Private-RT` has only the `local` route (no IGW route) |

### ⌨️ Option B: AWS CLI

```bash
VPC_ID=$(terraform output -raw vpc_id)

aws ec2 describe-vpcs --vpc-ids "$VPC_ID" \
  --query "Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,State:State}" --output table
```

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}" --output table
```

```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "RouteTables[].{RouteTableId:RouteTableId,Routes:Routes[].DestinationCidrBlock}" --output json
```

---

## 🧨 Step 12: Clean Up (Destroy)

When the lab is finished, remove everything to keep your AWS account clean:

```bash
cd ~/terraform-vpc-lab
terraform destroy
```

Review the planned deletions and type `yes`.

✅ **Expected result:**

```
Destroy complete! Resources: 8 destroyed.
```

Then open the VPC Console and confirm that `Student-VPC` is gone.

> ⚠️ `terraform destroy` removes only the resources managed by this configuration. If you later add a NAT Gateway or other paid resources, remember they can add AWS charges.

---

## 🛠️ Troubleshooting

| ❌ Problem | ✅ Solution |
|-----------|------------|
| `Failed to connect` / `Temporary failure in name resolution` during `wget` | Internet or DNS issue. Run `ping -c 3 8.8.8.8` and `ping -c 3 google.com`. If the second fails, set `nameserver 8.8.8.8` in `/etc/resolv.conf`. In a VM, set the network adapter to **NAT** or **Bridged** |
| `terraform: command not found` | Terraform is not installed. Repeat Step 1 and reopen the terminal |
| `aws: command not found` | AWS CLI is not installed. Repeat Step 2 |
| `No valid credential sources found` | Run `aws configure` again with correct keys |
| `UnauthorizedOperation` / `AccessDenied` | Attach `AmazonVPCFullAccess` (or equivalent) to your IAM user |
| `No configuration files` | You are in the wrong folder. Run `cd ~/terraform-vpc-lab` |
| `Error: Invalid CIDR` | Check the CIDR values (`10.0.0.0/16`, `10.0.1.0/24`, `10.0.10.0/24`) |
| HashiCorp repository error in `apt update` | Open `/etc/apt/sources.list.d/hashicorp.list` and confirm it uses `bookworm` |
| Terraform install through `apt` fails | Download the Linux AMD64 zip from the official Terraform website, then `unzip` it and run `sudo mv terraform /usr/local/bin/` |

---

## 🔒 Security Best Practices

- 🚫 **Never** upload your Access Key or Secret Key to GitHub.
- 🙈 Hide your AWS Account ID before sharing console screenshots.
- 👤 Use an **IAM user** with limited permissions instead of the root account.
- 🗂️ Do not commit the Terraform state or provider folders. Create a `.gitignore` file in the project folder:

```gitignore
# Terraform local files
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars

# OS files
.DS_Store
```

> 💡 Keep `.terraform.lock.hcl` in the repository. It records the exact provider version used.

---

## 📚 Key Learnings

- 🧱 How to describe a network (VPC, subnets, gateway, route tables) as code
- 🔁 The Terraform workflow: `init → fmt → validate → plan → apply → destroy`
- 🌍 The difference between a public and a private subnet, controlled only by the route table
- 🧪 How to verify cloud resources using both the Console and the AWS CLI
- 💸 Why cleaning up resources after a lab matters

---

## 📂 Project Structure

```
terraform-vpc-lab/
├── main.tf                 # Terraform configuration
├── .terraform.lock.hcl     # Provider version lock file (created by terraform init)
├── .gitignore              # Files to keep out of Git
└── README.md               # This documentation
```

---

## 🏁 Summary

| Command | Purpose |
|---------|---------|
| `terraform init` | Download providers and prepare the folder |
| `terraform fmt` | Format the code |
| `terraform validate` | Check the syntax |
| `terraform plan` | Preview the changes |
| `terraform apply` | Create the infrastructure |
| `terraform state list` | List managed resources |
| `terraform output` | Show output values |
| `terraform destroy` | Delete the infrastructure |

---

## 👨‍💻 Author

**Het Bhingradiya**

Terraform-Based Multi-Tier VPC Infrastructure on AWS
