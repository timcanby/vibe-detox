# Terraform State Management

> Terraform's memory of what infrastructure actually exists — the bridge between your code and reality.

## What is it?

**State** is Terraform's record of what resources it has created, their current configuration, and their real-world IDs. It's stored in a file called `terraform.tfstate` (JSON). Every time you run `terraform apply`, Terraform:

1. Reads the state file to learn what currently exists
2. Reads your `.tf` files to learn what *should* exist
3. Computes the diff
4. Makes API calls to create/update/destroy resources
5. Updates the state file to reflect the new reality

Without state, Terraform would have no way to know "does this S3 bucket already exist?" without querying AWS on every run — which would be slow and unreliable.

## Why does it matter?

State is the single most important — and most dangerous — concept in Terraform:

- **It's the source of truth.** If state says a resource exists, Terraform manages it. If state is wrong, Terraform may try to recreate (or destroy!) resources that already exist.
- **It may contain secrets.** State files can include passwords, API keys, and connection strings from resources Terraform created.
- **It must not be in git.** State changes on every `apply`. It's not version-controlled code — it's a live snapshot.
- **It must be shared across teams.** If two engineers run `terraform apply` simultaneously with different state files, they'll create conflicting resources.

## Mental model

Think of state as a **property deed**. It records: "Terraform owns this S3 bucket (ID: my-bucket-123), this Lambda function (ARN: ...), and this IAM role." When you change the blueprints (`.tf` files), Terraform checks the deed (state) to see what it already owns, then buys/sells (creates/destroys) to match the new blueprints.

If the deed is lost, Terraform doesn't know what it owns — it may try to buy something already owned (error: bucket exists) or ignore something it should manage (orphaned resource).

## How does it actually work?

### The state file (simplified)

```json
{
  "version": 4,
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "memory",
      "instances": [{
        "schema_version": 0,
        "attributes": {
          "bucket": "twin-dev-memory-123456",
          "id": "twin-dev-memory-123456",
          "region": "us-east-1"
        }
      }]
    },
    {
      "type": "aws_lambda_function",
      "name": "api",
      "instances": [{
        "attributes": {
          "function_name": "twin-dev-api",
          "arn": "arn:aws:lambda:us-east-1:123456:function:twin-dev-api"
        }
      }]
    }
  ]
}
```

### Local vs remote state

**Local state** (default): `terraform.tfstate` in your project directory. Fine for solo, throwaway work. Dangerous for teams.

**Remote state** (recommended): state stored in a shared backend (S3 + DynamoDB, Terraform Cloud, GCS, Azure Blob). Everyone on the team reads/writes the same state file, with locking to prevent simultaneous applies.

```hcl
# Configure remote state (in versions.tf or backend.tf)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "twin/dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### The .gitignore rules

```gitignore
# .gitignore for Terraform
*.tfstate          # NEVER commit state files
*.tfstate.*        # backup state files
.terraform/        # provider plugins (large, auto-downloaded)
*.tfvars           # may contain secrets (with exception below)
crash.log          # crash logs

# DO commit these:
!terraform.tfvars  # default variable values (non-secret)
!prod.tfvars       # production variable values (if no secrets)
.terraform.lock.hcl # provider version lock
```

## Common misconceptions

- **Misconception:** State is just a cache — you can delete it and Terraform will rebuild it.
  **Reality:** Deleting state orphans all resources. Terraform no longer knows they exist. It will try to recreate them and fail (names already taken). Recovering from lost state is painful — you must manually `terraform import` each resource.

- **Misconception:** State files should be in git for version history.
  **Reality:** State changes on every `apply`. Git history would be polluted with JSON diffs. And state may contain secrets. Use remote backends with versioning (S3 has versioning built in).

- **Misconception:** Remote state is only for teams.
  **Reality:** Even solo developers benefit. Remote state survives machine crashes, enables CI/CD pipelines, and provides locking (prevents you from running `apply` in two terminals simultaneously).

## What abstractions / AI tools often hide

When AI generates Terraform code, it hides:
- **State locking** — without a DynamoDB lock table (or equivalent), two simultaneous `terraform apply` runs can corrupt state. AI tools rarely set up locking.
- **State drift** — if someone manually changes a resource in the AWS console (not via Terraform), the state no longer matches reality. `terraform plan` will show the drift, but AI tools don't explain how to resolve it (usually: `terraform apply` to reconverge, or `terraform import` to bring the manual resource under Terraform's control).
- **`terraform import`** — bringing existing (manually created) resources under Terraform management. This is essential for migrating from ClickOps to IaC, but AI tools rarely mention it.
- **State contains secrets** — resources like RDS databases store their passwords in state. The state file must be encrypted (S3 server-side encryption) and access-controlled. AI tools don't warn about this.

## Practical engineering implications

- **Use remote state from day one.** S3 + DynamoDB for AWS. It's free (except the $0.023/month for the S3 bucket) and prevents 90% of state problems.
- **Never commit `.tfstate` to git.** Verify `.gitignore` includes `*.tfstate`.
- **Enable state locking.** Without it, concurrent `apply` runs corrupt state. DynamoDB is the standard for AWS.
- **Encrypt state.** S3 server-side encryption (SSE-S3 or SSE-KMS). State may contain secrets.
- **Run `terraform plan` before `apply`.** The plan shows the diff. Review it, especially in production.
- **Use `terraform import` to adopt existing resources.** Don't recreate — import them into state.
- **One state file per environment.** Don't put dev and prod in the same state file. Use workspaces or separate backends.

## Related topics

- [Terraform Fundamentals](terraform-fundamentals.md) — the tool this state belongs to
- [Terraform Workspaces](terraform-workspaces.md) — environment isolation via separate states
- [Infrastructure as Code](infrastructure-as-code.md) — the paradigm

## References

- [Terraform: State](https://developer.hashicorp.com/terraform/language/state)
- [Terraform: Remote State (S3)](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Terraform: Import](https://developer.hashicorp.com/terraform/cli/import)
- [State Locking](https://developer.hashicorp.com/terraform/language/state/locking)
