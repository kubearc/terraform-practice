# Terraform AWS Practice Lab  
Amazon Linux + S3 Backend + Application Load Balancer + SSH Key Access

This lab deploys a working AWS environment using Terraform.  
It includes a VPC, EC2 instance, load balancer, remote backend, and SSH access with a key pair.

---

## Project Structure
*Note*: Please create the file and directories according to below given Structure

```
terraform-aws-lab/
│── provider.tf
│── backend.tf
│── main.tf
│── variables.tf
│── outputs.tf
│
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── ec2/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── alb/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    └── s3_backend/
        ├── main.tf
        └── variables.tf
```

---

# Root Terraform Files

---

### provider.tf

```hcl
provider "aws" {
  region = var.aws_region
}
```

---

### variables.tf

```hcl
variable "aws_region" {
  default = "ap-south-1"   # Change if using a different AWS region
}

variable "project" {
  default = "terraform-lab"   # You can rename the project
}

variable "env" {
  default = "dev"   # Change to stage or prod if needed
}

variable "allowed_ssh_ip" {
  description = "Allowed SSH CIDR (default open to all)"
  default     = "0.0.0.0/0"
}

variable "public_key_path" {
  description = "Path to your SSH public key"
  default     = "~/.ssh/id_rsa.pub"   # Ensure this file exists before running Terraform
}
```

---

### backend.tf  
Enable only after running `terraform apply` once.

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state-manvir-lab"  # Must be globally unique
    key    = "env/dev/terraform.tfstate"
    region = "ap-south-1"
  }
}
```

---

### main.tf

```hcl
module "s3" {
  source     = "./modules/s3_backend"
  bucket_tag = "${var.project}-tf-state"
}

module "vpc" {
  source  = "./modules/vpc"
  project = var.project
  env     = var.env
}

module "ec2" {
  source            = "./modules/ec2"
  public_subnet_id  = module.vpc.public_subnets[0]
  allowed_ssh_ip    = var.allowed_ssh_ip
  public_key_path   = var.public_key_path
  project           = var.project
}

module "alb" {
  source             = "./modules/alb"
  public_subnet_ids  = module.vpc.public_subnets
  ec2_target_id      = module.ec2.instance_id
  vpc_id             = module.vpc.vpc_id
}
```

---

### outputs.tf

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "ec2_public_ip" {
  value = module.ec2.public_ip
}

output "ssh_command" {
  value = "ssh -i ~/.ssh/id_rsa ec2-user@${module.ec2.public_ip}"
}

output "alb_dns" {
  value = module.alb.alb_dns
}

output "state_bucket" {
  value = module.s3.bucket_name
}
```

---

# Modules

---

## Module: s3_backend

### main.tf

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = var.bucket_tag

  versioning {
    enabled = true
  }

  tags = {
    Name = var.bucket_tag
  }
}

output "bucket_name" {
  value = aws_s3_bucket.tf_state.bucket
}
```

### variables.tf

```hcl
variable "bucket_tag" {}
```

---

## Module: vpc

### variables.tf

```hcl
variable "project" {}
variable "env" {}
```

### main.tf

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "${var.project}-${var.env}-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  count                   = 2
  cidr_block              = "10.0.${count.index}.0/24"
  vpc_id                  = aws_vpc.main.id
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private" {
  count      = 2
  cidr_block = "10.0.${count.index + 10}.0/24"
  vpc_id     = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public_assoc" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### outputs.tf

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnets" {
  value = aws_subnet.public[*].id
}

output "private_subnets" {
  value = aws_subnet.private[*].id
}
```

---

## Module: ec2

### variables.tf

```hcl
variable "public_subnet_id" {}
variable "allowed_ssh_ip" {}
variable "project" {}
variable "public_key_path" {}
```

### main.tf

```hcl
data "aws_ami" "amazon_linux" {
  owners      = ["amazon"]
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}

resource "aws_key_pair" "ssh_key" {
  key_name   = "${var.project}-key"
  public_key = file(var.public_key_path)
}

resource "aws_security_group" "ec2_sg" {
  name = "${var.project}-ec2-sg"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_ip]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t2.micro"
  subnet_id              = var.public_subnet_id
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]
  key_name               = aws_key_pair.ssh_key.key_name
  associate_public_ip_address = true

  user_data = <<EOF
#!/bin/bash
yum update -y
yum install -y httpd
systemctl enable httpd
systemctl start httpd
echo "Terraform Amazon Linux Server with SSH Access Running" > /var/www/html/index.html
EOF
}

output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

## Module: alb

### variables.tf

```hcl
variable "public_subnet_ids" {}
variable "ec2_target_id" {}
variable "vpc_id" {}
```

### main.tf

```hcl
resource "aws_lb" "alb" {
  name               = "terraform-lab-alb"
  load_balancer_type = "application"
  subnets            = var.public_subnet_ids
}

resource "aws_lb_target_group" "tg" {
  name     = "terraform-alb-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

resource "aws_lb_target_group_attachment" "attach" {
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = var.ec2_target_id
  port             = 80
}

output "alb_dns" {
  value = aws_lb.alb.dns_name
}
```

---

# Validation Checklist

- Run `terraform init`
- Run `terraform apply`
- Check EC2 instance responds to SSH using the printed command
- Visit the EC2 public IP and confirm Apache is running
- Visit the ALB DNS name and confirm it displays the webpage
- Verify the Terraform state file is stored in S3
- Run `terraform destroy` when finished

---

# Completion

You now have a functioning AWS environment automated with Terraform, including a VPC, EC2 instance with SSH access, load balancer, and S3 remote backend.

---

