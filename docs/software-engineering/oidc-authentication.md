# OIDC Authentication — GitHub Actions to AWS

> Letting GitHub Actions deploy to AWS without storing long-lived access keys — using OpenID Connect (OIDC) for short-lived, scoped credentials.

## What is it?

**OIDC (OpenID Connect)** is an identity protocol that lets GitHub Actions authenticate to AWS without you creating static IAM access keys. Instead of storing `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as GitHub secrets (which never expire and can leak), GitHub requests short-lived credentials from AWS using a trust relationship called an **OIDC provider**.

You configure AWS to trust GitHub Actions (via an IAM role with a trust policy), and GitHub Actions "assumes" that role on each workflow run. The credentials are temporary — they expire after the workflow finishes.

## Why does it matter?

**Static keys are dangerous:**
- They never expire — a leaked key is valid forever until manually rotated
- They have whatever permissions the IAM user has — hard to scope down
- They're stored in GitHub secrets — another surface for leaks
- Rotating them is manual and disruptive

**OIDC is the modern approach:**
- No long-lived secrets to leak
- Credentials expire automatically (typically 1 hour)
- The trust policy can restrict *which* repo, *which* branch, *which* workflow can assume the role
- AWS CloudTrail logs every assumption — full audit trail

## Mental model

Think of it as a **valet key system**:
- **Static key:** you give the valet a copy of your house key. They can enter anytime, even after they quit. If they lose it, anyone can enter.
- **OIDC:** you call the front desk and say "the valet from GitHub is allowed to enter room AWS for the next hour." The valet shows their ID (OIDC token), gets a temporary key, enters, and the key expires.

No copy of your house key exists. The permission is scoped and temporary.

## How does it actually work?

```
GitHub Actions workflow starts
    ↓
GitHub generates an OIDC token (JWT) proving "I am workflow X from repo Y"
    ↓
Workflow sends token to AWS STS: "AssumeRoleWithWebIdentity"
    ↓
AWS checks: does this token match my trust policy?
  - Is the issuer github.com? ✓
  - Is the repo timcanby/twin? ✓
  - Is the branch main? ✓
    ↓
AWS issues temporary credentials (1 hour expiry)
    ↓
Workflow uses credentials to run Terraform / deploy
    ↓
Credentials expire automatically
```

### Setting up the trust (one-time, via Terraform)

```hcl
# github-oidc.tf (temporary — run once, then delete)

# 1. Create the OIDC provider (tells AWS to trust GitHub as an identity source)
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# 2. Create an IAM role that GitHub Actions can assume
resource "aws_iam_role" "github_actions" {
  name = "github-actions-deploy"

  # Trust policy: only allow GitHub Actions from your repo
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:sub" = "repo:YOUR_GITHUB_USERNAME/twin:ref:refs/heads/main"
        }
      }
    }]
  })
}

# 3. Attach policies (what the role can do in AWS)
resource "aws_iam_role_policy_attachment" "deploy" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"  # scope this down in production!
}
```

```bash
# Run it once (in default workspace)
cd terraform
terraform workspace select default
terraform init
terraform apply -target=aws_iam_openid_connect_provider.github -target=aws_iam_role.github_actions

# Get the role ARN (you'll need this for GitHub secrets)
terraform output
# → github_actions_role_arn = "arn:aws:iam::123456789:role/github-actions-deploy"
```

### Using it in a GitHub Actions workflow

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_DEFAULT_REGION }}

      - name: Deploy with Terraform
        run: ./scripts/deploy.sh
```

No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` anywhere. The `configure-aws-credentials` action handles the OIDC token exchange automatically.

## Common misconceptions

- **Misconception:** OIDC is more complex than static keys, so it's not worth it.
  **Reality:** The setup is a one-time Terraform run. After that, it's *simpler* than static keys — no rotation, no leaked secrets, no manual key management.

- **Misconception:** The OIDC token itself is a secret.
  **Reality:** The OIDC token is a JWT that's generated per-workflow-run and expires quickly. It's not stored anywhere — it's exchanged for temporary AWS credentials on the fly.

- **Misconception:** Any GitHub repo can assume your AWS role.
  **Reality:** The trust policy's `Condition` clause restricts which repos/branches can assume the role. `"repo:YOUR_USERNAME/twin:ref:refs/heads/main"` means only pushes to `main` in your `twin` repo can assume it.

## What abstractions / AI tools often hide

When AI generates OIDC setup code, it hides:
- **The thumbprint** — AWS requires the OIDC provider's certificate thumbprint. GitHub's thumbprint can change (rarely). AI tools hardcode it without explaining what happens if it changes (answer: OIDC breaks, you update the thumbprint).
- **The trust policy condition** — the `StringEquals` condition is the security boundary. AI tools often use `StringLike` with wildcards (`"repo:YOUR_USERNAME/*"`) which is less secure — it grants access from *any* of your repos, not just the one you want.
- **The `permissions: id-token: write`** — without this in the workflow YAML, GitHub won't generate the OIDC token. AI tools frequently omit this, causing cryptic "no identity token" errors.
- **AdministratorAccess is too broad** — AI-generated Terraform often attaches `AdministratorAccess` for convenience. In production, scope the policy to only the actions GitHub Actions actually needs (S3, Lambda, CloudFront, etc.).

## Practical engineering implications

- **Use OIDC over static keys, always.** The one-time setup cost is worth eliminating an entire class of secret-leak security incidents.
- **Scope the trust policy tightly.** Specify the exact repo and branch: `"repo:USERNAME/twin:ref:refs/heads/main"`. Don't use wildcards.
- **Scope the IAM role policies.** Don't use `AdministratorAccess` in production. Grant only the permissions the deployment actually needs.
- **Store the role ARN in GitHub secrets.** `AWS_ROLE_ARN` is the one secret you need — it's not sensitive (it's a public identifier), but it's how the workflow knows which role to assume.
- **Delete the temporary Terraform files.** The OIDC setup is a one-time bootstrap. After `terraform apply`, delete the setup `.tf` file — the resources persist in AWS, but the code doesn't clutter your repo.

## Related topics

- [GitHub Actions CI/CD](github-actions-cicd.md) — the workflows that use OIDC
- [Environment Variables & Secrets](environment-variables-and-secrets.md) — where the role ARN is stored
- [Terraform State Management](../system-design/terraform-state-management.md) — remote state for CI/CD

## References

- [GitHub: OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [AWS: Creating OIDC providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
