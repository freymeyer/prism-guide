# PRISM-Matutum — Complete Project Guide

> **PRISM** (Protected Area Resource Information System and Monitoring) for **Mt. Matutum Protected Landscape** — a Django-based web application for biodiversity data collection, validation, management, and analysis.

This guide is designed for developers joining the project. Each chapter explains the relevant **Django concepts** first, then shows how PRISM-Matutum applies them. By the end, you should understand every part of the system well enough to work on it — or to document a similar Django project of your own.

---

## Table of Contents

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 1 | [The `api` App](01-api-app.md) | Models, services, signals, REST API, Celery tasks — the data backbone |
| 2 | [The `dashboard` App](02-dashboard-app.md) | Views, templates, URL routing, forms, reports — the user-facing layer |
| 3 | [The `users` App](03-users-app.md) | Custom User model, RBAC, 2FA, account approval, organizations |
| 4 | [The `species` App](04-species-app.md) | Taxonomy hierarchy, checklists, media, analytics services |
| 5 | [Data Flow Pipeline](05-data-flow-pipeline.md) | End-to-end journey from KoboToolbox mobile → dashboard visualization |
| 6 | [Frontend Architecture](06-frontend-architecture.md) | Templates, CSS component system, JavaScript, Leaflet maps |
| 7 | [Configuration & Deployment](07-configuration-deployment.md) | Docker, settings, Celery, environment variables, management commands |
| 8 | [Django Admin](08-django-admin.md) | Admin site configuration, custom actions, GIS admin |

---

## What Is This Project?

PRISM-Matutum is a **biodiversity monitoring system** used by the Department of Environment and Natural Resources (DENR) and Protected Area (PA) staff to:

