# PRISM-Matutum System Architecture Overview

**Document Version:** 1.0  
**Last Updated:** January 6, 2026  
**Author:** Documentation for Capstone Project  

---

## Table of Contents

1. [Introduction](#introduction)
2. [System Architecture](#system-architecture)
3. [Technology Stack](#technology-stack)
4. [Module Structure](#module-structure)
5. [Data Flow Architecture](#data-flow-architecture)
6. [Component Integration](#component-integration)
7. [File Organization](#file-organization)
8. [Design Patterns](#design-patterns)

---

## 1. Introduction

**PRISM-Matutum** (Protected Area Biodiversity Monitoring System) is a comprehensive web-based platform designed for collecting, validating, managing, and analyzing biodiversity data from protected areas. The system integrates field data collection through KoboToolbox with a Django-based backend and PostgreSQL+PostGIS spatial database.

### System Purpose

- **Field Data Collection**: Enable PA Staff to collect wildlife observations, resource use incidents, disturbance records, and landscape monitoring data.
- **Data Validation**: Provide Validators with tools to review, approve, or reject submitted data.
- **Data Analysis**: Offer analytics, reporting, and visualization capabilities for decision-makers.
- **Multi-Role Access**: Support different user roles with appropriate permissions and workflows.

### Key Features

- Role-Based Access Control (RBAC) with 5 distinct user roles
- Real-time data synchronization with KoboToolbox
- Geographic Information System (GIS) integration with Leaflet.js
- Custom dashboard and widget system
- Comprehensive validation workflow
- Report generation and export functionality
- Species management and taxonomy system

---

## 2. System Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Web Browser  │  │   Mobile     │  │ KoboToolbox  │          │
│  │  (Desktop)   │  │   Browser    │  │   Collect    │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                  │
│         └──────────────────┴──────────────────┘                  │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             │
                             │ HTTPS
                             │
┌────────────────────────────┼─────────────────────────────────────┐
│                    PRESENTATION LAYER                            │
│  ┌──────────────────────────┴─────────────────────────────┐     │
│  │         Django Templates + Bootstrap 5                  │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │     │
│  │  │  Base    │  │ PA Staff │  │Validator │            │     │
│  │  │Templates │  │Templates │  │Templates │  + Others  │     │
│  │  └──────────┘  └──────────┘  └──────────┘            │     │
│  │                                                        │     │
│  │         JavaScript (Leaflet.js, Chart.js)             │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────┐
│                    APPLICATION LAYER                             │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              Django 5.2 Application                    │     │
│  │  ┌──────────────────────────────────────────────┐     │     │
│  │  │  dashboard/  (Main Module)                   │     │     │
│  │  │  • views_*.py      - View Controllers        │     │     │
│  │  │  • permissions.py  - Access Control          │     │     │
│  │  │  • forms.py        - Form Definitions        │     │     │
│  │  │  • models.py       - Dashboard Models        │     │     │
│  │  │  • urls.py         - URL Routing             │     │     │
│  │  └──────────────────────────────────────────────┘     │     │
│  │                                                        │     │
│  │  ┌──────────────────────────────────────────────┐     │     │
│  │  │  api/  (Core Data Module)                    │     │     │
│  │  │  • models/         - Data Models             │     │     │
│  │  │  • *_services.py   - Business Logic          │     │     │
│  │  │  • *_views.py      - API Endpoints           │     │     │
│  │  │  • tasks.py        - Celery Tasks            │     │     │
│  │  └──────────────────────────────────────────────┘     │     │
│  │                                                        │     │
│  │  ┌──────────────────────────────────────────────┐     │     │
│  │  │  Supporting Modules                          │     │     │
│  │  │  • users/      - User Management             │     │     │
│  │  │  • species/    - Species Data                │     │     │
│  │  └──────────────────────────────────────────────┘     │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────┐
│                      SERVICE LAYER                               │
│  ┌────────────────────┐  ┌─────────────────────┐                │
│  │  Celery Workers    │  │  Redis Message      │                │
│  │  • Async Tasks     │  │  Broker & Cache     │                │
│  │  • Data Sync       │  │                     │                │
│  │  • Report Gen      │  │                     │                │
│  └────────────────────┘  └─────────────────────┘                │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────┐
│                       DATA LAYER                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │        PostgreSQL 16 + PostGIS 3.x                     │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │     │
│  │  │ User & Auth  │  │ Submissions  │  │   Species    │ │     │
│  │  │    Tables    │  │   & Records  │  │   Taxonomy   │ │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │     │
│  │  │   Spatial    │  │  Dashboards  │  │   Reports    │ │     │
│  │  │   Geometry   │  │   & Widgets  │  │   & Logs     │ │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────┐
│                   EXTERNAL INTEGRATIONS                          │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  KoboToolbox API (https://eu.kobotoolbox.org)         │     │
│  │  • Form Definitions                                    │     │
│  │  • Field Submissions                                   │     │
│  │  • Choice Lists Sync                                   │     │
│  └────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```

### Architecture Layers Explained

#### 1. **Client Layer**
- Web browsers (desktop and mobile) for system access
- KoboToolbox Collect mobile app for field data collection
- All communication over HTTPS for security

#### 2. **Presentation Layer**
- Django Templates with Bootstrap 5 for responsive UI
- Role-specific template sets (PA Staff, Validator, Admin, etc.)
- JavaScript libraries: Leaflet.js (maps), Chart.js (analytics)

#### 3. **Application Layer**
- Django 5.2 framework
- Modular architecture with separated concerns:
  - **dashboard/**: Main user interface module
  - **api/**: Core data processing and business logic
  - **users/**: Authentication and authorization
  - **species/**: Biodiversity data management

#### 4. **Service Layer**
- **Celery**: Asynchronous task processing
- **Redis**: Message broker and caching
- Background jobs: data synchronization, report generation

#### 5. **Data Layer**
- **PostgreSQL 16**: Relational database
- **PostGIS 3.x**: Spatial data extension
- Stores all application data with geographic capabilities

#### 6. **External Integrations**
- **KoboToolbox**: Field data collection platform
- Bidirectional synchronization for forms and submissions

---

## 3. Technology Stack

### Backend Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| **Django** | 5.2 | Web framework and ORM |
| **Django REST Framework** | Latest | API development |
| **PostgreSQL** | 16 | Primary database |
| **PostGIS** | 3.x | Spatial database extension |
| **Celery** | Latest | Asynchronous task queue |
| **Redis** | Latest | Message broker & cache |
| **Python** | 3.14+ | Programming language |

### Frontend Technologies

| Technology | Purpose |
|------------|---------|
| **Bootstrap 5** | CSS framework for responsive design |
| **Leaflet.js** | Interactive mapping library |
| **Chart.js** | Data visualization and charts |
| **jQuery** | DOM manipulation (legacy support) |
| **Vanilla JavaScript** | Modern interactive features |

### Development & Deployment

| Tool | Purpose |
|------|---------|
| **Docker** | Containerization |
| **Docker Compose** | Multi-container orchestration |
| **Git** | Version control |
| **Nginx** | Web server (production) |
| **Gunicorn** | WSGI HTTP server |

### External Services

- **KoboToolbox**: Form builder and data collection
- **iNaturalist** (optional): Species image integration

---

## 4. Module Structure

### Dashboard Module (`dashboard/`)

The primary user-facing module containing all views, templates, and frontend logic.

#### Key Files and Their Purposes

```
dashboard/
├── views_*.py          → View controllers organized by feature
│   ├── views_dashboard.py      → Main dashboard views
│   ├── views_validation.py     → Validation queue & workspace
│   ├── views_pa_staff.py       → PA Staff home & submissions
│   ├── views_admin.py          → Admin management views
│   ├── views_species.py        → Species management
│   ├── views_gis.py            → GIS/Mapping features
│   ├── views_reports.py        → Report generation
│   ├── views_events.py         → Event management
│   ├── views_schedule.py       → Survey scheduling
│   ├── views_fgd.py            → Focus Group Discussions
│   ├── views_widgets.py        → Dashboard widgets
│   ├── views_taxonomy.py       → Taxonomy management
│   ├── views_articles.py       → Species articles
│   ├── views_profile.py        → User profile
│   ├── views_api.py            → API endpoints
│   ├── views_universal_*.py    → Universal observation views
│
├── permissions.py      → Role-based access control mixins
├── decorators.py       → Function-based permission decorators
├── models.py           → Dashboard-specific models (Reports, Dashboards, Widgets)
├── forms.py            → Form definitions (2126 lines - comprehensive)
├── urls.py             → URL routing configuration (314 lines)
├── filters.py          → QuerySet filters for data tables
├── tasks.py            → Celery async tasks
├── utils.py            → Utility functions
├── utils_geo.py        → Geospatial utilities
├── api_geo.py          → GeoJSON API endpoints
├── context_processors.py → Template context additions
├── middleware.py       → Custom middleware
├── health.py           → Health check endpoints
├── event_services.py   → Event business logic
├── report_services.py  → Report generation logic
│
├── templates/dashboard/  → Django templates
│   ├── base.html              → Base template
│   ├── *_home.html            → Role-specific home pages
│   ├── validation_*.html      → Validation views
│   ├── admin/                 → Admin templates
│   ├── reports/               → Report templates
│   ├── widgets/               → Widget templates
│   ├── errors/                → Error pages
│   └── ...                    → Feature templates
│
├── static/dashboard/     → Static assets
│   ├── css/                   → Stylesheets (component-based)
│   ├── js/                    → JavaScript files
│   └── images/                → Images and icons
│
├── management/           → Django management commands
│   └── commands/
│
├── tests/                → Test suite
│   ├── test_*.py              → Test modules
│   └── *_README.md            → Test documentation
│
└── docs/                 → Module documentation
    ├── USER_GUIDE.md
    ├── VALIDATION_WORKFLOW.md
    ├── *_TEST_CHECKLIST.md
    └── ...
```

#### View Files Organization

The dashboard module uses a **feature-based view organization** pattern:

- **views_dashboard.py**: Custom dashboards, role-specific dashboards (DENR, Collaborator)
- **views_validation.py**: Validation queue, workspace, history, audit trails
- **views_pa_staff.py**: PA Staff home, submissions list/detail, observations
- **views_admin.py**: User management, agency/department management, address management, choice lists
- **views_species.py**: Species CRUD, observations, media management
- **views_gis.py**: Interactive map, monitoring locations, transect routes
- **views_reports.py**: Report builder, templates, archive, export center
- **views_events.py**: Event CRUD operations
- **views_schedule.py**: Survey scheduling and calendar
- **views_fgd.py**: Focus Group Discussion sessions
- **views_widgets.py**: Dashboard widget CRUD and configuration
- **views_taxonomy.py**: Taxonomy hierarchy management
- **views_articles.py**: Species-related articles
- **views_profile.py**: User profile management
- **views_api.py**: API endpoints for AJAX requests
- **views_universal_observation.py**: Universal observation list views
- **views_universal_detail.py**: Universal observation detail views

### API Module (`api/`)

Core data processing and business logic module.

```
api/
├── models/                → Domain models (split by entity)
│   ├── submissions.py           → DataSubmission (raw Kobo data)
│   ├── records_*.py             → Processed records (Wildlife, Resource Use, etc.)
│   ├── choices.py               → Dynamic choice lists
│   ├── monitoring.py            → Monitoring locations & routes
│   └── ...
│
├── *_services.py          → Business logic services
│   ├── submission_services.py   → Data submission handling
│   ├── form_services.py         → KoboToolbox form management
│   ├── data_processing_services.py → Data processing & validation
│   ├── workflow_services.py     → Workflow state management
│   └── excel_processor.py       → Excel export functionality
│
├── *_views.py             → API view endpoints
│   ├── form_views.py            → Form-related endpoints
│   ├── submission_views.py      → Submission endpoints
│   ├── data_processing_views.py → Processing endpoints
│
├── *_serializers.py       → DRF serializers
│   ├── form_serializers.py
│   └── submission_serializers.py
│
├── tasks.py               → Celery background tasks
├── admin.py               → Django admin configuration
├── urls.py                → API URL routing
└── management/            → Management commands
    └── commands/
        ├── initialize_choice_lists.py
        ├── sync_kobotoolbox_data.py
        └── ...
```

#### Service Layer Pattern

The API module follows a **Service-Oriented Architecture**:

1. **Views** receive requests and delegate to **Services**
2. **Services** contain all business logic
3. **Models** are pure data structures (no complex logic)
4. **Serializers** handle data transformation

Example flow:
```
Request → View → Service → Model → Database
                   ↓
              (Business Logic)
```

### Users Module (`users/`)

Authentication, authorization, and user management.

```
users/
├── models.py          → User model with role-based permissions
├── forms.py           → Registration and profile forms
├── views.py           → Auth views (login, register, profile)
├── admin.py           → User admin interface
└── management/        → User-related commands
```

#### User Model Key Fields

- `role`: Enum (pa_staff, validator, denr, system_admin, collaborator)
- `account_status`: Enum (pending, approved, rejected)
- `agency`, `department`: Organization links
- Permission methods: `is_pa_staff()`, `is_validator()`, `can_validate()`, etc.

### Species Module (`species/`)

Biodiversity data and taxonomy management.

```
species/
├── models.py          → Species models (Fauna, Flora, Taxonomy)
├── admin.py           → Species admin interface
└── management/        → Species data commands
```

---

## 5. Data Flow Architecture

### Primary Data Flow: Field Data Collection → Validation → Analysis

```
┌─────────────────────────────────────────────────────────────────────┐
│                     STEP 1: DATA COLLECTION                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PA Staff in Field                                                  │
│       ↓                                                             │
│  KoboToolbox Collect App                                            │
│       ↓                                                             │
│  Form Submission (Wildlife, Resource Use, Disturbance, Landscape)   │
│       ↓                                                             │
│  KoboToolbox Server (https://eu.kobotoolbox.org)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                    (Celery Task - Periodic Sync)
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   STEP 2: DATA INGESTION                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Celery Task: sync_kobotoolbox_data                                 │
│       ↓                                                             │
│  DataSubmissionService.retrieve_submissions()                       │
│       ↓                                                             │
│  Create/Update DataSubmission records                               │
│       - form_id: Source form identifier                             │
│       - submission_id: Kobo submission UUID                         │
│       - content: Raw JSON payload from Kobo                         │
│       - validation_status: "---" (pending)                          │
│       - submitted_by: PA Staff user                                 │
│       - location: Point geometry (lat/long)                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   STEP 3: VALIDATION WORKFLOW                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Validator accesses Validation Queue                                │
│       ↓                                                             │
│  Filters pending submissions (validation_status = "---")            │
│       ↓                                                             │
│  Opens Validation Workspace for a submission                        │
│       ↓                                                             │
│  Reviews data:                                                      │
│    - Location accuracy                                              │
│    - Species identification                                         │
│    - Photo quality                                                  │
│    - Data completeness                                              │
│       ↓                                                             │
│  Makes decision:                                                    │
│    ┌─────────────────┬─────────────────┬─────────────────┐        │
│    │ APPROVE         │ REJECT          │ HOLD            │        │
│    │ (validation_ok) │ (validation_nok)│ (validation_hold)│        │
│    └────────┬────────┴────────┬────────┴────────┬────────┘        │
│             ↓                 ↓                  ↓                  │
│  Creates ValidationHistory record                                   │
│    - validator: User                                                │
│    - decision: Approved/Rejected/Hold                               │
│    - remarks: Validation comments                                   │
│    - timestamp: Now                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                    (If APPROVED only)
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                  STEP 4: DATA PROCESSING                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BMSDataProcessor.process_submission()                              │
│       ↓                                                             │
│  Parse JSON content based on form_id                                │
│       ↓                                                             │
│  Determine observation type:                                        │
│    ┌─────────────┬───────────────┬──────────────┬─────────────┐   │
│    │ Wildlife    │ Resource Use  │ Disturbance  │ Landscape   │   │
│    │ Observation │ Incident      │ Record       │ Monitoring  │   │
│    └──────┬──────┴───────┬───────┴──────┬───────┴──────┬──────┘   │
│           ↓              ↓              ↓              ↓            │
│  Create processed record:                                           │
│    - WildlifeObservation                                            │
│    - ResourceUseIncident                                            │
│    - DisturbanceRecord                                              │
│    - LandscapeMonitoring                                            │
│       ↓                                                             │
│  Link to DataSubmission (source_submission foreign key)            │
│       ↓                                                             │
│  Extract and store:                                                 │
│    - Species information                                            │
│    - Location (PostGIS Point geometry)                              │
│    - Timestamps                                                     │
│    - Environmental conditions                                       │
│    - Media attachments                                              │
│       ↓                                                             │
│  Update DataSubmission.processing_status = "completed"             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                  STEP 5: DATA AVAILABILITY                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Processed records now available for:                               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ 1. Analytics & Dashboards                                │     │
│  │    - Species frequency analysis                          │     │
│  │    - Population trends                                   │     │
│  │    - Spatial distribution maps                           │     │
│  │    - Dashboard widgets                                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ 2. Report Generation                                     │     │
│  │    - GIS-Enhanced Reports                                │     │
│  │    - Semestral BMS Reports                               │     │
│  │    - Custom Reports                                      │     │
│  │    - Data exports (Excel, CSV, Shapefile)                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ 3. Species Management                                    │     │
│  │    - Species observation history                         │     │
│  │    - Encounter rates                                     │     │
│  │    - Distribution patterns                               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ 4. GIS Mapping                                           │     │
│  │    - Interactive map layers                              │     │
│  │    - Spatial queries                                     │     │
│  │    - Heat maps                                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data States & Transitions

#### DataSubmission Status Flow

```
validation_status field (synced with Kobo):

    NEW             validation_ok      validation_nok
    "---" ────────→ "validation_ok" / "validation_nok" / "validation_hold"
  (Pending)         (Approved)         (Rejected)          (On Hold)
     │                   │                  │                  │
     │                   │                  │                  │
     ↓                   ↓                  ↓                  ↓
 Validation      Processing         Feedback to      Awaiting
   Queue         & Analysis          PA Staff      Clarification
```

#### Processing Status Flow

```
processing_status field:

  "pending" → "processing" → "completed" / "failed"
     │            │               │            │
     │            │               │            │
     ↓            ↓               ↓            ↓
  Awaiting    Being         Successfully   Processing
  Processing  Processed     Processed      Error
```

### Choice List Synchronization Flow

```
┌──────────────────────────────────────────────────────────────┐
│           BIDIRECTIONAL CHOICE LIST SYNC                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Django Admin                    KoboToolbox                 │
│      │                                │                      │
│      │  1. Create/Edit ChoiceList     │                      │
│      ├────────────────────────────────┼──→ API Call          │
│      │                                │   (POST/PATCH)       │
│      │                                │                      │
│      │  2. Map to Kobo Form Question  │                      │
│      ├────────────────────────────────┼──→ FormChoiceMapping │
│      │     (via FormChoiceMapping)    │                      │
│      │                                │                      │
│      │  3. Sync to Kobo               │                      │
│      ├────────────────────────────────┼──→ Update Form       │
│      │                                │   Definition         │
│      │                                │                      │
│      │ ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ←│                      │
│      │  4. Fetch from Kobo (periodic) │                      │
│      │     (Management Command)       │                      │
│      │                                │                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Component Integration

### Django + PostgreSQL + PostGIS

```python
# Example: Spatial query in Django ORM
from django.contrib.gis.geos import Point
from django.contrib.gis.measure import D  # Distance

# Find all wildlife observations within 5km of a point
location = Point(125.2744, 6.5059, srid=4326)  # Lon, Lat
nearby_observations = WildlifeObservation.objects.filter(
    location__distance_lte=(location, D(km=5))
)
```

### Django + Celery + Redis

```python
# Example: Async task for KoboToolbox sync
from celery import shared_task
from api.submission_services import DataSubmissionService

@shared_task
def sync_kobotoolbox_data():
    """Periodic task to sync data from KoboToolbox"""
    service = DataSubmissionService()
    result = service.sync_all_forms()
    return result

# Celery beat schedule (periodic execution)
CELERY_BEAT_SCHEDULE = {
    'sync-kobo-every-30-minutes': {
        'task': 'api.tasks.sync_kobotoolbox_data',
        'schedule': crontab(minute='*/30'),  # Every 30 minutes
    },
}
```

### Django + Bootstrap + Leaflet.js

```html
<!-- Example: Interactive map integration -->
<div id="map" style="height: 600px;"></div>

<script>
// Initialize Leaflet map
var map = L.map('map').setView([6.5059, 125.2744], 10);

// Add tile layer
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

// Fetch observations as GeoJSON
fetch('/dashboard/api/observations-geojson/')
    .then(response => response.json())
    .then(data => {
        L.geoJSON(data, {
            onEachFeature: function(feature, layer) {
                layer.bindPopup(feature.properties.species_name);
            }
        }).addTo(map);
    });
</script>
```

### Role-Based Access Control Integration

```python
# Example: Protecting a view with role-based mixin
from dashboard.permissions import ValidatorAccessMixin

class ValidationQueueView(ValidatorAccessMixin, ListView):
    """
    Validation queue - only accessible by Validators and Admins
    """
    model = DataSubmission
    template_name = 'dashboard/validation_queue.html'
    
    def get_queryset(self):
        # Only show pending submissions
        return DataSubmission.objects.filter(
            validation_status='---'
        ).order_by('-received_date')
```

---

## 7. File Organization

### Template Organization Strategy

Templates are organized by **feature and role**:

```
templates/dashboard/
├── base.html                    # Base template (all pages extend this)
│
├── *_home.html                  # Role-specific home pages
│   ├── pa_staff_home.html
│   ├── denr_dashboard.html
│   └── collaborator_dashboard.html
│
├── validation_*.html            # Validation feature
│   ├── validation_queue.html
│   ├── validation_workspace.html
│   ├── validation_history.html
│   └── submission_audit_trail.html
│
├── my_*.html                    # PA Staff features
│   ├── my_submissions.html
│   ├── my_submission_detail.html
│   └── my_observations.html
│
├── species_*.html               # Species management
│   ├── species_list.html
│   ├── species_detail.html
│   ├── species_observations.html
│   └── species_media.html
│
├── admin/                       # Admin-only templates
│   ├── admin_dashboard.html
│   ├── user_management.html
│   ├── agency_management.html
│   ├── choice_management.html
│   └── partials/                # Reusable table components
│       ├── _users_pending_table.html
│       ├── _agencies_table.html
│       └── ...
│
├── reports/                     # Report generation
│   ├── report_builder.html
│   ├── report_archive_list.html
│   └── data_export_center.html
│
├── widgets/                     # Dashboard widgets
│   ├── counter.html
│   ├── bar_chart.html
│   ├── map.html
│   └── ...
│
├── errors/                      # Error pages
│   ├── 403.html
│   ├── 404.html
│   └── 500.html
│
└── components/                  # Reusable UI components
    ├── widgets.html
    ├── quality_score.html
    └── issue_badge.html
```

### Static Files Organization (CSS)

The CSS follows a **component-based architecture**:

```
static/dashboard/css/
├── main.css                     # Main entry point (imports all)
│
├── variables.css                # Design tokens (colors, spacing)
│
├── components/                  # Reusable components
│   ├── cards.css                # .stat-card, .data-card
│   ├── tables.css               # .data-table
│   ├── buttons.css              # .btn-action-link
│   ├── badges.css               # .badge-status
│   ├── alerts.css               # .alert--verification
│   ├── typography.css           # .page-title, .section-title
│   └── ...
│
├── layouts/                     # Layout patterns
│   ├── layouts.css              # .page-header, .dashboard-grid
│   ├── navigation.css           # Sidebar, navbar
│   └── responsive.css           # Media queries
│
└── pages/                       # Page-specific styles
    ├── validation_queue.css
    ├── pa_staff_home.css
    ├── admin_dashboard.css
    └── ...
```

**Key Design Principle**: Never duplicate common patterns. Always use existing component classes before creating new ones.

---

## 8. Design Patterns

### Service Layer Pattern

**Problem**: Keep business logic separate from views and models.

**Solution**: Create service classes that encapsulate business logic.

```python
# api/submission_services.py
class DataSubmissionService:
    """Service for managing data submissions"""
    
    def retrieve_submissions(self, asset_uid: str) -> Dict:
        """Retrieve submissions from KoboToolbox"""
        # API call logic here
        pass
    
    def create_or_update_submission(self, data: dict) -> DataSubmission:
        """Create or update submission in database"""
        # Business logic here
        pass
```

**Usage**:
```python
# In a view
service = DataSubmissionService()
submissions = service.retrieve_submissions(asset_uid='abc123')
```

### Mixin-Based Permissions

**Problem**: Apply consistent role-based access control across many views.

**Solution**: Use class-based view mixins for permission checking.

```python
# dashboard/permissions.py
class ValidatorAccessMixin(DashboardAccessMixin):
    """Only Validators and Admins can access"""
    
    def dispatch(self, request, *args, **kwargs):
        if not (request.user.is_validator() or request.user.is_system_admin()):
            return self.handle_role_denied(request)
        return super().dispatch(request, *args, **kwargs)
```

**Usage**:
```python
class ValidationQueueView(ValidatorAccessMixin, ListView):
    # View automatically protected for Validators only
    pass
```

### Repository Pattern (Implicit via Django ORM)

**Pattern**: Use Django's ORM as a repository layer.

```python
# Custom QuerySets act as repositories
class DataSubmissionQuerySet(models.QuerySet):
    def pending(self):
        return self.filter(validation_status='---')
    
    def approved(self):
        return self.filter(validation_status__icontains='ok')

class DataSubmission(models.Model):
    objects = DataSubmissionQuerySet.as_manager()
    
# Usage
pending_submissions = DataSubmission.objects.pending()
```

### Template Inheritance Pattern

**Pattern**: DRY principle for templates.

```html
<!-- base.html -->
<!DOCTYPE html>
<html>
<head>
    {% block head %}{% endblock %}
</head>
<body>
    {% include 'dashboard/navbar.html' %}
    {% block content %}{% endblock %}
    {% include 'dashboard/footer.html' %}
</body>
</html>

<!-- validation_queue.html -->
{% extends 'dashboard/base.html' %}

{% block content %}
    <h1>Validation Queue</h1>
    <!-- Page-specific content -->
{% endblock %}
```

### Widget System Pattern

**Pattern**: Modular, reusable dashboard components.

```python
# Dashboard contains many Widgets
class Dashboard(models.Model):
    name = models.CharField(max_length=255)
    user = models.ForeignKey(User)

class DashboardWidget(models.Model):
    dashboard = models.ForeignKey(Dashboard)
    widget_type = models.CharField(choices=WIDGET_CHOICES)
    configuration = models.JSONField()  # Flexible config
    position = models.IntegerField()
```

Each widget type has its own template and rendering logic.

---

## Summary

PRISM-Matutum's architecture is built on:

1. **Layered Design**: Clear separation between presentation, application, service, and data layers
2. **Service-Oriented**: Business logic encapsulated in service classes
3. **Role-Based Security**: Comprehensive permission system using mixins and decorators
4. **Modular Frontend**: Component-based CSS and reusable templates
5. **Spatial-First**: PostGIS integration for geographic queries
6. **Async Processing**: Celery for background tasks (sync, reports)
7. **External Integration**: Seamless KoboToolbox data flow

The system handles the complete lifecycle of biodiversity data: collection → validation → processing → analysis → reporting.

---

**Next Steps**: Refer to the following documents for deeper dives:
- [RBAC_DOCUMENTATION.md](RBAC_DOCUMENTATION.md) - Role-based access control details
- [PERMISSION_SYSTEM.md](PERMISSION_SYSTEM.md) - Permission decorators and mixins
- [URL_ROUTING.md](URL_ROUTING.md) - Complete URL structure

---

*Document created as part of PRISM-Matutum Capstone Project Documentation*  
*For questions or updates, refer to the project repository.*
