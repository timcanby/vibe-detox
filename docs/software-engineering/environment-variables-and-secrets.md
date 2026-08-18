# Environment Variables & Secrets Management

> How to manage API keys, tokens, and configuration across local development, preview, and production — without leaking secrets into your repository.

## What is it?

**Environment variables** (env vars) are configuration values stored outside your code — in `.env` files locally, and in platform dashboards (Vercel, Fly.io, AWS) for production. They hold secrets (API keys, database URLs, auth tokens) and configuration (environment names, feature flags).

In a typical full-stack app, you have:
- **Frontend env vars** — prefixed with `NEXT_PUBLIC_` in Next.js, exposed to the browser
- **Backend env vars** — never exposed to the browser, only server-side
- **Secrets** — private keys, API tokens — must never be committed to git

## Why does it matter?

- **Security:** Committing a secret key to git is a critical breach. Anyone with repo access has your keys. GitHub scans for leaked secrets, but prevention is better.
- **Environment separation:** Development, preview, and production need different values. A dev key shouldn't work in prod (and vice versa).
- **Configuration without code changes:** Change API endpoints, feature flags, or provider keys without redeploying code.

The `.env.local` file (gitignored) holds secrets locally. The platform dashboard (e.g., Vercel) holds them for deployed environments. The code reads from `process.env` — the source is invisible to the code.

## Mental model

Think of env vars as a **safe with a combination lock**:
- The code knows *where* to look (the safe = `process.env.KEY_NAME`).
- The code doesn't know *what's inside* (the combination = the actual key value).
- Each environment has its own safe with different contents.
- The safe is never photographed (never committed to git).

## How does it actually work?

### Local development — `.env.local`

```bash
# .env.local (NEVER committed to git — in .gitignore by default in Next.js)

# Frontend (exposed to browser via NEXT_PUBLIC_ prefix)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxx

# Backend (server-only, never exposed to browser)
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxx
CLERK_JWT_URL=https://your-app.clerk.accounts.dev/.well-known/jwks.json
```

```tsx
// Frontend reads NEXT_PUBLIC_ vars at build time
const key = process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY;
```

```python
# Backend reads vars at runtime
import os
jwt_url = os.environ.get("CLERK_JWT_URL")
```

### Production — platform dashboard

```bash
# Set env vars on Vercel via CLI
vercel env add NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
# Paste the value, select "All environments" (preview + production)

vercel env add CLERK_SECRET_KEY
# Paste the value, select "All environments"

vercel env add CLERK_JWT_URL
# Paste the value, select "All environments"
```

Or set them in the Vercel dashboard → Settings → Environment Variables.

### The `.gitignore` safety net

Next.js's default `.gitignore` includes `.env*local`, so `.env.local` is automatically excluded from git. In editors like VS Code / Cursor, gitignored files appear dimmed — visual confirmation they won't be committed.

## Example: the three keys Clerk needs

| Variable | Where | Purpose |
|---|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Frontend + Vercel | Public key, safe to expose in browser |
| `CLERK_SECRET_KEY` | Backend + Vercel | Secret key, server-only, never in browser |
| `CLERK_JWT_URL` | Backend + Vercel | JWKS endpoint for backend JWT validation |

**Critical:** The key names must match exactly. A typo like `CLERK_SECRETE_KEY` will silently fail — the backend won't find the variable and auth breaks with no clear error.

## Common misconceptions

- **Misconception:** All env vars are secret.
  **Reality:** `NEXT_PUBLIC_` prefixed vars are embedded in the client bundle and visible to anyone. Only non-prefixed vars are server-only. Putting a secret in a `NEXT_PUBLIC_` var leaks it.

- **Misconception:** `.env.local` is enough for production.
  **Reality:** `.env.local` is for local dev only. Production needs env vars set on the hosting platform (Vercel, Fly.io, etc.). The `.env.local` file never reaches production.

- **Misconception:** If `.env.local` is in `.gitignore`, I'm safe.
  **Reality:** Mostly true, but check: (1) the `.gitignore` actually covers it, (2) you didn't accidentally `git add -f` it, (3) you didn't copy a secret into a non-ignored file (like a config file).

## What abstractions / AI tools often hide

When AI says "add your keys to `.env.local`," it hides:
- **The Vercel env var step** — setting keys locally is half the job. You must also set them on Vercel (via CLI or dashboard) for the deployed app to work. This is the #1 "it works locally but breaks in production" issue.
- **The `NEXT_PUBLIC_` prefix rule** — only vars with this prefix are available in the browser. Forgetting the prefix means `process.env.KEY` is `undefined` client-side. Adding it to a secret leaks it.
- **Environment scoping** — Vercel lets you set vars for Development, Preview, and Production separately. Setting a key for only one environment causes failures in others. "Select all environments" is the safe default for most keys.
- **Key name exactness** — `CLERK_SECRET_KEY` vs `CLERK_SECRETE_KEY` (typo) causes silent failures. The env var is simply `None`, and the error message rarely says "you misspelled the key name."

## Practical engineering implications

- **Never commit `.env` files.** Verify `.env*` is in `.gitignore`. If you accidentally commit a secret, rotate it immediately — assume it's compromised.
- **Use `NEXT_PUBLIC_` only for non-secret values.** Publishable keys, feature flags, public URLs. Never secret keys.
- **Set env vars on every deployment platform.** Local `.env.local` → Vercel dashboard/CLI → any other platform. Missing any one causes "works locally, breaks on deploy."
- **Double-check key names.** Copy-paste from the provider's dashboard. A single character difference = silent failure.
- **Use a `.env.example` file (committed) showing the required variable names** without values. This helps new contributors know what to set up.
- **Rotate keys periodically.** If a key leaks, revoke it in the provider dashboard and generate a new one. Update all environments.

## Related topics

- [Auth-as-a-Service](../backend/auth-as-a-service.md) — where the keys come from
- [Backend Auth Verification](../backend/backend-auth-verification.md) — uses `CLERK_JWT_URL`
- [Protected Routes](protected-routes.md) — uses `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`

## References

- [Next.js: Environment Variables](https://nextjs.org/docs/app/building-your-application/configuring/environment-variables)
- [Vercel: Environment Variables](https://vercel.com/docs/concepts/projects/environment-variables)
- [12-Factor App: Config](https://12factor.net/config)
- [Clerk: API Keys](https://clerk.com/docs/backend-requests/api-keys)
