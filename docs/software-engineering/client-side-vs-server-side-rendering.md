# Client-Side vs Server-Side Rendering

> Where does the HTML get built — in the user's browser (CSR) or on your server (SSR)? The choice affects performance, SEO, and architecture.

## What is it?

- **Client-Side Rendering (CSR):** The server sends a minimal HTML page (often just `<div id="root"></div>` and a `<script>`). The browser downloads the JavaScript, runs it, and the JS builds the UI. This is the "traditional" React/SPA approach.
- **Server-Side Rendering (SSR):** The server runs the React components, generates the full HTML, and sends a complete page. The browser shows content immediately. JavaScript then "hydrates" the page to make it interactive.

There are also hybrid strategies: **SSG** (Static Site Generation, build-time), **ISR** (Incremental Static Regeneration), and streaming SSR with React Server Components.

## Why does it matter?

- **Performance:** CSR shows a blank page while JS loads. SSR shows content immediately. On slow networks, the difference is seconds vs. blank screen.
- **SEO:** Search engine crawlers traditionally read HTML, not JavaScript. SSR/SSG gives them complete content. CSR may render as empty for crawlers.
- **Architecture:** SSR requires a running server (can't deploy as static files). CSR can be deployed to any CDN as static assets.

## Mental model

- **CSR** = a blank canvas + a paint kit shipped to the user. They paint it themselves.
- **SSR** = a pre-painted canvas delivered to the user. They just hang it up (hydration adds interactivity).

## How does it actually work?

### CSR flow

```
Browser requests /products
    ↓
Server sends: <div id="root"></div> + <script src="bundle.js">
    ↓
Browser downloads bundle (500KB)
    ↓
Browser executes JS → React renders components → DOM updates
    ↓
User sees content (several seconds later)
```

### SSR flow

```
Browser requests /products
    ↓
Server runs React components → generates full HTML
    ↓
Server sends: <div>...full product list...</div> + <script>
    ↓
Browser shows HTML immediately (content visible!)
    ↓
JS downloads, hydrates — makes page interactive
```

## Example

```tsx
// CSR (traditional React SPA) — client fetches data
import { useEffect, useState } from "react";

function Products() {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch("/api/products").then(r => r.json()).then(setData);
  }, []);
  if (!data) return <div>Loading...</div>;  // ← blank or spinner
  return <ProductList items={data} />;
}

// SSR (Next.js App Router) — server fetches, sends HTML
// No "use client" — runs on server
async function Products() {
  const data = await fetch("/api/products").then(r => r.json());
  return <ProductList items={data} />;  // ← HTML already has content
}
```

## Common misconceptions

- **Misconception:** SSR is always faster.
  **Reality:** SSR sends content faster (good for Time-to-Content), but the server does more work per request. For high-traffic sites, SSR server cost is higher. CSR static files can be cached on a CDN for free.

- **Misconception:** CSR is bad for SEO.
  **Reality:** Google can render JavaScript, but it's slower and less reliable. For content sites (blogs, e-commerce), SSR/SSG is safer. For app-like dashboards behind login, SEO doesn't matter — CSR is fine.

- **Misconception:** You must pick one for the whole app.
  **Reality:** Next.js lets you mix: some pages SSR, some SSG, some CSR. The choice is per-page.

## What abstractions / AI tools often hide

When AI says "use SSR for better performance," it hides:
- **Hydration cost** — SSR sends HTML, but the browser still downloads and runs the full JS bundle to make it interactive. If the bundle is large, hydration can cause a "blank but interactive-looking" pause.
- **Server infrastructure** — SSR requires a Node server running. You can't deploy to a static CDN. This changes your ops story (scaling, cold starts, cost).
- **Data fetching waterfalls** — in SSR, if components fetch data sequentially, each await blocks the whole page. React Server Components and streaming solve this, but AI tools often generate naive sequential fetches.

## Practical engineering implications

- **Content site (SEO matters)? → SSR or SSG.** Blogs, marketing pages, e-commerce. Users and crawlers need content fast.
- **App behind login (SEO irrelevant)? → CSR is fine.** Dashboards, admin panels. Simpler deployment (static CDN), no server cost.
- **Use SSG when possible.** If content doesn't change per-request, pre-render at build time. Best of both worlds: fast + cheap to serve.
- **Measure Time-to-First-Content, not just "fast."** A blank page that "loads fast" is worse than a content-rich page that takes an extra 200ms to hydrate.

## Related topics

- [Next.js Framework](nextjs-framework.md) — supports both CSR and SSR
- [Pages Router vs App Router](nextjs-routing.md) — App Router enables server components
- [React Components](../programming/react-components.md) — what gets rendered

## References

- [Next.js: Rendering](https://nextjs.org/docs/app/building-your-application/rendering)
- [React Server Components](https://react.dev/reference/rsc/server-components)
- [Web.dev: Rendering on the Web](https://web.dev/articles/rendering-on-the-web)
