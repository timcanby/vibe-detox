# Remote State Backend for CI/CD

> When Terraform runs in GitHub Actions instead of your laptop, state can't live in a local file — it needs shared, centralized storage that every workflow run can access.

## What is it?

A **remote state backend** is a shared storage location for Terraform state files that multiple machines and CI/CD pipelines can read and write to. When Terraform runs locally, state lives in `terraform.tfstate` on your disk. When Terraform runs in GitHub Actions, each workflow run is a fresh virtual machine — local state would be lost between runs.

The solution: store state in an **S3 bucket** (for the state file) and a **DynamoDB table** (for locking, preventing concurrent writes). Every GitHub Actions run reads from and writes to this shared backend.

## Why does it matter?

Without remote state, CI/CD Terraform is impossible:
- Each GitHub Actions run is a disposable VM. State written to disk is deleted when the VM shuts down.
- The next workflow run has no idea what infrastructure exists — it would try to recreate everything (and fail, because names are taken).
- Multiple workflow runs (e.g., a push while another deploy is still running) would create conflicting infrastructure.

Remote state solves all three: state persists across runs, is shared, and locking prevents concurrent modifications.

## Mental model

Think of it as a **shared notebook in a library**:
- Local state: you keep notes in your personal notebook. When you leave the library (VM shuts down), the notes are gone.
- Remote state: the notebook lives on a shelf in the library (S3). Anyone entering the library can read it. To write, you check out the notebook (DynamoDB lock) — no one else can write until you return it.

The S3 bucket is the shelf; the DynamoDB lock is the checkout system.

## How does it actually work?

### Step 1: Create the backend infrastructure (one-time)

You run a temporary Terraform config locally to create the S3 bucket and DynamoDB table *before* configuring the backend. This is a bootstrap — you need the bucket to exist before you can store state in it.

```hcl
# backend-setup.tf (TEMPORARY — run once, then delete this file)

resource "aws_s3_bucket" "terraform_state" {
  bucket = "twin-terraform-state"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"  # Version state history for rollback
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "twin-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```bash
# Run it once locally (in default workspace)
cd terraform
terraform workspace select default
terraform init
terraform apply -target=aws_s3_bucket.terraform_state -target=aws_dynamodb_table.terraform_locks

# Get the outputs (you'll need these)
terraform output
# → state_bucket = "twin-terraform-state"
# → locks_table = "twin-terraform-locks"

# DELETE the temporary file — it's done its job
rm backend-setup.tf
```

### Step 2: Configure the backend (permanent)

```hcl
# backend.tf (PERMANENT — stays in your repo)

terraform {
  backend "s3" {
    bucket         = "twin-terraform-state"
    key            = "twin/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "twin-terraform-locks"
    encrypt        = true
  }
}
```

Now every `terraform init` connects to the S3 backend. State is read from and written to S3, with DynamoDB locking.

### Step 3: Update deploy scripts for CI/CD

```bash
# scripts/deploy.sh (updated for remote backend)

cd terraform

# OLD (local state):
# terraform init

# NEW (remote backend with explicit config):
terraform init \
  -backend-config="bucket=twin-terraform-state" \
  -backend-config="key=twin/${ENVIRONMENT}/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=twin-terraform-locks" \
  -backend-config="encrypt=true"

terraform workspace select "$ENVIRONMENT"
terraform plan  -var-file="../terraform.tfvars"
terraform apply -auto-approve -var-file="../terraform.tfvars"
```

The `-backend-config` flags let you pass backend settings at runtime — essential for CI/CD where you can't hardcode values.

## Common misconceptions

- **Misconception:** The S3 state bucket needs to be in the same region as your infrastructure.
  **Reality:** The state bucket can be in any region. It's just storage. But keeping it in the same region reduces latency and simplifies permissions.

- **Misconception:** You need to create the backend infrastructure every time you deploy.
  **Reality:** The S3 bucket and DynamoDB table are created *once* (the bootstrap). They persist across all future deployments. The temporary Terraform file that creates them is deleted after the one-time run.

- **Misconception:** State locking prevents all concurrent issues.
  **Reality:** Locking prevents concurrent `terraform apply` runs. But if someone manually changes resources in the AWS console while a workflow is running, state can still drift. Locking prevents Terraform-vs-Terraform conflicts, not Terraform-vs-human conflicts.

## What abstractions / AI tools often hide

When AI generates backend configuration, it hides:
- **The bootstrap paradox** — you need the S3 bucket to exist before you can configure it as a backend. So you must create it *without* the backend configured (local state), then switch to remote. AI tools often generate both the backend config and the bucket creation in the same file, causing a chicken-and-egg error.
- **`-backend-config` at init time** — in CI/CD, you can't rely on the `backend.tf` file alone because environment-specific values (like the state key path) change per workspace. AI tools rarely show the `-backend-config` flag pattern.
- **State file path per workspace** — each workspace gets a separate state file in the bucket. The `key` in the backend config is the base path; Terraform appends `env:/WORKSPACE_NAME/` automatically. AI tools don't explain this path structure.
- **DynamoDB billing mode** — `PAY_PER_REQUEST` is correct for low-frequency Terraform runs. AI tools sometimes generate `PROVISIONED` which incurs ongoing costs even when unused.

## Practical engineering implications

- **Create the backend first, configure it second.** The S3 bucket and DynamoDB table must exist before `terraform init` can connect to them. Bootstrap locally, then switch to remote.
- **Enable S3 versioning on the state bucket.** If state is corrupted, you can roll back to a previous version. This is a one-line config that saves you from catastrophic state loss.
- **Use `encrypt = true`.** State files may contain secrets. S3 server-side encryption protects them at rest.
- **Pass backend config via `-backend-config` in CI/CD.** This lets you parameterize the state path per environment without changing the `backend.tf` file.
- **Delete temporary setup files.** The `backend-setup.tf` (and `github-oidc.tf`) are bootstrapping artifacts. After running them once, delete them — they clutter the repo and can confuse future `terraform plan` runs.
- **One state file per environment.** Use workspaces or separate state keys. Never share state between dev and prod — a `terraform destroy` in dev would destroy prod resources.

## Related topics

- [Terraform State Management](../system-design/terraform-state-management.md) — the state file itself
- [Terraform Workspaces](../system-design/terraform-workspaces.md) — environment isolation
- [OIDC Authentication](oidc-authentication.md) — how GitHub Actions gets AWS access
- [GitHub Actions CI/CD](github-actions-cicd.md) — where this backend is used

## References

- [Terraform: S3 Backend](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Terraform: Backend Configuration](https://developer.hashicorp.com/terraform/cli/init#backend-configuration)
- [AWS: DynamoDB for State Locking](https://developer.hashicorp.com/terraform/language/state/locking)
- [Terraform: Bootstrap S3 Backend](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-remote)
