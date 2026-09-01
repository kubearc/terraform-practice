# Practice Task: Loops, for_each, and Dynamic Blocks

## Scenario

Real infrastructure rarely has exactly one of anything — you need N subnets, N security group rules, N IAM users. This task moves you from `count` (which you've used before) to `for_each` and `dynamic` blocks, which handle these cases more safely and readably.

## Objectives

- Understand when `for_each` is a better fit than `count`
- Use `for_each` with a `map` to create multiple named resources
- Use a `dynamic` block to generate a variable number of nested blocks (e.g. security group rules) from a single variable
- Use conditional expressions (`? :`) to toggle resource behavior based on a variable
- Compare the diff behavior of `count` vs `for_each` when an item is removed from the middle of a list

## Requirements

### Part 1 — count's hidden problem

1. Create three S3 buckets using `count` and a list variable:
   ```hcl
   variable "bucket_names" {
     type    = list(string)
     default = ["logs", "backups", "uploads"]
   }

   resource "aws_s3_bucket" "count_demo" {
     count  = length(var.bucket_names)
     bucket = "tf-practice-${var.bucket_names[count.index]}-${random_id.suffix.hex}"
   }
   ```
2. Apply it. Then **remove the middle item** (`"backups"`) from the list and run `terraform plan`.
3. **Observe and record:** does Terraform only destroy the removed bucket, or does it also want to recreate/rename the ones after it? Explain why, based on how `count.index` works.

### Part 2 — for_each fixes it

4. Rewrite the same three buckets using `for_each` with a `set(string)` (or a `toset()` of the list) instead of `count`.
5. Repeat the same experiment: remove `"backups"` and run `terraform plan` again.
6. **Observe and record:** how is the plan different from Part 1? Why is `for_each` generally the safer choice when items might be added/removed from the middle of a collection?

### Part 3 — for_each with a map (named resources)

7. Create three IAM users using `for_each` over a **map**, where the map's keys are usernames and the values are a tag to apply:
   ```hcl
   variable "iam_users" {
     type = map(string)
     default = {
       alice = "engineering"
       bob   = "design"
       carol = "engineering"
     }
   }

   resource "aws_iam_user" "team" {
     for_each = var.iam_users
     name     = each.key
     tags = {
       Department = each.value
     }
   }
   ```
8. Apply, then check the AWS Console to confirm each user has the correct `Department` tag.

### Part 4 — dynamic blocks for variable-length nested config

9. Define a variable holding a list of ingress rules:
   ```hcl
   variable "ingress_rules" {
     type = list(object({
       port        = number
       protocol    = string
       cidr_blocks = list(string)
     }))
     default = [
       { port = 22, protocol = "tcp", cidr_blocks = ["YOUR_IP/32"] },
       { port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
       { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
     ]
   }
   ```
10. Use a `dynamic "ingress"` block inside a single `aws_security_group` resource to generate one `ingress {}` block per entry in `ingress_rules`, without writing three separate hardcoded blocks.
11. Add a fourth rule to the list (e.g. port 8080) and re-apply — confirm the security group picks it up with no changes to the resource block itself.

### Part 5 — Conditional expressions

12. Add a variable `enable_versioning` (bool). Use a ternary expression inside `aws_s3_bucket_versioning` so that `status = var.enable_versioning ? "Enabled" : "Suspended"`.
13. Toggle the variable and confirm the plan shows the versioning status flipping accordingly.

## Notes

- Rule of thumb taught here: use `count` only for near-identical resources where you'll never need to remove one from the middle; use `for_each` for almost everything else, especially anything with meaningful names (users, buckets, per-environment resources).
- `dynamic` blocks are for repeating a *nested block* (like `ingress {}` inside a security group), not top-level resources — top-level repetition is what `for_each`/`count` on the resource itself are for.

---

<details>
<summary><b>Hint 1: Why count shifts everything</b></summary>

`count.index` is purely positional. If you delete index 1 from a 3-item list, item 2 shifts down to become the new index 1, and Terraform sees that as "index 1 changed" — it may destroy and recreate it even though conceptually nothing about that resource changed.

</details>

<details>
<summary><b>Hint 2: for_each with a set built from a list</b></summary>

```hcl
resource "aws_s3_bucket" "foreach_demo" {
  for_each = toset(var.bucket_names)
  bucket   = "tf-practice-${each.value}-${random_id.suffix.hex}"
}
```
Note `each.value` (and `each.key`, identical for a set) replaces `count.index` + list indexing.

</details>

<details>
<summary><b>Hint 3: dynamic block syntax</b></summary>

```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.port
    to_port     = ingress.value.port
    protocol    = ingress.value.protocol
    cidr_blocks = ingress.value.cidr_blocks
  }
}
```
The block label (`ingress`) inside `dynamic "ingress"` becomes the loop variable name you reference as `ingress.value` — this trips people up because it looks like it should be `each.value`, but `dynamic` blocks use the block's own label instead.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**variables.tf**
```hcl
variable "bucket_names" {
  type    = list(string)
  default = ["logs", "backups", "uploads"]
}

variable "iam_users" {
  type = map(string)
  default = {
    alice = "engineering"
    bob   = "design"
    carol = "engineering"
  }
}

variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { port = 22, protocol = "tcp", cidr_blocks = ["203.0.113.5/32"] },
    { port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
  ]
}

variable "enable_versioning" {
  type    = bool
  default = true
}
```

**main.tf**
```hcl
resource "random_id" "suffix" {
  byte_length = 4
}

# Part 2 - for_each instead of count
resource "aws_s3_bucket" "foreach_demo" {
  for_each = toset(var.bucket_names)
  bucket   = "tf-practice-${each.value}-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "foreach_demo" {
  for_each = aws_s3_bucket.foreach_demo
  bucket   = each.value.id
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

# Part 3 - for_each over a map
resource "aws_iam_user" "team" {
  for_each = var.iam_users
  name     = each.key
  tags = {
    Department = each.value
  }
}

# Part 4 - dynamic block
resource "aws_security_group" "dynamic_demo" {
  name = "dynamic-rules-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
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
}
```

**Verify:**
```bash
terraform init
terraform apply
# Part 1/2 experiment: remove "backups" from bucket_names, then:
terraform plan
# compare behavior between a count-based version and this for_each version
terraform destroy
```

</details>
