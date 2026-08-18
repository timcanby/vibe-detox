# Terraform Workspaces

> One codebase, multiple environments — dev, test, and prod each get their own isolated state without duplicating configuration.

## What is it?

A **workspace** in Terraform is an isolated state container. The same `.tf` files can produce different infrastructure in different workspaces. Each workspace has its own `terraform.tfstate` — they don't interfere with each other.

You switch between workspaces (`terraform workspace select dev`), and Terraform operates on a completely different set of resources. Dev and prod can share the same code but have different buckets, different Lambda functions, and different variables — all from one codebase.

## Why does it matter?

Without workspaces, managing multiple environments means either:
1. **Duplicating the entire config** — one folder for dev, one for prod. Code drift is inevitable.
2. **Using the same state for everything** — dev and prod share resources. A `terraform destroy` in dev wipes production.

Workspaces solve this: one config, N isolated states. You write the infrastructure once, then apply it to dev, test, and prod as separate workspaces. Each workspace can use different variables (via `terraform.workspace` in your config).

## Mental model

Think of workspaces as **parallel universes**. The laws of physics (your `.tf` config) are the same, but the physical reality (state + resources) is different. In universe "dev", the S3 bucket is `twin-dev-memory`. In universe "prod", it's `twin-prod-memory`. Same code, different outcomes.

Switching workspaces = traveling between universes. You can only be in one at a time.

## How does it actually work?

```bash
# Create workspaces for each environment
terraform workspace new dev
terraform workspace new test
terraform workspace new prod

# Switch between them
terraform workspace select dev    # Now operating in dev
terraform workspace select prod   # Now operating in prod

# List all workspaces
terraform workspace list
```

### Using workspace name in config

```hcl
# variables.tf
variable "prefix" {
  type = string
}

# main.tf — resources are named based on current workspace
resource "aws_s3_bucket" "memory" {
  # In "dev" workspace: twin-dev-memory-123456
  # In "prod" workspace: twin-prod-memory-123456
  bucket = "twin-${terraform.workspace}-memory-${var.account_id}"
}

resource "aws_lambda_function" "api" {
  function_name = "twin-${terraform.workspace}-api"
  # ...
}

# Use workspace for environment-specific config
locals {
  environment_config = {
    dev  = { instance_type = "t3.micro",    min_capacity = 1 }
    prod = { instance_type = "t3.large",    min_capacity = 3 }
  }
  config = local.environment_config[terraform.workspace]
}
```

### The workflow

```bash
# Deploy to dev
terraform workspace select dev
terraform plan    # Shows changes for dev environment
terraform apply   # Creates dev resources

# Deploy to prod
terraform workspace select prod
terraform plan    # Shows changes for prod environment (different resources!)
terraform apply   # Creates prod resources

# Clean up dev (doesn't touch prod!)
terraform workspace select dev
terraform destroy # Only destroys dev resources
```

## Common misconceptions

- **Misconception:** Workspaces provide security isolation.
  **Reality:** Workspaces isolate *state*, not access. If your AWS credentials have access to both dev and prod resources, a mistake in the dev workspace can still affect prod resources if you reference them incorrectly. Use separate AWS accounts for true isolation.

- **Misconception:** You should always use workspaces for environments.
  **Reality:** For simple projects, workspaces are great. For complex projects with very different dev vs. prod configurations, separate directories (or separate repos) may be clearer. Workspaces shine when configs are *mostly* the same with minor differences.

- **Misconception:** `terraform.workspace` is a variable you set.
  **Reality:** It's a built-in attribute that reflects the current workspace. You don't set it — you switch to a workspace, and `terraform.workspace` reflects that.

## What abstractions / AI tools often hide

When AI generates Terraform code with workspaces, it hides:
- **The state file per workspace** — each workspace gets its own state file in the same backend. If using S3 backend, the state path includes the workspace name. AI tools don't explain this path structure.
- **The default workspace** — Terraform always has a `default` workspace. If you never create workspaces, you're using `default`. Forgetting this leads to accidentally applying prod changes to the default workspace.
- **Workspace limitations** — not all Terraform backends support workspaces. S3 does, but some legacy backends don't. AI tools may assume workspace support without checking.
- **The `terraform.workspace` interpolation** — using `terraform.workspace` in resource names is powerful but means your resource names are coupled to workspace names. Renaming a workspace is now a destructive operation.

## Practical engineering implications

- **Name resources with `terraform.workspace`.** `bucket = "twin-${terraform.workspace}-memory"` ensures each environment gets unique, non-colliding resource names.
- **Use separate AWS accounts for prod.** Workspaces isolate state, not access. For true production safety, use separate AWS accounts (or at minimum, separate IAM roles with scoped permissions).
- **Never `terraform destroy` in the prod workspace.** Workspaces make it easy to destroy — and easy to destroy the wrong one. Always verify `terraform workspace show` before any destructive command.
- **Use workspace-specific `.tfvars` files.** `terraform apply -var-file=prod.tfvars` combined with workspace selection gives you both environment-specific variables and isolated state.
- **Document which workspace is "active."** In a team, it's easy to forget which workspace you're in. Add `terraform workspace show` to your CI/CD pipeline to verify before applying.

## Related topics

- [Terraform Fundamentals](terraform-fundamentals.md) — the tool workspaces belong to
- [Terraform State Management](terraform-state-management.md) — what workspaces isolate
- [Monorepo vs Multi-Repo](monorepo-vs-multi-repo.md) — a similar trade-off at the repo level

## References

- [Terraform: Workspaces](https://developer.hashicorp.com/terraform/language/workspaces)
- [Terraform: Workspace Commands](https://developer.hashicorp.com/terraform/cli/commands/workspace)
- [When to use workspaces vs. directories](https://developer.hashicorp.com/terraform/language/state/workspaces#when-not-to-use-workspaces)
