# Next.js — The React Application Framework

> A full-featured framework built on top of React that bundles routing, rendering, transpiling, and deployment into one package — so you can build production apps without assembling the parts yourself.

## What is it?

Next.js is a meta-framework for React, created by **Vercel** (the company behind the Vercel cloud platform). It takes React — which is "just" a UI library — and adds everything a real web application needs: file-based routing, server-side rendering, API routes, image optimization, code splitting, and build tooling. Instead of choosing and configuring a router, a bundler, a transpiler, and a deployment pipeline separately, Next.js gives you all of them, pre-configured and working together.

## Why does it matter?

React alone is not an application — it's a library for rendering UI. To build a real app you need routing (multi-page navigation), server-side rendering (SEO, performance), API endpoints, and build optimization. Before Next.js, teams spent days assembling these. Next.js makes the "hello world to production" path a single command: `npx create-next-app`.

It's also the most popular React framework, with strong defaults and an opinionated structure that keeps teams aligned.

## Mental model

Think of Next.js as **Rails for React**. Ruby on Rails became famous for "convention over configuration" — instead of wiring up every piece manually, you follow conventions and get a working app. Next.js does the same for React: put a file in `pages/` and it becomes a route; put a file in `app/` and it's a server component. The framework handles the plumbing.

## How does it actually work?

```bash
npx create-next-app@latest my-app
# → React + TypeScript + routing + bundling + dev server
```

Project structure:

```
my-app/
├── app/          # or pages/ — each file is a route
│   ├── layout.tsx
│   ├── page.tsx      # → /
│   └── about/
│       └── page.tsx  # → /about
├── public/          # static assets
├── next.config.js   # configuration
└── package.json
```

Key features out of the box:
- **File-based routing** — files map to URLs automatically
- **Code splitting** — each route loads only its own code
- **SSR/SSG** — render on server or at build time
- **API routes** — backend endpoints in the same project
- **Image optimization** — responsive images, lazy loading
- **Fast Refresh** — instant feedback during development

## Example

```tsx
// app/page.tsx — this file is automatically served at "/"
export default function Home() {
  return (
    <main>
      <h1>Welcome to my Next.js app</h1>
    </main>
  );
}

// app/api/hello/route.ts — API endpoint at "/api/hello"
export async function GET() {
  return Response.json({ message: "Hello from the server" });
}
```

No router configuration. No webpack config. Files in folders become routes.

## Common misconceptions

- **Misconception:** Next.js is just React with extra stuff.
  **Reality:** Next.js changes the mental model: pages are file-based, rendering can happen on the server, and components have different rules (server vs client components). It's a different paradigm.

- **Misconception:** You must use Vercel to deploy Next.js.
  **Reality:** Next.js can be self-hosted (Docker, Node server, static export). Vercel is the easiest path, not the only one.

- **Misconception:** Next.js is only for big apps.
  **Reality:** For small projects, Next.js's zero-config setup is faster than setting up React + router + bundler manually.

## What abstractions / AI tools often hide

When AI generates a Next.js app, it hides:
- **Server vs client components** — in the App Router, components are server-side by default. Adding `"use client"` opts into client rendering. This distinction is invisible until it breaks (e.g., `useState` in a server component → error).
- **The build pipeline** — Next.js uses SWC (a Rust-based compiler) for transpilation and Turbopack/webpack for bundling. These are powerful but opaque. When builds fail, understanding the pipeline matters.
- **Data fetching changes** — Next.js 13+ changed data fetching paradigms (server components fetch data directly, no `useEffect` needed). AI tools may generate older patterns.

## Practical engineering implications

- **Start with the App Router.** It's the recommended approach and the future. Pages Router is legacy (but still common in older tutorials).
- **Understand server vs client components.** This is the #1 source of confusion. Server components can't use hooks or browser APIs. `useState`, `useEffect`, event handlers require `"use client"`.
- **Use the built-in optimizations.** `next/image` for images, `next/font` for fonts, `next/link` for navigation. These are not cosmetic — they meaningfully improve performance.
- **Co-locate API routes.** Next.js API routes let you build a full-stack app in one repo. For AI apps, this is where your LLM calls live.

## Related topics

- [React Components](../programming/react-components.md) — what Next.js renders
- [Pages Router vs App Router](nextjs-routing.md) — the two routing paradigms
- [Client-Side vs Server-Side Rendering](client-side-vs-server-side-rendering.md) — rendering strategies
- [Transpiling & Bundling](transpiling-and-bundling.md) — the build pipeline

## References

- [Next.js Documentation](https://nextjs.org/docs)
- [Next.js App Router](https://nextjs.org/docs/app)
- [Vercel](https://vercel.com/) — the company behind Next.js
