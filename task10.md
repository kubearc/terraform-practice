# Practice Task: Creating a GitHub Repository and Files with Terraform

## Scenario

Beyond just reading GitHub data, Terraform can fully manage a repository's *contents* — creating the repo itself, then writing files into it (README, config files, even source code) with specific content, all from a single `apply`. This task builds directly on the earlier GitHub provider task, this time focused entirely on repo creation and file management.

## Objectives

- Create a GitHub repository from scratch with `github_repository`
- Create files inside that repository with specific content using `github_repository_file`
- Update a file's content and watch Terraform diff and recommit it
- Create multiple files at once using `for_each`
- Understand what `overwrite_on_create` and `commit_message` actually control

## Requirements

Needs a free GitHub account and a Personal Access Token (classic, scope `repo`), set as an environment variable:
```bash
export GITHUB_TOKEN="ghp_xxxxxxxxxxxx"
```

### Part 1 — Create the repository

1. Configure the `github` provider (reuse from the earlier task if you still have it).
2. Create a repository using `github_repository`:
   - `name` from a variable (e.g. `"tf-file-practice-<yourname>"`)
   - `visibility = "public"`
   - `auto_init = true` (this matters — without it, the repo has no default branch yet, and file creation will fail because there's nothing to commit onto)

### Part 2 — Create a single file with content

3. Use `github_repository_file` to create a `README.md` in that repo:
   - `repository` = the repo's name (reference the resource, don't hardcode the string)
   - `file = "README.md"`
   - `content` = a short Markdown string written directly in the `.tf` file
   - `commit_message = "Add README via Terraform"`
   - `overwrite_on_create = true` (since `auto_init` already created a default README/branch structure — this tells Terraform it's fine to overwrite what's there rather than error out)
4. Apply, then check the repository on GitHub — confirm the commit shows up in the repo's commit history with your commit message.

### Part 3 — Update the file's content

5. Change the `content` argument to something different (add a new paragraph). Run `terraform plan` — confirm Terraform shows a content diff, not a destroy/recreate.
6. Apply, then check GitHub's commit history again — you should see a **second** commit, not the file being deleted and remade.

### Part 4 — Create multiple files from a map

7. Define a variable holding several files and their content:
   ```hcl
   variable "project_files" {
     type = map(string)
     default = {
       "docs/setup.md"    = "# Setup\n\nRun `terraform init` to get started."
       "docs/usage.md"    = "# Usage\n\nRun `terraform apply` to deploy."
       ".gitignore"        = "*.tfstate\n*.tfstate.backup\n.terraform/\n"
     }
   }
   ```
8. Use `for_each` over that map to create one `github_repository_file` resource per entry, where the map's key is the file path and the value is its content.
9. Apply, then confirm all three files exist in the repo with the correct content and that they were all created in a single commit-per-file pattern (check the commit history — how many commits did this add?).

### Part 5 — Read a file back with a data source

10. Use the `github_repository_file` **data source** (same resource type name, different block type) to read back the content of `README.md` after your Part 3 update, and output it.
11. **Question to answer:** why might reading a file back via `data` be useful before you overwrite it with a resource — think about a scenario where two people might be editing the same file.

## Notes

- `github_repository_file` manages one file per resource block — there's no "upload a whole folder" resource, which is exactly why `for_each` over a map is the natural pattern for multiple files.
- Every `apply` that changes a file's `content` creates a **real commit** in the repo's history, authored as whatever identity your `GITHUB_TOKEN` belongs to. This is genuinely useful for scaffolding template repos, but also means you shouldn't run `apply` in a tight loop while testing — you'll spam the commit history.
- `branch` defaults to the repo's default branch if not specified — set it explicitly if you want files created on something other than `main`.

---

<details>
<summary><b>Hint 1: Why auto_init + overwrite_on_create go together</b></summary>

`auto_init = true` makes GitHub create an initial commit (usually with its own README) so the repo has a default branch at all. Without a default branch, `github_repository_file` has nothing to commit onto and fails. `overwrite_on_create = true` on the file resource then tells Terraform "if a file already exists at this path (like GitHub's auto-generated README), overwrite it instead of erroring."

</details>

<details>
<summary><b>Hint 2: Multiline content strings</b></summary>

Use a heredoc for multi-line file content instead of `\n` escapes when it's more than a line or two:
```hcl
content = <<-EOT
  # Setup

  Run `terraform init` to get started.
EOT
```

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**variables.tf**
```hcl
variable "repo_name" {
  type    = string
  default = "tf-file-practice"
}

variable "project_files" {
  type = map(string)
  default = {
    "docs/setup.md" = "# Setup\n\nRun `terraform init` to get started.\n"
    "docs/usage.md" = "# Usage\n\nRun `terraform apply` to deploy.\n"
    ".gitignore"    = "*.tfstate\n*.tfstate.backup\n.terraform/\n"
  }
}
```

**main.tf**
```hcl
provider "github" {
  owner = "your-github-username"
}

resource "github_repository" "practice" {
  name        = var.repo_name
  description = "Created and managed via Terraform"
  visibility  = "public"
  auto_init   = true
}

resource "github_repository_file" "readme" {
  repository          = github_repository.practice.name
  file                = "README.md"
  content             = "# ${var.repo_name}\n\nThis repo's README is fully managed by Terraform.\n"
  commit_message      = "Add README via Terraform"
  overwrite_on_create = true
}

resource "github_repository_file" "project_files" {
  for_each            = var.project_files
  repository          = github_repository.practice.name
  file                = each.key
  content             = each.value
  commit_message      = "Add ${each.key} via Terraform"
  overwrite_on_create = true
}

data "github_repository_file" "readme_check" {
  repository = github_repository.practice.name
  file       = "README.md"

  depends_on = [github_repository_file.readme]
}
```

**outputs.tf**
```hcl
output "repo_url" {
  value = github_repository.practice.html_url
}

output "readme_content_on_github" {
  value = data.github_repository_file.readme_check.content
}
```

**Verify:**
```bash
export GITHUB_TOKEN="ghp_xxxxxxxxxxxx"
terraform init
terraform apply
# Check the repo on GitHub: README.md, docs/setup.md, docs/usage.md, .gitignore should all exist
# Change the README content in main.tf, re-apply, check commit history for a second commit
terraform destroy
```

</details>
