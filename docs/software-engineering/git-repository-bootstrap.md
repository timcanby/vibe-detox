# Git Repository Bootstrap — From Project to Repo

> The one-time process of turning a local project directory into a proper GitHub repository: `.gitignore`, `git init`, first commit, remote, and push.

## What is it?

**Repository bootstrap** is the sequence of steps that takes a local project folder (code on your laptop) and turns it into a version-controlled GitHub repository that others can clone, contribute to, and that CI/CD pipelines can trigger on.

This is a one-time setup per project. Once bootstrapped, the repo lives on GitHub and local changes flow through `git push`.

## Why does it matter?

Without proper git setup:
- No version history — you can't see what changed when, or roll back
- No collaboration — others can't clone or contribute
- No CI/CD — GitHub Actions only works on GitHub repos
- No backup — if your laptop dies, the project is gone
- No code review — changes go in without oversight

Bootstrapping the repo is the gateway to all DevOps practices: PRs, CI/CD, code review, issue tracking, and automated deployment.

## Mental model

Think of it as **incorporating a business**. Your project has been operating as a sole proprietorship (files on your laptop). Incorporating (creating a GitHub repo) gives it:
- A legal identity (repo URL)
- A record book (git history)
- Shareholders (collaborators)
- Public filing (public repo) or private records (private repo)

After incorporation, every change goes through formal channels (commits → PRs → review → merge).

## How does it actually work?

### Step 1: Create `.gitignore`

```gitignore
# .gitignore — controls what NOT to commit

# Terraform (state and secrets — never commit)
*.tfstate
*.tfstate.*
.terraform/
*.tfvars          # may contain secrets
crash.log

# But DO commit these:
!terraform.tfvars  # default variable values (non-secret)
.terraform.lock.hcl

# Build artifacts
lambda_package.zip
lambda_package/
frontend/out/

# Environment files (secrets)
.env
.env.*
!.env.example      # DO commit the example template

# Dependencies
node_modules/
.venv/
__pycache__/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

### Step 2: Create `.env.example`

```bash
# .env.example — template for new contributors (committed to git)
# Copy to .env.local and fill in real values

NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_your_key_here
CLERK_SECRET_KEY=sk_test_your_key_here
CLERK_JWT_URL=https://your-app.clerk.accounts.dev/.well-known/jwks.json
AWS_ACCOUNT_ID=123456789012
AWS_DEFAULT_REGION=us-east-1
```

### Step 3: Remove nested `.git` directories

```bash
# If subdirectories have their own .git (e.g., from create-next-app),
# remove them — you want ONE repo, not nested repos

# Mac/Linux:
rm -rf frontend/.git backend/.git

# Windows (PowerShell):
# Remove-Item -Recurse -Force frontend\.git, backend\.git
```

### Step 4: Initialize the repo

```bash
# Create a new git repo with 'main' as the default branch
git init -b main

# Configure your identity (if not already set globally)
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

### Step 5: Stage and commit

```bash
# Stage all files (respecting .gitignore)
git add .

# Verify what will be committed
git status

# Create the first commit
git commit -m "Initial commit: digital twin full-stack AI app"
```

### Step 6: Create the GitHub repo and push

```bash
# On github.com: create a new EMPTY repo (no README, no .gitignore, no license)
# Repository name: twin
# Visibility: public or private

# Connect your local repo to GitHub
git remote add origin https://github.com/YOUR_USERNAME/twin.git

# Push to GitHub
git push -u origin main
```

The `-u` flag sets `origin/main` as the upstream, so future `git push` commands work without arguments.

## Common misconceptions

- **Misconception:** You should initialize the GitHub repo with a README and .gitignore.
  **Reality:** If you already have a local project, creating a GitHub repo with a README causes merge conflicts on first push. Always create an *empty* repo on GitHub and push your local code to it.

- **Misconception:** `.gitignore` is only for secrets.
  **Reality:** `.gitignore` is for anything that shouldn't be version-controlled: build artifacts, dependencies (`node_modules/`), local config (`.env`), OS files (`.DS_Store`), and IDE settings. Committing these clutters the repo and causes conflicts.

- **Misconception:** Nested `.git` directories are harmless.
  **Reality:** If `frontend/` has its own `.git`, GitHub treats it as a "submodule" — a repo inside a repo. The frontend code won't be visible in the parent repo. Always remove nested `.git` directories before the parent `git init`.

## What abstractions / AI tools often hide

When AI generates git bootstrap commands, it hides:
- **The `.env.example` pattern** — committing `.env.example` (non-secret template) while gitignoring `.env` (real secrets) is a best practice. AI tools create `.gitignore` but rarely mention the example file.
- **`git init -b main`** — the `-b main` flag sets the initial branch name. Without it, older git versions default to `master`, which then requires renaming. AI tools use `git init` without `-b`.
- **The upstream flag (`-u`)** — `git push -u origin main` sets the tracking relationship. Without `-u`, every future push needs `git push origin main` (full form). AI tools omit `-u`, causing confusion later.
- **Nested repo detection** — `create-next-app` creates a `.git` inside `frontend/`. If not removed, the frontend becomes a submodule. AI tools don't warn about this common trap.
- **`rm -rf` danger** — removing nested `.git` uses `rm -rf`, the most dangerous command in Unix. Being in the wrong directory or mistyping the path can delete your entire project. AI tools generate `rm -rf` commands without warnings.

## Practical engineering implications

- **Create `.gitignore` before `git init`.** If you `git add .` without a `.gitignore`, you'll commit `node_modules/` and `.env` — removing them later requires rewriting git history.
- **Always create an empty GitHub repo.** No README, no .gitignore, no license. Push your local code to avoid merge conflicts.
- **Provide `.env.example`.** New contributors know exactly which environment variables to set up. It's a documentation tool that doubles as an onboarding checklist.
- **Remove nested `.git` directories.** Check every subdirectory before `git init`. `find . -name ".git" -type d` reveals all nested repos.
- **Use `git init -b main`.** Modern git defaults to `main`, but older versions don't. Explicit is better.
- **Commit `.terraform.lock.hcl` but not `.terraform/`.** The lock file pins provider versions (should be shared). The `.terraform/` directory contains downloaded plugins (should be re-fetched via `terraform init`).

## Related topics

- [Environment Variables & Secrets](environment-variables-and-secrets.md) — `.env.example` vs `.env.local`
- [Deploy & Destroy Workflows](deploy-destroy-workflows.md) — what the repo enables
- [Monorepo vs Multi-Repo](monorepo-vs-multi-repo.md) — repo structure decisions

## References

- [Git: Getting Started](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control)
- [GitHub: Creating a new repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository)
- [gitignore.io — Generate .gitignore files](https://www.toptal.com/developers/gitignore)
- [Pro Git Book (free)](https://git-scm.com/book)
