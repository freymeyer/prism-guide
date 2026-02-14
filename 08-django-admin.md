# Chapter 8 — Django Admin

> Django ships with a powerful auto-generated admin interface. PRISM-Matutum extends it heavily with custom actions, inline editors, map widgets, KoboToolbox integration views, and image previews. This chapter documents what's available in the admin and how it's configured.

---

## Table of Contents

- [8.1 What Is Django Admin?](#81-what-is-django-admin)
- [8.2 Admin Architecture](#82-admin-architecture)
- [8.3 API App Admin (17 models)](#83-api-app-admin-17-models)
- [8.4 Users App Admin (16 models)](#84-users-app-admin-16-models)
- [8.5 Species App Admin (33 models)](#85-species-app-admin-33-models)
- [8.6 Dashboard App Admin (9 models)](#86-dashboard-app-admin-9-models)
- [8.7 Notable Patterns](#87-notable-patterns)

---

## 8.1 What Is Django Admin?

Django Admin is an auto-generated web interface at `/admin/` for managing data. It reads your models and creates:

- **List views** — searchable, filterable tables of records
- **Detail views** — forms for viewing/editing individual records
- **Actions** — bulk operations on selected records
- **Inlines** — edit related records on the same page

To access it, you must be a **staff user** (`is_staff=True`) or **superuser** (`is_superuser=True`).

### How Admin Classes Work

```python
# admin.py (simplified example)
from django.contrib import admin
from .models import WildlifeObservation

@admin.register(WildlifeObservation)
class WildlifeObservationAdmin(admin.ModelAdmin):
    list_display = ['species', 'species_count', 'username', 'processed_at']
    list_filter = ['species_type', 'processed_at']
    search_fields = ['species', 'username']
    actions = ['reprocess_observations']

    def reprocess_observations(self, request, queryset):
        """Re-run processing on selected observations."""
        for obs in queryset:
            # reprocess logic here
            pass
    reprocess_observations.short_description = "Reprocess selected observations"
```

This registers `WildlifeObservation` with:
- A table showing species, count, username, and date
- Sidebar filters for species type and date
- A search box that searches species/username
- A "Reprocess" action in the action dropdown

---

## 8.2 Admin Architecture

The admin is split across all four apps with a total of **75 models** registered:

```
/admin/
├── api/admin/                    ← Modular package (5 submodules)
│   ├── choice_admin.py           ← Choice lists & KoboToolbox form integration
│   ├── submission_admin.py       ← Data submissions & validation history
│   ├── monitoring_admin.py       ← Processed records (wildlife, disturbance, etc.)
│   ├── location_admin.py         ← Spatial models (locations, transects, surveys)
│   └── fgd_admin.py              ← Focus group discussion sessions
│
├── users/admin.py                ← Users, profiles, organizations, geography
├── species/admin.py              ← Taxonomy hierarchy, checklists, media
└── dashboard/admin.py            ← Reports, events, system messages, logs
```

| App | Custom ModelAdmin Classes | Bulk-Registered (No Custom Admin) | Total |
|-----|--------------------------|-----------------------------------|-------|
| `api` | 17 | 0 | 17 |
| `users` | 16 | 0 | 16 |
| `species` | 17 | 16 | 33 |
| `dashboard` | 9 | 0 | 9 |
| **Total** | **59** | **16** | **75** |

---

## 8.3 API App Admin (17 models)

The `api/admin/` directory is a **Python package** — split into 5 files for maintainability. The main `api/admin.py` contains `from .admin import *` which triggers `api/admin/__init__.py` to import all submodules.

### Choice & Form Integration (`choice_admin.py`)

These models manage the bidirectional sync between Django choice lists and KoboToolbox forms.

| Model | Key Features |
|-------|-------------|
| **ChoiceList** | Shows choice count. Inline editor for choices. Actions: "Populate from registered model" (pulls species/locations from Django models), "Sync to KoboToolbox forms" |
| **Choice** | Shows linked species/location if any. Filter by list and active status |
| **FormChoiceMapping** | Links a KoboToolbox form question to a Django choice list. Actions: "Sync choices from Kobo", "Update KoboToolbox forms", "Test Kobo connection". **Custom admin views**: health check page, list Kobo assets, browse/import Kobo forms |
| **FormUpdateLog** | Read-only log of form update operations. Action: "Retry failed updates" |

> **Custom Admin Views** — `FormChoiceMappingAdmin` adds custom URLs to the admin: `/admin/api/formchoicemapping/browse-kobo-forms/` renders a full HTML page for browsing and importing KoboToolbox forms. This is unusual — most admin classes only use the built-in list/detail views.

### Data Submissions (`submission_admin.py`)

The **most heavily customized** admin in the project:

| Model | Key Features |
|-------|-------------|
| **DataSubmission** | 9 custom actions covering the full lifecycle. Filter by validation status, processing state, asset_uid. Filter-horizontal for assigned_staff. Custom admin views for sync, validation stats, and retrieval |
| **ValidationHistory** | Shows status transitions (old → new) with who and when |
| **RecordValidationHistory** | Shows validator reassignment history. All fields read-only |
| **DataRetrievalLog** | Read-only sync operation logs. Shows duration (calculated). Actions: retry failed, trigger manual sync |

#### DataSubmission Admin Actions

These are the most important actions in the entire admin:

| Action | What It Does |
|--------|-------------|
| `approve_submissions` | Sets validation_status to "approved" on selected rows |
| `process_validated_submissions` | Runs `BMSDataProcessor` on selected approved submissions |
| `commit_processed_submissions` | Runs `BMSDataCommitService` to create processed records |
| `process_and_commit_submissions` | Combines process + commit in one step |
| `reset_processing_status` | Resets `is_processed` and `database_committed` flags |
| `sync_from_kobo` | Pulls latest data from KoboToolbox API |
| `update_validation_status_kobo` | Pushes local validation status to KoboToolbox |
| `bulk_approve_and_sync` | Approves + syncs to Kobo in one step |
| `detect_orphaned_submissions` | Finds submissions with no matching Kobo record |

### Processed Records (`monitoring_admin.py`)

Four admin classes for the processed data models, all following a consistent pattern:

| Model | Special Features |
|-------|-----------------|
| **WildlifeObservation** | Color-coded species badge (linked=green, unlinked=gray). Image preview via Kobo proxy URL. Autocomplete for fauna/flora species linking. Actions: reprocess, auto-link species, validate |
| **ResourceUseIncident** | Image preview. Actions: reprocess, validate |
| **DisturbanceRecord** | Image preview. Actions: reprocess, validate, mark follow-up required |
| **LandscapeMonitoring** | Uses `LeafletGeoAdmin` (map widget for editing geometry). Actions: reprocess, validate |

> **Image Previews** — All four models have an `image_tag()` method that generates an `<img>` HTML tag. The image URL goes through a Kobo proxy: `/api/kobo-image/?url=<kobo-attachment-url>`, which authenticates with the KoboToolbox API and streams the image.

### Spatial/Location Models (`location_admin.py`)

| Model | Special Features |
|-------|-----------------|
| **MonitoringLocation** | `LeafletGeoAdmin` with map centered on Mount Matutum (zoom 18) |
| **TransectRoute** | `LeafletGeoAdmin` with custom inline map preview showing the route line, station markers, and a legend. Actions: regenerate geometry from stations, generate transect segments |
| **TransectSegment** | Read-only `length_meters` and observation count (queries 4 model types). Inline map preview showing segment path |
| **ScheduledSurvey** | Filter-horizontal for staff and submissions. Actions: assign matching submissions (by time range), mark in-progress/completed, link submissions by date |

### Focus Group Discussions (`fgd_admin.py`)

| Model | Special Features |
|-------|-----------------|
| **FGDSession** | Three tabular inlines (participants, findings, attachments) on the same page. Auto-sets `created_by` on save |
| **FGDParticipant** | `raw_id_fields` for session and barangay (faster than dropdown for large datasets) |
| **FGDFinding** | `filter_horizontal` for related species (fauna and flora). Auto-sets `recorded_by` on save |
| **FGDAttachment** | Auto-sets `uploaded_by` on save |

---

## 8.4 Users App Admin (16 models)

### User Management

| Model | Key Features |
|-------|-------------|
| **User** | Extends Django's `BaseUserAdmin`. Shows role, account status (colored HTML badge), agency, years of service, field verification status. Custom fieldsets add organization, 2FA, account approval sections. Actions: approve users, reject users (both check `can_manage_users()` permission) |
| **Profile** | Basic profile fields |
| **PAStaffProfile** | Filterable by training_completed, specialization |
| **ValidatorProfile** | Shows validation statistics (total, approved, rejected, pending counts) |

### Organizations

| Model | Key Features |
|-------|-------------|
| **Agency** | Simple list with search |
| **Department** | Simple list with search, filter by agency |

### Geographic Hierarchy

| Model | Key Features |
|-------|-------------|
| **Country → Region → Province → Municipality → Barangay** | Each level has search fields and filters by parent level. Simple CRUD admin |
| **Address** | Joins all levels together |
| **ProtectedArea** | Uses `GISModelAdmin` (OpenLayers map widget) with default center on Mount Matutum. Fieldsets for basic info + geographic boundary drawing |
| **Landscape** | Simple admin |

### Activity & Notifications

| Model | Key Features |
|-------|-------------|
| **UserActivityLog** | List display with user, action, timestamp |
| **UserNotification** | List display with user, notification type, read status |

---

## 8.5 Species App Admin (33 models)

### Taxonomy Hierarchy (7 models)

`Kingdom`, `Phylum`, `TaxonomyClass`, `TaxonomyOrder`, `Family`, `Genus`, `Taxonomy` — each with basic list/search. The hierarchy chain allows drilling down from Kingdom to individual taxonomic entries.

### Conservation Status (2 models)

| Model | Key Features |
|-------|-------------|
| **Endemicity** | Endemic status options (endemic, native, introduced, etc.) |
| **ThreatAssessment** | IUCN threat categories |

### Characteristics (10 models)

`FaunalCharacteristics`, `FloralCharacteristics` — Main characteristic models with custom admin.

**8 Flora Lookup Models** (bulk-registered with no custom admin):
`LeafType`, `LeafShape`, `LeafMargin`, `LeafApex`, `LeafBase`, `StemType`, `FlowerType`, `FruitType`

### Species Checklists (2 models)

| Model | Key Features |
|-------|-------------|
| **FaunaChecklist** | Shows scientific name, taxonomy, endemicity, threat category, recorded population. Inline tabular editor for common names. Search by scientific name |
| **FloraChecklist** | Same pattern as Fauna. Inline for flora common names |

### Relationships (8 models, all bulk-registered)

`FaunaCommonName`, `FloraCommonName`, `RelatedFaunaResearch`, `RelatedFloraResearch`, `FaunaRelatedArticle`, `FloraRelatedArticle`, `FaunaFieldMedia`, `FloraFieldMedia`

### Content & Media (4 models)

| Model | Key Features |
|-------|-------------|
| **SpeciesCommonName** | Common names in different languages |
| **RelatedResearch** | Academic research references |
| **Articles** | News/blog articles about species |
| **FieldMedia** | **Richest admin in species app**. Colored download status badge (green=downloaded, yellow=pending, red=failed). Human-readable file size (KB/MB). Image preview (thumbnail). Actions: download images (queues Celery task), retry failed downloads, reset download status |

---

## 8.6 Dashboard App Admin (9 models)

### Reports (4 models)

| Model | Key Features |
|-------|-------------|
| **ReportTemplate** | Template configuration for report generation |
| **GeneratedReport** | Shows report type, status, requester, file count, total file size. Date hierarchy on requested_at. Fieldsets cover date range, geographic scope, configuration, files (PDF/Excel/GeoJSON/Shapefile/Markdown), access control (is_public, allowed_users) |
| **ReportArchive** | Archived report storage |
| **ReportDownloadLog** | Tracks who downloaded what report and when |

### Communication (2 models)

| Model | Key Features |
|-------|-------------|
| **SystemMessage** | Targeting: all users, specific roles, or specific users (filter_horizontal). Delivery: email and/or notification, with expiration date |
| **MessageRead** | Tracks which users have read which messages |

### Security & Logging (2 models)

| Model | Key Features |
|-------|-------------|
| **SecurityEvent** | Shows event type, user, timestamp, risk level, reviewed status, IP address. Fieldsets include context (user agent, details) and review section (reviewed by/at/notes) |
| **ErrorLog** | Shows severity level, module, function, timestamp, user, resolved status. Fieldsets: error info (message, traceback), source (module, function, line), context (URL, method, request data), resolution |

### Events (1 model)

| Model | Key Features |
|-------|-------------|
| **Event** | Shows title, type, datetime, location, organizer, public flag, attendee count. Filter-horizontal for attendees. Date hierarchy on start_datetime |

---

## 8.7 Notable Patterns

### LeafletGeoAdmin vs GISModelAdmin

Two different map widgets are used in admin:

| Widget | Models | Map Library | Use Case |
|--------|--------|-------------|----------|
| `LeafletGeoAdmin` | MonitoringLocation, TransectRoute, TransectSegment, LandscapeMonitoring | Leaflet.js | Drawing/editing points, lines, polygons on an interactive map |
| `GISModelAdmin` | ProtectedArea | OpenLayers | Drawing MultiPolygon boundaries |

Both provide map-based geometry editing instead of raw coordinate text fields.

### Custom Admin Views

`FormChoiceMappingAdmin` extends the admin with full custom views:

- `/admin/api/formchoicemapping/kobo-health-check/` — Tests KoboToolbox API connectivity
- `/admin/api/formchoicemapping/list-kobo-assets/` — Lists all forms in KoboToolbox
- `/admin/api/formchoicemapping/browse-kobo-forms/` — Full HTML browser for KoboToolbox forms
- `/admin/api/formchoicemapping/import-kobo-form/<uid>/` — Import a form configuration

### Celery Tasks from Admin Actions

Several admin actions dispatch tasks to Celery instead of running synchronously:

| Admin | Action | Celery Task |
|-------|--------|-------------|
| `FieldMediaAdmin` | "Download images" | `download_multiple_images` |
| `DataSubmissionAdmin` | "Process validated submissions" | Uses `BMSDataProcessor` (inline, not via Celery) |
| `DataRetrievalLogAdmin` | "Trigger manual sync" | `sync_all_kobotoolbox_data` |

### Inline Admins

Inline admins let you edit related records on the parent's page:

| Parent Model | Inline Model | Type |
|-------------|-------------|------|
| ChoiceList | Choice | TabularInline |
| FaunaChecklist | FaunaCommonName | TabularInline |
| FloraChecklist | FloraCommonName | TabularInline |
| FGDSession | FGDParticipant | TabularInline |
| FGDSession | FGDFinding | TabularInline |
| FGDSession | FGDAttachment | TabularInline |

### Read-Only Admins

Some models are logging/audit models that should not be manually edited:

| Model | Read-Only Fields |
|-------|-----------------|
| `FormUpdateLog` | All fields (no add/change permissions) |
| `DataRetrievalLog` | All fields |
| `RecordValidationHistory` | All fields |
| `ValidationHistory` | All fields |

### Auto-Set Fields on Save

Several admin classes auto-populate fields:

```python
def save_model(self, request, obj, form, change):
    if not change:  # Only on creation, not edit
        obj.created_by = request.user
    super().save_model(request, obj, form, change)
```

Models using this pattern: `FGDSession` (created_by), `FGDFinding` (recorded_by), `FGDAttachment` (uploaded_by)

---

**Previous:** [← Chapter 7: Configuration & Deployment](07-configuration-deployment.md)

**Back to:** [Index](README.md)
