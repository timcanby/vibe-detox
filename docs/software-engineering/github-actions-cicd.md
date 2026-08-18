# GitHub Actions CI/CD

> Automated workflows that run on GitHub's servers when you push code — build, test, and deploy your app without manual intervention.

## What is it?

**GitHub Actions** is GitHub's built-in CI/CD (Continuous Integration / Continuous Deployment) platform. You define workflows in YAML files (`.github/workflows/`), and GitHub automatically runs them on its servers when events happen — pushes, pull requests, releases, schedules, or manual triggers.

A **workflow** is a series of **jobs**, each running on a **runner** (a virtual machine). Jobs contain **steps** — shell commands or pre-built actions. The most common use case: when you `git push`, GitHub Actions builds your app, runs tests, and deploys to production if tests pass.

## Why does it matter?

Without CI/CD, deployment is manual: you SSH into a server, pull code, run migrations, restart the process. This is:
- **Error-prone** — you forget a step, production breaks
- **Unrepeatable** — "it works on my machine" but not on the server
- **Slow** — minutes to hours of manual work per deploy
- **Risky** — no automated tests gate production

CI/CD makes deployment a single `git push` — the pipeline handles everything. If tests fail, production is protected. If they pass, deployment is automatic. This is the backbone of modern software delivery.

## Mental model

Think of it as a **factory assembly line triggered by a commit**:

1. You push code → the conveyor belt starts
2. **Build station** — compile, install dependencies
3. **Test station** — run unit tests, integration tests
4. **Package station** — build Docker image, bundle assets
5. **Deploy station** — push to staging, then production

If any station fails, the belt stops. Nothing reaches production.

## How does it actually work?

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]  # Trigger on push to main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest  # GitHub's virtual machine
    steps:
      # Step 1: Get your code
      - uses: actions/checkout@v4

      # Step 2: Set up Python
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      # Step 3: Install dependencies
      - run: pip install -r requirements.txt

      # Step 4: Run tests
      - run: pytest

      # Step 5: Build Docker image
      - run: docker build -t myapp .

      # Step 6: Deploy (only if tests passed)
      - run: |
          echo "Deploying to production..."
          # Deploy command here (e.g., push to Vercel, Fly.io, etc.)
```

When you `git push` to `main`:
1. GitHub detects the push event
2. Spins up an Ubuntu VM
3. Runs each step in order
4. If any step fails → workflow stops, you get an email
5. If all pass → your app is deployed

## Example: deploy to Vercel

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Vercel CLI
        run: npm install -g vercel

      - name: Deploy to Vercel
        run: vercel --prod --token=${{ secrets.VERCEL_TOKEN }}
        env:
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
```

## Common misconceptions

- **Misconception:** CI/CD is only for big teams.
  **Reality:** A solo developer benefits more — it eliminates manual deployment, catches bugs early, and makes deploys reproducible. GitHub Actions is free for public repos and has 2,000 free minutes/month for private.

- **Misconception:** GitHub Actions is just for deployment.
  **Reality:** It runs any automation — linting, testing, releasing, code scanning, issue triage, scheduled tasks. CI is about automation; CD is one use case.

- **Misconception:** Workflows must run on GitHub's servers.
  **Reality:** You can use **self-hosted runners** — your own machines — for free, with no minute limits. Useful for private data or custom hardware (GPUs).

## What abstractions / AI tools often hide

When AI generates a workflow file, it hides:
- **Secrets management** — `secrets.VERCEL_TOKEN` must be set in GitHub's repository settings. AI generates the YAML referencing secrets, but you must manually add them. A common deployment failure.
- **Runner environment** — GitHub runners have specific software pre-installed. Missing dependencies (Docker, specific Python version) cause failures that AI-generated workflows don't anticipate.
- **Caching** — without caching, every workflow reinstalls all dependencies (slow). AI rarely generates caching steps, which can turn a 30-second workflow into a 3-minute one.
- **Matrix builds** — testing across multiple Python/Node versions. AI usually generates single-environment workflows, missing compatibility bugs.

## Practical engineering implications

- **Fail fast.** Put the cheapest, fastest checks first (lint, type-check) and slowest last (integration tests, deploy). If lint fails, don't wait 5 minutes for tests.
- **Protect your main branch.** Use branch protection rules: PRs must pass CI before merge. Never push directly to `main`.
- **Cache dependencies.** Use `actions/cache` or built-in caching in setup actions. Saves minutes per run.
- **Keep secrets in GitHub Secrets.** Never hardcode tokens in workflow files. `secrets.NAME` is the only correct way.
- **Use environments for staging vs production.** GitHub supports environments with separate secrets and approval gates. Deploy to `staging` automatically, require manual approval for `production`.

## Related topics

- [Monorepo vs Multi-Repo](monorepo-vs-multi-repo.md) — how repo structure affects CI/CD
- [FastAPI Deep Dive](../backend/fastapi-deep-dive.md) — the backend you'd deploy

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
