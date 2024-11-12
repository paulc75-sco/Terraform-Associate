# Week 5: State Management and Collaboration - Daily Training Exercises

## Day 1: Remote State Management
### Learning Objectives
- Master remote state configuration
- Understand state locking
- Practice state migration

### Exercise 1.1: S3 Backend Configuration
```hcl
# backend-setup/main.tf
provider "aws" {
  region = "us-west-2"
}

# Create S3 bucket for state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-${random_id.suffix.hex}"

  # Prevent accidental deletion
  lifecycle {
    prevent_destroy = true
  }
}

# Enable versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Create DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

# Random suffix for unique naming
resource "random_id" "suffix" {
  byte_length = 4
}

# Output the backend configuration
output "backend_config" {
  value = {
    bucket         = aws_s3_bucket.terraform_state.id
    dynamodb_table = aws_dynamodb_table.terraform_locks.name
    region         = "us-west-2"
  }
}
```

### Exercise 1.2: Implementing Remote State
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-xxxx"  # Use bucket from previous exercise
    key            = "terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# main.tf
# Example infrastructure using remote state
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  
  tags = {
    Name = "public-subnet"
  }
}
```

Tasks:
1. Initialize backend infrastructure
2. Migrate local state to remote
3. Test state locking
4. Practice state recovery

## Day 2: State Data Sources and Dependencies
### Learning Objectives
- Understand remote state data sources
- Practice cross-project state references
- Learn state dependency management

### Exercise 2.1: Remote State Data Sources
```hcl
# network/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.public[*].id
}

# app/main.tf
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "terraform-state-xxxx"
    key    = "network/terraform.tfstate"
    region = "us-west-2"
  }
}

resource "aws_instance" "app" {
  count         = 2
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = data.terraform_remote_state.network.outputs.subnet_ids[count.index]

  tags = {
    Name = "app-server-${count.index + 1}"
  }
}
```

### Exercise 2.2: Cross-Project Dependencies
```hcl
# shared-services/main.tf
resource "aws_kms_key" "shared" {
  description = "Shared encryption key"
  
  tags = {
    Environment = "shared"
  }
}

output "kms_key_arn" {
  value = aws_kms_key.shared.arn
}

# app/main.tf
data "terraform_remote_state" "shared" {
  backend = "s3"
  config = {
    bucket = "terraform-state-xxxx"
    key    = "shared-services/terraform.tfstate"
    region = "us-west-2"
  }
}

resource "aws_s3_bucket" "app_data" {
  bucket = "app-data-${random_id.suffix.hex}"

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        kms_master_key_id = data.terraform_remote_state.shared.outputs.kms_key_arn
        sse_algorithm     = "aws:kms"
      }
    }
  }
}
```

## Day 3: State Operations and Manipulation
### Learning Objectives
- Master state commands
- Practice state operations
- Learn state troubleshooting

### Exercise 3.1: State Operations
```bash
# Practice these state operations
terraform state list
terraform state show aws_instance.app[0]
terraform state mv aws_instance.app[0] aws_instance.web
terraform state rm aws_instance.app[1]
terraform state pull > backup.tfstate
terraform state push backup.tfstate
```

### Exercise 3.2: State Repair Scenarios
```hcl
# Scenario 1: Importing existing resources
resource "aws_s3_bucket" "imported" {
  bucket = "existing-bucket-name"
}

# Import command
# terraform import aws_s3_bucket.imported existing-bucket-name

# Scenario 2: Fixing state drift
resource "aws_security_group" "web" {
  name_prefix = "web-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# terraform refresh
```

## Day 4: Collaboration and Team Workflows
### Learning Objectives
- Understand team workflows
- Practice GitOps principles
- Learn collaboration patterns

### Exercise 4.1: GitOps Workflow
```bash
# Create branch structure
git checkout -b feature/add-rds
```

```hcl
# Feature branch changes
resource "aws_db_instance" "database" {
  identifier        = "app-database"
  engine            = "postgres"
  engine_version    = "13.7"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  
  username = "dbadmin"
  password = var.db_password

  skip_final_snapshot = true

  tags = {
    Environment = terraform.workspace
  }
}
```

### Exercise 4.2: CI/CD Pipeline Configuration
```yaml
# .github/workflows/terraform.yml
name: 'Terraform CI'

on:
  pull_request:
    branches: [ main ]

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1

    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan
```

## Day 5: Advanced State Management
### Learning Objectives
- Practice state file recovery
- Learn state backup strategies
- Master state troubleshooting

### Exercise 5.1: State Recovery Procedures
```hcl
# 1. Create state backup mechanism
resource "aws_s3_bucket" "state_backup" {
  bucket = "terraform-state-backup-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "state_backup" {
  bucket = aws_s3_bucket.state_backup.id
  versioning_configuration {
    status = "Enabled"
  }
}

# 2. Create backup policy
resource "aws_s3_bucket_lifecycle_configuration" "state_backup" {
  bucket = aws_s3_bucket.state_backup.id

  rule {
    id     = "archive-old-versions"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}
```

### Exercise 5.2: State Management Scenarios
```bash
# Scenario 1: Corrupted State
cp terraform.tfstate terraform.tfstate.backup
rm terraform.tfstate
terraform init

# Scenario 2: State Version Recovery
aws s3api get-object-versions --bucket terraform-state-xxxx --key terraform.tfstate

# Scenario 3: State Lock Breaking
terraform force-unlock LOCK_ID
```

Tasks:
1. Practice state recovery scenarios
2. Implement backup procedures
3. Document recovery processes
4. Test lock breaking

## Remember:
- Always backup state before operations
- Use meaningful commit messages
- Document state changes
- Test state operations in isolation
- Follow team workflows
- Use consistent naming conventions
- Keep state files secure

## Best Practices:
1. State Management
   - Regular backups
   - Version control
   - Access controls
   - Encryption

2. Team Collaboration
   - Branch protection
   - Code review
   - Documentation
   - Communication

3. Security
   - Encryption at rest
   - Access logging
   - Minimal permissions
   - Secure credentials

4. Recovery
   - Backup procedures
   - Recovery testing
   - Documentation
   - Team training