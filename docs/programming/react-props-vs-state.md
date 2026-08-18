# Props vs State

> The two inputs to every React component: props come from outside (configuration), state lives inside (memory).

## What is it?

- **Props** (short for "properties") are data passed *down* from a parent component to a child. They're read-only — the child cannot modify its props. Props are how a parent configures a child.
- **State** is data managed *inside* a component. The component owns it, can change it, and React re-renders when it does. State is a component's private memory.

Together, props and state are the two things that determine what a component renders.

## Why does it matter?

Every React bug eventually comes down to one question: **"Is this data props or state?"** Getting it wrong causes:
- **Stale UI** — data in state that should be props (derived from parent), so it doesn't update when the parent changes.
- **Broken data flow** — child mutating props (which it can't), or parent reading child state (which it shouldn't).
- **Redundant re-renders** — duplicating the same data as both props and state.

Understanding the distinction is the single most important React skill.

## Mental model

- **Props = function arguments.** A component is a function; props are the parameters the caller passes in. The function can't change its own arguments.
- **State = local variables.** State is the component's private memory. Only the component itself can change it.

```
Parent(props) → Child(props)    // props flow down, read-only
Child(state) ↻ Child(state)    // state lives inside, mutable by owner
```

## How does it actually work?

```tsx
// Parent owns the data, passes it as props
function ProductList() {
  const [products, setProducts] = useState(["Apple", "Banana"]);

  return (
    <div>
      {products.map(p => (
        <ProductCard key={p} name={p} />  // ← 'name' is a prop
      ))}
    </div>
  );
}

// Child receives props — cannot modify them
function ProductCard({ name }: { name: string }) {
  const [liked, setLiked] = useState(false);  // ← 'liked' is state

  return (
    <div>
      <h3>{name}</h3>
      <button onClick={() => setLiked(!liked)}>
        {liked ? "♥" : "♡"}
      </button>
    </div>
  );
}
```

**Data flow:** `ProductList` owns `products` → passes `name` as prop → `ProductCard` can't change `name`, but can manage its own `liked` state.

## Example: lifting state up

When two children need to share data, the state moves up to their common parent:

```tsx
function SearchContainer() {
  const [query, setQuery] = useState("");

  return (
    <>
      <SearchBar value={query} onChange={setQuery} />
      <Results query={query} />
    </>
  );
}
```

`query` lives in the parent. Both children receive it as props. This is "lifting state up" — the canonical React pattern.

## Common misconceptions

- **Misconception:** State should be stored in the parent always.
  **Reality:** State should live in the *lowest common ancestor* of all components that need it. Too high = prop drilling; too low = duplicated state.

- **Misconception:** Props are immutable data.
  **Reality:** Props can be any value — strings, numbers, objects, arrays, even functions (callbacks). "Read-only" means the *child* can't modify them, not that they can't be objects.

- **Misconception:** `useState` is the only way to hold state.
  **Reality:** `useReducer` (complex state), `useRef` (mutable, non-rendering), `useContext` (shared state) are all state tools.

## What abstractions / AI tools often hide

When AI generates a component, it hides:
- **Where state should live** — AI defaults to putting state wherever it's first needed, creating prop-drilling and duplication. The "lift state up" decision is an architectural choice AI often gets wrong.
- **Derived state anti-pattern** — copying props into state (`const [local, setLocal] = useState(props.value)`) breaks when props change. This is a common AI-generated bug.
- **State management libraries** (Redux, Zustand, Jotai) — AI tools may pull these in unnecessarily. Not all state needs a library; often `useState` + context is enough.

## Practical engineering implications

- **Don't copy props into state.** If a value comes from props, use it directly. If you need a derived value, compute it: `const filtered = items.filter(...)` not `useState(items.filter(...))`.
- **Lift state up only as high as needed.** The lowest common ancestor. Don't put everything in a global store by default.
- **State changes must be immutable.** `setArr([...arr, newItem])` not `arr.push(newItem)`. React relies on reference equality to detect changes.
- **Props drilling past 2-3 levels is a smell.** Consider context, composition, or a state library.

## Related topics

- [React Components](react-components.md) — the functions that receive props and hold state
- [Declarative UI](react-declarative-ui.md) — why props + state → UI is the core idea
- [React Ecosystem](react-component-libraries.md) — libraries that build on this model

## References

- [React: State: A Component's Memory](https://react.dev/learn/state-a-components-memory)
- [React: Sharing State Between Components](https://react.dev/learn/sharing-state-between-components)
- [Lifting State Up (React docs)](https://react.dev/learn/choosing-the-state-structure)
