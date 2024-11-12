# Week 1: Terraform Fundamentals - Daily Training Exercises

## Day 1: Introduction to Infrastructure as Code
### Learning Objectives
- Set up a working Terraform environment
- Understand basic Terraform workflow
- Create and manage simple resources

### Exercise 1.1: Environment Setup
1. Install Terraform:
   ```bash
   # Download and install Terraform
   brew install terraform   # For MacOS
   # or
   choco install terraform # For Windows
   
   # Verify installation
   terraform version
   ```

2. Configure AWS Provider:
   ```bash
   # Set up AWS CLI
   aws configure
   
   # Create a workspace directory
   mkdir terraform-training
   cd terraform-training
   ```

### Exercise 1.2: First Configuration
1. Create your first Terraform file (main.tf):
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "first_bucket" {
  bucket = "your-unique-name-terraform-training"
  
  tags = {
    Environment = "Training"
    Day         = "1"
  }
}
```

Tasks:
- Initialize the directory (`terraform init`)
- Validate the configuration (`terraform validate`)
- Review the execution plan (`terraform plan`)
- Apply the configuration (`terraform apply`)
- Verify the bucket in AWS Console
- Destroy the resources (`terraform destroy`)

## Day 2: Basic Resource Management
### Learning Objectives
- Create multiple related resources
- Understand resource dependencies
- Work with tags and metadata

### Exercise 2.1: VPC Creation
```hcl
# main.tf
resource "aws_vpc" "training_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "training-vpc"
    Environment = "training"
    Day         = "2"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.training_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-west-2a"

  tags = {
    Name = "training-public-subnet"
  }
}
```

Tasks:
- Create the VPC and subnet
- Use AWS Console to verify the resources
- Practice modifying tags
- Try changing the CIDR block and observe the plan

### Exercise 2.2: Resource Dependencies
Add an Internet Gateway and Route Table:
```hcl
resource "aws_internet_gateway" "training_igw" {
  vpc_id = aws_vpc.training_vpc.id

  tags = {
    Name = "training-igw"
  }
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.training_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.training_igw.id
  }

  tags = {
    Name = "training-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}
```

Tasks:
- Observe implicit dependencies
- Try removing resources in different orders
- Understand the dependency chain

## Day 3: Working with Multiple Resource Types
### Learning Objectives
- Create different types of resources
- Understand resource attributes
- Practice resource referencing

### Exercise 3.1: EC2 Instance Setup
```hcl
resource "aws_security_group" "allow_ssh" {
  name        = "allow_ssh"
  description = "Allow SSH inbound traffic"
  vpc_id      = aws_vpc.training_vpc.id

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
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

resource "aws_instance" "training_instance" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI ID
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet.id
  
  vpc_security_group_ids = [aws_security_group.allow_ssh.id]

  tags = {
    Name = "training-instance"
  }
}
```

Tasks:
- Create the security group and EC2 instance
- Modify security group rules
- Add additional tags to the instance
- Try changing the instance type

## Day 4: State Management Basics
### Learning Objectives
- Understand Terraform state
- Practice state examination
- Learn basic state commands

### Exercise 4.1: State Manipulation
1. Create a DynamoDB table:
```hcl
resource "aws_dynamodb_table" "training_table" {
  name           = "terraform-training-table"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "ID"

  attribute {
    name = "ID"
    type = "S"
  }

  tags = {
    Environment = "training"
  }
}
```

Tasks:
- Apply the configuration
- Examine the state file (`terraform show`)
- List resources in state (`terraform state list`)
- Show specific resource details (`terraform state show aws_dynamodb_table.training_table`)

### Exercise 4.2: State Practice
Tasks:
- Make a manual change in AWS Console
- Run `terraform refresh`
- Observe state differences
- Run `terraform plan` to see drift detection

## Day 5: Terraform Commands and Workflow
### Learning Objectives
- Master essential Terraform commands
- Understand command outputs
- Practice workflow scenarios

### Exercise 5.1: Command Practice
Create this configuration:
```hcl
resource "aws_s3_bucket" "command_practice" {
  bucket = "terraform-command-practice-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

Tasks for each command:
1. `terraform init`:
   - Delete .terraform directory
   - Reinitialize
   - Observe downloaded providers

2. `terraform plan`:
   - Run plan with -out flag
   - Run plan with different variables
   - Understand plan output

3. `terraform apply`:
   - Apply with auto-approve
   - Apply targeting specific resources
   - Apply saved plan file

4. `terraform fmt`:
   - Intentionally mess up formatting
   - Run fmt
   - Observe changes

5. `terraform validate`:
   - Introduce syntax errors
   - Run validate
   - Fix errors

## Remember:
- Always run `terraform destroy` after completing exercises
- Save your configurations in version control
- Take notes on command behaviors
- Experiment with different options and flags

Each exercise builds upon the previous ones, creating a complete training environment. Ensure you understand each concept before moving to the next exercise.