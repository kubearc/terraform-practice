# Practice Task: S3 Bucket with IAM Access Policy

## Scenario

Most real AWS projects need storage plus tightly scoped permissions, not root-level access. This task has you provision an S3 bucket with proper modern configuration (versioning, encryption, public access blocked) and an IAM role that can access *only* that bucket — nothing else.

## Objectives

- Create an S3 bucket with versioning and encryption enabled using the current (post-v4) AWS provider resource split
- Block all public access on the bucket explicitly
- Create an IAM role and a scoped policy using the `aws_iam_policy_document` data source
- Attach the policy to the role
- Practice using `jsonencode`-free policy authoring via the data source (safer than hand-written JSON strings)

## Requirements

1. **Provider**
   - Configure the AWS provider with a region of your choice via a variable (`aws_region`), not hardcoded.

2. **S3 Bucket**
   - Create `aws_s3_bucket` with a globally unique name (use a variable `bucket_name`, e.g. `"tf-practice-${var.bucket_suffix}"` where `bucket_suffix` could be `random_id`).
   - Enable versioning via a **separate** `aws_s3_bucket_versioning` resource (not an inline `versioning {}` block — that's deprecated).
   - Enable default encryption (SSE-S3) via `aws_s3_bucket_server_side_encryption_configuration`.
   - Block all public access via `aws_s3_bucket_public_access_block` (all four settings `true`).

3. **IAM Role + Policy**
   - Write an `aws_iam_policy_document` data source that allows `s3:GetObject`, `s3:PutObject`, and `s3:ListBucket` — scoped to only this bucket's ARN (and `/*` for object-level actions).
   - Create an `aws_iam_role` with an assume-role policy for the EC2 service principal (`ec2.amazonaws.com`).
   - Create an `aws_iam_policy` from the document, and an `aws_iam_role_policy_attachment` linking the two.

4. **Outputs**
   - Output the bucket's ARN, the role's ARN, and the policy's ARN.

5. **Verify**
   - After `apply`, check in the AWS Console that the bucket shows "Block all public access: On" and the IAM role's permissions tab shows only the scoped policy — nothing else.

## Notes

- Using `aws_iam_policy_document` instead of a raw heredoc JSON string catches typos and invalid actions at `plan` time, not at `apply` time.
- This is the same pattern (least-privilege role scoped to one resource) you'd use before attaching a role to any real EC2 instance, Lambda function, or ECS task.

---

<details>
<summary><b>Hint 1: Getting a unique bucket name</b></summary>

Use the `random` provider's `random_id` resource (`byte_length = 4`) and interpolate its `hex` output into the bucket name. This avoids "bucket already exists" errors when everyone in class runs the same code.

</details>

<details>
<summary><b>Hint 2: Referencing the bucket ARN inside the policy document</b></summary>

`aws_s3_bucket.this.arn` gives you the bucket-level ARN (for `s3:ListBucket`). For object-level actions you need `"${aws_s3_bucket.this.arn}/*"` — a common mistake is using the bucket ARN for `GetObject`/`PutObject`, which will fail at runtime even though `plan` won't catch it.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**variables.tf**
```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "bucket_prefix" {
  type    = string
  default = "tf-practice"
}
```

**main.tf**
```hcl
provider "aws" {
  region = var.aws_region
}

resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "this" {
  bucket = "${var.bucket_prefix}-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

data "aws_iam_policy_document" "bucket_access" {
  statement {
    sid       = "ListBucket"
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.this.arn]
  }

  statement {
    sid       = "ReadWriteObjects"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["${aws_s3_bucket.this.arn}/*"]
  }
}

data "aws_iam_policy_document" "assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "bucket_role" {
  name               = "s3-practice-role"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

resource "aws_iam_policy" "bucket_policy" {
  name   = "s3-practice-policy"
  policy = data.aws_iam_policy_document.bucket_access.json
}

resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.bucket_role.name
  policy_arn = aws_iam_policy.bucket_policy.arn
}
```

**outputs.tf**
```hcl
output "bucket_arn" {
  value = aws_s3_bucket.this.arn
}

output "role_arn" {
  value = aws_iam_role.bucket_role.arn
}

output "policy_arn" {
  value = aws_iam_policy.bucket_policy.arn
}
```

**Verify:**
```bash
terraform init
terraform apply
# Check AWS Console: S3 → bucket → Permissions tab → "Block all public access: On"
# Check IAM → Roles → s3-practice-role → Permissions → only s3-practice-policy attached
terraform destroy
```

</details>
