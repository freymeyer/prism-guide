# Chapter 7 — Configuration & Deployment

> This chapter explains how PRISM-Matutum is configured, how Docker containers are set up, what environment variables control behavior, and how to deploy the system to production.

---

## Table of Contents

- [7.1 Settings System](#71-settings-system)
- [7.2 Docker Architecture](#72-docker-architecture)
- [7.3 Python Dependencies](#73-python-dependencies)
- [7.4 Celery & Task Queues](#74-celery--task-queues)
- [7.5 URL Routing (Top Level)](#75-url-routing-top-level)
- [7.6 Environment Variables Reference](#76-environment-variables-reference)
- [7.7 Management Commands](#77-management-commands)
- [7.8 Development Workflow](#78-development-workflow)
- [7.9 Production Deployment](#79-production-deployment)

---

## 7.1 Settings System

### What Are Django Settings?

In Django, `settings.py` is a Python file containing all configuration for your project — database credentials, installed apps, middleware, time zones, and more. Django reads these on startup and uses them everywhere.

### PRISM-Matutum Settings Architecture

This project has **two settings systems** (a flat and a modular approach):

```
prism/
├── settings.py              ← Flat development settings (used by Docker dev)
├── settings_production.py   ← Production overlay (imports settings.py, overrides)
│
└── settings/                ← Modular settings (environment-based routing)
    ├── __init__.py          ← Reads ENVIRONMENT env var → loads correct module
    ├── base.py              ← Shared foundation settings
    ├── development.py       ← Dev overrides (DEBUG=True, eager Celery, etc.)
    └── production.py        ← Prod overrides (security, Sentry, retention policies)
```

> **Which one is active?** Docker uses `prism.settings` (dev) and `prism.settings_production` (prod). The modular `prism.settings` package approach would be activated by renaming/removing the flat file and setting `ENVIRONMENT=development` or `ENVIRONMENT=production`.

### Key Settings Explained

| Setting | Dev Value | Prod Value | What It Does |
|---------|-----------|------------|--------------|
| `DEBUG` | `True` | `False` | Shows detailed error pages (NEVER True in production!) |
| `SECRET_KEY` | Hardcoded | From env var | Cryptographic signing key for sessions/CSRF |
| `ALLOWED_HOSTS` | `localhost`, `*.ngrok-free.app` | From env var | Which domains can access the site |
| `AUTH_USER_MODEL` | `users.User` | Same | Custom user model (see Chapter 3) |
| `TIME_ZONE` | `Asia/Manila` | Same | PHP/UTC+8 timezone |
| `AUTO_COMMIT_ON_APPROVAL` | `True`* | `True` | Auto-process data when approved |

### Installed Apps

```python
INSTALLED_APPS = [
    # Django built-in
    'django.contrib.admin',          # Admin interface
    'django.contrib.auth',           # Authentication
    'django.contrib.contenttypes',   # Content type framework
    'django.contrib.sessions',       # Session management
    'django.contrib.messages',       # Flash messages
    'django.contrib.staticfiles',    # Static file serving
    'django.contrib.gis',            # PostGIS/GeoDjango

    # Third-party
    'django_otp',                    # Two-factor authentication core
    'django_otp.plugins.otp_totp',   # TOTP (Google Authenticator)
    'django_otp.plugins.otp_static', # Static backup codes
    'leaflet',                       # Leaflet.js map integration
    'rest_framework',                # Django REST Framework
    'django_filters',                # Queryset filtering
    'multiselectfield',              # Multi-select model fields
    'crispy_forms',                  # Better form rendering
    'crispy_bootstrap5',             # Bootstrap 5 form templates

    # Project apps
    'species',                       # Taxonomy & species data
    'api',                           # REST API & data processing
    'users',                         # User management & auth
    'dashboard',                     # Web dashboard & UI
]
```

### Middleware Stack

Middleware wraps every HTTP request/response. Order matters:

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',    # HTTPS, HSTS
    'django.contrib.sessions.middleware.SessionMiddleware',  # Session handling
    'django.middleware.common.CommonMiddleware',        # URL normalization
    'django.middleware.csrf.CsrfViewMiddleware',       # CSRF protection
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # User auth
    'django_otp.middleware.OTPMiddleware',              # 2FA verification
    'django.contrib.messages.middleware.MessageMiddleware',     # Flash messages
    'django.middleware.clickjacking.XFrameOptionsMiddleware',  # Clickjacking protection
    'dashboard.middleware.DashboardAccessMiddleware',   # Role-based access
    'dashboard.middleware.DashboardActivityLogMiddleware',  # Activity logging
]
```

### Database Configuration

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.contrib.gis.db.backends.postgis',  # PostGIS, not plain PostgreSQL
        'NAME': 'prismdb',
        'USER': 'prismuser',
        'PASSWORD': 'prismpassword',      # Overridden by env var in production
        'HOST': os.environ.get('DATABASE_HOST', 'localhost'),
        'PORT': os.environ.get('DATABASE_PORT', '5433'),
    }
}
```

> **Why PostGIS?** PostGIS adds spatial types (`PointField`, `LineStringField`, `MultiPolygonField`) and spatial queries (`within`, `distance`, `intersects`) to PostgreSQL. Required for maps and geographic data.

### Celery Beat Schedule (Periodic Tasks)

```python
CELERY_BEAT_SCHEDULE = {
    'auto-sync-kobotoolbox': {
        'task': 'api.tasks.sync_all_kobotoolbox_data',
        'schedule': timedelta(minutes=10),           # Every 10 minutes
    },
    'check-overdue-surveys': {
        'task': 'api.tasks.check_overdue_surveys',
        'schedule': timedelta(hours=1),              # Every hour
    },
    'update-validation-statistics': {
        'task': 'dashboard.tasks.update_validation_statistics',
        'schedule': timedelta(minutes=30),            # Every 30 minutes
    },
    'sync-validation-status': {
        'task': 'api.tasks.sync_all_validation_statuses',
        'schedule': timedelta(hours=2),               # Every 2 hours
    },
    'cleanup-old-logs': {
        'task': 'dashboard.tasks.cleanup_old_logs',
        'schedule': timedelta(days=7),                # Weekly
    },
}
```

---

## 7.2 Docker Architecture

### What Is Docker Compose?

Docker Compose lets you define and run multi-container applications. Instead of installing PostgreSQL, Redis, and Python separately, you describe all services in a YAML file and start them with one command.

### Development Compose (`docker-compose.yml`)

```
┌───────────────────────────────────────────────────────┐
│                Docker Compose Network                 │
│                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐    │
│  │    db     │    │  redis   │    │    ngrok     │    │
│  │ PostGIS  │    │  7-alpine │    │   tunnel    │    │
│  │ 16-3.4   │    │          │    │   to web    │    │
│  │ :5433    │    │ :6379    │    │             │    │
│  └─────┬────┘    └────┬─────┘    └──────┬──────┘    │
│        │              │                 │            │
│        │    ┌─────────┴─────────┐       │            │
│        └────┤       web         ├───────┘            │
│             │   Django + runserver                    │
│             │   :8000                                │
│             │   (code mounted via volume)             │
│             └─────────┬─────────┘                    │
│                       │ same image                    │
│             ┌─────────┴─────────┐                    │
│             │      celery       │                    │
│             │   worker process  │                    │
│             └───────────────────┘                    │
└───────────────────────────────────────────────────────┘
```

| Service | Image | Exposed Port | Purpose |
|---------|-------|-------------|---------|
| **db** | `postgis/postgis:16-3.4` | `5433:5432` | PostgreSQL + PostGIS database |
| **redis** | `redis:7-alpine` | `6379:6379` | Message broker for Celery + cache |
| **web** | Local build | `8000:8000` | Django development server |
| **celery** | Local build | — | Background task worker |
| **ngrok** | `ngrok/ngrok:latest` | — | Public tunnel (for KoboToolbox webhooks) |

Key details:
- **`web`** mounts `.:/app` so code changes are live-reloaded
- **`web`** depends on `db` being healthy (PostgreSQL ready check)
- **`celery`** uses the same image and environment as `web`
- **`ngrok`** tunnels external requests to `web:8000` for KoboToolbox callbacks

### Production Compose (`docker-compose.prod.yml`)

Key differences from development:

| Aspect | Development | Production |
|--------|------------|------------|
| **Image** | Built locally | `freymeyer/prism-matutum:latest` (from registry) |
| **Server** | `runserver` (Django dev) | `gunicorn` (4 workers) |
| **Code** | Mounted volume (live-reload) | Baked into image (immutable) |
| **Settings** | `prism.settings` | `prism.settings_production` |
| **Secret key** | Hardcoded | `${SECRET_KEY}` env var |
| **Celery Beat** | Not separate | Dedicated `celery-beat` service |
| **ngrok** | Included | Not included |
| **DB password** | Hardcoded | `${POSTGRES_PASSWORD}` env var |

### Dockerfile

```dockerfile
FROM python:3.11-slim

# Install spatial libraries (for PostGIS)
RUN apt-get update && apt-get install -y \
    gdal-bin libgdal-dev binutils libproj-dev \
    gcc libpq-dev git

# Set spatial library paths
ENV GDAL_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/libgdal.so
ENV GEOS_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/libgeos_c.so

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .
RUN python manage.py collectstatic --noinput

EXPOSE 8000
```

---

## 7.3 Python Dependencies

### Core Framework

| Package | Version | What It Does |
|---------|---------|-------------|
| `Django` | 5.2.7 | The web framework — handles HTTP, ORM, templates, admin |
| `djangorestframework` | 3.16.1 | Adds REST API capabilities (serializers, viewsets, routers) |
| `psycopg2` | 2.9.11 | PostgreSQL database adapter — lets Python talk to PostgreSQL |
| `gunicorn` | 23.0.0 | Production web server (replaces Django's development server) |
| `whitenoise` | 6.8.2 | Serves static files efficiently in production |

### Async Task Processing

| Package | What It Does |
|---------|-------------|
| `celery` | Background task queue — runs KoboToolbox sync, report generation, etc. |
| `redis` | Python client for Redis (used as Celery message broker) |
| `django-redis` | Redis cache backend for Django |
| `kombu` | Messaging library (Celery dependency) |

### Geospatial

| Package | What It Does |
|---------|-------------|
| `django-leaflet` | Integrates Leaflet.js maps into Django templates |
| `shapely` | Python library for geometric operations (buffer, intersect, etc.) |
| `fiona` | Reads/writes geospatial file formats (GeoJSON, Shapefile) |

### Data Processing

| Package | What It Does |
|---------|-------------|
| `pandas` | DataFrame library for data manipulation and analysis |
| `numpy` | Numerical computing (pandas dependency) |
| `openpyxl` | Reads/writes Excel (.xlsx) files |
| `pillow` | Image processing (thumbnails, format conversion) |
| `weasyprint` | Converts HTML to PDF (for report generation) |

### Authentication & Security

| Package | What It Does |
|---------|-------------|
| `django-otp` | Two-factor authentication framework |
| `pyotp` | Generates TOTP codes (Google Authenticator compatible) |
| `qrcode` | Generates QR codes for 2FA setup |
| `django-ratelimit` | Rate limits views to prevent abuse |
| `django-cors-headers` | Handles CORS headers for cross-origin API requests |

### Forms & Filtering

| Package | What It Does |
|---------|-------------|
| `django-crispy-forms` | Renders Django forms with Bootstrap styling |
| `crispy-bootstrap5` | Bootstrap 5 template pack for crispy-forms |
| `django-filter` | Adds filtering to DRF views (FilterSet, SearchFilter) |
| `django-multiselectfield` | Multi-select field for Django models |

### Utilities

| Package | What It Does |
|---------|-------------|
| `requests` | HTTP client for KoboToolbox API calls |
| `python-decouple` | Reads config from `.env` files and environment variables |
| `python-dateutil` | Flexible date parsing |
| `Markdown` | Renders Markdown to HTML |

---

## 7.4 Celery & Task Queues

### What Is Celery?

Celery is a **distributed task queue**. Instead of making the user wait while the server syncs with KoboToolbox or generates a report, Celery runs these tasks **in the background**.

```
User clicks "Generate Report"
    │
    ├── Django view sends task to Redis:
    │     generate_report.delay(report_id)
    │
    ├── Returns immediately:
    │     "Report is being generated..."
    │
    └── Celery worker picks up task:
          generate_report(report_id)
          → Queries database
          → Creates Excel/PDF
          → Saves to media/reports/
          → Marks complete
```

### Architecture

```
┌───────────────┐     ┌───────────┐     ┌─────────────────┐
│  Django views  │────►│   Redis   │────►│  Celery Worker  │
│  (producers)   │     │  (broker) │     │  (consumer)     │
│                │     │           │     │                 │
│  .delay()      │     │  Queue:   │     │  Picks up tasks │
│  .apply_async()│     │  message  │     │  Runs them      │
│                │     │  storage  │     │  Stores results │
└───────────────┘     └───────────┘     └─────────────────┘
```

### Task Queues

Tasks are routed to specific queues by app:

| Queue | App | Example Tasks |
|-------|-----|---------------|
| `data_processing` | `api` | `sync_all_kobotoolbox_data`, `process_bms_data`, `reprocess_submission` |
| `validation` | `dashboard` | `update_validation_statistics`, `generate_report`, `cleanup_old_logs` |
| `media_downloads` | `species` | `download_field_media_image`, `download_inaturalist_images` |

### Task Routing Configuration

```python
# In prism/celery.py:
app.conf.task_routes = {
    'dashboard.tasks.*': {'queue': 'validation'},
    'api.tasks.*': {'queue': 'data_processing'},
    'species.tasks.*': {'queue': 'media_downloads'},
}
```

### Celery Beat (Periodic Tasks)

Celery Beat is a scheduler that sends tasks at defined intervals:

| Task | Schedule | Purpose |
|------|----------|---------|
| `sync_all_kobotoolbox_data` | Every 10 min | Pull new submissions from KoboToolbox |
| `check_overdue_surveys` | Every hour | Flag overdue scheduled surveys |
| `update_validation_statistics` | Every 30 min | Refresh cached validation counts |
| `sync_all_validation_statuses` | Every 2 hours | Sync status changes to/from KoboToolbox |
| `cleanup_old_logs` | Weekly | Remove old activity/retrieval logs |

### Worker Configuration

```python
# prism/celery.py
app.conf.worker_prefetch_multiplier = 1    # Fetch 1 task at a time (fair scheduling)
app.conf.task_acks_late = True             # Acknowledge after completion (not before)
app.conf.worker_max_tasks_per_child = 1000 # Restart worker after 1000 tasks (memory leak prevention)
app.conf.result_expires = 3600             # Results expire after 1 hour

# Production (settings_production.py)
CELERY_TASK_TIME_LIMIT = 1800              # Hard kill at 30 minutes
CELERY_TASK_SOFT_TIME_LIMIT = 1500         # Soft warning at 25 minutes
```

---

## 7.5 URL Routing (Top Level)

Django routes URLs hierarchically. The root router is `prism/urls.py`:

```python
urlpatterns = [
    path('', home_redirect, name='home'),           # → role-based redirect
    path('admin/', admin.site.urls),                 # Django admin
    path('api/', include('api.urls')),               # REST API
    path('accounts/', include('users.urls')),        # Login/register/profile
    path('accounts/login/', LoginView.as_view()),    # Login page
    path('accounts/logout/', LogoutView.as_view()),  # Logout
    path('dashboard/', include('dashboard.urls')),   # Dashboard UI
]
```

### The `home_redirect` View

When users visit `/`, they're redirected based on their role:

| Role | Redirected To |
|------|--------------|
| `system_admin` | `/dashboard/admin/` |
| `pa_staff` | `/dashboard/pa-staff-home/` |
| `validator` | `/dashboard/queue/` |
| `denr` | `/dashboard/reports/archives/` |
| Anonymous | `/accounts/login/` |
| Other | `/dashboard/` |

### Custom Error Handlers

```python
handler404 = 'dashboard.error_views.custom_404'
handler403 = 'dashboard.error_views.custom_403'
handler500 = 'dashboard.error_views.custom_500'
```

---

## 7.6 Environment Variables Reference

### Essential Variables (Must Be Set in Production)

| Variable | Example | Purpose |
|----------|---------|---------|
| `SECRET_KEY` | `your-random-50-char-string` | Django cryptographic signing |
| `DEBUG` | `False` | Disable debug mode |
| `ALLOWED_HOSTS` | `prism-matutum.org,www.prism-matutum.org` | Accepted hostnames |
| `DATABASE_HOST` | `db` | PostgreSQL hostname |
| `DATABASE_PORT` | `5432` | PostgreSQL port |
| `DATABASE_NAME` | `prismdb` | Database name |
| `DATABASE_USER` | `prismuser` | Database user |
| `DATABASE_PASSWORD` | `strong-password-here` | Database password |
| `CELERY_BROKER_URL` | `redis://redis:6379/0` | Redis URL for Celery |
| `CELERY_RESULT_BACKEND` | `redis://redis:6379/0` | Redis URL for task results |
| `KOBOTOOLBOX_TOKEN` | `abc123def456` | KoboToolbox API authentication token |

### Optional Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ENVIRONMENT` | `development` | Selects settings module (`development`/`production`) |
| `DJANGO_SETTINGS_MODULE` | `prism.settings` | Which settings file Django uses |
| `KOBOTOOLBOX_BASE_URL` | `https://kf.kobotoolbox.org` | KoboToolbox API base URL |
| `AUTO_COMMIT_ON_APPROVAL` | `True` | Auto-process approved submissions |
| `REDIS_URL` | `redis://localhost:6379/1` | Redis cache URL |
| `CELERY_ALWAYS_EAGER` | `False` | Run Celery tasks synchronously (for testing) |
| `NGROK_AUTHTOKEN` | — | ngrok tunnel authentication |

### Email (Production)

| Variable | Default | Purpose |
|----------|---------|---------|
| `EMAIL_HOST` | `smtp.gmail.com` | SMTP server |
| `EMAIL_PORT` | `587` | SMTP port |
| `EMAIL_HOST_USER` | — | SMTP username |
| `EMAIL_HOST_PASSWORD` | — | SMTP password |
| `DEFAULT_FROM_EMAIL` | `noreply@prism-matutum.org` | Sender address |

### Monitoring (Production)

| Variable | Default | Purpose |
|----------|---------|---------|
| `SENTRY_DSN` | — | Sentry error tracking URL |
| `SENTRY_TRACES_SAMPLE_RATE` | `0.1` | Performance monitoring sample rate |
| `APP_VERSION` | `1.0.0` | App version tag for Sentry |
| `LOG_LEVEL` | `INFO` | Logging verbosity |

---

## 7.7 Management Commands

All commands are run via: `docker-compose exec web python manage.py <command>`

### Data Processing (`api/management/commands/`)

| Command | Purpose |
|---------|---------|
| `sync_kobotoolbox_data` | Pull new submissions from KoboToolbox API into `DataSubmission` table |
| `process_bms_data` | Process raw submissions through the complete BMS workflow |
| `process_validated_submissions` | Process and commit only the validated (approved) submissions |
| `initialize_choice_lists` | Set up choice lists for KoboToolbox form integration |

### Data Maintenance (`api/management/commands/`)

| Command | Purpose |
|---------|---------|
| `assign_observations_to_segments` | Link observations to nearest transect segments via PostGIS |
| `backfill_assigned_staff` | Populate assigned_staff on submissions from scheduled surveys |
| `generate_transect_segments` | Auto-generate segments for all active transect routes |
| `populate_observation_types` | Seed `ObservationType` table with default modes |
| `migrate_observation_modes` | Migrate data from old field to new ManyToMany relationship |
| `update_survey_statuses` | Transition surveys: scheduled → in_progress → overdue |

### Debugging (`api/management/commands/`)

| Command | Purpose |
|---------|---------|
| `debug_timezone_issues` | Diagnose timezone problems with submissions and surveys |

### Dashboard Setup (`dashboard/management/commands/`)

| Command | Purpose |
|---------|---------|
| `setup_validator_dashboard` | Production setup: create superuser, validator user, directories, check config |
| `generate_test_report` | Run test suite and generate HTML/JSON/MD test reports |

### Species Data (`species/management/commands/`)

| Command | Purpose |
|---------|---------|
| `add_species_byjson` | Populate fauna/flora checklists from a JSON file |
| `download_field_images` | Download field media images from KoboToolbox to local storage |

---

## 7.8 Development Workflow

### First-Time Setup

```bash
# 1. Clone the repository
git clone <repo-url>
cd PRISM-Matutum

# 2. Start all services
docker-compose up -d --build

# 3. Run database migrations
docker-compose exec web python manage.py migrate

# 4. Create a superuser
docker-compose exec web python manage.py createsuperuser

# 5. Initialize choice lists (KoboToolbox integration)
docker-compose exec web python manage.py initialize_choice_lists

# 6. (Optional) Load species data
docker-compose exec web python manage.py add_species_byjson path/to/species.json

# 7. Access the site
# http://localhost:8000
```

### Daily Development

```bash
# Start services
docker-compose up -d

# View live logs
docker-compose logs -f web

# Run Django shell
docker-compose exec web python manage.py shell

# Make/apply migrations after model changes
docker-compose exec web python manage.py makemigrations
docker-compose exec web python manage.py migrate

# Run tests
docker-compose exec web python manage.py test

# Stop services
docker-compose down
```

### Useful Docker Commands

```bash
# Rebuild after requirements.txt change
docker-compose up -d --build web celery

# View Celery worker logs
docker-compose logs -f celery

# Access database directly
docker-compose exec db psql -U prismuser prismdb

# Restart a single service
docker-compose restart web
```

---

## 7.9 Production Deployment

### Pre-Deployment Checklist

1. **Set all required environment variables** (see Section 7.6)
2. **Set `DEBUG=False`**
3. **Set a strong, random `SECRET_KEY`** (at least 50 characters)
4. **Configure `ALLOWED_HOSTS`** with your actual domain
5. **Set database credentials** via environment variables (not hardcoded)
6. **Configure email** settings for notifications
7. **Run `collectstatic`** to gather all static files

### Deploying with Docker

```bash
# 1. Pull the latest image
docker pull freymeyer/prism-matutum:latest

# 2. Create a .env file with production variables
cat > .env << EOF
SECRET_KEY=your-strong-random-key-here
POSTGRES_PASSWORD=strong-db-password
DEBUG=False
ALLOWED_HOSTS=prism-matutum.org
DJANGO_SETTINGS_MODULE=prism.settings_production
DATABASE_HOST=db
DATABASE_PORT=5432
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0
KOBOTOOLBOX_TOKEN=your-token
EOF

# 3. Start with production compose
docker-compose -f docker-compose.prod.yml up -d

# 4. Run migrations
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate

# 5. Run production setup
docker-compose -f docker-compose.prod.yml exec web python manage.py setup_validator_dashboard

# 6. Verify health
curl https://your-domain/health/
```

### Production Security Settings

When `settings_production.py` is active, these security features are enabled:

| Feature | Setting | Effect |
|---------|---------|--------|
| **HSTS** | 1 year | Browser enforces HTTPS for 1 year |
| **SSL Redirect** | Enabled | HTTP → HTTPS automatic redirect |
| **Secure Cookies** | Enabled | Cookies sent only over HTTPS |
| **CSRF Secure** | Enabled | CSRF cookie only over HTTPS |
| **X-Frame-Options** | `DENY` | Prevents clickjacking |
| **Static Files** | WhiteNoise | Compressed, cached for 1 year |
| **DB Connections** | Pooled, `CONN_MAX_AGE=600` | Connection reuse |
| **Logging** | RotatingFileHandler | 15MB/file, 10 backups, admin email on ERROR |

### Production Compose Services

```yaml
# docker-compose.prod.yml services:
services:
  db:          # PostgreSQL + PostGIS (persistent volume)
  redis:       # Message broker
  web:         # gunicorn with 4 workers
  celery:      # Background task worker
  celery-beat: # Periodic task scheduler (separate service)
```

> **Note:** Production uses a separate `celery-beat` service for the periodic scheduler, while development runs it within the celery worker.

---

**Next:** [Chapter 8 — Django Admin →](08-django-admin.md)

**Previous:** [← Chapter 6: Frontend Architecture](06-frontend-architecture.md)
