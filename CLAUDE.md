# CLAUDE.md — Golden Template Build Rules

This file defines the architecture, invariants, and anti-patterns for apps built from this golden template. Claude MUST follow these rules when transforming or extending this codebase.

## What This Repo Is

A FastAPI SaaS application with:
- **viv-auth** — Magic link authentication (session cookie `viv_session`)
- **viv-pay** — Stripe billing (checkout, webhooks, subscription gating)
- **PostgreSQL** — Cloud SQL via `DATABASE_URL` env var
- **Gemini AI** — via `google-genai` SDK + Vertex AI ADC
- **Jinja2 templates** — Server-rendered HTML with embedded CSS
- **Coolify deployment** — Docker container on port 8000

## Fork-Transform Rules

When transforming this app into a new application:

1. **PRESERVE** the following files and patterns exactly — do NOT rewrite from scratch:
   - `Dockerfile` — keep as-is, only change if adding system packages
   - `requirements.txt` — keep all `git+https://` URLs exactly as-is, add new deps at the end
   - `app/main.py` — keep auth/pay initialization pattern, change app-specific routers
   - `app/routes/__init__.py` — keep dependency injection pattern
   - `app/database.py` — keep as-is
   - `app/routes/billing.py` — keep as-is (Stripe billing is universal)

2. **REPLACE** domain-specific content:
   - `app/models.py` — new SQLAlchemy models for the target domain
   - `app/routes/` — new route files for the target domain (keep billing.py)
   - `app/templates/` — new HTML templates for the target domain
   - `app/seed.py` — new seed data for the target domain
   - `app/routes/ai.py` — adapt AI features for the new domain

