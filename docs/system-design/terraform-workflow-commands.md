# Terraform Workflow Commands

> The four commands that drive the entire Terraform lifecycle: `init`, `plan`, `apply`, `destroy`.

## What is it?

Terraform's command-line interface (CLI) has many commands, but the core workflow is just four:

1. **`terraform init`** — initialize the project: download providers, configure backend
2. **`terraform plan`** — preview changes: show what will be created, changed, or destroyed
3. **`terraform apply`** — execute the plan: make API calls to create/update/destroy resources
4. **`terraform destroy`** — tear everything down: remove all resources managed by this config

Every Terraform session follows this cycle. You run `init` once (or after adding new providers), then `plan` → `apply` for every change, and `destroy` when you're done with the environment.

## Why does it matter?

These commands are the interface between your code (`.tf` files) and reality (cloud resources). Understanding each one — when to run it, what it does, and what can go wrong — is the difference between confident infrastructure management and "I ran a command and now production is down."

- **`plan` is your safety net.** It shows exactly what will change before anything actually changes. Never skip it.
- **`apply` is irreversible.** It creates, modifies, and destroys real resources that cost real money.
- **`destroy` is nuclear.** It removes everything. In production, it should be guarded behind manual approval.

## Mental model

Think of it as a **construction project**:
- `init` = set up the construction site (deliver materials, hire workers)
- `plan` = review the blueprints with the architect (what will we build?)
- `apply` = build it (pour concrete, erect walls)
- `destroy` = demolish everything (bulldoze the site)

You only set up the site once. You plan before every build. You build when ready. You demolish at the end.

## How does it actually work?

### `terraform init` — one-time setup

```bash
terraform init
```

What it does:
- Reads `versions.tf` to find required providers (e.g., `hashicorp/aws`)
- Downloads provider plugins to `.terraform/` directory
- Configures the backend (remote state on S3, Terraform Cloud, etc.)
- Creates `.terraform.lock.hcl` (provider version lock file)

When to run it:
- First time in a project
- After adding a new provider
- After changing backend configuration
- After a colleague adds new providers (pull their `.terraform.lock.hcl`)

```bash
# Typical output
Initializing the backend...
Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Installing hashicorp/aws v5.20.0...
- Installed hashicorp/aws v5.20.0

Terraform has been successfully initialized!
```

### `terraform plan` — preview changes

```bash
terraform plan
# or with variables:
terraform plan -var-file=prod.tfvars
```

What it does:
- Reads `.tf` files → builds desired state
- Reads `terraform.tfstate` → builds current state
- Calls cloud APIs to check for drift (reality vs. state)
- Computes the diff: what to create (+), change (~), or destroy (-)

```bash
# Output shows a color-coded diff:
Terraform will perform the following actions:

  # aws_s3_bucket.memory will be created
  + resource "aws_s3_bucket" "memory" {
      + bucket = "twin-dev-memory-123456"
      + id     = (known after apply)
    }

  # aws_lambda_function.api will be created
  + resource "aws_lambda_function" "api" {
      + function_name = "twin-dev-api"
      + runtime       = "python3.11"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

**Always read the plan.** Especially the "to destroy" count.

### `terraform apply` — execute

```bash
terraform apply
# Terraform shows the plan and asks for confirmation:
#   Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
#   Enter a value: yes

# Or skip the prompt (for CI/CD):
terraform apply -auto-approve
```

What it does:
- Runs `plan` internally (unless you pass a saved plan file)
- Makes API calls to AWS/GCP/Azure to create/update/destroy resources
- Updates `terraform.tfstate` to reflect the new reality
- Outputs any `output` blocks (URLs, ARNs)

### `terraform destroy` — tear down

```bash
terraform destroy
# Shows what will be destroyed, asks for confirmation

# For a specific workspace only:
terraform workspace select dev
terraform destroy  # Only destroys dev resources
```

What it does:
- Reads state to find all managed resources
- Destroys them in reverse dependency order (Lambda before IAM role, etc.)
- Updates state to empty

**This is permanent.** Resources are deleted. Data in S3 buckets is lost (unless backed up). Use with extreme caution in production.

## Common misconceptions

- **Misconception:** `apply` automatically runs `plan` first.
  **Reality:** `apply` does run an internal plan, but it's not shown in detail unless you pass `-auto-approve=false` (default). For safety, always run `plan` separately first, review, then `apply`.

- **Misconception:** `destroy` only removes resources you explicitly list.
  **Reality:** `destroy` removes *everything* in the state file. If your state accidentally includes production resources, `destroy` will delete them. Always check `terraform workspace show` and review the plan before destroying.

- **Misconception:** `init` re-runs on every command.
  **Reality:** `init` is only needed once (or after provider/backend changes). `plan`, `apply`, and `destroy` work without re-running `init`.

## What abstractions / AI tools often hide

When AI generates Terraform commands, it hides:
- **The `-auto-approve` flag danger** — in CI/CD, `terraform apply -auto-approve` skips the confirmation prompt. If the plan is wrong, there's no human in the loop. AI tools generate this for "automation" without mentioning the risk.
- **`terraform plan -out=tfplan`** — you can save a plan to a file and pass it to `apply`. This ensures the exact plan you reviewed is what gets applied (nothing changes between plan and apply). AI tools rarely use this pattern.
- **`terraform fmt` and `terraform validate`** — formatting and syntax validation commands that should run before every commit. AI tools generate `.tf` code without running these checks.
- **`terraform taint` / `terraform untaint`** — marking a resource for recreation on next apply. Useful for forcing a redeploy without changing code. AI tools don't mention this operational tool.
- **The refresh step** — before planning, Terraform queries cloud APIs to check for drift (manual changes). This can be slow. `terraform plan -refresh=false` skips it (faster but less safe).

## Practical engineering implications

- **Always `plan` before `apply`.** Review the output, especially the "to destroy" count. If you see unexpected destroys, stop.
- **Use `-out` for safety in CI/CD.** `terraform plan -out=tfplan && terraform apply tfplan` ensures the reviewed plan is what gets applied.
- **Run `terraform fmt` and `terraform validate` in pre-commit hooks.** Catches formatting and syntax errors before they reach CI.
- **Guard `destroy` behind manual approval.** In production, never use `terraform destroy -auto-approve`. Require a human to type "yes."
- **Check `terraform workspace show` before any command.** Ensure you're in the right workspace (dev vs. prod) before applying.
- **Run `terraform init` after pulling new code.** If a colleague added a provider, you need to `init` to download it.

## Related topics

- [Terraform Fundamentals](terraform-fundamentals.md) — what these commands operate on
- [Terraform State Management](terraform-state-management.md) — what `apply` and `destroy` modify
- [Terraform Workspaces](terraform-workspaces.md) — switching environments
- [GitHub Actions CI/CD](../software-engineering/github-actions-cicd.md) — automating the Terraform workflow

## References

- [Terraform CLI Commands](https://developer.hashicorp.com/terraform/cli/commands)
- [Terraform: plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [Terraform: apply](https://developer.hashicorp.com/terraform/cli/commands/apply)
- [Terraform: destroy](https://developer.hashicorp.com/terraform/cli/commands/destroy)
