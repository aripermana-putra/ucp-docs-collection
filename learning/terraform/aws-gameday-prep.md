# Terraform AWS GameDay — 1-Hour Prep

**Context:** You know Crossplane. Terraform solves the same problem (declarative infra) but from the CLI and files,
not from Kubernetes controllers. The mental model transfer is fast — the main shift is: there's no reconciler running
in a cluster, Terraform runs as a one-shot CLI process that computes a diff and applies it.

---

## Mental Model: Crossplane vs Terraform

| Concept | Crossplane | Terraform |
|---|---|---|
| Desired state | Kubernetes manifest (YAML) | `.tf` file (HCL) |
| Actual state | Etcd via managed resource | State file (`terraform.tfstate`) |
| Reconciliation | Continuous controller loop | On-demand `terraform apply` |
| Drift correction | Automatic | Manual (`terraform apply` re-runs) |
| Provider | ProviderConfig + provider pod | `terraform { required_providers {} }` block |
| Dependency | Crossplane compositions | `depends_on`, implicit ref via `resource.type.name.attr` |
| Destroy | Delete the CR | `terraform destroy` |

The biggest practical difference: **state is a file**. In a team, that file lives in a remote backend (S3 + DynamoDB lock).
In the GameDay you will likely be given an AWS account — treat state as critical; do not let two people run apply simultaneously.

---

## Prerequisites

### 1. Terraform CLI

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

terraform version
# Terraform v1.x.x
```

### 2. AWS CLI

```bash
brew install awscli

aws --version
# aws-cli/2.x.x
```

### 3. Configure AWS credentials

The GameDay will give you temporary credentials. When you have them:

```bash
# Option A: env vars (preferred for temp creds)
export AWS_ACCESS_KEY_ID=AKIAxxx
export AWS_SECRET_ACCESS_KEY=xxx
export AWS_SESSION_TOKEN=xxx          # if using assumed role / SSO
export AWS_DEFAULT_REGION=ap-southeast-1

# Verify
aws sts get-caller-identity
```

```bash
# Option B: named profile
aws configure --profile gameday
# Then prefix every command: AWS_PROFILE=gameday terraform apply
```

### 4. tfenv (optional but useful — lets you switch Terraform versions)

```bash
brew install tfenv
tfenv install 1.9.0
tfenv use 1.9.0
```

### 5. Editor

VS Code with the **HashiCorp Terraform** extension gives you autocomplete, inline docs, and format-on-save.

```bash
code --install-extension hashicorp.terraform
```

### 6. Colima

Colima is not required for Terraform itself, but if the GameDay has any Docker-based tasks (e.g., pushing a container image):

```bash
colima start
docker ps   # sanity check
```

---

## Project Structure (what you'll see or create)

```
my-infra/
├── main.tf          # resources
├── variables.tf     # input variables
├── outputs.tf       # output values
├── providers.tf     # provider + backend config
└── terraform.tfvars # variable values (do not commit secrets)
```

For a team scenario add:

```
└── backend.tf       # remote state: S3 bucket + DynamoDB lock table
```

---

## HCL Syntax Crash Course

### Provider block

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
  region = var.aws_region
}
```

### Resource block

```hcl
resource "<TYPE>" "<LOCAL_NAME>" {
  <argument> = <value>
}

# Example
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "gameday-vpc"
  }
}
```

### Reference another resource

```hcl
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id     # <TYPE>.<LOCAL_NAME>.<ATTRIBUTE>
  cidr_block = "10.0.1.0/24"
}
```

### Variables

```hcl
# variables.tf
variable "aws_region" {
  type    = string
  default = "ap-southeast-1"
}

# usage
provider "aws" {
  region = var.aws_region
}
```

### Outputs

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

### Data sources (read existing infra, don't manage it)

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

### Locals (computed values, not inputs)

```hcl
locals {
  name_prefix = "gameday-${var.env}"
}

resource "aws_s3_bucket" "assets" {
  bucket = "${local.name_prefix}-assets"
}
```

---

