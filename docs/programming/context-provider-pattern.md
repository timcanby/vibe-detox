# The Context Provider Pattern

> A React pattern for sharing state — like auth status — across an entire component tree without prop-drilling through every layer.

## What is it?

The **Context Provider pattern** in React lets you share data (auth state, theme, user preferences, locale) across many components without passing props through every intermediate level. A **Provider** component wraps your app and makes shared data available to any descendant that asks for it.

In auth, `ClerkProvider` wraps your entire app and makes auth state (is the user signed in? who are they?) available to every component — without you manually threading `isSignedIn={true}` through every component layer.

## Why does it matter?

Without Context, sharing auth state across your component tree requires **prop drilling**: passing `user` from `App` → `Layout` → `Header` → `Avatar` → `ProfileButton`. Five levels deep, every intermediate component receives a prop it doesn't use, just to pass it down.

With Context, the Provider holds the state once, and any component in the tree can read it directly:

```tsx
<ClerkProvider>
  <App />     // doesn't need user prop
</ClerkProvider>

// Deep inside App, any component can access auth:
function ProfileButton() {
  const { user } = useUser();  // reads from Clerk context, no prop
  return <button>{user.name}</button>;
}
```

## Mental model

Think of it as a **walkie-talkie system in a building**:
- Without context: every message must be relayed floor by floor (prop drilling) — receptionist → floor manager → desk → person. Slow and error-prone.
- With context: everyone in the building is on the same channel (Provider). Anyone can listen or speak directly (useContext hook). No relay needed.

The Provider is the radio tower; `useContext` is the receiver in each room.

## How does it actually work?

### The pattern in 3 steps

1. **Create a context** — defines what data will be shared
2. **Wrap your app in a Provider** — supplies the data
3. **Read it in any child** — using the context hook

```tsx
// Step 1: Clerk creates the context (inside @clerk/nextjs)
// You don't write this — Clerk does.

// Step 2: Wrap your app (pages/_app.tsx)
import { ClerkProvider } from "@clerk/nextjs";

export default function App({ Component, pageProps }) {
  return (
    <ClerkProvider>
      <Component {...pageProps} />
    </ClerkProvider>
  );
}
```

```tsx
// Step 3: Read auth state in any component
import { SignedIn, SignedOut, useUser } from "@clerk/nextjs";

function Header() {
  return (
    <>
      <SignedOut>
        <SignInButton><button>Sign In</button></SignInButton>
      </SignedOut>
      <SignedIn>
        <UserButton />  {/* Shows avatar, manages account */}
      </SignedIn>
    </>
  );
}

function ProfilePage() {
  const { user, isLoaded, isSignedIn } = useUser();

  if (!isLoaded) return <Spinner />;
  if (!isSignedIn) return <p>Please sign in</p>;

  return <h1>Hello, {user.firstName}</h1>;
}
```

### What Clerk's `<SignedIn>` / `<SignedOut>` do

These are convenience wrappers around `useUser()`. Internally:

```tsx
// Simplified: what Clerk's SignedIn does under the hood
function SignedIn({ children }) {
  const { isSignedIn } = useUser();
  return isSignedIn ? <>{children}</> : null;
}

function SignedOut({ children }) {
  const { isSignedIn } = useUser();
  return !isSignedIn ? <>{children}</> : null;
}
```

They read from context (the `ClerkProvider`) and conditionally render. You never pass `isSignedIn` as a prop — the context handles it.

## Example: building your own context

```tsx
import { createContext, useContext, useState } from "react";

// 1. Create context
const ThemeContext = createContext("light");

// 2. Provider component
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. Consumer hook
function useTheme() {
  return useContext(ThemeContext);
}

// Usage: any component can access theme without prop drilling
function Button() {
  const { theme } = useTheme();
  return <button className={theme}>Click</button>;
}

// App root
function App() {
  return (
    <ThemeProvider>
      <Button />  {/* No theme prop! Reads from context. */}
    </ThemeProvider>
  );
}
```

## Common misconceptions

- **Misconception:** Context replaces all state management.
  **Reality:** Context is for sharing state that many components need. For local component state, `useState` is simpler. For complex state logic, `useReducer` or a state library (Zustand, Redux) may be better.

- **Misconception:** Context causes re-renders of the entire tree.
  **Reality:** Only components that **consume** the context re-render when it changes. A component that doesn't call `useContext` is unaffected, even if it's inside the Provider.

- **Misconception:** `ClerkProvider` makes API calls to Clerk on every render.
  **Reality:** ClerkProvider initializes once, manages auth state internally, and only re-renders consumers when auth state actually changes (sign-in, sign-out, token refresh).

## What abstractions / AI tools often hide

When AI generates `<ClerkProvider>` wrapping the app, it hides:
- **The context boundary** — components inside `<ClerkProvider>` can use auth hooks; components outside cannot. Placing the Provider at the root (`_app.tsx`) ensures all pages are covered. Placing it deeper means only some routes have auth.
- **The re-render scope** — when auth state changes (sign-in), all components using `useUser()` re-render. If your app has hundreds of components using auth context, this can cause performance issues. Splitting contexts (auth state vs. user profile) mitigates this.
- **Server vs client context** — in Next.js App Router, context works differently for server components (context doesn't propagate across server/client boundaries). Clerk handles this, but if you build your own, you need to understand the boundary.
- **The initialization timing** — `useUser()` returns `isLoaded: false` until Clerk fetches auth state. Not handling this causes flash-of-wrong-content (showing "signed out" for a frame before "signed in" loads).

## Practical engineering implications

- **Place the Provider at the root.** `ClerkProvider` goes in `_app.tsx` (Pages Router) or `layout.tsx` (App Router). Every page needs auth context.
- **Handle the loading state.** `isLoaded` is `false` initially. Show a loading state, not "signed out," while waiting.
- **Don't over-use Context.** For state that only 2-3 nearby components share, prop drilling is simpler. Context shines when many components across many levels need the same data.
- **Split contexts for performance.** If your auth state has both "is signed in" (changes rarely) and "current user profile" (changes often), consider splitting them into separate contexts to avoid re-rendering the whole tree on profile updates.
- **Use the Provider's built-in components.** Clerk's `<SignedIn>`, `<SignedOut>`, `<UserButton>`, `<SignInButton>` are pre-built and tested. Use them instead of reimplementing the same logic.

## Related topics

- [Props vs State](../programming/react-props-vs-state.md) — the alternative to context (prop drilling)
- [Auth-as-a-Service](../backend/auth-as-a-service.md) — Clerk, which uses this pattern
- [Protected Routes](protected-routes.md) — where auth context is consumed

## References

- [React: Context API](https://react.dev/learn/passing-data-deeply-with-context)
- [Clerk: ClerkProvider](https://clerk.com/docs/references/nextjs/clerk-provider)
- [Clerk: useUser hook](https://clerk.com/docs/references/react/use-user)
- [React: Prop Drilling vs Context](https://react.dev/learn/passing-data-deeply-with-context#the-problem-with-prop-drilling)
