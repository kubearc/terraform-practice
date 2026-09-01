# Practice Task: Deploying an AWS Lambda Function with Terraform

## Scenario

Serverless functions are one of the most common things teams provision with Terraform. This task has you package a small Python function, give it a scoped IAM execution role, and deploy it as a Lambda function — entirely from Terraform, with zero manual zipping or console clicking.

## Objectives

- Use the `archive_file` data source to zip Lambda source code automatically as part of `plan`/`apply`
- Create a minimal IAM execution role for Lambda (trust policy + basic logging permissions)
- Deploy an `aws_lambda_function` resource
- Invoke it manually to confirm it works
- Practice passing environment variables into a Lambda function via Terraform

## Requirements

1. **Provider**
   - Add the `archive` provider (`hashicorp/archive`) alongside `aws` in `required_providers`.

2. **Function source code**
   - Create a folder `lambda_src/` containing a file `handler.py`:
     ```python
     import os

     def handler(event, context):
         name = os.environ.get("GREETING_NAME", "World")
         return {"statusCode": 200, "body": f"Hello, {name}! Deployed via Terraform."}
     ```

3. **Zip it with Terraform**
   - Use `data "archive_file"` to zip `lambda_src/` into `lambda.zip`, with `output_path` pointing at a local build folder.

4. **IAM role**
   - Create an `aws_iam_role` with an assume-role policy trusting `lambda.amazonaws.com`.
   - Attach the AWS-managed policy `AWSLambdaBasicExecutionRole` (by ARN) so the function can write CloudWatch logs.

5. **Lambda function**
   - Create `aws_lambda_function` using:
     - `filename` and `source_code_hash` from the `archive_file` data source (this is what tells Terraform to redeploy when your code changes)
     - `runtime = "python3.12"`
     - `handler = "handler.handler"`
     - An `environment { variables = { GREETING_NAME = var.greeting_name } }` block, where `greeting_name` is a variable with a default

6. **Verify**
   - Run `aws lambda invoke --function-name <name> --payload '{}' response.json` (or use the AWS Console's Test feature) and confirm the response body greets whatever name you set in the variable.
   - Change `greeting_name` in `terraform.tfvars`, re-apply, and re-invoke — confirm the response changes without you touching the zip manually.

## Notes

- `source_code_hash = data.archive_file.lambda_zip.output_base64sha256` is the key line that makes Terraform detect code changes and redeploy — forgetting it means Terraform won't notice you edited `handler.py`.
- This same `archive_file` → IAM role → `aws_lambda_function` pattern scales to real projects; the only thing that changes is the handler code and permissions scope.

---

<details>
<summary><b>Hint 1: archive_file output values</b></summary>

`data.archive_file.lambda_zip.output_path` is the local zip file path, and `output_base64sha256` is a hash Terraform uses to detect changes. You need both: one for `filename`, one for `source_code_hash`.

</details>

<details>
<summary><b>Hint 2: Handler naming</b></summary>

`handler = "handler.handler"` means: file `handler.py`, function named `handler` inside it. If you rename either, update this string to match — a mismatch causes a runtime "Unable to import module" error that Terraform won't catch at plan time.

</details>

<details>
<summary><b>Solution walkthrough</b></summary>

**lambda_src/handler.py**
```python
import os

def handler(event, context):
    name = os.environ.get("GREETING_NAME", "World")
    return {"statusCode": 200, "body": f"Hello, {name}! Deployed via Terraform."}
```

**providers.tf**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

**variables.tf**
```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "greeting_name" {
  type    = string
  default = "Class"
}
```

**main.tf**
```hcl
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/lambda_src"
  output_path = "${path.module}/build/lambda.zip"
}

data "aws_iam_policy_document" "lambda_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_exec" {
  name               = "lambda-practice-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json
}

resource "aws_iam_role_policy_attachment" "basic_logs" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "greeting" {
  function_name    = "terraform-practice-greeting"
  role             = aws_iam_role.lambda_exec.arn
  runtime          = "python3.12"
  handler          = "handler.handler"
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = {
      GREETING_NAME = var.greeting_name
    }
  }
}
```

**outputs.tf**
```hcl
output "function_name" {
  value = aws_lambda_function.greeting.function_name
}

output "function_arn" {
  value = aws_lambda_function.greeting.arn
}
```

**Verify:**
```bash
terraform init
terraform apply
aws lambda invoke --function-name terraform-practice-greeting --payload '{}' --cli-binary-format raw-in-base64-out response.json
cat response.json
terraform destroy
```

</details>
