# Virtual DOM & Reconciliation

> The algorithm that lets you "describe the whole screen" without paying the cost of redrawing it.

## What is it?

The **Virtual DOM** is an in-memory tree of plain JavaScript objects that represents what the UI *should* look like. **Reconciliation** is the algorithm React runs to compare the new virtual tree against the previous one, compute the minimal set of changes, and apply only those to the real browser DOM.

## Why does it matter?

DOM mutations are the most expensive operation in a web app — each one can trigger layout reflow, repaint, and style recalculation. Naively redrawing the entire page on every state change would be visibly slow. Reconciliation is the trick that makes declarative UI practical: you get the ergonomics of "just describe the whole screen" at the cost of a cheap in-memory diff instead of a full DOM rebuild.

## Mental model

Think of it like **git diff for UI**. You don't apply the entire file every commit — git computes the changed lines and patches only those. React does the same for the DOM: it diffs the old tree against the new tree and patches only the changed nodes.

## How does it actually work?

```
State changes
    ↓
React calls your component functions → new Virtual DOM tree
    ↓
Diff: compare new tree vs previous tree
    ↓
Compute minimal patch (which nodes added/removed/changed)
    ↓
Apply patches to real DOM (the expensive part — minimized)
```

The diffing algorithm makes assumptions to stay fast (O(n)):

1. **Different element types → tear down and rebuild.** If a `<div>` becomes a `<span>`, React destroys the old subtree and builds a new one. No attempt to morph.
2. **Same element type → keep the DOM node, update changed attributes.** `<div className="a">` → `<div className="b">` keeps the node, swaps the class.
3. **List children → use `key` to match.** Without keys, React matches by position, which causes subtle bugs when items reorder. Keys give React stable identity.

```tsx
// ❌ No keys — React matches by index, may misidentify items
{items.map(item => <li>{item}</li>)}

// ✅ Keys — React tracks identity across reorders
{items.map(item => <li key={item.id}>{item}</li>)}
```

## Example

```tsx
// Before state change: virtual tree
<div>
  <h1>Title</h1>
  <p>Old text</p>
</div>

// After state change: new virtual tree
<div>
  <h1>Title</h1>
  <p>New text</p>   ← only this text node changed
</div>

// React's patch: DOM mutation
// document.querySelector("p").textContent = "New text"
// (nothing else touched)
```

## Common misconceptions

- **Misconception:** The Virtual DOM is faster than the real DOM.
  **Reality:** It adds overhead (the diff). The Virtual DOM is faster than *naive full re-renders*, but slower than hand-optimized direct DOM manipulation. Its value is ergonomics, not raw speed.

- **Misconception:** React re-renders the whole component tree on every change.
  **Reality:** React re-*calls* functions, but `memo` and component boundaries can skip subtrees whose props haven't changed. Reconciliation is about DOM patches, not function calls.

## What abstractions / AI tools often hide

When AI writes React code that "just works," it hides:
- **The diffing heuristics** — why changing a `<div>` to a `<span>` destroys state, why missing keys cause bugs. These aren't cosmetic — they cause real production failures.
- **`memo` / `useMemo` / `useCallback`** — performance escape hatches. AI tools add them liberally "just in case," but unnecessary memoization *adds* overhead. Understanding when to memoize is a skill AI frequently lacks.
- **Render phase purity** — your component function must be pure (same input → same output). Side effects in render break reconciliation's assumptions. AI-generated code sometimes violates this.

## Practical engineering implications

- **Always provide stable, unique keys for lists.** Index-as-key is a common source of subtle state bugs when items are added/removed/reordered.
- **Don't fight the diff.** If you're doing manual DOM manipulation alongside React, you're working against the model. Use refs sparingly.
- **Profile before optimizing.** React DevTools Profiler shows which components re-render and why. Don't guess — measure.
- **Keep render pure.** No API calls, no mutations, no random values in render. Side effects go in effects or event handlers.

## Related topics

- [Declarative UI](react-declarative-ui.md) — the paradigm this enables
- [React Components](react-components.md) — produce the virtual trees
- [Props vs State](react-props-vs-state.md) — what triggers re-renders

## References

- [React: Render and Commit](https://react.dev/learn/render-and-commit) — official explanation
- [Reconciliation (React docs)](https://legacy.reactjs.org/docs/reconciliation.html) — the original algorithm description
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture) — the modern reconciler internals
