# Declarative UI — React's Paradigm

> You describe *what* the UI should look like for each state; the framework figures out *how* to get there.

## What is it?

In a **declarative** UI, you write code that says: "given this data, the screen should look like *this*." You never write instructions for *how* to change the screen from one state to another — no "remove this div, add that class, update this text." The framework compares the before and after, and does the DOM surgery for you.

The opposite is **imperative** UI: you manually select DOM nodes and mutate them step by step.

## Why does it matter?

Imperative DOM manipulation is the single biggest source of front-end bugs:
- **State drift:** you forget to update one of 5 places when data changes; the UI shows stale info.
- **Race conditions:** two rapid updates overwrite each other.
- **Unreadable code:** the "how" buries the "what."

Declarative UI eliminates an entire class of bugs by making the UI a pure function of state. If the state is correct, the UI is correct. Always.

## Mental model

Think of it as a **spreadsheet**. You don't tell Excel how to redraw cell C4 when A1 changes — you write a formula. React is Excel for UI: the "formula" is your component, the "cells" are state variables.

## How does it actually work?

```tsx
// Declarative: describe what the UI looks like for each state
function TrafficLight({ color }: { color: "red" | "green" | "yellow" }) {
  return (
    <div style={{ background: color, width: 60, height: 60, borderRadius: "50%" }} />
  );
}

// Imperative equivalent (what you'd do WITHOUT React):
// const el = document.getElementById("light");
// el.style.background = newColor;
// el.style.width = "60px";  // etc. — manual, error-prone
```

Behind the scenes:
1. React calls your component function → gets a tree of virtual nodes.
2. React compares this tree to the last one.
3. React computes the minimal set of DOM mutations.
4. React applies those mutations.

You never see steps 2–4. You only write step 1.

## Example

```tsx
// The UI is a pure function of state — no DOM manipulation
function TodoList() {
  const [todos, setTodos] = useState(["Learn React", "Ship app"]);

  return (
    <ul>
      {todos.map((todo, i) => (
        <li key={i}>{todo}</li>
      ))}
    </ul>
  );
}
```

Add an item → React re-runs `TodoList()` → diffs → adds one `<li>`. You didn't write "append a child to the ul." You just said "here's the list."

## Common misconceptions

- **Misconception:** Declarative means slower because React re-renders everything.
  **Reality:** React re-*calls* your function, but only patches the real DOM where it differs. The virtual DOM comparison is cheap; DOM mutations are expensive.

- **Misconception:** You can never touch the DOM in React.
  **Reality:** You can (refs), but 95% of UIs never need to. The declarative path handles it.

## What abstractions / AI tools often hide

When AI generates a React component for you, it hides:
- **The reconciliation algorithm** — the diffing logic that makes "describe the whole screen" affordable. Without it, re-rendering everything would be visibly slow.
- **Key-based identity** — React uses the `key` prop to match elements between renders. AI rarely explains why missing keys cause state bugs.
- **Side-effect timing** — `useEffect` runs *after* render, not during. This is declarative: you describe the effect, React schedules it.

## Practical engineering implications

- **Never read from the DOM to determine state.** The DOM is a *reflection* of state, not a source of truth. If you find yourself doing `document.querySelector` to check what's shown, your state model is wrong.
- **State should be the single source of truth.** Derive everything else from it. Don't keep a parallel "is this visible" flag — derive it from state.
- **Beware implicit ordering.** Declarative code is easier to read but hides execution timing. Effects and callbacks have subtle ordering rules — understand the lifecycle.

## Related topics

- [React Components](react-components.md) — the units that make declarative UI possible
- [Virtual DOM & Reconciliation](react-reconciliation.md) — the engine behind the abstraction
- [Props vs State](react-props-vs-state.md) — the inputs to the declarative function

## References

- [React: Describing the UI](https://react.dev/learn/describing-the-ui) — official guide
- [Declarative UI (Wikipedia)](https://en.wikipedia.org/wiki/Declarative_programming)
- [The reactive revolution](https://www.youtube.com/watch?v=1VwREAmAOxM) — historical context
