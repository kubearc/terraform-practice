# Practice Task: Managing a Local Git Repository with Terraform (git Provider)

## Scenario

So far you've used Terraform to manage cloud infrastructure and local files. This task introduces the **`git` provider** (`metio/git`), which lets Terraform inspect and manage an actual Git repository on your own machine — remotes, commit history, and branches — as if it were just another kind of infrastructure.

This provider works against a **local** `.git` repository (a path on disk), not directly against GitHub/GitLab servers — that's what the `github` provider from an earlier task is for. Think of `git` (this task) as "manage the repo on disk" and `github` (earlier task) as "manage the repo on the platform."

> **Note:** This provider is community-maintained (not by HashiCorp), and its exact resources/attributes can change between versions. Always check `registry.terraform.io/providers/metio/git/latest/docs` for the exact schema of the version you install — don't blindly trust a version pin from an old tutorial.

## Objectives

- Configure the `git` provider alongside `local`
- Create a real Git repository manually and commit something to it
- Use Terraform **data sources** to read that repository's commit log and latest commit
- Use a Terraform **resource** to manage a Git remote
- Feed values read from Git into Terraform outputs

## Requirements

1. **Set up a real repo first (outside Terraform)**

   Terraform's `git` provider reads/manages an *existing* repository — it doesn't replace `git init` and your first commit. Before writing any `.tf` files:

   ```bash
   mkdir git-provider-practice && cd git-provider-practice
   git init
   echo "# Git Provider Practice" > README.md
   git add README.md
   git commit -m "Initial commit"
   ```

2. **Provider setup**

   In `providers.tf`, declare the `git` provider from `metio/git`. Since this is a smaller community provider, don't hardcode a specific patch version from memory — look up the current latest version on the Terraform Registry and pin using a `~>` constraint on the minor version.

   ```hcl
   terraform {
     required_providers {
       git = {
         source  = "metio/git"
         version = "~> 0.x"  # replace 0.x with whatever the registry shows as latest
       }
     }
   }

   provider "git" {}
   ```

   Run `terraform init` and confirm it downloads successfully.

3. **Read the commit log with a data source**

   Use the `git_log` data source to list commits in your repo:

   ```hcl
   data "git_log" "history" {
     directory = "${path.cwd}/.."   # adjust so this points at your git-provider-practice repo
   }

   output "commit_shas" {
     value = data.git_log.history.commits
   }
   ```

   Run `terraform plan` and confirm your one commit's SHA shows up in the output.

4. **Read details of the latest commit**

   Use the `git_commit` data source to fetch details about `HEAD`:

   ```hcl
   data "git_commit" "latest" {
     directory = "${path.cwd}/.."
     revision  = "HEAD"
   }

   output "latest_commit_message" {
     value = data.git_commit.latest.message
   }

   output "latest_commit_author" {
     value = data.git_commit.latest.author
   }
   ```

   Apply and confirm the output matches your `"Initial commit"` message.

5. **Manage a remote with a resource**

   Add a fake or real remote URL to your repo using the `git_remote` resource (no need to actually push anywhere — a fake URL like `https://github.com/your-username/git-provider-practice.git` works fine for this exercise):

   ```hcl
   resource "git_remote" "origin" {
     directory = "${path.cwd}/.."
     name      = "origin"
     urls      = ["https://github.com/your-username/git-provider-practice.git"]
   }
   ```

   Apply, then run `git remote -v` inside `git-provider-practice/` (outside Terraform) to confirm Terraform actually wrote the remote into your real Git config.

6. **Second commit, re-read the log**

   Outside Terraform, make a second commit:

   ```bash
   echo "more content" >> README.md
   git add README.md
   git commit -m "Second commit"
   ```

   Re-run `terraform plan`. **Question to answer:** Does Terraform show any changes for your `data` blocks even though you didn't touch your `.tf` files? Why does that make sense given what data sources are for?

## Notes

- Keep your `.tf` files in a *separate* folder from the repository they're managing (as shown above) to avoid Terraform accidentally treating its own `.terraform/` state as part of the Git history you're inspecting.
- This provider is a good example of Terraform's scope going beyond "cloud resources" — anything with a plugin can become something Terraform can read or manage, including your own dev tooling.

---

<details>
<summary><b>Hint 1: Getting the directory path right</b></summary>

`directory` must point at the folder that actually contains the `.git` subfolder — not a subfolder inside the repo. If you get a "not a git repository" error, double check you're pointing one level in the right direction relative to where your `.tf` files live.

</details>

<details>
<summary><b>Hint 2: git_remote vs. actually pushing</b></summary>

`git_remote` just writes an entry into `.git/config` — it doesn't validate that the URL is reachable or push any code. That's why a fake URL is fine for this exercise.

</details>

<details>
<summary><b>Hint 3: Why data sources "change" without a code change</b></summary>

Data sources are re-read on every `plan`/`apply` — they reflect the *current* state of whatever they point at, not a snapshot from when you wrote the `.tf` file. A new commit changes what `git_log` and `git_commit` return, even though your Terraform code is untouched. This is the same reason a `data "aws_ami"` block can return a different AMI ID over time as new images are published.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**Folder layout:**
```
git-provider-practice/        ← the actual Git repo (git init here)
└── README.md

tf-git-provider/               ← your Terraform config, separate folder
├── providers.tf
└── main.tf
```

**providers.tf**
```hcl
terraform {
  required_providers {
    git = {
      source  = "metio/git"
      version = "~> 0.1"   # check registry.terraform.io/providers/metio/git for the real latest
    }
  }
}

provider "git" {}
```

**main.tf**
```hcl
locals {
  repo_dir = "${path.cwd}/../git-provider-practice"
}

data "git_log" "history" {
  directory = local.repo_dir
}

data "git_commit" "latest" {
  directory = local.repo_dir
  revision  = "HEAD"
}

resource "git_remote" "origin" {
  directory = local.repo_dir
  name      = "origin"
  urls      = ["https://github.com/your-username/git-provider-practice.git"]
}

output "commit_shas" {
  value = data.git_log.history.commits
}

output "latest_commit_message" {
  value = data.git_commit.latest.message
}

output "latest_commit_author" {
  value = data.git_commit.latest.author
}
```

**Verify:**
```bash
cd git-provider-practice && git init && echo "# demo" > README.md && git add . && git commit -m "Initial commit"
cd ../tf-git-provider
terraform init
terraform apply
cd ../git-provider-practice && git remote -v   # should show origin now
```

</details>