## Core CLI Commands

```bash
# Initialize — downloads providers, sets up backend
terraform init

# Preview changes — never skip this
terraform plan

# Preview and save to file (use in CI or for review)
terraform plan -out=tfplan
terraform apply tfplan

# Apply changes interactively
terraform apply

# Apply without prompt (use carefully)
terraform apply -auto-approve

# Destroy everything
terraform destroy

# Destroy a specific resource
terraform destroy -target=aws_instance.web

# Show current state
terraform show

# List managed resources
terraform state list

# Inspect a single resource in state
terraform state show aws_vpc.main

# Pull remote state locally
terraform state pull

# Format all .tf files
terraform fmt

# Validate syntax
terraform validate

# Graph dependencies (pipe to dot for visual)
terraform graph | dot -Tsvg > graph.svg

# Override a variable at runtime
terraform apply -var="aws_region=us-east-1"
terraform apply -var-file="prod.tfvars"
```

---

## Remote State Backend (critical for team scenarios)

If the GameDay has a shared AWS account, you need a remote backend to avoid state conflicts.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "gameday-tfstate-bucket"
    key            = "team-X/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "gameday-tfstate-lock"
    encrypt        = true
  }
}
```

After adding or changing backend config, run `terraform init` again — it will prompt to migrate state.

---

## Scenario-Based Practice

These mirror the GameDay objectives. Run these against a personal AWS account or use LocalStack (see below) for cost-free practice.

---

### Scenario 1 — VPC + Subnet + Internet Gateway

**Goal:** Provision a functional public network.

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "gameday-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "gameday-igw" }
}

resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-southeast-1a"
  map_public_ip_on_launch = true
  tags                    = { Name = "public-a" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}
```

```bash
terraform init && terraform plan && terraform apply
```

---

### Scenario 2 — EC2 with Security Group

**Goal:** Deploy a web server with controlled ingress.

```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "al2" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
  EOF

  tags = { Name = "web-server" }
}

output "web_public_ip" {
  value = aws_instance.web.public_ip
}
```

---

### Scenario 3 — Simulate misconfiguration and fix it

**Goal:** Practice the troubleshoot-and-fix cycle (a core GameDay skill).

1. Intentionally set a wrong port in the security group (e.g., `8080` instead of `80`)
2. `terraform apply`
3. Notice the site doesn't respond
4. Fix the port back in `.tf`
5. `terraform plan` — confirm only the security group rule changes
6. `terraform apply`

Key lesson: `terraform plan` always shows exactly what will change. Read it before every apply.

---

### Scenario 4 — RDS in a private subnet (connectivity failure simulation)

**Goal:** Deploy RDS with proper network isolation, simulate a connectivity failure.

```hcl
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "ap-southeast-1a"
  tags              = { Name = "private-a" }
}

resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "ap-southeast-1b"
  tags              = { Name = "private-b" }
}

resource "aws_db_subnet_group" "main" {
  name       = "gameday-db-subnet-group"
  subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_b.id]
}

resource "aws_security_group" "rds" {
  name   = "rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]   # only from web tier
  }
}

resource "aws_db_instance" "main" {
  identifier        = "gameday-db"
  engine            = "postgres"
  engine_version    = "15"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  db_name           = "appdb"
  username          = "admin"
  password          = var.db_password   # never hardcode

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  skip_final_snapshot    = true
}
```

**Simulate failure:** Remove `aws_security_group.web.id` from RDS ingress, apply, observe connection refused from the web tier. Add it back, apply.

---

### Scenario 5 — High Availability with Auto Scaling Group

**Goal:** Understand the HA pattern before the GameDay asks for it.

```hcl
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = data.aws_ami.al2.id
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = base64encode(<<-EOF
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
  EOF
  )
}

resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  min_size            = 2
  max_size            = 4
  desired_capacity    = 2
  vpc_zone_identifier = [aws_subnet.public_a.id]   # add subnets in multiple AZs for real HA

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "web-asg-instance"
    propagate_at_launch = true
  }
}
```