1. **Collect** field data (wildlife sightings, resource use incidents, disturbances, landscape monitoring) using [KoboToolbox](https://www.kobotoolbox.org/) on mobile devices
2. **Validate** that data through a multi-stage review process (PA Staff review → Validator approval)
3. **Store** processed records in a PostgreSQL database with PostGIS spatial support
4. **Analyze** biodiversity trends through dashboards, maps, charts, and exported reports
5. **Manage** species checklists, taxonomic data, monitoring locations, transect routes, and organizational structure

### Who Uses It?

| Role | What They Do |
|------|-------------|
| **PA Staff** | Collect field data via KoboToolbox, review own submissions |
| **Validator** | Review and approve/reject submissions, manage species data |
| **DENR Personnel** | View reports and analytics, oversee protected area management |
| **System Admin** | Manage users, organizations, system configuration |
| **Collaborator** | Read-only access to species data and observations |

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Backend** | Django 5.2, Python 3.x | Web framework, ORM, template rendering |
| **REST API** | Django REST Framework (DRF) | JSON API endpoints for data operations |
| **Database** | PostgreSQL 16 + PostGIS 3.4 | Relational data + spatial/geographic queries |
| **Task Queue** | Celery 5.4 + Redis 7 | Async background tasks (data sync, reports) |
| **Frontend** | Django Templates + Bootstrap 5 | Server-rendered HTML with responsive design |
| **Maps** | Leaflet.js + django-leaflet | Interactive maps with layers, clustering, heatmaps |
| **PDF Reports** | WeasyPrint 63 | HTML-to-PDF report generation |
| **Data Processing** | Pandas, NumPy, OpenPyXL | Excel/CSV generation, data analysis |
| **Spatial** | Fiona, Shapely | Shapefile export, geometric operations |
| **External API** | KoboToolbox API v2 | Field data collection platform |
| **Auth** | PyOTP, QRCode | TOTP-based two-factor authentication |
| **Containerization** | Docker, Docker Compose | Development and deployment environment |

---

## Project Structure — The Big Picture

A Django **project** is a collection of **apps**. Each app is a self-contained module that handles one area of functionality. The project configuration lives in the `prism/` directory.

```
PRISM-Matutum/
│
├── prism/                  # Project configuration (settings, URLs, WSGI/ASGI, Celery)
│   ├── settings.py         #   All Django settings (database, apps, middleware, etc.)
│   ├── urls.py             #   Root URL routing — connects URLs to apps
│   ├── celery.py           #   Celery (background task) configuration
│   └── wsgi.py / asgi.py   #   Web server entry points
│
├── api/                    # APP: Data backbone — models, services, REST API
│   ├── models/             #   Database models (split across multiple files)
│   ├── *_services.py       #   Business logic (service layer pattern)
│   ├── *_serializers.py    #   DRF serializers for API input/output
│   ├── *_views.py          #   API view endpoints
│   ├── signals.py          #   Auto-processing triggers
│   ├── tasks.py            #   Celery background tasks
│   └── management/         #   Custom manage.py commands
│
├── dashboard/              # APP: User-facing layer — views, templates, reports
│   ├── views_*.py          #   20+ view modules by feature area
│   ├── templates/          #   80+ HTML templates
│   ├── static/             #   CSS (53 files), JS (4 files), images
│   ├── forms.py            #   Django forms for user input
│   ├── filters.py          #   Data filtering for list views
│   ├── permissions.py      #   Access control mixins
│   ├── decorators.py       #   Permission decorators
│   ├── middleware.py       #   Request processing pipeline
│   ├── context_processors.py  # Template context injection
│   ├── report_services.py  #   Report generation logic
│   └── templatetags/       #   Custom template filters/tags
│
├── users/                  # APP: Authentication & user management
│   ├── models.py           #   Custom User, profiles, organizations, geography
│   ├── views.py            #   Login, registration, 2FA, password management
│   └── admin.py            #   Django admin configuration
│
├── species/                # APP: Taxonomic reference data
│   ├── models.py           #   Full taxonomy hierarchy, checklists, media
│   ├── services.py         #   Taxonomy and biodiversity analytics services
│   ├── tasks.py            #   Image download background tasks
│   └── admin.py            #   Species admin with image previews
│
├── docker-compose.yml      # Docker services: web, db, redis, celery, ngrok
├── requirements.txt        # Python package dependencies
├── manage.py               # Django's command-line utility
└── docs/                   # Documentation (you are here)
```

---

## How the Apps Connect

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SYSTEMS                             │
│   KoboToolbox (mobile data collection) ◄──── PA Staff in the field  │
└─────────────────────┬───────────────────────────────────────────────┘
                      │ REST API (sync every 10 min)
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         api app                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │    Models     │  │   Services   │  │    Signals (auto-process) │  │
│  │ DataSubmission│  │ DataSubmission│  │ approved → BMSDataProc   │  │
│  │ Wildlife Obs  │  │ BMSDataProc  │  │          → CommitService  │  │
│  │ Resource Use  │  │ BMSWorkflow  │  └───────────────────────────┘  │
│  │ Disturbance   │  │ FormServices │                                 │
│  │ Landscape     │  └──────────────┘  ┌───────────────────────────┐  │
│  │ Choices/Forms │                    │  Celery Tasks (periodic)  │  │
│  │ Monitoring    │                    │  sync_all_kobotoolbox     │  │
│  │ Schedules     │                    │  check_overdue_surveys    │  │
│  │ FGD Sessions  │                    └───────────────────────────┘  │
│  └──────┬───────┘                                                    │
│         │ provides data to                                           │
└─────────┼────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       dashboard app                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │    Views      │  │  Templates   │  │      Report Services      │  │
│  │ (20+ modules) │  │  (80+ HTML)  │  │  GIS, Semestral, BMS,    │  │
│  │ Validation    │  │  Role-based  │  │  Digital Backup, PDF      │  │
│  │ Species CRUD  │  │  dashboards  │  └───────────────────────────┘  │
│  │ GIS/Maps      │  │  Widgets     │                                 │
│  │ Reports       │  │  Components  │  ┌───────────────────────────┐  │
│  │ Admin Panel   │  └──────────────┘  │  Permissions & Middleware │  │
│  │ Events/FGD    │                    │  Role-based access control│  │
│  │ Analytics     │                    └───────────────────────────┘  │
│  └──────┬───────┘                                                    │
│         │ depends on                                                 │
└─────────┼────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────┐    ┌──────────────────────────────────────┐
│       users app           │    │           species app                 │
│  Custom User (5 roles)    │    │  Taxonomy (Kingdom→Genus)            │
│  Organization structure   │    │  FaunaChecklist / FloraChecklist     │
│  Address geography        │    │  Characteristics & morphology        │
│  2FA authentication       │    │  Field media with download pipeline  │
│  Account approval flow    │    │  Biodiversity analytics services     │
│  Activity logging         │    │  Image download Celery tasks         │
└──────────────────────────┘    └──────────────────────────────────────┘
```

---

## Django Concepts Primer

If you're new to Django, here are the key concepts you'll encounter throughout this guide:

| Concept | What It Is | Where You'll See It |
|---------|-----------|-------------------|
| **Model** | A Python class that maps to a database table. Each attribute = a column. | `api/models/`, `users/models.py`, `species/models.py` |
| **View** | A function or class that takes a web request and returns a response. | `dashboard/views_*.py`, `api/*_views.py` |
| **Template** | An HTML file with Django template tags (`{% %}`, `{{ }}`). | `dashboard/templates/` |
| **URL Configuration** | Maps URL patterns (like `/dashboard/species/`) to views. | `prism/urls.py`, `dashboard/urls.py` |
| **Migration** | Auto-generated SQL that creates/alters database tables from models. | `*/migrations/` |
| **Serializer** | Converts Django models to/from JSON (used by DRF for APIs). | `api/*_serializers.py` |
| **Signal** | A hook that runs code automatically when something happens (e.g., model saved). | `api/signals.py` |
| **Middleware** | Code that runs on every request/response (authentication, logging). | `dashboard/middleware.py` |
| **Context Processor** | Injects variables into every template automatically. | `dashboard/context_processors.py` |
| **Management Command** | Custom `python manage.py <command>` scripts. | `*/management/commands/` |
| **Celery Task** | Background function that runs asynchronously (outside web request). | `api/tasks.py`, `dashboard/tasks.py` |
| **Form** | A Python class for validating and processing user input from HTML forms. | `dashboard/forms.py` |
| **Mixin** | A class that adds reusable behavior to views (like permission checking). | `dashboard/permissions.py` |

---

## Existing Documentation

This project guide is **additive** — the `docs/` folder also contains deeper reference documentation:

| Folder | Contents |
|--------|----------|
| `docs/architecture/` | System architecture, URL routing reference, RBAC documentation, permission system details |
| `docs/features/` | Deep dives into validation, KoboToolbox integration, dashboard widgets, GIS features, reports, species management, events, user/org management |
| `docs/testing/` | Test plans, UTAUT integration testing, test script tracking |
| `docs/user-guides/` | Per-role user guides (PA Staff, Validator, DENR, Collaborator, System Admin, Deployment) |

---

## Getting Started

To run the project locally, see [Chapter 7: Configuration & Deployment](07-configuration-deployment.md). The quick version:

```bash
# Start all services (Django, PostgreSQL, Redis, Celery)
docker-compose up --build

# Run database migrations
docker-compose exec web python manage.py migrate

# Create initial data
docker-compose exec web python manage.py initialize_choice_lists
docker-compose exec web python manage.py create_test_users

# Access the application
# http://localhost:8000
```

---

**Next:** [Chapter 1 — The `api` App →](01-api-app.md)
