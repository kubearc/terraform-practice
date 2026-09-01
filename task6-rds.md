# Practice Task: Provisioning an RDS Database with Terraform

## Scenario

Databases need more careful networking than a plain EC2 instance — subnet groups, security groups scoped to your app tier only, and secrets that shouldn't be hardcoded in plaintext. This task has you provision a small MySQL RDS instance the way a real project would.

## Objectives

- Create a DB subnet group spanning at least two availability zones (RDS requires this even for a single-AZ instance)
- Create a security group that only allows inbound MySQL traffic (port 3306) from a specific app-tier security group, not from `0.0.0.0/0`
- Provision an `aws_db_instance` using the free-tier-eligible `db.t3.micro` class
- Handle the master password properly using a variable marked `sensitive`, not a hardcoded string
- Confirm connectivity from an EC2 instance in the same VPC

## Requirements

1. **Networking prerequisites**
   - Reuse or create a VPC with at least 2 subnets in different AZs (you can adapt this from an earlier VPC task).

2. **DB Subnet Group**
   - Create `aws_db_subnet_group` referencing both subnet IDs.

3. **Security Groups**
   - Create an `app_sg` security group (this represents your application tier — no inbound rules needed for this exercise beyond what you already use for SSH/HTTP).
   - Create a `db_sg` security group whose only ingress rule allows port 3306 **from `app_sg`'s security group ID** (not a CIDR block) — this is the key teaching point: security-group-to-security-group references instead of IP ranges.

4. **RDS Instance**
   - `engine = "mysql"`, a recent `engine_version`, `instance_class = "db.t3.micro"`, `allocated_storage = 20`.
   - `db_name`, `username` from variables.
   - `password` from a variable declared with `sensitive = true`, supplied via `TF_VAR_db_password` (never committed to `tfvars`).
   - `db_subnet_group_name` and `vpc_security_group_ids` wired to the resources above.
   - `skip_final_snapshot = true` (so `destroy` doesn't hang waiting for a snapshot name — fine for practice, **not** for production).

5. **Outputs**
   - Output the RDS endpoint. Mark it `sensitive = false` (endpoints aren't secret) but do **not** output the password.

6. **Verify**
   - From an EC2 instance in `app_sg`, install a MySQL client and confirm you can connect: `mysql -h <rds_endpoint> -u <username> -p`.

## Notes

- `terraform plan` will show the password as `(sensitive value)` in the CLI output once the variable is marked `sensitive` — a good moment to show students what accidentally leaks in a non-sensitive variable's plan output vs. a properly marked one.
- This is one of the few tasks where you genuinely should **not** put the value in `terraform.tfvars` even for practice — get in the habit now of using environment variables or a secrets manager for anything password-shaped.

---

<details>
<summary><b>Hint 1: Security-group-to-security-group reference</b></summary>

Instead of `cidr_blocks = ["10.0.0.0/16"]`, write `security_groups = [aws_security_group.app_sg.id]` inside the `db_sg`'s ingress block. This means "allow traffic from anything that has `app_sg` attached," and it stays correct even if your VPC's CIDR range changes later.

</details>

<details>
<summary><b>Hint 2: Supplying the sensitive password safely</b></summary>

```bash
export TF_VAR_db_password="SomeStrongPassword123!"
terraform apply
```
Never put this in `terraform.tfvars` if that file might get committed to Git.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**variables.tf**
```hcl
variable "db_username" {
  type    = string
  default = "admin"
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "db_name" {
  type    = string
  default = "practicedb"
}
```

**main.tf**
```hcl
resource "aws_db_subnet_group" "this" {
  name       = "practice-db-subnet-group"
  subnet_ids = [aws_subnet.public[0].id, aws_subnet.public[1].id]
}

resource "aws_security_group" "app_sg" {
  name   = "app-tier-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "db_sg" {
  name   = "db-tier-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_db_instance" "practice" {
  identifier             = "terraform-practice-db"
  engine                 = "mysql"
  engine_version         = "8.0"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  db_name                = var.db_name
  username               = var.db_username
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  skip_final_snapshot    = true
}
```

**outputs.tf**
```hcl
output "rds_endpoint" {
  value = aws_db_instance.practice.endpoint
}
```

**Verify:**
```bash
export TF_VAR_db_password="SomeStrongPassword123!"
terraform init
terraform apply
# from an EC2 instance attached to app_sg:
mysql -h <rds_endpoint> -u admin -p
terraform destroy
```

</details>
