# Week 6: Terraform Associate Certification (003) - Final Review and Practice

## Day 1: Core Terraform Workflow Review
### Learning Objectives
- Review core Terraform commands
- Practice full deployment lifecycle
- Master initialization and planning

### Exercise 1.1: Comprehensive Infrastructure Setup
```hcl
# main.tf
provider "aws" {
  region = "us-west-2"
}

# VPC Configuration
resource "aws_vpc" "exam_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "exam-vpc"
  }
}

# Subnet Configuration
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.exam_vpc.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# Security Group
resource "aws_security_group" "exam_sg" {
  name        = "exam-sg"
  description = "Exam security group"
  vpc_id      = aws_vpc.exam_vpc.id

  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Variables
variable "allowed_ports" {
  type    = list(number)
  default = [22, 80, 443]
}

# Data Sources
data "aws_availability_zones" "available" {
  state = "available"
}
```

Practice Tasks:
1. Initialize with different backend configurations
2. Run plans with various targets
3. Apply with partial configurations
4. Practice state commands

### Exercise 1.2: Command Practice Scenarios
```bash
# Scenario 1: Backend Configuration
terraform init \
  -backend-config="bucket=my-terraform-state" \
  -backend-config="key=exam/terraform.tfstate" \
  -backend-config="region=us-west-2"

# Scenario 2: Planning with Variables
terraform plan \
  -var="allowed_ports=[22,80,443,8080]" \
  -out=exam.plan

# Scenario 3: Targeted Apply
terraform apply -target=aws_vpc.exam_vpc

# Scenario 4: State Operations
terraform state list
terraform state show aws_vpc.exam_vpc
terraform state pull > backup.tfstate
```

## Day 2: Variables and Outputs Review
### Learning Objectives
- Master variable types and usage
- Practice output configurations
- Understand data types and validation

### Exercise 2.1: Complex Variable Configurations
```hcl
# variables.tf
variable "vpc_config" {
  type = object({
    cidr_block = string
    subnets = list(object({
      cidr_block = string
      public     = bool
      tags       = map(string)
    }))
    enable_dns = bool
  })

  default = {
    cidr_block = "10.0.0.0/16"
    subnets = [
      {
        cidr_block = "10.0.1.0/24"
        public     = true
        tags = {
          Type = "Public"
        }
      },
      {
        cidr_block = "10.0.2.0/24"
        public     = false
        tags = {
          Type = "Private"
        }
      }
    ]
    enable_dns = true
  }

  validation {
    condition     = can(regex("^\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}/\\d{1,2}$", var.vpc_config.cidr_block))
    error_message = "VPC CIDR block must be a valid IPv4 CIDR."
  }
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### Exercise 2.2: Output Configurations
```hcl
# outputs.tf
output "vpc_details" {
  value = {
    id         = aws_vpc.exam_vpc.id
    cidr_block = aws_vpc.exam_vpc.cidr_block
    subnets    = {
      for subnet in aws_subnet.public :
      subnet.availability_zone => {
        id         = subnet.id
        cidr_block = subnet.cidr_block
        public_ip  = subnet.map_public_ip_on_launch
      }
    }
  }

  description = "VPC configuration details"
}

output "security_group_rules" {
  value = {
    ingress = [
      for rule in aws_security_group.exam_sg.ingress :
      {
        port        = rule.from_port
        protocol    = rule.protocol
        cidr_blocks = rule.cidr_blocks
      }
    ]
  }
  description = "Security group rules"
}
```

## Day 3: State Management and Workspace Review
### Learning Objectives
- Review state operations
- Master workspace management
- Practice state troubleshooting

### Exercise 3.1: State Management Scenarios
```hcl
# Example Infrastructure for State Practice
resource "aws_s3_bucket" "state_practice" {
  bucket = "state-practice-${random_id.suffix.hex}"
}

resource "aws_dynamodb_table" "state_lock" {
  name         = "state-lock-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

Practice Tasks:
1. State Migration:
```bash
# Export current state
terraform state pull > current.tfstate

# Import into new backend
terraform init -migrate-state

# List and verify resources
terraform state list
```

2. Workspace Management:
```bash
# Create and manage workspaces
terraform workspace new development
terraform workspace new staging
terraform workspace new production

# List and select workspaces
terraform workspace list
terraform workspace select development
```

## Day 4: Mock Exam Practice
### Learning Objectives
- Test exam readiness
- Practice time management
- Review weak areas

### Exercise 4.1: Sample Exam Questions
```hcl
# Question 1: Resource Configuration
# What's wrong with this configuration?
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  count         = 2
  
  tags = {
    Name = count.index
  }
}

# Question 2: Variable Declaration
# Which variable type is most appropriate?
variable "instance_types" {
  type = # Complete this
  default = {
    small  = "t2.micro"
    medium = "t2.small"
    large  = "t2.medium"
  }
}

# Question 3: Output Configuration
# How would you output the instance IDs?
output "instance_ids" {
  value = # Complete this
}
```

### Exercise 4.2: Troubleshooting Scenarios
```hcl
# Scenario 1: State Lock
# How would you handle a stuck state lock?

# Scenario 2: Resource Dependencies
# Fix the dependency cycle in this configuration
resource "aws_instance" "app" {
  depends_on = [aws_instance.db]
}

resource "aws_instance" "db" {
  depends_on = [aws_instance.app]
}

# Scenario 3: Variable Validation
# Add appropriate validation
variable "environment" {
  type = string
}
```

## Day 5: Final Review and Practice
### Learning Objectives
- Comprehensive review
- Time management practice
- Exam strategies

### Exercise 5.1: Comprehensive Infrastructure
```hcl
# Complete Infrastructure Setup
provider "aws" {
  region = var.region
}

module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = var.vpc_cidr
  environment = var.environment
}

module "security" {
  source = "./modules/security"
  
  vpc_id = module.vpc.vpc_id
  environment = var.environment
}

module "compute" {
  source = "./modules/compute"
  
  subnet_ids = module.vpc.private_subnet_ids
  security_group_ids = [module.security.security_group_id]
  environment = var.environment
}

terraform {
  backend "s3" {}
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

### Exercise 5.2: Final Practice Items
1. Core Workflow Practice:
   - Full initialization
   - Plan and apply
   - State management
   - Cleanup

2. Variable Management:
   - Complex type handling
   - Validation rules
   - Dynamic values

3. State Operations:
   - Remote state
   - State locking
   - Import/export

4. Module Usage:
   - Source types
   - Variable passing
   - Output handling

## Exam Day Preparation Checklist
1. Technical Setup:
   - Clean workspace
   - Stable internet connection
   - Required documentation bookmarked

2. Time Management:
   - 1 minute per question average
   - Mark difficult questions for review
   - Leave time for review

3. Key Topics Review:
   - Core workflow
   - State management
   - Variables and outputs
   - Modules
   - Workspaces
   - Provider configuration

4. Common Commands:
   ```bash
   terraform init
   terraform plan
   terraform apply
   terraform destroy
   terraform state
   terraform workspace
   terraform import
   terraform output
   ```

## Remember:
- Read questions carefully
- Understand the context
- Double-check answers
- Use process of elimination
- Trust your preparation
- Stay calm and focused