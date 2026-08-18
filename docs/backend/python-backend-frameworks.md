# Python Backend Frameworks: Django vs Flask vs FastAPI

> Three frameworks, three philosophies: batteries-included (Django), micro (Flask), and modern async (FastAPI).

## What is it?

Python has three dominant web backend frameworks, each representing a different philosophy:

- **Django** — "Batteries included." Comes with an ORM, authentication, admin panel, templating, migrations — everything a full web app needs. Opinionated and heavy.
- **Flask** — "Micro framework." Provides only the essentials: routing and request/response handling. You add everything else (ORM, auth, forms) by choosing and integrating libraries yourself.
- **FastAPI** — "Modern, async, typed." Built on Starlette (ASGI server) and Pydantic (data validation). Optimized for building API endpoints with automatic type checking and documentation. The newest, fastest-growing.

## Why does it matter?

Your framework choice shapes your entire backend architecture: how you define routes, how you validate data, how you handle concurrency, what you get for free vs. what you build yourself. It's the most consequential backend decision and one of the hardest to reverse.

## Mental model

| Framework | Philosophy | Analogy |
|---|---|---|
| Django | Batteries included | A fully equipped kitchen — stove, fridge, utensils, recipes |
| Flask | Micro framework | An empty kitchen — you buy each appliance yourself |
| FastAPI | Modern, async, typed | A smart kitchen — modern appliances, automated, efficient |

## How does it actually work?

### Django — everything included

```python
# models.py — built-in ORM
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

# views.py — request handler
from django.shortcuts import render
from .models import Product

def product_list(request):
    products = Product.objects.all()
    return render(request, "products/list.html", {"products": products})

# urls.py — routing
from django.urls import path
from . import views

urlpatterns = [
    path("products/", views.product_list),
]
```

Django gives you: ORM, admin panel (`/admin/`), auth, sessions, templates, migrations, CSRF protection — all built-in.

### Flask — minimal, you assemble

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

# A simple route — that's it. No ORM, no auth, no admin.
@app.route("/products", methods=["GET"])
def get_products():
    # You'd need SQLAlchemy for DB, Flask-Login for auth,
    # Flask-WTF for forms — all added separately
    return jsonify({"products": []})

if __name__ == "__main__":
    app.run(debug=True)
```

Flask gives you: routing, request/response. Everything else is an add-on you choose.

### FastAPI — typed, async, documented

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# Pydantic model = automatic validation + OpenAPI docs
class Product(BaseModel):
    name: str
    price: float

@app.post("/products")
async def create_product(product: Product):
    # 'product' is already validated and typed
    # OpenAPI docs auto-generated at /docs
    return {"created": product}
```

FastAPI gives you: async routing, Pydantic validation, auto-generated OpenAPI/Swagger docs, dependency injection.

## Common misconceptions

- **Misconception:** Flask is too simple for production.
  **Reality:** Flask powers major production apps (Pinterest, LinkedIn used it). Its minimalism means you build exactly what you need. The "micro" refers to the core, not to production-readiness.

- **Misconception:** Django is slow.
  **Reality:** Django is synchronous and not optimized for high-concurrency APIs, but for most CRUD apps it's fast enough. The bottleneck is usually the database, not the framework.

- **Misconception:** FastAPI is only for APIs.
  **Reality:** FastAPI can serve templates and HTML, but its sweet spot is indeed API endpoints. For a full-stack app with admin panels, Django is more productive.

## What abstractions / AI tools often hide

When AI generates backend code, it hides:
- **The sync vs async boundary** — Django and Flask are synchronous (one request per thread). FastAPI is async (thousands of concurrent requests on one thread). Mixing sync DB calls in async code blocks the event loop — a common AI-generated bug.
- **ORM choice implications** — Django's ORM is tightly coupled to Django. SQLAlchemy (used with Flask/FastAPI) is more powerful but requires more setup. Switching ORMs is a rewrite.
- **Deployment differences** — Django deploys as WSGI (gunicorn/uwsgi). FastAPI deploys as ASGI (uvicorn). The server choice matters for performance.

## Practical engineering implications

- **Content-heavy site (admin, CMS, blog)? → Django.** The admin panel and ORM save weeks.
- **Simple API or microservice? → Flask or FastAPI.** Don't need Django's weight.
- **High-concurrency API (AI, real-time, streaming)? → FastAPI.** Async is the reason.
- **Type safety and auto-docs? → FastAPI + Pydantic.** The OpenAPI docs are generated from your type annotations — no manual Swagger files.
- **Team familiar with Django? → Django.** Framework familiarity beats theoretical optimization.

## Related topics

- [FastAPI Deep Dive](fastapi-deep-dive.md) — the framework we use in detail
- [React Components](../programming/react-components.md) — what the frontend talks to

## References

- [Django Documentation](https://docs.djangoproject.com/)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
