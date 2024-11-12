# Week 4: Modules and Workspaces - Daily Training Exercises

## Day 1: Introduction to Modules
### Learning Objectives
- Understand module structure
- Learn module creation
- Practice module usage

### Exercise 1.1: Create Basic Module Structure
```bash
# Create module directory structure
mkdir -p modules/vpc
cd modules/vpc
touch main.tf variables.tf outputs.tf README.md
```

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for VPC"
}

variable "environment" {
  type        = string
  description = "Environment name"
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "List of public subnet CIDR blocks"
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "List of private subnet CIDR blocks"
}

variable "availability_zones" {
  type        = list(string)
  description = "List of availability zones"
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-public-subnet-${count.index + 1}"
    Environment = var.environment
    Type        = "Public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-private-subnet-${count.index + 1}"
    Environment = var.environment
    Type        = "Private"
  }
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "List of public subnet IDs"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "List of private subnet IDs"
}
```

### Exercise 1.2: Module Usage
```hcl
# root/main.tf
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = "10.0.0.0/16"
  environment          = "development"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.3.0/24", "10.0.4.0/24"]
  availability_zones   = ["us-west-2a", "us-west-2b"]
}

output "vpc_details" {
  value = {
    vpc_id            = module.vpc.vpc_id
    public_subnets    = module.vpc.public_subnet_ids
    private_subnets   = module.vpc.private_subnet_ids
  }
}
```

Tasks:
1. Initialize and apply the module
2. Test different variable combinations
3. Add more outputs
4. Document module usage

## Day 2: Advanced Module Concepts
### Learning Objectives
- Understand module composition
- Practice module versioning
- Learn module data sharing

### Exercise 2.1: Create a Security Module
```hcl
# modules/security/main.tf
resource "aws_security_group" "this" {
  name_prefix = "${var.name_prefix}-sg"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = var.tags
}
```

```hcl
# modules/security/variables.tf
variable "name_prefix" {
  type        = string
  description = "Prefix for security group name"
}

variable "vpc_id" {
  type        = string
  description = "VPC ID where security group will be created"
}

variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  description = "List of ingress rules"
}

variable "tags" {
  type        = map(string)
  description = "Tags for security group"
  default     = {}
}
```

### Exercise 2.2: Module Composition
```hcl
# root/main.tf
module "web_vpc" {
  source = "./modules/vpc"
  # ... vpc variables ...
}

module "web_security" {
  source = "./modules/security"
  
  name_prefix = "web"
  vpc_id      = module.web_vpc.vpc_id
  
  ingress_rules = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]

  tags = {
    Environment = "development"
    Purpose     = "web"
  }
}
```

## Day 3: Module Sources and Versioning
### Learning Objectives
- Understand module sources
- Practice version constraints
- Learn registry modules

### Exercise 3.1: Public Registry Modules
```hcl
# Using public registry modules
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = "development"
    Project     = "TerraformTraining"
  }
}
```

### Exercise 3.2: Git Source Modules
```hcl
module "s3_bucket" {
  source = "git::https://github.com/example/terraform-aws-s3-bucket.git?ref=v2.1.0"

  bucket_name = "my-secure-bucket"
  versioning  = true
  
  tags = {
    Environment = "development"
  }
}
```

Tasks:
1. Practice with different module sources
2. Test version constraints
3. Fork and modify public modules
4. Create module documentation

## Day 4: Workspaces
### Learning Objectives
- Understand workspace concept
- Practice workspace management
- Learn environment separation

### Exercise 4.1: Workspace Management
```bash
# Create and list workspaces
terraform workspace new development
terraform workspace new staging
terraform workspace new production
terraform workspace list
```

```hcl
# main.tf with workspace-specific configs
locals {
  environment = terraform.workspace

  # Environment-specific variables
  environment_configs = {
    development = {
      instance_type = "t2.micro"
      instance_count = 1
    }
    staging = {
      instance_type = "t2.small"
      instance_count = 2
    }
    production = {
      instance_type = "t2.medium"
      instance_count = 3
    }
  }

  # Current environment config
  config = local.environment_configs[local.environment]
}

resource "aws_instance" "app" {
  count         = local.config.instance_count
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = local.config.instance_type

  tags = {
    Name        = "app-${local.environment}-${count.index + 1}"
    Environment = local.environment
  }
}
```

### Exercise 4.2: Workspace-Specific Variables
```hcl
# development.tfvars
vpc_cidr    = "10.0.0.0/16"
environment = "development"
region      = "us-west-2"

# staging.tfvars
vpc_cidr    = "10.1.0.0/16"
environment = "staging"
region      = "us-west-2"

# production.tfvars
vpc_cidr    = "10.2.0.0/16"
environment = "production"
region      = "us-west-2"
```

## Day 5: Workspace Best Practices
### Learning Objectives
- Learn workspace organization
- Practice state management
- Understand backend configuration

### Exercise 5.1: Remote Backend with Workspaces
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### Exercise 5.2: Workspace Isolation
```hcl
# main.tf
locals {
  workspace_configs = {
    development = {
      state_bucket = "dev-terraform-state"
      lock_table   = "dev-terraform-locks"
    }
    staging = {
      state_bucket = "staging-terraform-state"
      lock_table   = "staging-terraform-locks"
    }
    production = {
      state_bucket = "prod-terraform-state"
      lock_table   = "prod-terraform-locks"
    }
  }
}

terraform {
  backend "s3" {}
}

# backend-config/development.hcl
bucket         = "dev-terraform-state"
key            = "terraform.tfstate"
region         = "us-west-2"
dynamodb_table = "dev-terraform-locks"
encrypt        = true
```

Tasks:
1. Create workspace-specific backends
2. Practice workspace switching
3. Implement state isolation
4. Document workspace procedures

## Remember:
- Always use consistent naming conventions
- Document module usage and requirements
- Test modules thoroughly before use
- Use workspace-specific variable files
- Keep workspaces synchronized
- Clean up unused workspaces
- Back up state files regularly