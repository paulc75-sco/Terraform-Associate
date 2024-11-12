# Week 3: Resource Configuration and Dependencies - Daily Training Exercises

## Day 1: Advanced Resource Configuration
### Learning Objectives
- Master resource meta-arguments
- Understand resource behaviors
- Practice resource lifecycle rules

### Exercise 1.1: Meta-Arguments
```hcl
# main.tf
provider "aws" {
  region = "us-west-2"
}

# Count Example
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index + 1}"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# For_each with Map
resource "aws_s3_bucket" "data" {
  for_each = {
    logs    = "company-logs-bucket"
    backups = "company-backups-bucket"
    assets  = "company-assets-bucket"
  }

  bucket = "${each.value}-${random_id.suffix.hex}"

  tags = {
    Purpose = each.key
    Type    = "storage"
  }
}

# For_each with Set
resource "aws_iam_user" "developers" {
  for_each = toset(["john", "jane", "mike"])
  name     = each.key

  tags = {
    Role = "Developer"
    Team = "Engineering"
  }
}
```

### Exercise 1.2: Lifecycle Rules
```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "app-server"
  }

  lifecycle {
    create_before_destroy = true
    prevent_destroy      = true
    ignore_changes      = [tags]
  }
}

resource "aws_s3_bucket" "important" {
  bucket = "critical-data-${random_id.suffix.hex}"

  lifecycle {
    prevent_destroy = true
  }

  versioning {
    enabled = true
  }
}
```

Tasks:
1. Practice different meta-arguments usage
2. Test lifecycle rules
3. Understand creation/destruction order
4. Handle resource updates

## Day 2: Resource Dependencies
### Learning Objectives
- Master explicit and implicit dependencies
- Understand depends_on usage
- Practice dependency management

### Exercise 2.1: Implicit Dependencies
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "app" {
  vpc_id     = aws_vpc.main.id  # Implicit dependency
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "app-subnet"
  }
}

resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "Application security group"
  vpc_id      = aws_vpc.main.id  # Implicit dependency

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]  # Implicit dependency
  }
}
```

### Exercise 2.2: Explicit Dependencies
```hcl
resource "aws_s3_bucket" "log_bucket" {
  bucket = "app-logs-${random_id.suffix.hex}"
}

resource "aws_s3_bucket" "app_bucket" {
  bucket = "app-data-${random_id.suffix.hex}"

  depends_on = [aws_s3_bucket.log_bucket]
}

resource "aws_lambda_function" "processor" {
  filename         = "function.zip"
  function_name    = "log-processor"
  role            = aws_iam_role.lambda_role.arn
  handler         = "index.handler"
  runtime         = "nodejs14.x"

  depends_on = [
    aws_iam_role_policy_attachment.lambda_logs,
    aws_cloudwatch_log_group.lambda_log_group,
  ]
}
```

Tasks:
1. Map out resource dependencies
2. Test dependency chains
3. Practice with circular dependencies
4. Optimize resource creation order

## Day 3: Data Sources
### Learning Objectives
- Understand data source usage
- Practice data filtering
- Learn dynamic data retrieval

### Exercise 3.1: Basic Data Sources
```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_instance" "app" {
  ami               = data.aws_ami.amazon_linux_2.id
  instance_type     = "t2.micro"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "app-server"
  }
}
```

### Exercise 3.2: Advanced Data Source Usage
```hcl
data "aws_vpc" "existing" {
  tags = {
    Environment = "Production"
  }
}

data "aws_subnet_ids" "private" {
  vpc_id = data.aws_vpc.existing.id

  tags = {
    Tier = "Private"
  }
}

data "aws_security_groups" "web" {
  tags = {
    Application = "Web"
  }

  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
}

resource "aws_instance" "web" {
  count             = 2
  ami               = data.aws_ami.amazon_linux_2.id
  instance_type     = "t2.micro"
  subnet_id         = tolist(data.aws_subnet_ids.private.ids)[count.index % length(data.aws_subnet_ids.private.ids)]
  security_groups   = data.aws_security_groups.web.ids

  tags = {
    Name = "web-${count.index + 1}"
  }
}
```

Tasks:
1. Practice different data source types
2. Filter and query existing resources
3. Combine data sources with resources
4. Handle data source errors

## Day 4: Provisioners
### Learning Objectives
- Understand provisioner types
- Practice provisioner configurations
- Learn provisioner best practices

### Exercise 4.1: File and Local Provisioners
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name

  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("${path.module}/deployer.pem")
      host        = self.public_ip
    }
  }

  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }
}

resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("${path.module}/deployer.pub")
}
```

### Exercise 4.2: Remote Provisioners
```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y httpd",
      "sudo systemctl start httpd",
      "sudo systemctl enable httpd"
    ]

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("${path.module}/deployer.pem")
      host        = self.public_ip
    }
  }

  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Destroying instance ${self.id}' >> destroy_log.txt"
  }
}
```

Tasks:
1. Practice different provisioner types
2. Handle provisioner failures
3. Test connection configurations
4. Implement cleanup operations

## Day 5: Resource Import and State Operations
### Learning Objectives
- Master resource importing
- Practice state operations
- Learn resource migration

### Exercise 5.1: Resource Import
```hcl
# Existing resource configuration
resource "aws_instance" "imported" {
  # Configuration to match existing instance
  instance_type = "t2.micro"
  
  tags = {
    Name = "imported-instance"
  }
}

# Import command
# terraform import aws_instance.imported i-1234567890abcdef0
```

### Exercise 5.2: State Operations
```hcl
# Resource to be renamed/moved
resource "aws_s3_bucket" "old_name" {
  bucket = "my-bucket-${random_id.suffix.hex}"
}

# State move commands
# terraform state mv aws_s3_bucket.old_name aws_s3_bucket.new_name

# State remove command
# terraform state rm aws_s3_bucket.old_name
```

Tasks:
1. Import existing resources
2. Practice state moves
3. Handle state conflicts
4. Perform resource migrations

## Remember:
- Test provisioners in a safe environment
- Always backup state before operations
- Document all imports and moves
- Clean up resources after exercises
- Use version control for configurations