For true HA: `vpc_zone_identifier` should include subnets in at least two AZs.

---

### Scenario 6 — Targeted destroy and import (recovery skills)

```bash
# Destroy only one resource without touching the rest
terraform destroy -target=aws_instance.web

# Import an existing resource into state
# (if infra was created outside Terraform — common in GameDay catch-up situations)
terraform import aws_vpc.main vpc-0abc123456789

# After import, run plan — it should show no changes if your .tf matches reality
terraform plan
```

---

## LocalStack — Practice Without an AWS Account

LocalStack emulates AWS API endpoints locally (`localhost:4566`). Terraform and the AWS CLI talk to it exactly as they would to real AWS — no account, no cost.

### Prerequisites

```bash
# Install Colima if not already present
brew install colima
colima --version
```

LocalStack runs as a Docker container (image `localstack/localstack:2.3.2`) — no LocalStack CLI or account required.

### Practice environment setup

A dedicated practice environment lives in the kitchen-sink repo at `terraform-gameday/`. It uses a named Colima instance so it stays isolated from any other Docker workloads.

```bash
cd __Work/others/source/kitchen-sink/terraform-gameday

# Start Colima (named instance) + LocalStack
make up

# Check Colima + LocalStack container status
make status

# Check Colima instance directly
colima list

# Confirm LocalStack is responding
curl http://localhost:4566/_localstack/health

# List available services and their status
curl -s http://localhost:4566/_localstack/health | python3 -m json.tool

# Stop everything when done
make down

# Delete the Colima VM entirely after the workshop
colima delete terraform-gameday
```

`make up` creates the `terraform-gameday` Colima instance (2 CPU, 4GB RAM, 20GB disk) and starts LocalStack as a Docker container in the background.

### Terraform provider config for LocalStack

Use this provider block in any scenario under `terraform-gameday/scenarios/`:

```hcl
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    ec2 = "http://localhost:4566"
    s3  = "http://localhost:4566"
    rds = "http://localhost:4566"
    iam = "http://localhost:4566"
  }
}
```

Before running `terraform` commands, export the Docker socket so the CLI targets the right Colima instance:

```bash
source terraform-gameday/scripts/env.sh
```

Note: LocalStack free tier supports core services (S3, EC2, IAM, SQS, VPC). RDS and some advanced services require Pro.

---

## GameDay Tactics

**Before the game starts:**
- Confirm credentials work: `aws sts get-caller-identity`
- Set up a remote backend with your team immediately — do this before writing any resource
- Agree on a team key prefix in the S3 backend to avoid state collisions
- Assign one person as "state owner" early to avoid concurrent apply conflicts

**During the game:**
- Always `terraform plan` before `terraform apply` — read the diff
- When something breaks, `terraform state show <resource>` is your first debugging step
- `terraform console` lets you test expressions interactively (useful for debugging interpolations)

```bash
# Sync state to actual cloud state without making changes
terraform apply -refresh-only
```

**Common traps:**
- Hardcoding AZs — use `data "aws_availability_zones" "available" {}` instead
- Forgetting egress rules on security groups — AWS denies all egress by default on custom SGs
- Missing `depends_on` when implicit dependency isn't captured (rare but happens with modules)
- Running `apply` from two terminals simultaneously without a DynamoDB state lock in place

---

## Quick Reference Card

| Command | Purpose |
|---|---|
| `terraform init` | Bootstrap — download providers, init backend |
| `terraform plan` | Diff — always run before apply |
| `terraform apply` | Converge — make reality match config |
| `terraform destroy` | Teardown |
| `terraform fmt` | Format all `.tf` files |
| `terraform validate` | Syntax check |
| `terraform state list` | List managed resources |
| `terraform state show <resource>` | Inspect a resource in state |
| `terraform import <resource> <id>` | Adopt existing infra into state |
| `terraform console` | REPL for expressions |
| `terraform apply -refresh-only` | Sync state without making changes |
| `terraform destroy -target=<resource>` | Destroy one resource |
