# JSX and TSX — Hybrid Markup

> A syntax extension that lets you write HTML-like markup directly inside JavaScript (JSX) or TypeScript (TSX).

## What is it?

**JSX** (JavaScript XML) is a syntax extension that looks like HTML but compiles to JavaScript function calls. **TSX** is JSX with TypeScript type annotations. Together, they let you write UI markup and logic in the same file — markup and code interleave naturally.

Before JSX, you built UIs by concatenating strings: `` `<div class="card">` + title + `</div>` `` — error-prone, no syntax highlighting, no type checking. JSX makes markup a first-class part of the language.

## Why does it matter?

JSX is the reason React won. Before JSX, template languages (Handlebars, EJS, Jinja) were separate mini-languages with their own syntax, scoping rules, and limitations. JSX says: *the template IS JavaScript.* You get full language power — conditionals, loops, variables, functions — with zero template-engine indirection.

TSX adds TypeScript's type safety on top: your markup is type-checked. A `<Button title={42}>` where `title` expects a string is a compile-time error, not a runtime surprise.

## Mental model

JSX is **syntactic sugar for function calls**. This:

```tsx
<h1 className="title">Hello</h1>
```

Compiles to:

```ts
React.createElement("h1", { className: "title" }, "Hello");
```

Every JSX element is a function call that returns a plain object (a "React element"). JSX is just a prettier way to write those calls.

## How does it actually work?

```tsx
// TSX: markup + logic in one file
function UserCard({ user, isAdmin }: { user: User; isAdmin: boolean }) {
  return (
    <div className="card">
      {/* Conditional rendering with real JS */}
      {isAdmin && <span className="badge">Admin</span>}

      {/* Loop with map — no template 'for-each' needed */}
      <ul>
        {user.roles.map(role => (
          <li key={role.id}>{role.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

Key differences from HTML:
- `class` → `className` (because `class` is a JS keyword)
- `for` → `htmlFor`
- Style is an object: `style={{ color: "red" }}` not `style="color: red"`
- Events are camelCase: `onClick` not `onclick`
- Self-closing tags required: `<img src="..." />` not `<img>`

## Example

```tsx
// Expression interpolation — anything in {} is JavaScript
function Greeting({ name, count }: { name: string; count: number }) {
  const message = count > 0
    ? `Welcome back, ${name}! (${count} visits)`
    : `Welcome, ${name}!`;

  return (
    <div>
      <h1>{message}</h1>
      <p>Today is {new Date().toLocaleDateString()}</p>
    </div>
  );
}
```

## Common misconceptions

- **Misconception:** JSX is HTML.
  **Reality:** JSX looks like HTML but is JavaScript. `style` is an object, not a string. Attributes are camelCase. Self-closing tags are required.

- **Misconception:** JSX is a template engine.
  **Reality:** There's no separate template language. `{}` inside JSX runs real JavaScript — you have the full language, not a limited template DSL.

- **Misconception:** TSX is slower because of types.
  **Reality:** TypeScript types are erased at compile time. The runtime output is identical to JSX.

## What abstractions / AI tools often hide

When AI generates TSX, it hides:
- **The compilation step** — JSX must be transpiled (by Babel, SWC, or TypeScript) before browsers can run it. You never ship raw JSX to production.
- **The `createElement` underlying model** — understanding that `<div />` is `createElement("div", ...)` explains why you can store elements in variables, pass them as props, and return them from functions.
- **Type inference boundaries** — TSX gives you types on props, but generic component inference (`<Select<string>>`) and complex prop types require explicit annotation. AI tools often generate `any`-typed props, defeating the purpose of TSX.

## Practical engineering implications

- **Use TSX, not JSX.** The type safety catches real bugs: wrong prop types, missing required props, typos in event handlers.
- **Keep expressions in `{}` simple.** Complex logic inside JSX is hard to read. Extract to a variable or a helper function.
- **Extract reusable markup into components.** If you copy-paste the same JSX block twice, it's a component.
- **Fragments over wrapper divs.** Use `<>...</>` to group without adding a DOM node: `<><h1>A</h1><p>B</p></>`.

## Related topics

- [React Components](react-components.md) — the functions you write with JSX
- [Transpiling & Bundling](../software-engineering/transpiling-and-bundling.md) — how JSX becomes vanilla JS
- [React Ecosystem](react-component-libraries.md) — libraries that provide pre-built JSX components

## References

- [React: Writing Markup with JSX](https://react.dev/learn/writing-markup-with-jsx)
- [React: JavaScript in JSX with Curly Braces](https://react.dev/learn/javascript-in-jsx-with-curly-braces)
- [TypeScript JSX (TSX) handbook](https://www.typescriptlang.org/docs/handbook/jsx.html)
