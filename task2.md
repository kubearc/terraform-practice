# Terraform Practice: Providers & Variables

**No cloud account needed** — these exercises use the `random` and `local` providers, so every student can run them on their own laptop.

---

## Setup (do this first)

Create a new folder and one file:

```bash
mkdir tf-practice && cd tf-practice
touch providers.tf variables.tf terraform.tfvars main.tf
```

---

## Task 1 — Write your first providers.tf

In `providers.tf`, declare the `random` provider (from `hashicorp/random`) and the `local` provider (from `hashicorp/local`).

**Requirements:**
- Use a `required_providers` block inside `terraform {}`
- Pin `random` to version `~> 3.6`
- Pin `local` to version `~> 2.5`

Then run:
```bash
terraform init
```

**Check yourself:** You should see a `.terraform/` folder and a `.terraform.lock.hcl` file appear. Open the lock file and find where the exact provider version got recorded.

---

## Task 2 — Declare your first variables

In `variables.tf`, declare these variables:

| Name | Type | Default | Description |
|---|---|---|---|
| `project_name` | `string` | `"class-demo"` | Name used to tag generated resources |
| `word_length` | `number` | `8` | Length of a random string to generate |
| `use_special_chars` | `bool` | `false` | Whether the random string includes symbols |

Then in `main.tf`, create a `random_string` resource that uses all three variables:

```hcl
resource "random_string" "demo" {
  length  = var.word_length
  special = var.use_special_chars
}
```

Run `terraform plan` and confirm it works with just the defaults (don't apply yet).

---

## Task 3 — Remove a default, watch what happens

Delete the `default` from `word_length` in `variables.tf`. Run `terraform plan` again.

**Question to answer:** What did Terraform do differently? Why is relying on interactive prompts a bad idea for real projects?

---

## Task 4 — Use a `.tfvars` file

In `terraform.tfvars`, set:
```hcl
project_name = "student-lab"
word_length  = 12
```

Run `terraform plan` again — no prompt this time. Explain in one sentence *why* Terraform picked this up automatically without you passing any flag.

---

## Task 5 — Override from the command line

Without editing any files, override `word_length` to `20` using a CLI flag:

```bash
terraform plan -var="word_length=20"
```

**Question to answer:** If both `terraform.tfvars` and the CLI flag set `word_length`, which one wins? (Hint: think about precedence order — try it and see.)

---

## Task 6 — List and map variables

Add a new variable to `variables.tf`:

```hcl
variable "environments" {
  type    = list(string)
  default = ["dev", "staging", "prod"]
}
```

Then use `count` to create one `local_file` resource per environment, where each file:
- Is named `output-<environment>.txt` (e.g. `output-dev.txt`)
- Contains the text: `Environment: <environment>, Project: <project_name>`

Run `terraform apply` and confirm three files were created in your folder.

**Hint:** You'll need `count = length(var.environments)` and `var.environments[count.index]` inside the resource.

---

## Task 7 — Environment variable override

Destroy your resources (`terraform destroy`), then re-apply while overriding `project_name` using an environment variable instead of a flag or tfvars file:

```bash
export TF_VAR_project_name="env-var-test"
terraform apply
```

Check the content of one of the generated files to confirm the override worked.

---

## Challenge Task — Provider alias

Add a **second** instance of the `random` provider using an alias (e.g. `alias = "backup"`), and create a second `random_string` resource that explicitly uses that aliased provider via `provider = random.backup`.

**Question to answer:** Why might a real project need two configurations of the same provider? (Think about the AWS multi-region use case.)

---

