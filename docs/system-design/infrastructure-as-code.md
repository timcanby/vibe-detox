# Infrastructure as Code (IaC)

> Defining your cloud infrastructure — servers, databases, networks, load balancers — in text files that can be versioned, reviewed, and replayed, instead of clicking through cloud consoles.

## What is it?

**Infrastructure as Code (IaC)** is the practice of defining infrastructure resources (S3 buckets, Lambda functions, API gateways, databases) in declarative configuration files rather than creating them manually through a cloud provider's web console. Instead of clicking through 20 AWS screens to set up an environment, you write code that describes the desired state, and a tool (Terraform, CDK, Pulumi) creates or updates the infrastructure to match.

## Why does it matter?

Manual infrastructure ("ClickOps") is:
- **Not reproducible** — you click through screens, and if you need another environment, you do it all again, differently.
- **Not versioned** — there's no git history of what changed when. "Who opened port 22 to the world?" is unanswerable.
- **Error-prone** — a typo in a region name (`us-east-1` vs `us-eat-1`) silently breaks things or creates resources in the wrong place.
- **Not reviewable** — no PR, no code review, no diff. A junior engineer can open a production database to the internet with one click.

IaC solves all of this: infrastructure changes go through git, get reviewed, can be diffed, and are exactly reproducible. Build a test environment once, then create production with the same code and different variables.

## Mental model

Think of it as a **vending machine** (the course's analogy). You select what you want (write IaC code), press a button (`terraform apply`), and the infrastructure pops out. Press the same button again — you get the same result. No human clicking through screens.

## How does it actually work?

```
You write .tf files (declarative description)
    ↓
terraform plan (shows what will change)
    ↓
terraform apply (creates/updates resources to match code)
    ↓
terraform destroy (tears everything down)
```

### IaC vs manual — the same S3 bucket

```python
# Manual: log into AWS console → S3 → Create bucket
# → name: "my-bucket" → region: us-east-1 → click Create
# → go to permissions → block public access → save
# → repeat for every environment, every time, hoping you don't miss a step
```

```hcl
# IaC: write it once, apply it anywhere
resource "aws_s3_bucket" "memory" {
  bucket = "${var.prefix}-memory-${var.account_id}"
  region = "us-east-1"
}

resource "aws_s3_bucket_public_access_block" "memory" {
  bucket         = aws_s3_bucket.memory.id
  block_public_acls   = true
  block_public_policy = true
}
```

The IaC version is versioned, reviewable, and reusable across dev/test/prod.

## Example: the IaC tools landscape

| Tool | Creator | Language | Cloud-agnostic? | Notes |
|---|---|---|---|---|
| **Terraform** | HashiCorp | HCL (declarative) | ✅ AWS, GCP, Azure | Most popular, industry standard |
| **AWS CDK** | Amazon | TypeScript/Python | ❌ AWS only | Feels like programming, not config |
| **Pulumi** | Pulumi | TypeScript/Python/Go | ✅ | Like CDK but multi-cloud |
| **CloudFormation** | Amazon | YAML/JSON | ❌ AWS only | Native but verbose |
| **Ansible** | Red Hat | YAML | Config management | Not for provisioning |

Terraform is the most widely adopted because it's cloud-agnostic — learn it once, use it for AWS, GCP, and Azure.

## Common misconceptions

- **Misconception:** IaC means no manual steps at all.
  **Reality:** IAM setup (creating users, setting permissions) is still manual in most cases — it's the bootstrap. Once IAM is set up, everything else can be IaC.

- **Misconception:** IaC is only for large companies.
  **Reality:** Even a solo project benefits. Reproducible environments, cost cleanup (`terraform destroy` to remove everything), and no "what did I click?" debugging.

- **Misconception:** You should use the cloud-native tool (CDK for AWS, Deployment Manager for GCP).
  **Reality:** Cloud-native tools lock you in. Terraform works across all clouds — one skill, many providers.

## What abstractions / AI tools often hide

When AI generates Terraform code, it hides:
- **The state file** — IaC tools track what they've created in a state file. If the state is corrupted or out of sync with reality, `terraform apply` can destroy resources or create duplicates. Understanding state is critical.
- **Dependency resolution** — Terraform auto-detects dependencies between resources (Lambda needs the IAM role, CloudFront needs S3). The `depends_on` block is for edge cases. AI tools add it unnecessarily, hiding the auto-resolution.
- **The destroy path** — `terraform destroy` removes *everything*. In production, you rarely want this. AI tools don't warn about the irreversibility.
- **Cost implications** — IaC code can create expensive resources (e.g., a NAT gateway at $30/month). AI tools generate resources without mentioning cost.

## Practical engineering implications

- **Start with IaC from day one.** Even for prototypes. `terraform destroy` to clean up saves money and prevents orphaned resources.
- **Keep state files out of git.** State contains secrets and changes constantly. Use remote state (S3 + DynamoDB, or Terraform Cloud).
- **Use variables for environment separation.** One codebase, different `terraform.tfvars` for dev/test/prod. Don't duplicate the whole config.
- **Review Terraform PRs like code PRs.** A `terraform plan` output in the PR shows exactly what will change. Review it.
- **Check AWS billing after destroying.** Orphaned resources (EIPs, unused NAT gateways) still cost money. Verify your bill is zero.

## Related topics

- [Terraform Fundamentals](terraform-fundamentals.md) — the tool that implements IaC
- [Terraform State Management](terraform-state-management.md) — the state file
- [Environment Variables & Secrets](environment-variables-and-secrets.md) — variables in IaC

## References

- [What is Infrastructure as Code? (HashiCorp)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/what-is-terraform)
- [Terraform vs CloudFormation](https://developer.hashicorp.com/terraform/intro/vs-cloud-formation)
- [Gruntwork: IaC Best Practices](https://gruntwork.io/)
- [The Terraform Book](https://www.terraform.io/)
