# Protected Routes

> Pages that only signed-in users can access — the frontend gatekeeper between public and private parts of your app.

## What is it?

A **protected route** is a page in your web app that requires authentication to view. If a user is not signed in, they're redirected to a sign-in page (or shown a "please sign in" message). Only after successful authentication can they access the page.

In Next.js with Clerk, this is done by wrapping protected pages in auth-guarding components or using middleware. The `pages/product.tsx` is only accessible to authenticated users; unauthenticated users see the landing page with a "Sign In" button instead.

## Why does it matter?

Without protected routes, every page is public — anyone can see any URL, including user dashboards, billing pages, and admin panels. Protected routes are the **frontend layer of access control**: they prevent casual access and provide a good UX (redirect to sign-in instead of a blank page or error).

However, protected routes are **not security** — they're UX. The real security check must happen on the backend. Frontend route protection is about user experience; backend token validation is about security.

## Mental model

Think of a **club with a VIP lounge**:
- The main entrance is open to anyone (landing page, `/`).
- The VIP lounge has a bouncer at the door (protected route, `/product`).
- If you're not on the list (not signed in), the bouncer sends you to the front desk (sign-in page).
- Once you have a wristband (token), the bouncer lets you through.

But — the bouncer is just for show. The real security is the locked door behind the bar (backend validation). Someone with a fake wristband gets past the bouncer but not the locked door.

## How does it actually work?

### With Clerk in Next.js (Pages Router)

```tsx
// pages/_app.tsx — wrap everything in ClerkProvider
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
// pages/index.tsx — public landing page
import { SignInButton, SignedIn, SignedOut } from "@clerk/nextjs";

export default function Home() {
  return (
    <>
      <SignedOut>
        <h1>Generate your next big business idea</h1>
        <SignInButton><button>Get Started Free</button></SignInButton>
      </SignedOut>
      <SignedIn>
        <p>Welcome back!</p>
        <Link href="/product">Go to app →</Link>
      </SignedIn>
    </>
  );
}
```

```tsx
// pages/product.tsx — protected page
import { useUser } from "@clerk/nextjs";
import { useRouter } from "next/router";
import { useEffect } from "react";

export default function ProductPage() {
  const { isLoaded, isSignedIn, user } = useUser();
  const router = useRouter();

  useEffect(() => {
    if (isLoaded && !isSignedIn) {
      router.push("/");  // Redirect to landing if not signed in
    }
  }, [isLoaded, isSignedIn]);

  if (!isSignedIn) return <p>Redirecting...</p>;

  return (
    <div>
      <h1>Business Idea Generator</h1>
      {/* App content here */}
    </div>
  );
}
```

Clerk's `<SignedIn>` and `<SignedOut>` components handle conditional rendering automatically based on auth state. For route-level protection, you add a redirect check.

### Clerk's built-in middleware (App Router)

```tsx
// middleware.ts — protects all routes except public ones
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isProtectedRoute = createRouteMatcher(["/product(.*)", "/dashboard(.*)"]);

export default clerkMiddleware((auth, req) => {
  if (isProtectedRoute(req)) auth().protect();
});
```

## Example: conditional UI based on auth state

```tsx
import { SignedIn, SignedOut, UserButton } from "@clerk/nextjs";

function Header() {
  return (
    <nav>
      <SignedOut>
        <SignInButton mode="modal">
          <button>Sign In</button>
        </SignInButton>
      </SignedOut>
      <SignedIn>
        <UserButton />  {/* Avatar + dropdown menu */}
        <Link href="/product">My App</Link>
      </SignedIn>
    </nav>
  );
}
```

Clerk automatically determines which branch to render. You never manually check `isSignedIn` for basic UI toggling — the components handle it.

## Common misconceptions

- **Misconception:** Protected routes are security.
  **Reality:** They're UX. A user can bypass frontend route protection by directly calling your API. The backend must independently validate the token. Frontend protection improves UX; backend validation provides security.

- **Misconception:** You need to manually check auth state on every page.
  **Reality:** Clerk's middleware (App Router) or `<SignedIn>`/`<SignedOut>` components handle this declaratively. You wrap routes, not check tokens.

- **Misconception:** Redirecting to sign-in is enough.
  **Reality:** You must also handle the "signed in but loading" state. If you redirect before Clerk finishes loading auth state, users get stuck in a redirect loop.

## What abstractions / AI tools often hide

When AI generates "just add `<SignedIn>`," it hides:
- **The loading state** — Clerk needs time to fetch auth state on page load. During this window, `isSignedIn` is `undefined`. Not handling this causes flash-of-wrong-content or redirect loops.
- **The token in the request** — protected routes show/hide UI, but the API call from the protected page still needs to include the JWT. Clerk handles this via cookies automatically, but if you're using `fetch()` directly, you must ensure cookies are sent (`credentials: "include"`).
- **Server-side vs client-side protection** — in App Router, middleware can protect routes on the server (before HTML is sent). In Pages Router, protection is client-side (React checks auth state). The difference matters for SEO and initial load security.

## Practical engineering implications

- **Protect routes on both layers.** Frontend: redirect to sign-in (UX). Backend: validate JWT (security). Both are needed.
- **Handle the loading state.** Always check `isLoaded` before `isSignedIn`. Show a loading spinner, not a redirect, while auth state is loading.
- **Use Clerk's middleware for App Router.** It protects routes at the server level — users never even receive the HTML for protected pages if not authenticated.
- **Separate public and protected pages clearly.** Landing page (`/`) is public. App pages (`/product`, `/dashboard`) are protected. This mental model keeps your routing clean.

## Related topics

- [Auth-as-a-Service](../backend/auth-as-a-service.md) — Clerk, the provider
- [JWT (JSON Web Tokens)](../backend/jwt-json-web-tokens.md) — the token that proves identity
- [Backend Auth Verification](../backend/backend-auth-verification.md) — the real security layer
- [Context Provider Pattern](context-provider-pattern.md) — how `ClerkProvider` shares auth state

## References

- [Clerk: Protecting Routes (Next.js)](https://clerk.com/docs/references/nextjs/protect-routes)
- [Clererk: SignedIn / SignedOut components](https://clerk.com/docs/references/react/signed-in)
- [Next.js Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
