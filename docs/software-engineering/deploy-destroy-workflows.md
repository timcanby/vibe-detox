# Deploy & Destroy Workflows — GitHub Actions for Terraform

> Two YAML files that automate your entire infrastructure lifecycle: `deploy.yml` creates everything on push, `destroy.yml` tears it down on demand.

## What is it?

In a GitHub Actions-driven Terraform workflow, you define two workflow files in `.github/workflows/`:

- **`deploy.yml`** — triggered on push to `main`, runs `terraform init → plan → apply` to create or update infrastructure
- **`destroy.yml`** — triggered manually (workflow_dispatch), runs `terraform destroy` to tear down an environment

Together, these replace all manual `terraform` commands. You push code → infrastructure updates automatically. You click a button → environment is destroyed (for cost cleanup or teardown).

## Why does it matter?

Without automated workflows:
- You run `terraform apply` locally — but your local machine has different state, different credentials, and different variables than production
- Deploying requires a human to run commands — no audit trail, no review, no automation
- Destroying environments is forgotten — orphaned resources accrue AWS charges

With automated workflows:
- Every infrastructure change goes through git → PR → review → push → automatic deploy
- The same GitHub Actions VM runs Terraform every time — consistent state, credentials, and variables
- Destroy is a button click — clean up dev/test environments over the weekend to save money

## Mental model

Think of it as a **factory with two buttons**:
- **Green button (deploy):** push to main → factory builds your infrastructure automatically
- **Red button (destroy):** click in GitHub UI → factory demolishes everything

The factory always uses the same tools, the same blueprints (your `.tf` files), and the same shared workspace (remote state). No human touches the machinery.

## How does it actually work?

### deploy.yml — the green button

```yaml
# .github/workflows/deploy.yml
name: Deploy Digital Twin

on:
  push:
    branches: [main]  # Trigger on push to main

# OIDC permissions (no static AWS keys)
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # 1. Get the code
      - uses: actions/checkout@v4

      # 2. Configure AWS credentials via OIDC
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_DEFAULT_REGION }}

      # 3. Set up Python (for Lambda packaging)
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      # 4. Set up Terraform
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.13"

      # 5. Set up Node.js (for frontend build)
      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      # 6. Run the deploy script
      - name: Deploy
        env:
          AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
        run: |
          chmod +x scripts/deploy.sh
          ./scripts/deploy.sh
```

### destroy.yml — the red button

```yaml
# .github/workflows/destroy.yml
name: Destroy Environment

on:
  workflow_dispatch:  # Manual trigger only
    inputs:
      environment:
        description: "Environment to destroy (dev/test/prod)"
        required: true
        default: "dev"

permissions:
  id-token: write
  contents: read

jobs:
  destroy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_DEFAULT_REGION }}

      - uses: hashicorp/setup-terraform@v3

      - name: Destroy
        env:
          ENVIRONMENT: ${{ github.event.inputs.environment }}
          AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
        run: |
          chmod +x scripts/destroy.sh
          ./scripts/destroy.sh
```

### The deploy script (what the workflow runs)

```bash
# scripts/deploy.sh
#!/bin/bash
set -e

ENVIRONMENT=${ENVIRONMENT:-dev}
cd terraform

# Connect to remote state (S3 backend)
terraform init \
  -backend-config="bucket=twin-terraform-state" \
  -backend-config="key=twin/${ENVIRONMENT}/terraform.tfstate" \
  -backend-config="region=${AWS_DEFAULT_REGION}" \
  -backend-config="dynamodb_table=twin-terraform-locks"

# Switch to the target workspace
terraform workspace select "$ENVIRONMENT"

# Apply
terraform apply -auto-approve -var-file="terraform.tfvars"
```

## Common misconceptions

- **Misconception:** Deploy and destroy are symmetric — just opposite commands.
  **Reality:** Deploy is triggered automatically (on push). Destroy should *never* be automatic — it's irreversible. Use `workflow_dispatch` (manual trigger) for destroy, and require environment approvals in production.

- **Misconception:** The deploy script is just `terraform apply`.
  **Reality:** The script also handles: packaging the Lambda function (zipping Python code), building the frontend (npm build), uploading artifacts to S3, and configuring the backend. Terraform apply is one step in a multi-step process.

- **Misconception:** You need separate workflow files for dev, test, and prod.
  **Reality:** One `deploy.yml` can deploy to any environment — the environment is parameterized via workspace selection in the deploy script. You can add environment-specific triggers (e.g., branch-based: `dev` branch → dev environment).

## What abstractions / AI tools often hide

When AI generates workflow YAML, it hides:
- **The `permissions` block** — `id-token: write` is required for OIDC. `contents: read` is required for `actions/checkout`. Missing either causes cryptic permission errors. AI tools frequently omit these.
- **The `setup-terraform` action** — GitHub runners don't have Terraform pre-installed. You need `hashicorp/setup-terraform@v3` to install it. AI tools sometimes assume Terraform is available.
- **`chmod +x` before running scripts** — scripts checked into git may lose their executable permission. AI tools generate `./scripts/deploy.sh` without the `chmod`, causing "permission denied" errors.
- **The Lambda packaging step** — before Terraform can create the Lambda function, the Python code must be zipped and uploaded. This is a build step, not a Terraform step. AI tools often skip it, causing Terraform to fail ("file not found: lambda_package.zip").
- **Environment protection rules** — GitHub supports "environments" (Settings → Environments) with required reviewers. In production, you should require manual approval before the deploy job runs. AI tools don't set this up.

## Practical engineering implications

- **Trigger deploy on push to main, never on every branch.** Otherwise, every feature branch triggers a deployment. Use branch protection + PRs to control what reaches main.
- **Make destroy manual-only (`workflow_dispatch`).** Never auto-trigger destroy. Add a confirmation input (environment name) to prevent accidental destruction.
- **Keep deploy scripts cross-platform.** GitHub Actions runs Linux. Your local machine might be Mac or Windows. Keep the `.sh` scripts (GitHub uses them) even if you also have `.ps1` versions for local Windows use.
- **Use environment protection rules for production.** In GitHub repo settings, create a "production" environment with required reviewers. The deploy job to prod won't run until a human approves.
- **Log Terraform output in the workflow.** The workflow run logs are your audit trail. If something goes wrong, the logs show exactly what Terraform planned and applied.
- **Cache dependencies.** Use `actions/cache` for Python packages and npm modules. Speeds up workflow runs significantly.

## Related topics

- [GitHub Actions CI/CD](github-actions-cicd.md) — the platform these workflows run on
- [OIDC Authentication](oidc-authentication.md) — how the workflow gets AWS access
- [Remote State Backend](remote-state-backend.md) — where state lives across runs
- [Terraform Workflow Commands](../system-design/terraform-workflow-commands.md) — what the scripts run

## References

- [GitHub Actions: Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [hashicorp/setup-terraform](https://github.com/hashicorp/setup-terraform)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub: Manual Triggers (workflow_dispatch)](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)
