#  AWS Terraform Practice — 

---

##  Overview

This project helps you practice Terraform by deploying an AWS EC2 instance using various input types, including:

- Standard variables (string)
- List variables
- Map variables
- Boolean variables

---

## Project Structure
```
aws-terraform-practice/
│
├── main.tf
├── variables.tf
├── terraform.tfvars
└── outputs.tf
```

---

## Prerequisites

| Requirement | Version |
|------------|---------|
| Terraform  | v1.3+   |
| AWS CLI    | Configured IAM credentials |

---

##  Step 1 — variables.tf

```hcl
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
}

variable "aws_accsess_key" {
  description = "AWS Access Key"
    type        = string    
}
variable "aws_secret_key" {
  description = "AWS Secret Key"
    type        = string
  
}
variable "ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "key_name" {
  description = "AWS SSH keypair name"
  type        = string
}

variable "allowed_ssh_cidr" {
  description = "List of CIDRs allowed to SSH"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "instance_tags" {
  description = "Tags to apply on EC2 instance"
  type        = map(string)
  default = {
    Name        = "Terraform-Learning-Instance"
    Environment = "Dev"
    Owner       = "Student"
  }
}

variable "associate_public_ip" {
  description = "Assign Public IP to instance?"
  type        = bool
  default     = true
}
```

# Step 2:-  main.tf
```
provider "aws" {
  region = var.aws_region
access_key = var.aws_accsess_key
  secret_key = var.aws_secret_key
}

resource "aws_security_group" "practice_sg" {
  name        = "practice-security-group"
  description = "Security group created for Terraform practice"

  dynamic "ingress" {
    for_each = var.allowed_ssh_cidr
    content {
      description = "Allow SSH"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "practice_ec2" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.practice_sg.id]

  associate_public_ip_address = var.associate_public_ip

  tags = var.instance_tags
}
```

# Step 3 — terraform.tfvars
```
aws_region      = "ap-south-1"
ami_id          = "ami-0e306788ff2473ccb"
instance_type   = "t2.micro"
key_name        = "~/.ssh/*.pub"
aws_accsess_key = "value_here"
aws_secret_key  = "value_here"

allowed_ssh_cidr = [
  "0.0.0.0/0",
  "192.168.1.0/24"
]

instance_tags = {
  Name        = "Terraform-EC2-Demo"
  Owner       = "Manvir"
  Environment = "Training"
}

associate_public_ip = true
```


# Step 4 — outputs.tf
```
output "public_ip" {
  description = "Public IP of instance (if enabled)"
  value       = aws_instance.practice_ec2.public_ip
}

output "instance_id" {
  description = "EC2 Instance ID"
  value       = aws_instance.practice_ec2.id
}

output "security_group_id" {
  description = "Security Group ID"
  value       = aws_security_group.practice_sg.id
}
```
## Terraform Commands

```
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply -auto-approve
```
Destroy:
```
terraform destroy -auto-approve
```
