# Terraform / HCL / AWS Cheatsheet

---

## HCL Syntax

### Block types

```hcl
# Provider
provider "aws" {
  region = "ap-southeast-1"
}

# Resource
resource "<TYPE>" "<LOCAL_NAME>" {
  <argument> = <value>
}

# Data source (read existing infra, not managed)
data "<TYPE>" "<LOCAL_NAME>" {
  <filter>
}

# Variable (input)
variable "<NAME>" {
  type    = string
  default = "value"
}

# Output
output "<NAME>" {
  value = <expression>
}

# Local (computed value)
locals {
  name_prefix = "gameday-${var.env}"
}
```

### Reference syntax

```hcl
resource.TYPE.LOCAL_NAME.ATTRIBUTE     # resource attribute
data.TYPE.LOCAL_NAME.ATTRIBUTE         # data source attribute
var.NAME                               # input variable
local.NAME                             # local value
module.NAME.OUTPUT                     # module output
```

### String interpolation

```hcl
name   = "prefix-${var.env}-suffix"
bucket = "${local.name_prefix}-assets"
```

### Types

```hcl
variable "example" {
  type = string        # string
  type = number        # number
  type = bool          # bool
  type = list(string)  # list
  type = map(string)   # map
  type = object({      # object
    name = string
    port = number
  })
}
```

### Conditionals and loops

```hcl
# Conditional
name = var.env == "prod" ? "production" : "staging"

# Count (create N copies)
resource "aws_instance" "web" {
  count = 3
  tags  = { Name = "web-${count.index}" }
}

# For each (create from a map or set)
resource "aws_subnet" "public" {
  for_each          = { a = "10.0.1.0/24", b = "10.0.2.0/24" }
  cidr_block        = each.value
  availability_zone = "ap-southeast-1${each.key}"
}
```

### Functions

```hcl
length(var.list)                        # list/string length
toset(["a", "b"])                       # convert to set
flatten([["a"], ["b", "c"]])            # flatten nested lists
merge(map1, map2)                       # merge maps
lookup(map, key, default)               # safe map lookup
base64encode("string")                  # base64 encode
jsonencode({ key = "value" })           # encode to JSON string
cidrsubnet("10.0.0.0/16", 8, 1)        # compute subnet CIDR
```

---

## Terraform CLI

### Workflow

```bash
terraform init                          # download providers, init backend
terraform validate                      # syntax check
terraform fmt                           # format all .tf files
terraform plan                          # preview changes
terraform plan -out=tfplan              # save plan to file
terraform apply                         # apply changes (interactive)
terraform apply tfplan                  # apply saved plan
terraform apply -auto-approve           # apply without prompt
terraform destroy                       # destroy all resources
terraform destroy -target=TYPE.NAME     # destroy one resource
```

### State

```bash
terraform state list                    # list all managed resources
terraform state show TYPE.NAME          # inspect a resource in state
terraform state pull                    # dump remote state locally
terraform state rm TYPE.NAME            # remove resource from state (does not destroy)
terraform state mv TYPE.OLD TYPE.NEW    # rename resource in state
terraform import TYPE.NAME ID           # import existing resource into state
terraform apply -refresh-only           # sync state to real cloud without making changes
```

### Debugging

```bash
terraform console                       # REPL — test expressions interactively
terraform graph | dot -Tsvg > graph.svg # visualize dependency graph
TF_LOG=DEBUG terraform apply            # verbose logging
TF_LOG=ERROR terraform apply            # errors only
```

### Variables at runtime

```bash
terraform apply -var="region=us-east-1"
terraform apply -var-file="prod.tfvars"
```

### Workspace (environments)

```bash
terraform workspace list
terraform workspace new staging
terraform workspace select staging
terraform workspace show
```

---

## AWS CLI

### Identity

```bash
aws sts get-caller-identity             # who am I?
aws configure list                      # show active profile/credentials
aws configure --profile NAME            # set up a named profile
```

### EC2

```bash
aws ec2 describe-instances
aws ec2 describe-instances --filters "Name=tag:Name,Values=web-server"
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' --output table
aws ec2 start-instances --instance-ids i-0abc123
aws ec2 stop-instances --instance-ids i-0abc123
aws ec2 terminate-instances --instance-ids i-0abc123
aws ec2 describe-security-groups --group-ids sg-0abc123
```

### VPC / Networking

```bash
aws ec2 describe-vpcs
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0abc123"
aws ec2 describe-internet-gateways
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0abc123"
```

### S3

```bash
aws s3 ls                               # list buckets
aws s3 ls s3://bucket-name              # list bucket contents
aws s3 cp file.txt s3://bucket-name/    # upload
aws s3 sync ./dir s3://bucket-name/     # sync directory
aws s3 rb s3://bucket-name --force      # delete bucket and contents
```

### IAM

```bash
aws iam get-user
aws iam list-roles
aws iam list-attached-role-policies --role-name NAME
aws iam get-role --role-name NAME
```

### RDS

```bash
aws rds describe-db-instances
aws rds describe-db-instances --db-instance-identifier NAME
```

### Logs

```bash
aws logs describe-log-groups
aws logs tail /aws/ec2/my-log-group --follow
```

### Output formatting

```bash
--output table                          # human-readable table
--output json                           # JSON
--output text                           # plain text (for scripting)
--query 'KEY[*].FIELD'                  # JMESPath filter
```

---

## Common Patterns

### Remote state backend (team setup)

```hcl
terraform {
  backend "s3" {
    bucket         = "tfstate-bucket"
    key            = "team-X/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "tfstate-lock"
    encrypt        = true
  }
}
```

### Data source — look up existing resource

```hcl
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["my-vpc"]
  }
}

resource "aws_subnet" "new" {
  vpc_id     = data.aws_vpc.existing.id
  cidr_block = "10.0.99.0/24"
}
```

### Dynamic AZ lookup (avoid hardcoding)

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count             = 2
  availability_zone = data.aws_availability_zones.available.names[count.index]
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  vpc_id            = aws_vpc.main.id
}
```

### Security group — allow only from another SG

```hcl
ingress {
  from_port       = 5432
  to_port         = 5432
  protocol        = "tcp"
  security_groups = [aws_security_group.web.id]
}
```

### User data (bootstrap script)

```hcl
user_data = base64encode(<<-EOF
  #!/bin/bash
  dnf install -y nginx
  systemctl enable --now nginx
EOF
)
```

---

## Common Traps

| Trap | Fix |
|------|-----|
| Security group missing egress | Add explicit egress block — AWS denies all egress on custom SGs by default |
| Hardcoded AZs | Use `data "aws_availability_zones"` |
| Two people running apply simultaneously | Set up DynamoDB state lock in the S3 backend |
| Running `apply` without reading `plan` | Always `terraform plan` first — read the diff |
| Resource created outside Terraform | Use `terraform import` to adopt it into state |
| State out of sync with reality | Run `terraform apply -refresh-only` to resync |
