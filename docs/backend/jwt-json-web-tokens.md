# JWT (JSON Web Tokens)

> A compact, self-contained token format that lets your backend verify a user's identity without calling the auth provider on every request.

## What is it?

A **JWT** (JSON Web Token) is a standardized way to encode claims (user identity, expiration, scope) into a signed string. It has three parts — header, payload, signature — separated by dots: `eyJhbG...eyJzdW...SflKx...`.

Auth providers like Clerk issue a JWT after sign-in. The client sends it with each API request. The backend verifies the JWT's signature using the provider's public keys — **without calling the provider's API**.

## Why does it matter?

Without JWT, every API request would need a round-trip to the auth provider ("is this session valid?"). That's slow and creates a dependency. JWT lets the backend **verify locally** — the signature proves the token was issued by Clerk, without calling Clerk.

- **Stateless:** backend doesn't need a session store. The token itself carries the identity.
- **Verifiable locally:** public-key cryptography lets you trust the token without network calls.
- **Compact:** designed to travel in cookies, headers, or URL parameters.

## Mental model

Think of a JWT as a **passport**:
- Issued by a trusted authority (Clerk = issuing country).
- Contains identity claims (user ID, expiration = photo, nationality).
- Verified by anyone with the public key (your backend = border control).
- Border control doesn't call the issuing country to verify — they check the hologram (signature).

## How does it actually work?

### Token structure

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImV4cCI6MTcwMH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
│          header          │       payload           │     signature     │
```

- **Header:** algorithm + token type (`{"alg": "HS256", "typ": "JWT"}`)
- **Payload:** claims — user ID, expiration, issuer (`{"sub": "user_123", "exp": 1700, "iss": "clerk"}`)
- **Signature:** `HMAC(header + payload, secret_key)` — proves it wasn't tampered with

### Validation flow

```
Client sends JWT in Authorization: Bearer <token> header
    ↓
Backend fetches provider's public keys (JWKS URL) — cached!
    ↓
Verify signature: recompute HMAC with public key, compare
    ↓
Check claims: not expired? correct issuer?
    ↓
Extract user ID → process request
```

### In a FastAPI app

```python
from fastapi_clerk_auth import GetClerkUser, ClerkConfig

# Configure with Clerk's JWKS URL
clerk_config = ClerkConfig(jwks_url="https://your-app.clerk.accounts.dev/.well-known/jwks.json")

@app.post("/api/generate")
async def generate(user = Depends(GetClerkUser)):
    # Token already validated — user is authenticated
    user_id = user.user_id  # Extracted from JWT payload
    return {"result": generate_idea(user_id)}
```

The `fastapi-clerk-auth` library handles JWKS fetching, signature verification, and claim checking. If the token is invalid or missing, the route returns 401 automatically.

## Example: what's in a JWT

```json
// Header (base64-decoded)
{"alg": "RS256", "typ": "JWT", "kid": "key-123"}

// Payload (base64-decoded)
{
  "sub": "user_2AbCdEf",     // user ID
  "iss": "https://saas.clerk.accounts.dev",
  "exp": 1700000000,             // expiration timestamp
  "iat": 1699900000,           // issued at
  "email": "user@example.com"
}
```

Anyone can read the payload (it's just base64, not encrypted). The **signature** is what proves authenticity — only Clerk can create valid tokens, but anyone with the public key can verify them.

## Common misconceptions

- **Misconception:** JWTs are encrypted.
  **Reality:** JWTs are *signed*, not encrypted. The payload is base64-encoded — anyone can read it. Never put secrets in a JWT. The signature prevents *tampering*, not *reading*.

- **Misconception:** JWTs are always stateless.
  **Reality:** In practice, many systems maintain a revocation list (for logout, password changes). This reintroduces some server-side state. Pure stateless JWTs can't be revoked before expiry.

- **Misconception:** You should always use JWT.
  **Reality:** JWT shines for stateless, cross-service auth. For a simple server-rendered app, server-side sessions may be simpler and more secure.

## What abstractions / AI tools often hide

When AI says "Clerk uses JWT," it hides:
- **The JWKS caching layer** — your backend fetches Clerk's public keys once and caches them. If the cache is misconfigured (stale keys), token validation breaks silently. Understanding this cache is critical for debugging.
- **Token expiration and refresh** — JWTs expire (often in minutes). The frontend must silently refresh them. Clerk handles this via cookies, but if you're building custom flows, you manage refresh yourself.
- **The `kid` (key ID)** — the JWT header specifies which key signed it. Your backend must support key rotation — Clerk rotates keys periodically. If your JWKS cache is stale, all auth breaks.
- **Cross-origin cookies** — if your frontend (Vercel) and backend (separate API) are on different domains, cookies may be blocked by CORS/SameSite policies. This is a common, painful debugging session.

## Practical engineering implications

- **Never put sensitive data in the payload.** It's base64, not encrypted. User ID and roles are fine; passwords and secrets are not.
- **Cache JWKS keys.** Fetching them on every request kills performance. Cache with a TTL and refresh on key rotation.
- **Set short expiration times.** A stolen JWT is valid until it expires. Short-lived tokens (minutes) + refresh tokens are safer than long-lived tokens.
- **Use HTTPS.** JWTs in cookies over HTTP can be intercepted. `Secure` + `HttpOnly` cookie flags are mandatory.
- **Validate on the backend, always.** Frontend auth state is UX. The JWT must be verified on every protected API request.

## Related topics

- [Auth-as-a-Service](auth-as-a-service.md) — providers like Clerk that issue JWTs
- [Backend Auth Verification](backend-auth-verification.md) — how FastAPI validates JWTs
- [Protected Routes](protected-routes.md) — frontend route guarding

## References

- [JWT.io — Interactive decoder](https://jwt.io)
- [RFC 7519: JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [Clerk: JWT Templates](https://clerk.com/docs/backend-requests/jwt-templates)
- [JWKS Explained](https://auth0.com/docs/secure/tokens/json-web-keys)
