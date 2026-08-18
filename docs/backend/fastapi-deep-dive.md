# FastAPI Deep Dive

> A modern Python backend framework built on Starlette (ASGI) and Pydantic (validation), optimized for building typed, async API endpoints with automatic documentation.

## What is it?

FastAPI is a Python web framework that uses Python's type hints as the source of truth for:
1. **Request validation** — incoming JSON is validated against Pydantic models
2. **Response serialization** — outgoing data is typed and serialized
3. **Documentation** — OpenAPI/Swagger docs are auto-generated from type annotations
4. **Editor support** — IDE autocomplete works because types are real

It's built on **Starlette** (an ASGI framework for async web servers) and **Pydantic** (a data validation library). This combination gives you async performance with type safety.

## Why does it matter?

Traditional Python frameworks (Django, Flask) are synchronous — one request blocks one worker. For I/O-heavy workloads (database queries, API calls, LLM calls), this wastes resources. FastAPI's async model handles thousands of concurrent I/O operations on a single worker.

For AI applications specifically, FastAPI is the de facto standard: LLM calls are slow (seconds), and async lets one server handle many concurrent requests while waiting for LLM responses.

## Mental model

FastAPI = **Starlette (async web) + Pydantic (types) + dependency injection + auto-docs**. You define routes with type-hinted parameters; FastAPI validates, routes, documents, and executes them asynchronously.

## How does it actually work?

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel

app = FastAPI()

# 1. Pydantic model = schema + validation
class ProductCreate(BaseModel):
    name: str
    price: float
    stock: int = 0  # default value

# 2. Route with type hints
@app.post("/products")
async def create_product(product: ProductCreate):
    # 'product' is already validated — no manual checking
    # If invalid, FastAPI returns 422 with error details
    return {"created": product}

# 3. Dependency injection
async def get_db():
    db = DatabaseSession()
    try:
        yield db
    finally:
        db.close()

@app.get("/products/{product_id}")
async def get_product(product_id: int, db = Depends(get_db)):
    # 'product_id' is validated as int (string → 422 if not)
    # 'db' is injected by FastAPI
    product = await db.get(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Not found")
    return product
```

When you run this, FastAPI automatically:
- Validates `product_id` is an integer (returns 422 if "abc")
- Validates `product` body matches `ProductCreate` (returns 422 if missing `name`)
- Generates interactive docs at `/docs` (Swagger UI)
- Generates machine-readable spec at `/openapi.json`

## Example: async LLM endpoint

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    model: str = "gpt-4"

@app.post("/chat")
async def chat(request: ChatRequest):
    # Async: other requests aren't blocked while we wait
    response = await call_llm(request.model, request.message)
    return {"reply": response}
```

This is why FastAPI dominates AI backends: the `await call_llm()` doesn't block the server — other users' requests are served concurrently.

## Common misconceptions

- **Misconception:** FastAPI is async so everything is fast.
  **Reality:** Async only helps with I/O-bound work (network, disk, DB). CPU-bound work (image processing, ML inference) still blocks. For CPU work, use `run_in_executor` or a task queue.

- **Misconception:** Pydantic is just for API models.
  **Reality:** Pydantic validates any data — config files, environment variables, internal data structures. It's a general validation library.

- **Misconception:** Auto-docs replace writing docs.
  **Reality:** OpenAPI docs show the *shape* of the API. They don't explain *why* or *how to use* it. Human-written docs are still needed.

## What abstractions / AI tools often hide

When AI generates FastAPI code, it hides:
- **The async/sync boundary** — calling a synchronous library (e.g., `requests.get()` instead of `httpx.AsyncClient()`) inside an async route blocks the event loop. This is the #1 FastAPI performance bug. AI tools frequently generate sync calls in async routes.
- **Pydantic v1 vs v2** — Pydantic v2 (2023) is a complete rewrite with different APIs. AI tools may generate v1 syntax that doesn't work in v2.
- **Dependency injection complexity** — `Depends` is powerful but subtle: scoped lifetimes, yield-based cleanup, nested dependencies. AI-generated DI code often leaks resources.
- **Background tasks vs async** — `BackgroundTasks` runs after the response, not concurrently. It's not a replacement for a real task queue (Celery, RQ).

## Practical engineering implications

- **Use `async def` only when you `await` inside.** If your route is purely synchronous (no `await`), declare it as `def` — FastAPI runs it in a threadpool, which is faster than faking async.
- **Always use async HTTP clients.** `httpx.AsyncClient` not `requests`. `asyncpg` not `psycopg2` (unless using psycopg3 async).
- **Leverage Pydantic for everything.** Settings (`BaseSettings`), config, env vars. Don't hand-validate.
- **Version your API from day one.** `/api/v1/products` not `/products`. FastAPI `APIRouter` makes this clean.
- **Test with `TestClient`.** FastAPI provides `fastapi.testclient.TestClient` — no need for a running server.

## Related topics

- [Python Backend Frameworks](python-backend-frameworks.md) — comparison with Django and Flask
- [React Components](../programming/react-components.md) — what calls these APIs
- [GitHub Actions CI/CD](../software-engineering/github-actions-cicd.md) — deploying FastAPI

## References

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Starlette Documentation](https://www.starlette.io/)
- [FastAPI + LLM tutorial](https://fastapi.tiangolo.com/tutorials/)
