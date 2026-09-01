# terraform-practice

Hands-on Terraform practice tasks, ordered roughly by difficulty. Each task is self-contained with a scenario, objectives, requirements, hints (collapsed), and a full solution walkthrough (collapsed).

| Task | Topic | Cloud account needed? |
|---|---|---|
| [task0](task0.md) | Hardcoded AWS EC2 web server (basics) | AWS |
| [task01](task01.md) | Same, plus VPC/subnet/IGW/route table from scratch | AWS |
| [task1](task1.md) | AWS EC2 with variables (string/list/map/bool) | AWS |
| [task2](task2.md) | Providers & variables deep dive (local/random only) | No |
| [task3](task3-git-provider.md) | Managing a local Git repo with the `git` provider | No |
| [task4](task4-s3-iam.md) | S3 bucket + scoped IAM role/policy | AWS |
| [task5](task5-lambda.md) | Lambda function with `archive_file` + IAM execution role | AWS |
| [task6](task6-rds.md) | RDS MySQL with subnet groups, SG-to-SG rules, sensitive vars | AWS |
| [task7](task7-remote-state-backend.md) | S3 remote backend, DynamoDB locking, `terraform_remote_state` | AWS |
| [task8](task8-loops-dynamic-blocks.md) | `count` vs `for_each`, `dynamic` blocks, conditionals | AWS |
| [lab1](lab1.md) | Full lab: VPC + EC2 + ALB + S3 backend, modularized | AWS |

## Suggested order for a class

1. task0 / task01 — get comfortable with raw resource blocks first
2. task1 — introduce variables
3. task2 — providers & variables without needing AWS (good for take-home practice)
4. task3 — a different kind of provider entirely (git), reinforces that "provider" isn't just "cloud"
5. task4, task5, task6 — AWS services beyond EC2 (storage, serverless, databases)
6. task7 — remote state, once students have more than one stack to manage
7. task8 — loops and dynamic blocks, ideally revisited after task4–7 so students have real resources to apply it to
8. lab1 — capstone, combines everything into one modularized project
