# Pages Router vs App Router

> Next.js has two routing systems: the older **Pages Router** (simpler, battle-tested) and the newer **App Router** (more powerful, recommended).

## What is it?

Next.js provides two ways to handle routing — the mechanism that maps URLs to the code that renders each page:

- **Pages Router:** Create a file in `pages/` → it becomes a route. `pages/about.tsx` → `/about`. Simple, file-based, the original system.
- **App Router:** Create a file in `app/` → it becomes a route, with nested layouts, server components, and streaming. `app/about/page.tsx` → `/about`. Newer, more capable, now the recommended default.

## Why does it matter?

The router choice affects your entire project structure, how data flows, how layouts nest, and whether you can use server components. It's a foundational decision that's hard to reverse. Many existing projects use Pages Router (it was the only option for years); new projects should default to App Router.

## Mental model

- **Pages Router** = a flat file system. Each file is one page. Simple, but layouts and shared data require manual wiring.
- **App Router** = a nested file system. Folders nest, layouts wrap children automatically, and components can render on the server without JavaScript shipped to the client.

## How does it actually work?

### Pages Router

```
pages/
├── index.tsx        →  /
├── about.tsx        →  /about
├── products/
│   ├── index.tsx    →  /products
│   └── [id].tsx     →  /products/:id
└── _app.tsx         →  wraps every page (custom layout)
```

```tsx
// pages/about.tsx
export default function AboutPage() {
  return <h1>About Us</h1>;
}
```

Layouts require a custom `_app.tsx` wrapper or a shared component imported into every page.

### App Router

```
app/
├── layout.tsx       →  root layout (wraps everything)
├── page.tsx         →  /
├── about/
│   └── page.tsx     →  /about
├── products/
│   ├── layout.tsx   →  wraps all /products/* pages
│   ├── page.tsx     →  /products
│   └── [id]/
│       └── page.tsx →  /products/:id
```

```tsx
// app/layout.tsx — automatically wraps every page
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Navbar />
        {children}
      </body>
    </html>
  );
}

// app/about/page.tsx
export default function AboutPage() {
  return <h1>About Us</h1>;
}
```

Nested layouts wrap automatically — no manual import needed.

## Example: data fetching

```tsx
// Pages Router — client-side fetching with useEffect
import { useEffect, useState } from "react";

export default function Products() {
  const [products, setProducts] = useState([]);
  useEffect(() => {
    fetch("/api/products").then(r => r.json()).then(setProducts);
  }, []);
  return <ProductList products={products} />;
}

// App Router — server component, fetch directly
// (no "use client", no useEffect, runs on server)
export default async function Products() {
  const res = await fetch("/api/products");
  const products = await res.json();
  return <ProductList products={products} />;
}
```

## Common misconceptions

- **Misconception:** Pages Router is deprecated.
  **Reality:** It's still maintained and widely used. Many libraries (especially older auth libraries) only support Pages Router. It's not deprecated, just not the recommended default for new projects.

- **Misconception:** App Router is always better.
  **Reality:** App Router has a steeper learning curve (server vs client components, async components, caching). For a simple site, Pages Router can be faster to ship.

- **Misconception:** You can mix both routers.
  **Reality:** You can have both `pages/` and `app/` in the same project (for migration), but a given route should only be in one. Mixing is a migration strategy, not a permanent state.

## What abstractions / AI tools often hide

When AI generates Next.js code, it hides:
- **Which router it's using** — AI may mix Pages Router patterns (useEffect fetching) with App Router file structure, creating broken code.
- **Server component constraints** — in App Router, server components can't use `useState`, `useEffect`, event handlers, or browser APIs. AI often generates client code in server components, causing runtime errors.
- **Caching semantics** — App Router has aggressive caching (fetch caching, route caching). AI tools rarely explain why data seems stale or why revalidation is needed.

## Practical engineering implications

- **New project? Use App Router.** It's the future, has better performance defaults, and is officially recommended.
- **Legacy project on Pages Router?** Don't rush to migrate. Use `next/dev` docs to plan incremental migration. Both routers can coexist during migration.
- **Auth libraries may constrain your choice.** Some older auth libraries only support Pages Router. Check before committing.
- **Learn the "use client" directive.** In App Router, this is the boundary between server and client. Understanding it is the key to avoiding 80% of App Router bugs.

## Related topics

- [Next.js Framework](nextjs-framework.md) — the framework these routers belong to
- [Client-Side vs Server-Side Rendering](client-side-vs-server-side-rendering.md) — rendering strategies both routers support
- [React Components](../programming/react-components.md) — what the routes render

## References

- [Next.js: Pages Router](https://nextjs.org/docs/pages)
- [Next.js: App Router](https://nextjs.org/docs/app)
- [Upgrading from Pages to App Router](https://nextjs.org/docs/app/building-your-application/upgrading)
