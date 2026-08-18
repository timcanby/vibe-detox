# React Components — The Building Blocks

> A reusable, self-contained piece of UI that bundles markup, styles, and logic into one unit.

## What is it?

A React component is a JavaScript/TypeScript function (or class) that returns a description of what should appear on screen. Think of it as a custom HTML tag you invent — `<Button>`, `<ProductCard>`, `<ChatWindow>` — composed of markup, styling, and behavior sealed together. You assemble complex interfaces by stacking these blocks like Lego.

## Why does it matter?

Before components, front-end code lived in three separate worlds: HTML for structure, CSS for style, JS for behavior. Changing one meant hunting through three files. Components collapse that separation — each UI piece is a single, portable, testable unit. This is why entire organizations can ship UIs with hundreds of engineers: the component boundary is the team boundary.

## Mental model

Think of a component as a **function: input → UI**. The inputs are "props" (configuration data) and "state" (internal data). The output is a tree of DOM elements. React calls your function whenever inputs change and diff-updates only what changed.

## How does it actually work?

A component is just a function returning JSX:

```tsx
// A simple component
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}!</h1>;
}

// Compose components together
function App() {
  return (
    <div>
      <Greeting name="World" />
      <Greeting name="React" />
    </div>
  );
}
```

React calls `App()`, which calls `Greeting()` twice. The result is a tree of "virtual" nodes. React compares this tree to the previous one and patches the real DOM only where they differ.

## Example

```tsx
// Counter component — has its own state
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  );
}
```

## Common misconceptions

- **Misconception:** Components must be small.
  **Reality:** A component should represent one conceptual unit. A 300-line form can be a valid component if it has one job. Splitting for the sake of "smallness" creates indirection.

- **Misconception:** Components are classes.
  **Reality:** Since 2018 (Hooks), function components are the default. Class components are legacy.

## What abstractions / AI tools often hide

When an AI generates `<ProductCard />`, it hides:
- **Props drilling** — data must be threaded through every intermediate component
- **Re-render cost** — every state change re-runs the component function; unoptimized components in a large tree cause jank
- **Component lifecycle** — when the component mounts, updates, and unmounts determines side-effect timing (fetches, subscriptions, cleanup)

## Practical engineering implications

- **Naming matters.** A component name is documentation. `<UserAvatarRow>` tells you what it is; `<Row>` does not.
- **Colocate related code.** Keep a component's styles, tests, and sub-components near it. Feature folders beat type folders.
- **Design the props API deliberately.** Props are the public interface. Too many props = configuration hell; too few = inflexible.

## Related topics

- [Declarative UI](react-declarative-ui.md) — the paradigm components enable
- [Props vs State](react-props-vs-state.md) — the two inputs every component has
- [JSX/TSX](react-jsx-tsx.md) — the syntax components are written in

## References

- [React: Quick Start](https://react.dev/learn) — official conceptual intro
- [Thinking in React](https://react.dev/learn/thinking-in-react) — the canonical mental model
