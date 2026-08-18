# Backend Auth Verification

> The real security layer — validating the JWT on every API request, independently of what the frontend says.

## What is it?

**Backend auth verification** is the process of checking that every incoming API request includes a valid, unexpired JWT token issued by your auth provider (Clerk). If the token is missing, expired, or tampered with, the backend rejects the request with a 401 Unauthorized.

This is distinct from frontend route protection: the frontend hides UI, but a malicious user can bypass the frontend and call your API directly. The backend verification is the actual security boundary.

## Why does it matter?

Frontend auth is **UX** — it improves the user experience. Backend auth is **security** — it prevents unauthorized access.

Without backend verification:
- Anyone can call your API endpoints directly (e.g., `curl https://your-api.com/generate`).
- No rate limiting per user — anonymous abuse.
- No way to identify users for personalization or billing.
- No way to check subscription plans for feature gating.

Backend verification ensures that even if someone bypasses the frontend entirely, they still can't access protected resources without a valid token.

## Mental model

Think of a **bank vault**:
- The lobby has a greeter who asks "are you a customer?" (frontend protection).
- The vault has a second guard who checks your ID and vault key (backend verification).

A smooth-talker might get past the greeter, but the vault guard checks the actual credentials. The greeter is courtesy; the vault guard is security.

## How does it actually work?

### In FastAPI with Clerk

```python
# requirements.txt
fastapi-clerk-auth  # Backend library for Clerk JWT validation

# index.py (backend)
from fastapi import FastAPI, Depends
from fastapi_clerk_auth import GetClerkUser, ClerkConfig

app = FastAPI()

# Configure with Clerk's JWKS URL (from Clerk dashboard → API keys)
clerk_config = ClerkConfig(
    jwks_url="https://your-app.clerk.accounts.dev/.well-known/jwks.json"
)

@app.post("/api/generate")
async def generate_idea(user = Depends(GetClerkUser)):
    # If we reach here, the JWT was valid.
    # If not, FastAPI returned 401 before this code runs.

    user_id = user.user_id  # Extract identity from the token

    # You can also check subscription plans:
    # if user.subscription_plan == "free":
    #     raise HTTPException(403, "Upgrade to Pro")

    return {"result": generate(user_id)}
```

### What `Depends(GetClerkUser)` does:

```
Request arrives → Authorization: Bearer eyJhb...
    ↓
Extract JWT from header or cookie
    ↓
Fetch Clerk's public keys (JWKS) — cached after first fetch
    ↓
Verify signature: HMAC(header + payload, public_key) == signature?
    ↓
Check claims: exp not expired? iss = clerk?
    ↓
✓ Valid → inject user object, run route
✗ Invalid → return 401 Unauthorized
```

The key insight: **your backend never calls Clerk's API**. It validates the JWT locally using cached public keys. This is fast and doesn't create a dependency on Clerk's availability.

## Example: extracting user identity

```python
@app.get("/api/profile")
async def get_profile(user = Depends(GetClerkUser)):
    user_id = user.user_id      # "user_2AbCdEf"
    email = user.email          # "user@example.com"
    name = user.name            # "John Doe"

    # Fetch user-specific data from your database
    profile = await db.get_profile(user_id)
    return profile

@app.post("/api/save")
async def save_data(data: MyData, user = Depends(GetClerkUser)):
    user_id = user.user_id
    await db.save(user_id, data)  # Data is scoped to this user
    return {"status": "saved"}
```

## Common misconceptions

- **Misconception:** If the frontend is protected, the backend is safe.
  **Reality:** The frontend can be bypassed entirely. `curl -X POST https://your-api.com/generate` works without a browser. Backend verification is the only real security.

- **Misconception:** You should call Clerk's API to validate each token.
  **Reality:** That would be slow and create a dependency. JWTs are designed for local verification via public keys (JWKS). Your backend fetches the keys once, caches them, and validates locally.

- **Misconception:** Once the user is authenticated, you don't need to check again.
  **Reality:** Every request must be verified. Tokens expire, users log out, sessions get revoked. Each API call is independent — there's no persistent connection.

## What abstractions / AI tools often hide

When AI generates `Depends(GetClerkUser)`, it hides:
- **The JWKS caching layer** — the backend fetches Clerk's public keys and caches them. If the cache is stale (key rotation), validation breaks. The `fastapi-clerk-auth` library handles this, but understanding it matters for debugging "why did auth suddenly stop working?" (answer: JWKS cache stale, keys rotated).
- **Environment variables on the backend** — the backend needs `CLERK_JWT_URL` (different from the frontend keys). Missing this causes 500 errors.
- **Token in cookies vs headers** — Clerk's Next.js SDK puts the JWT in an httpOnly cookie. But if frontend and backend are on different domains (e.g., Vercel frontend + Fly.io backend), cookies don't cross domains automatically. You may need to manually forward the token in the `Authorization` header.
- **Subscription/role checks** — `GetClerkUser` verifies identity but not permissions. Checking subscription plans (free vs pro) or admin roles requires additional logic in your route. AI tools often skip this, leaving paid features accessible to free users.

## Practical engineering implications

- **Protect every sensitive endpoint.** Use `Depends(GetClerkUser)` (or equivalent) on every route that touches user data. It's easy to forget one.
- **Extract user identity for personalization.** `user.user_id` lets you scope database queries to the current user. Never trust a user ID sent in the request body — always extract from the token.
- **Check permissions, not just identity.** Authenticated ≠ authorized. If you have subscription tiers, add a second check: `if user.plan != "pro": raise 403`.
- **Handle 401 gracefully on the frontend.** When the backend rejects a token (expired), the frontend should redirect to sign-in, not show a blank error.
- **Forward tokens correctly across domains.** If frontend and backend are on different domains, ensure the JWT is sent in the `Authorization` header, not just cookies.

## Related topics

- [JWT (JSON Web Tokens)](../backend/jwt-json-web-tokens.md) — what the backend validates
- [Auth-as-a-Service](../backend/auth-as-a-service.md) — Clerk as the provider
- [Protected Routes](protected-routes.md) — the frontend layer (UX)
- [Environment Variables & Secrets](environment-variables-and-secrets.md) — where the JWKS URL lives

## References

- [fastapi-clerk-auth on PyPI](https://pypi.org/project/fastapi-clerk-auth/)
- [Clerk: Backend JWT Validation](https://clerk.com/docs/backend-requests/overview)
- [Clerk: JWKS Endpoint](https://clerk.com/docs/backend-requests/jwt-templates)
- [FastAPI: Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
