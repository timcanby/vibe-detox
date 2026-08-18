# Terraform Fundamentals

> The most popular Infrastructure as Code tool — write declarative `.tf` files, and Terraform creates, updates, and destroys cloud resources for you.

## What is it?

**Terraform** is an open-source IaC tool created by Mitchell Hashimoto and HashiCorp. You write configuration files in **HCL** (HashiCorp Configuration Language) describing the infrastructure you want — S3 buckets, Lambda functions, API gateways, databases. Terraform reads these files, figures out what already exists (via state), computes the diff, and makes API calls to your cloud provider to reach the desired state.

It is **cloud-agnostic**: the same tool works with AWS, GCP, Azure, and hundreds of other providers. Learn it once, use it everywhere.

## Why does it matter?

Terraform is the industry standard for IaC. Knowing it is:
- **A career skill** — "Terraform" on your resume is recognized everywhere. It's the most-requested IaC tool in job postings.
- **Cloud-agnostic** — unlike AWS CDK (AWS only) or CloudFormation (AWS only), Terraform works across all major clouds. One skill, many providers.
- **Declarative** — you describe *what* you want, not *how* to build it. Terraform figures out the API calls, ordering, and dependencies.
- **Reproducible** — the same `.tf` files create identical environments. Dev, test, prod — same code, different variables.

## Mental model

Think of Terraform as a **smart contractor**. You hand the contractor blueprints (`.tf` files). The contractor:
1. Checks what's already built (reads state file)
2. Compares blueprints to reality (plans the diff)
3. Builds/modifies/destroys to match the blueprints (applies)
4. Records what was built (updates state file)

You never tell the contractor *which nails to hammer first* — the contractor figures out the order (dependency resolution).

## How does it actually work?

### File structure

```
terraform/
├── versions.tf      # Provider versions (AWS, required Terraform version)
├── variables.tf     # Input variables (model ID, timeout, environment name)
├── main.tf          # Resources (S3, Lambda, API Gateway, CloudFront)
├── outputs.tf       # Outputs (URLs, ARNs to display after apply)
├── terraform.tfvars # Default variable values
└── prod.tfvars      # Production-specific values
```

Terraform merges all `.tf` files into one logical configuration. You can split into many files or use one big file — Terraform doesn't care. Convention is to split by concern.

### versions.tf — setting up providers

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### variables.tf — input parameters

```hcl
variable "bedrock_model_id" {
  type    = string
  default = "anthropic.claude-3-haiku-20240307-v1:0"
}

variable "lambda_timeout" {
  type    = number
  default = 60
}

variable "environment" {
  type    = string
  default = "dev"
}
```

### main.tf — resources (the core)

```hcl
# An S3 bucket for memory storage
resource "aws_s3_bucket" "memory" {
  bucket = "${var.prefix}-memory-${var.account_id}"
}

# A Lambda function
resource "aws_lambda_function" "api" {
  filename         = "lambda_package.zip"
  function_name    = "twin-${var.environment}-api"
  role             = aws_iam_role.lambda_role.arn
  handler          = "handler.handler"
  runtime          = "python3.11"
  timeout          = var.lambda_timeout
  source_code_hash = filebase64sha256("lambda_package.zip")
}
```

## Example: the three core terms

| Term | What it is | Example |
|---|---|---|
| **Provider** | The cloud vendor plugin | `provider "aws"` — tells Terraform to talk to AWS |
| **Variable** | An input parameter | `variable "bedrock_model_id"` — configurable model ID |
| **Resource** | An infrastructure building block | `resource "aws_s3_bucket" "memory"` — an S3 bucket |

These three are the essentials. If you remember nothing else, remember: **provider + variables + resources = your infrastructure**.

## Common misconceptions

- **Misconception:** Terraform is a programming language.
  **Reality:** HCL is declarative configuration, not a programming language. No loops (well, limited `for_each`), no conditionals in the traditional sense. You describe state, not logic.

- **Misconception:** You need one `.tf` file per resource.
  **Reality:** Terraform merges all `.tf` files in a directory. You can have one giant file or many small ones. Convention is to split by concern (versions, variables, main, outputs).

- **Misconception:** Terraform is only for cloud infrastructure.
  **Reality:** Terraform has providers for DNS (Cloudflare), databases (MySQL, PostgreSQL), monitoring (Datadog), SaaS (GitHub, Slack), and more. Anything with an API can have a Terraform provider.

## What abstractions / AI tools often hide

When AI generates Terraform code, it hides:
- **HCL vs JSON** — Terraform supports both HCL and JSON. AI tools may generate HCL (human-friendly) but the actual state and plan files are JSON. Understanding both helps debugging.
- **The `terraform.lock.hcl` file** — Terraform generates a lock file for provider versions. This *should* be committed (unlike state files). AI tools rarely explain this distinction.
- **Resource naming conventions** — `aws_s3_bucket.memory` (the logical name in Terraform) vs `bucket = "my-bucket"` (the physical name in AWS). These are different things. Confusing them causes state mismatches.
- **Implicit dependencies** — `aws_lambda_function.api` references `aws_iam_role.lambda_role.arn`. Terraform auto-detects this dependency and creates the role first. Explicit `depends_on` is rarely needed but AI tools add it unnecessarily.

## Practical engineering implications

- **Split files by concern.** `versions.tf`, `variables.tf`, `main.tf`, `outputs.tf`. Don't put everything in one 500-line file.
- **Use variables for everything environment-specific.** Bucket names, model IDs, timeouts — all should be variables with defaults. Hardcoding values makes the config non-reusable.
- **Name resources consistently.** `aws_s3_bucket.memory`, `aws_lambda_function.api`, `aws_cloudfront_distribution.frontend`. The logical name is documentation.
- **Read the plan before applying.** `terraform plan` shows exactly what will change. Always review it, especially in production.
- **Commit `.tf` files, not `.tfstate`.** Code goes in git; state goes in remote storage (S3, Terraform Cloud).

## Related topics

- [Infrastructure as Code](infrastructure-as-code.md) — the concept Terraform implements
- [Terraform State Management](terraform-state-management.md) — the state file
- [Terraform Workspaces](terraform-workspaces.md) — environment isolation

## References

- [Terraform Documentation](https://developer.hashicorp.com/terraform)
- [Terraform: Getting Started (AWS)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)
- [HCL Syntax Guide](https://developer.hashicorp.com/terraform/language)
- [Terraform Registry (providers)](https://registry.terraform.io/)
