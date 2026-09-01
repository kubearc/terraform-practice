# Practice Task: Using Git From Terraform (Module Sources & Provisioners)

## Scenario

Task 3 covered the `git` provider (`metio/git`), which manages a Git repo as a resource. But there are two other, more common ways Git shows up directly in a Terraform workflow: pulling **modules straight from a Git repository** instead of the local filesystem or registry, and running **git commands as part of `apply`** via a provisioner. This task covers both — plus why the second one comes with a warning label.

## Objectives

- Source a Terraform module directly from a public Git repository, pinned to a specific ref
- Understand the `git::` source syntax, subdirectory syntax, and `ref` query parameter
- Use `local-exec` to run a git command as part of a resource's lifecycle
- Understand why provisioners are considered "last resort" in Terraform, and what the safer alternatives are
- Combine this with the `git` provider from Task 3 to close the loop: generate a file with Terraform, then commit it

## Requirements

### Part 1 — Consume a module from a public Git repo

1. Create a fresh project folder. In `main.tf`, reference a public module directly from GitHub instead of the registry or a local path. Any small public module works — for example, HashiCorp's own example modules, or one you find on the [Terraform Registry](https://registry.terraform.io/) that lists a GitHub source.

   ```hcl
   module "example" {
     source = "git::https://github.com/hashicorp/terraform-guides.git//path/to/some-module?ref=main"
   }
   ```

2. Identify the three parts of that source string:
   - The protocol prefix (`git::`)
   - The repo URL
   - The subdirectory (after `//`) and the `ref` query parameter

3. Run `terraform init`. Confirm Terraform clones the repo into `.terraform/modules/`.

4. Change `ref=main` to a specific tagged release or commit SHA instead, and re-run `terraform init -upgrade`. **Question to answer:** why would pinning to a specific tag/SHA be safer for a real project than tracking a branch like `main`?

### Part 2 — Private repos and SSH-style sources

5. Without necessarily running it (a private repo you don't have access to is fine for this), write what the `source` line would look like for a **private** repo accessed over SSH instead of HTTPS:
   ```hcl
   module "private_example" {
     source = "git::[email protected]:your-org/your-private-repo.git?ref=v1.0.0"
   }
   ```
6. **Question to answer:** what has to be true about the machine running `terraform init` for this to work (hint: think about what `git clone` itself would need).

### Part 3 — Running git commands with local-exec

7. Create a `null_resource` that uses a `local-exec` provisioner to run a git command after another resource is created — for example, tagging a local repo with the ID of a resource you just made:
   ```hcl
   resource "random_pet" "release_name" {}

   resource "null_resource" "tag_release" {
     triggers = {
       release_name = random_pet.release_name.id
     }

     provisioner "local-exec" {
       command = "git -C ${path.module}/../git-provider-practice tag -f ${random_pet.release_name.id}"
     }
   }
   ```
8. Apply it, then check `git tag` inside the target repo to confirm the tag was created.
9. Change nothing and re-apply. **Question to answer:** does the provisioner re-run? Why does the `triggers` block matter here — what would happen if you removed it?

### Part 4 — Why this is a last resort (discussion + refactor)

10. List at least two problems with using `local-exec` to run git commands as part of `apply` (think about: what happens in CI/CD where there's no local git config, what happens on `terraform destroy`, whether Terraform tracks the git tag as a real "resource" it can reconcile).
11. Refactor Part 3 to use the `git` provider's `git_remote` or a tagging-capable resource (from Task 3) instead of `local-exec`, if the provider version you're using supports it. If it doesn't support tagging yet, explain in a sentence why a dedicated provider resource would still be preferable in principle even though `local-exec` "works."

## Notes

- `git::` sourcing works for modules regardless of whether the repo is on GitHub, GitLab, Bitbucket, or a self-hosted server — the prefix is what matters, not the host.
- HashiCorp's own docs list `local-exec`/`remote-exec` provisioners as a "last resort" specifically because Terraform can't model what they do — it can't diff, plan, or safely destroy the side effects of an arbitrary shell command the way it can a real resource.
- The `triggers` map is what makes a `null_resource` re-run its provisioner — without it, Terraform has no reason to think anything changed and won't re-execute the command on a later `apply`.

---

<details>
<summary><b>Hint 1: The double-slash in git:: sources</b></summary>

`git::https://github.com/org/repo.git//subdir/path?ref=v1.0.0` — everything after the first `//` following `.git` is treated as a subdirectory inside the cloned repo, not part of the clone URL itself. Forgetting this and only using a single `/` is a very common mistake.

</details>

<details>
<summary><b>Hint 2: SSH source requirements</b></summary>

The environment running `terraform init` needs a working SSH key already loaded (via `ssh-agent` or `~/.ssh/config`) with access to that repo — Terraform doesn't manage SSH credentials itself, it just shells out to `git`, which needs to already be authenticated the same way a manual `git clone` would.

</details>

<details>
<summary><b>Hint 3: Why triggers matter</b></summary>

Without `triggers`, a `null_resource`'s provisioner only runs once, at creation — Terraform has no attribute change to compare against on subsequent applies, so it just leaves the existing `null_resource` alone. Adding `triggers` gives Terraform something to diff, so a change to `random_pet.release_name.id` forces recreation of the `null_resource` and therefore reruns the provisioner.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**Part 1 — main.tf**
```hcl
module "example" {
  source = "git::https://github.com/hashicorp/terraform-guides.git//path/to/some-module?ref=v1.2.0"
}
```
```bash
terraform init
ls .terraform/modules/   # confirm the cloned module is there
```

**Part 2 — private repo over SSH (reference only)**
```hcl
module "private_example" {
  source = "git::[email protected]:your-org/your-private-repo.git?ref=v1.0.0"
}
```

**Part 3 — main.tf**
```hcl
resource "random_pet" "release_name" {}

resource "null_resource" "tag_release" {
  triggers = {
    release_name = random_pet.release_name.id
  }

  provisioner "local-exec" {
    command = "git -C ${path.module}/../git-provider-practice tag -f ${random_pet.release_name.id}"
  }
}
```
```bash
terraform apply
git -C ../git-provider-practice tag   # confirm the new tag exists
```


</details>