3. **UPDATE** these in `app/main.py`:
   - Router imports and `include_router()` calls for new route modules
   - `app_name=` parameter in `init_auth()` and `init_pay()` calls
   - Root redirect URL (`/` should redirect to the app's main page)

4. **UPDATE** the pricing page template (`templates/pricing.html`):
   - Replace app name, feature list, and price to match the new app
   - Do NOT leave the golden template's "AI Notes" pricing copy

5. **READ each existing file before modifying.** Understand the architecture first.

## Architecture Invariants

### Import Pattern (CRITICAL)
```python
import app.routes as routes_module
```
NOT `import app.routes` or `from app import routes` — FastAPI shadows the module name.

### Initialization Order
```python
User, require_auth = init_auth(app, engine, Base, get_db, app_name="...")
create_checkout, get_customer, require_subscription = init_pay(app, engine, Base, get_db, app_name="...")
```
`init_auth()` MUST be called BEFORE `init_pay()`. Pay depends on auth tables.

### Dependency Injection Pattern
Routes use placeholder functions in `app/routes/__init__.py`:
```python
def get_current_user():
    pass

def get_active_subscription():
    pass
```
`main.py` overrides these via `app.dependency_overrides[routes_module.get_current_user] = require_auth`.

All route files use `Depends(routes.get_current_user)` — NEVER `Depends(routes.require_auth)`.

### Auth-Billing Bridge
```python
async def require_active_subscription(request: Request, user=Depends(require_auth)):
    return await require_subscription(request, user_id=user.id)
```
viv-auth uses encrypted session cookies, so `require_subscription` can't find `user_id` on its own.

### User ID Type
`user.id` is a string (Clerk ID format). When storing in database columns or comparing:
```python
user_id=str(user.id)
```
Always use `str(user.id)` in ALL route files. The `user_id` column in app models should be `String`, not `Integer`.

### ForeignKey Relationships
For EVERY `ForeignKey` column, define a SQLAlchemy `relationship()` on BOTH sides:
```python
class Parent(Base):
    children = relationship("Child", back_populates="parent")

class Child(Base):
    parent_id = Column(Integer, ForeignKey("parents.id"))
    parent = relationship("Parent", back_populates="children")
```
Templates access related objects via dot notation (e.g. `item.category.name`). Without `relationship()`, this crashes at runtime with `UndefinedError`.

### Static Files Directory
`app/static/` directory MUST exist (create with `.gitkeep` if empty) for the `StaticFiles` mount in `main.py`. Without it, the app crashes on startup.

### Stripe Price ID
ALWAYS read from environment:
```python
price_id = os.environ.get("STRIPE_PRICE_ID")
```
NEVER hardcode a Stripe price ID in source code.

## Infrastructure Rules

### Port
Python apps MUST listen on **port 8000**. Never 3000, 5000, 8080, or 8081.

### Health Endpoint
`GET /health` MUST return `{"status": "ok"}` with HTTP 200.
It MUST be the FIRST route defined in `main.py` (before any other routes).

### Dockerfile
- Base image: `python:3.11-slim-bookworm`
- NEVER add a `HEALTHCHECK` instruction — infrastructure polls `/health` externally via HTTP. Docker HEALTHCHECK uses `curl` which is not installed in slim images, causing containers to be killed as "unhealthy".
- `git` package is required for `pip install git+https://` dependencies.
- CMD: `uvicorn app.main:app --host 0.0.0.0 --port 8000`

### Templates
- ALL CSS MUST be embedded in `<style>` tags in `templates/base.html`
- Do NOT create separate CSS files or a `static/css/` directory
- All templates extend `base.html`: `{% extends "base.html" %}`
- Do NOT use `get_flashed_messages()` — that is Flask-only. Pass messages via template context.

### Database
- `DATABASE_URL` is read from environment in `app/database.py`
- PostgreSQL in production (Cloud SQL), SQLite fallback for local dev
- `Base.metadata.create_all()` runs on startup
- Seed data function runs on startup, checks if tables are empty before inserting
- Use realistic seed data (not "test1", "test2")

## Dependency Rules

### requirements.txt
Keep these lines EXACTLY as they are (pinned working versions):
```
git+https://github.com/ooda-AI-GB/viv-auth.git
git+https://github.com/ooda-AI-GB/viv-pay.git@854f785
```

### Never Use These Packages
| Package | Why | Use Instead |
|---------|-----|-------------|
| `resend` (explicit pin) | Transitive dep of viv-auth. Pinning causes version conflicts. | Don't add it at all |
| `passlib` | Broken with modern bcrypt versions | `bcrypt` directly |
| `xhtml2pdf` | Heavy, breaks in Docker | `reportlab` |
| `weasyprint` | Requires system libs not in slim images | `reportlab` |
| `gitpython` | Gemini corruption artifact. Never needed. | Native `git` CLI |
| `tensorflow` | 2GB+ download, timeout | Smaller ML libs |
| `torch` / `pytorch` | 2GB+ download, timeout | Smaller ML libs |
| `gunicorn` | Not needed for single-container deploys | `uvicorn` directly |

### Correct Package Names
If you need these libraries, use the correct pip package name:
| Import | Pip Package |
|--------|-------------|
| `sklearn` | `scikit-learn` |
| `cv2` | `opencv-python-headless` |
| `yaml` | `pyyaml` |
| `PIL` | `Pillow` |
| `dotenv` | `python-dotenv` |
| `bs4` | `beautifulsoup4` |
| `dateutil` | `python-dateutil` |

### Pinned Versions (known working)
```
fastapi==0.115.0
uvicorn[standard]==0.34.0
jinja2==3.1.4
python-multipart==0.0.18
sqlalchemy==2.0.36
psycopg2-binary==2.9.9
google-genai==1.62.0
httpx==0.28.1
bcrypt==4.2.1
python-dateutil==2.9.0
itsdangerous==2.2.0
reportlab==4.2.5
cloudinary==1.41.0
```

## Self-Verification Checklist

Before finishing, verify ALL of these:

1. `Dockerfile` exists, has NO `HEALTHCHECK`, uses `python:3.11-slim-bookworm`, exposes port 8000
2. `requirements.txt` exists, contains both `git+https://` lines for viv-auth and viv-pay unchanged
3. `app/main.py` has `/health` as first route, calls `init_auth()` before `init_pay()`
4. `app/main.py` uses `import app.routes as routes_module` pattern
5. `app/routes/__init__.py` has `get_current_user()` and `get_active_subscription()` placeholders
6. `app/routes/billing.py` exists and handles Stripe checkout/webhooks
7. `app/static/` directory exists (with `.gitkeep` if empty)
8. All route files use `str(user.id)` when storing user IDs
9. All ForeignKey columns have `relationship()` on both sides
10. No `HEALTHCHECK` in Dockerfile
11. No `passlib`, `resend`, `gitpython`, or `gunicorn` in requirements.txt
12. CSS is embedded in `base.html`, not in separate files
13. `STRIPE_PRICE_ID` read from `os.environ.get()`, never hardcoded
14. All templates extend `base.html`
15. Seed data is realistic and checks for empty tables before inserting

## Environment Variables (set by deployment, not in code)

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string |
| `SESSION_SECRET` | Cookie encryption key |
| `RESEND_API_KEY` | Email sending (magic links) |
| `FROM_EMAIL` | Sender address for auth emails |
| `STRIPE_SECRET_KEY` | Stripe API key |
| `STRIPE_PRICE_ID` | Monthly subscription price |
| `STRIPE_WEBHOOK_SECRET` | Webhook signature verification |
| `APP_URL` | Public URL (for Stripe redirects) |
| `DEV_AUTH_BYPASS` | Skip auth in dev (`true`/`false`) |
| `GOOGLE_CLOUD_PROJECT` | GCP project for Vertex AI |
| `GOOGLE_CLOUD_LOCATION` | GCP region for Vertex AI |
