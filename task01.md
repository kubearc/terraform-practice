# Practice Task: Launch an AWS EC2 Web Server with Terraform (Basics)

## Scenario

You've learned how to write a single `main.tf` file with hardcoded values to create AWS resources. Now practice that by provisioning a small EC2 instance that runs a web server, using only what you've covered so far — no variables, no outputs, just resource blocks with values written directly in.

## Objectives

- Write a single `main.tf` with the AWS provider configured directly (hardcoded region)
- Create a security group with hardcoded inbound rules
- Launch an EC2 instance with a hardcoded AMI ID and instance type
- Use `user_data` to install and start a web server
- Run the full `init` → `plan` → `apply` → `destroy` lifecycle

## Requirements

1. **Provider**
   - Configure the AWS provider directly with `region = "us-east-1"` (or a region of your choice — just write it in, don't parameterize).

2. **Networking**
   - Create a VPC with CIDR block `10.0.0.0/16`, hardcoded directly in the resource block.
   - Create a subnet inside that VPC with CIDR block `10.0.1.0/24`, in availability zone `us-east-1a`, with `map_public_ip_on_launch = true`.
   - Create an Internet Gateway and attach it to the VPC.
   - Create a route table for the VPC with a default route (`0.0.0.0/0`) pointing at the Internet Gateway, and associate it with the subnet.

3. **Security Group**
   - Create a security group named `web-sg`, attached to the VPC you created.
   - Allow inbound port 22 (SSH) from your IP, written directly as a CIDR block (e.g. `"49.36.XX.XX/32"`).
   - Allow inbound port 80 (HTTP) from `"0.0.0.0/0"`.
   - Allow all outbound traffic.

4. **EC2 Instance**
   - Launch a `t2.micro` instance inside the subnet you created.
   - Use a real Amazon Linux 2023 AMI ID for your region (look one up manually in the AWS Console — just paste the ID in as a string, don't use a `data` block yet).
   - Attach the security group you created.
   - Add a `user_data` script that installs `httpd`, starts it, and writes a simple HTML page saying "Hello from Terraform".
   - Give the instance a `Name` tag.

5. **Lifecycle**
   - Run `terraform init`, `terraform plan`, `terraform apply`.
   - Find the instance's public IP in the AWS Console (or `terraform show`) and confirm the web page loads in a browser.
   - Run `terraform destroy` and confirm everything is cleaned up.

## Notes

- Everything in this task goes into one file: `main.tf`. No `variables.tf`, no `outputs.tf` yet — that's for a later exercise.
- All values (region, AMI ID, instance type, IP, ports) should be typed directly into the resource blocks.

---

<details>
<summary><b>Hint 1: Finding an AMI ID manually</b></summary>

In the AWS Console, go to EC2 → Launch Instance → search "Amazon Linux 2023" and copy the AMI ID shown (it looks like `ami-0abcdef1234567890`). AMI IDs are region-specific, so make sure you're looking in the same region you hardcoded in the provider block.

</details>

<details>
<summary><b>Hint 2: Getting your IP for the SSH rule</b></summary>

Search "what is my ip" in a browser and add `/32` after it, e.g. `"103.25.14.9/32"`. Never use `"0.0.0.0/0"` for SSH.

</details>

<details>
<summary><b>Hint 3: Wiring the VPC, subnet, IGW, and route table together</b></summary>

Four resources, four connections: the subnet references the VPC's `id` via `vpc_id`, the Internet Gateway also references the VPC's `id`, the route table references the VPC's `id` and its `route` block points `gateway_id` at the Internet Gateway's `id`, and finally an `aws_route_table_association` ties the subnet to the route table. Without that last association, the subnet has no route to the internet even though the IGW exists.

</details>

<details>
<summary><b>Hint 4: user_data</b></summary>

The script must start with `#!/bin/bash`. Use `dnf install -y httpd` and `systemctl enable --now httpd` for Amazon Linux 2023. `user_data` only runs once, at first boot.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**main.tf**
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "practice-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "practice-public-subnet"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "practice-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "practice-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["103.25.14.9/32"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
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

resource "aws_instance" "web" {
  ami                    = "your ami id"
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              dnf install -y httpd
              systemctl enable --now httpd
              echo "Hello from Terraform" > /var/www/html/index.html
              EOF

  tags = {
    Name = "practice-web-instance"
  }
}
```

**Verify:**
```bash
terraform init
terraform plan
terraform apply
# copy the public IP from the AWS Console or `terraform show`
curl http://<PUBLIC_IP>
terraform destroy
```

Replace the AMI ID and the SSH CIDR block with your own values before running this.

</details>
