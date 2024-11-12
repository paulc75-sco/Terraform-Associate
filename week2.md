# Week 2: Variables, Outputs, and State Management - Daily Training Exercises

## Day 1: Basic Variables
### Learning Objectives
- Understand variable types
- Learn variable declaration methods
- Practice variable referencing

### Exercise 1.1: Variable Types and Declarations
Create a new directory structure:
```bash
mkdir -p week2/day1
cd week2/day1
touch main.tf variables.tf terraform.tfvars
```

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Environment name"
  default     = "development"
}

variable "instance_count" {
  type        = number
  description = "Number of instances to launch"
  default     = 1
}

variable "enable_monitoring" {
  type        = bool
  description = "Enable detailed monitoring"
  default     = false
}

variable "allowed_ports" {
  type        = list(number)
  description = "List of allowed ports"
  default     = [22, 80, 443]
}

variable "resource_tags" {
  type        = map(string)
  description = "Tags for resources"
  default     = {
    Project = "TerraformTraining"
    Week    = "2"
  }
}
```

```hcl
# terraform.tfvars
environment      = "staging"
instance_count   = 2
enable_monitoring = true
allowed_ports    = [22, 80, 443, 8080]
resource_tags    = {
  Project     = "TerraformTraining"
  Week        = "2"
  Owner       = "StudentName"
  Environment = "Training"
}
```

### Exercise 1.2: Using Variables
```hcl
# main.tf
provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  count = var.instance_count

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  monitoring    = var.enable_monitoring

  tags = merge(var.resource_tags, {
    Name = "web-${var.environment}-${count.index + 1}"
  })
}

resource "aws_security_group" "web" {
  name        = "web-${var.environment}"
  description = "Security group for web servers"

  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

Tasks:
1. Practice different ways to set variables:
   - Using terraform.tfvars
   - Using -var command line flag
   - Using environment variables (TF_VAR_)
   - Using .tfvars.json format

2. Experiment with variable validation:
```hcl
variable "environment" {
  type        = string
  description = "Environment name"
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}
```

## Day 2: Complex Variable Types
### Learning Objectives
- Work with complex variable structures
- Understand variable validation
- Practice type constraints

### Exercise 2.1: Complex Variable Structures
```hcl
# variables.tf
variable "vpc_configuration" {
  type = object({
    cidr_block = string
    subnets = list(object({
      cidr_block        = string
      availability_zone = string
      public           = bool
    }))
    enable_dns = bool
    tags       = map(string)
  })

  default = {
    cidr_block = "10.0.0.0/16"
    subnets = [
      {
        cidr_block        = "10.0.1.0/24"
        availability_zone = "us-west-2a"
        public           = true
      },
      {
        cidr_block        = "10.0.2.0/24"
        availability_zone = "us-west-2b"
        public           = false
      }
    ]
    enable_dns = true
    tags = {
      Project = "TerraformTraining"
      Week    = "2"
    }
  }
}
```

### Exercise 2.2: Using Complex Variables
```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_configuration.cidr_block
  enable_dns_hostnames = var.vpc_configuration.enable_dns
  enable_dns_support   = var.vpc_configuration.enable_dns

  tags = var.vpc_configuration.tags
}

resource "aws_subnet" "subnets" {
  count             = length(var.vpc_configuration.subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.vpc_configuration.subnets[count.index].cidr_block
  availability_zone = var.vpc_configuration.subnets[count.index].availability_zone

  tags = merge(var.vpc_configuration.tags, {
    Name   = "subnet-${count.index + 1}"
    Public = var.vpc_configuration.subnets[count.index].public
  })
}
```

Tasks:
1. Create different environment configurations
2. Add validation rules for CIDR blocks
3. Practice accessing nested variable values
4. Add more subnet configurations

## Day 3: Outputs
### Learning Objectives
- Understand output values
- Learn output formatting
- Practice output dependencies

### Exercise 3.1: Basic Outputs
```hcl
# outputs.tf
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "List of subnet IDs"
  value       = aws_subnet.subnets[*].id
}

output "subnet_details" {
  description = "Detailed information about subnets"
  value = [
    for subnet in aws_subnet.subnets : {
      id                = subnet.id
      cidr_block       = subnet.cidr_block
      availability_zone = subnet.availability_zone
    }
  ]
}

output "sensitive_example" {
  description = "Example of sensitive output"
  value       = "secret-value"
  sensitive   = true
}
```

### Exercise 3.2: Advanced Output Usage
```hcl
locals {
  public_subnets  = [for subnet in aws_subnet.subnets : subnet.id if subnet.tags["Public"] == true]
  private_subnets = [for subnet in aws_subnet.subnets : subnet.id if subnet.tags["Public"] == false]
}

output "network_summary" {
  description = "Summary of network configuration"
  value = {
    vpc_id          = aws_vpc.main.id
    public_subnets  = local.public_subnets
    private_subnets = local.private_subnets
    total_subnets   = length(aws_subnet.subnets)
  }
}
```

Tasks:
1. Create outputs for all resources
2. Practice with sensitive outputs
3. Use output dependencies
4. Format complex outputs

## Day 4: State Management
### Learning Objectives
- Understand state file structure
- Practice state commands
- Learn state backup strategies

### Exercise 4.1: State Commands
```hcl
# main.tf
resource "aws_s3_bucket" "state_practice" {
  bucket = "terraform-state-practice-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_dynamodb_table" "state_lock" {
  name           = "terraform-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

Tasks:
1. Practice state commands:
   ```bash
   terraform state list
   terraform state show aws_s3_bucket.state_practice
   terraform state pull > state-backup.tfstate
   ```

2. Set up remote state:
```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-practice-xxxx"
    key            = "week2/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

## Day 5: State Operations
### Learning Objectives
- Practice state migrations
- Learn state manipulation
- Understand state locking

### Exercise 5.1: State Operations
Tasks:
1. Create multiple workspaces:
   ```bash
   terraform workspace new development
   terraform workspace new staging
   terraform workspace new production
   ```

2. Practice state moves:
   ```bash
   terraform state mv aws_s3_bucket.state_practice aws_s3_bucket.new_name
   ```

3. Import existing resources:
   ```bash
   terraform import aws_s3_bucket.imported existing-bucket-name
   ```

### Exercise 5.2: State Troubleshooting
Scenarios to practice:
1. Recover from state file loss
2. Resolve state conflicts
3. Fix state inconsistencies
4. Practice state file repairs

## Remember:
- Always backup state before operations
- Use proper state locking in team environments
- Practice safe state manipulation
- Document all state changes
- Clean up resources after exercises