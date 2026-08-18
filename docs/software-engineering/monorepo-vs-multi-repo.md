# Monorepo vs Multi-Repo Strategy

> Should all your projects live in one repository (monorepo) or each in its own (multi-repo)? The decision affects CI/CD, deployment, code sharing, and team workflow.

## What is it?

- **Monorepo** — a single Git repository containing multiple projects or services. All code — frontend, backend, shared libraries, configs — lives together.
- **Multi-repo** — each project or service has its own Git repository. The frontend is one repo, the backend another, shared libraries are published as packages.

## Why does it matter?

This decision affects:
- **CI/CD complexity** — one pipeline (monorepo) vs. N pipelines (multi-repo)
- **Code sharing** — trivial (monorepo, just import) vs. package publishing (multi-repo)
- **Deployment** — one deploy target (monorepo) vs. independent deploys (multi-repo)
- **Team autonomy** — shared (monorepo) vs. independent (multi-repo)
- **Access control** — everyone sees everything (monorepo) vs. per-repo permissions (multi-repo)

## Mental model

- **Monorepo** = a single house. Everything under one roof. Easy to share, but everyone's in the same space.
- **Multi-repo** = an apartment building. Each unit is independent. More setup per unit, but clear boundaries.

## How does it actually work?

### Monorepo structure

```
my-monorepo/
├── apps/
│   ├── web/          # Next.js frontend
│   ├── api/          # FastAPI backend
│   └── worker/       # Background job processor
├── packages/
│   ├── ui/           # Shared UI components
│   ├── types/        # Shared TypeScript types
│   └── config/       # Shared config
├── package.json      # Root workspace config
└── turbo.json        # Build pipeline (Turborepo)
```

Importing shared code: `import { Button } from "@my-org/ui"` — it's just a local path, no package publishing.

### Multi-repo structure

```
# 3 separate repos:
github.com/my-org/web-app      → deployed to Vercel
github.com/my-org/api-server    → deployed to Fly.io
github.com/my-org/shared-types  → published to npm as @my-org/types

# To use shared types:
npm install @my-org/types      # version-pinned, published package
```

## Common misconceptions

- **Misconception:** Monorepos are only for big companies (Google, Meta).
  **Reality:** Tools like Turborepo, Nx, and pnpm workspaces make monorepos practical for teams of any size. Even solo developers benefit from having frontend + backend in one repo.

- **Misconception:** Multi-repo means no code sharing.
  **Reality:** You share via published packages (npm, PyPI). It's more ceremony but works fine. The trade-off is version management overhead.

- **Misconception:** Monorepos are slower because you clone everything.
  **Reality:** Modern tools (sparse checkout, shallow clone, Turborepo caching) make monorepos fast even at large scale. You don't build everything — only what changed.

## What abstractions / AI tools often hide

When AI suggests a project structure, it hides:
- **The CI/CD implication** — a monorepo needs a smart CI pipeline that only builds/deploy the changed apps. Without this, every push triggers N builds. Tools like Turborepo and Nx handle this, but AI rarely configures them.
- **Dependency management** — in a monorepo, upgrading a shared dependency requires updating one package and testing all dependents. In multi-repo, each repo upgrades independently (drift risk) but can pin versions (stability).
- **Deploy coupling** — in a monorepo, a broken shared library can block all apps from deploying. In multi-repo, a broken shared package can be pinned to the last working version by consumers.

## Practical engineering implications

- **One app, one repo (multi-repo) is simpler for courses and learning.** Each repo = one deploy target = one CI workflow. This is why the course uses one repo per week.
- **Multiple apps sharing code (monorepo) is better for products.** Shared types, components, and configs stay in sync. Use Turborepo or Nx.
- **Start multi-repo, migrate to monorepo if sharing becomes painful.** The reverse (splitting a monorepo) is much harder.
- **If using monorepo, invest in a build system early.** Turborepo (JS), Nx (JS/any), or just-pnpm-workspaces. A monorepo without smart builds is just a slow multi-repo.
- **Consider access control needs.** Monorepo = everyone sees everything. If you have contractors or external collaborators, multi-repo with per-repo permissions is safer.

## Related topics

- [GitHub Actions CI/CD](github-actions-cicd.md) — how repo structure affects CI
- [Next.js Framework](nextjs-framework.md) — a frontend that might live in a monorepo
- [FastAPI Deep Dive](../backend/fastapi-deep-dive.md) — a backend that might be in the same or separate repo

## References

- [Turborepo](https://turbo.build/repo)
- [Nx](https://nx.dev/)
- [pnpm Workspaces](https://pnpm.io/workspaces)
- [Monorepo tools comparison](https://monorepo.tools/)
