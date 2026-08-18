# Transpiling & Bundling

> The invisible pipeline that turns your modern TypeScript/JSX into the vanilla JavaScript a browser can actually run.

## What is it?

- **Transpiling** (transform + compile) = converting source code from one language/format to another. TypeScript → JavaScript. JSX → `createElement()` calls. Modern ES2024 syntax → older syntax that more browsers understand.
- **Bundling** = taking many separate files (your code + dependencies) and combining them into fewer, optimized files for delivery. Tree-shaking removes unused code, minification strips whitespace/renames variables, and code-splitting produces per-route bundles.

## Why does it matter?

You write `.tsx` files with TypeScript types, JSX syntax, and `import` statements. **Browsers cannot run any of that directly.** Browsers understand:
- Plain JavaScript (ES2015+ for modern browsers)
- No JSX, no TypeScript types, no `import` from `node_modules` paths

Without transpiling and bundling, modern web development is impossible. Every production React app goes through this pipeline. Understanding it is the difference between "my build failed" panic and "I know what went wrong."

## Mental model

Think of it as a **translation pipeline**:

1. You write in a convenient language (TypeScript + JSX)
2. A **transpiler** (SWC, Babel, esbuild) converts it to plain JavaScript
3. A **bundler** (webpack, Turbopack, Rollup, Vite) combines all files into optimized bundles
4. The browser receives vanilla JS — fast, compatible, minimal

## How does it actually work?

```
Your Source Code (.tsx files)
    ↓
[Transpiler: SWC / Babel / esbuild]
    TypeScript types stripped
    JSX → createElement() calls
    Modern syntax → compatible syntax
    ↓
[Bundler: webpack / Turbopack / Vite]
    Resolve imports (follow dependency graph)
    Tree-shake (remove unused exports)
    Code-split (per-route bundles)
    Minify (strip whitespace, rename variables)
    ↓
Output: /static/js/chunk-abc123.js (vanilla JS)
    ↓
Browser loads optimized bundle
```

### Example: what transpiling does

```tsx
// What you write (TSX)
const Button = ({ label }: { label: string }) => {
  return <button className="btn">{label}</button>;
};

// After transpiling (JavaScript)
const Button = ({ label }) => {
  return React.createElement("button", { className: "btn" }, label);
};
```

### Example: what bundling does

```
Input: 200 files, 1.5MB total (with unused code)
    ↓ bundling + tree-shaking + minification
Output: 3 chunk files, 180KB total (only used code)
```

## Common misconceptions

- **Misconception:** Transpiling and compiling are the same.
  **Reality:** Compiling converts to machine code (or bytecode). Transpiling converts between similar-level languages (TS → JS, ES2024 → ES2015). Same source-level, different syntax.

- **Misconception:** Bundling just concatenates files.
  **Reality:** Bundling resolves the module graph, removes unused code (tree-shaking), splits into cacheable chunks, and minifies. It's a graph optimization problem.

- **Misconception:** You need to understand webpack to use React.
  **Reality:** Next.js abstracts the entire pipeline. You write TSX; Next.js handles transpiling and bundling. You only need to dig in when builds break or performance needs tuning.

## What abstractions / AI tools often hide

When AI says "just run `npm run build`," it hides:
- **The toolchain** — SWC (Rust, fast) vs Babel (JS, slower) vs esbuild (Go, fast). Each has different plugin ecosystems and trade-offs.
- **Source maps** — the bundled/minified code is unreadable. Source maps connect errors back to your original source. Without them, debugging production errors is guesswork.
- **Bundle size analysis** — `npm run build` silently produces a bundle. Whether it's 50KB or 5MB is invisible until you analyze it. `@next/bundle-analyzer` or `webpack-bundle-analyzer` reveals what's in the bundle.
- **Polyfills** — some features need runtime polyfills for older browsers. The build pipeline may or may not include them.

## Practical engineering implications

- **Know your bundler.** Next.js uses Turbopack (newer, Rust) or webpack. Vite uses esbuild + Rollup. The choice affects build speed and plugin availability.
- **Analyze your bundle regularly.** A dependency that seemed small can pull in a huge tree. Run bundle analysis before shipping.
- **Use dynamic imports for large, rarely-used features.** `const Chart = dynamic(() => import("./Chart"))` splits the chart library into a separate chunk loaded on demand.
- **Don't fight tree-shaking.** Write ES modules (`import/export`), not CommonJS (`require`). Tree-shaking only works on static imports.
- **Source maps in production.** Enable them for error tracking (Sentry, Bugsnag). They don't affect runtime performance.

## Related topics

- [JSX/TSX](../programming/react-jsx-tsx.md) — the syntax that gets transpiled
- [Next.js Framework](nextjs-framework.md) — handles this pipeline for you
- [React Components](../programming/react-components.md) — produce the code that gets bundled

## References

- [SWC (Speedy Web Compiler)](https://swc.rs/)
- [esbuild](https://esbuild.github.io/)
- [webpack: Tree Shaking](https://webpack.js.org/guides/tree-shaking/)
- [Vite](https://vitejs.dev/)
