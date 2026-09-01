# Practice Task: Remote State with S3 Backend, DynamoDB Locking, and Cross-Stack Data

## Scenario

So far every task has used local state (`terraform.tfstate` sitting in your project folder). That breaks down the moment more than one person touches the same infrastructure. This task has you set up a real remote backend with state locking, then read one stack's outputs from a completely separate Terraform project.

## Objectives

- Provision the S3 bucket + DynamoDB table that Terraform itself will use to store and lock state (the classic "bootstrap" problem)
- Migrate a project from local to remote state
- Prove state locking actually works by triggering a lock conflict
- Use `terraform_remote_state` to read outputs from one stack inside a completely separate stack

## Requirements

### Part 1 — Bootstrap the backend infrastructure

1. In a folder called `backend-bootstrap/`, write a **local-state** configuration (yes, local — this one config has to exist before remote state can exist) that creates:
   - An `aws_s3_bucket` for state storage, with versioning enabled (`aws_s3_bucket_versioning`).
   - An `aws_dynamodb_table` with a string hash key named `LockID`, billing mode `PAY_PER_REQUEST`.
2. Apply it. This bucket and table now exist independently of any other Terraform project you write.

### Part 2 — Migrate a real stack to remote state

3. Take any earlier task's configuration (e.g. your S3+IAM task) and add a `backend "s3"` block referencing the bucket and table from Part 1:
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "your-bootstrap-bucket-name"
       key            = "s3-iam-task/terraform.tfstate"
       region         = "us-east-1"
       dynamodb_table = "your-bootstrap-lock-table"
       encrypt        = true
     }
   }
   ```
4. Run `terraform init` — Terraform will detect existing local state and ask if you want to migrate it to the new backend. Say yes.
5. Confirm the state file now appears in the S3 bucket (check the AWS Console) and that your local `terraform.tfstate` is gone or empty.

### Part 3 — Prove locking works

6. Open two terminals in the same project folder. In terminal 1, run `terraform apply` and pause at the confirmation prompt (don't type `yes` yet). In terminal 2, run `terraform plan`.
7. **Observe and record:** what message does terminal 2 show while terminal 1 is holding the lock?

### Part 4 — Cross-stack data with terraform_remote_state

8. In a brand-new folder (`network-consumer/`), write a config that uses `data "terraform_remote_state"` to read the VPC ID and subnet IDs from an earlier VPC-creating stack's remote state (adjust the `key` to match wherever that stack's state lives).
9. Use the fetched VPC ID to create a single new security group in that VPC — proving you can build on top of another team's/stack's infrastructure without ever touching their `.tf` files.

## Notes

- The bootstrap bucket/table is usually created **once** per organization, by hand or via a one-time `apply`, and then never destroyed by the same automation that uses it — destroying your own state backend while depending on it is a classic self-inflicted outage.
- `terraform_remote_state` only exposes values that were explicitly defined in an `output` block in the source stack — if a value you need isn't there, you have to add an output to that stack first.

---

<details>
<summary><b>Hint 1: DynamoDB table schema for locking</b></summary>

```hcl
resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```
The attribute name **must** be exactly `LockID` — Terraform's S3 backend hardcodes this.

</details>

<details>
<summary><b>Hint 2: What the lock conflict message looks like</b></summary>

You should see something like `Error: Error acquiring the state lock` with a `Lock Info` block showing who holds it, when, and the operation in progress. This is Terraform's built-in protection against two people applying at the same time and corrupting state.

</details>

<details>
<summary><b>Hint 3: terraform_remote_state syntax</b></summary>

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "your-bootstrap-bucket-name"
    key    = "vpc-task/terraform.tfstate"
    region = "us-east-1"
  }
}
```
Reference values as `data.terraform_remote_state.network.outputs.vpc_id` — the `outputs` key is required.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**backend-bootstrap/main.tf**
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "tf_state" {
  bucket = "your-org-tf-state-bootstrap"
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Migrating an existing stack — add to top of main.tf:**
```hcl
terraform {
  backend "s3" {
    bucket         = "your-org-tf-state-bootstrap"
    key            = "s3-iam-task/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
```bash
terraform init
# Terraform asks: "Do you want to copy existing state to the new backend?" -> yes
```

**network-consumer/main.tf**
```hcl
provider "aws" {
  region = "us-east-1"
}

data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "your-org-tf-state-bootstrap"
    key    = "vpc-task/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_security_group" "consumer_sg" {
  name   = "consumer-sg"
  vpc_id = data.terraform_remote_state.network.outputs.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Verify:**
```bash
cd backend-bootstrap && terraform init && terraform apply
cd ../s3-iam-task && terraform init   # answer yes to state migration prompt
# Terminal A:
terraform apply    # pause at the yes/no prompt
# Terminal B, same folder:
terraform plan      # should show a state lock error
cd ../network-consumer && terraform init && terraform apply
```

</details>
