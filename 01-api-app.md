# Chapter 1 — The `api` App

> The `api` app is the **data backbone** of PRISM-Matutum. It defines all the core database models, handles communication with KoboToolbox, processes raw submissions into structured records, and exposes REST API endpoints. Everything else in the system reads from or writes to models defined here.

---

## Table of Contents

- [1.1 Why Split Models Across Multiple Files?](#11-why-split-models-across-multiple-files)
- [1.2 Models Walkthrough](#12-models-walkthrough)
  - [DataSubmission — The Starting Point](#datasubmission--the-starting-point)
  - [BaseProcessedSubmission — The Abstract Base](#baseprocessedsubmission--the-abstract-base)
  - [Processed Record Types](#processed-record-types)
  - [Choice System](#choice-system)
  - [Monitoring & Spatial Models](#monitoring--spatial-models)
  - [Scheduling](#scheduling)
  - [Focus Group Discussions (FGD)](#focus-group-discussions-fgd)
  - [Observation Media — Generic Relations](#observation-media--generic-relations)
  - [Validation History](#validation-history)
  - [Dashboard Models (Deprecated)](#dashboard-models-deprecated)
- [1.3 The Service Layer](#13-the-service-layer)
  - [DataSubmissionService](#datasubmissionservice)
  - [BMSDataProcessor](#bmsdataprocessor)
  - [BMSDataCommitService](#bmsdatacommitservice)
  - [BMSWorkflowService](#bmsworkflowservice)
  - [Form Services](#form-services)
- [1.4 Signals — Auto-Processing Pipeline](#14-signals--auto-processing-pipeline)
- [1.5 REST API Endpoints](#15-rest-api-endpoints)
- [1.6 Serializers](#16-serializers)
- [1.7 Celery Tasks](#17-celery-tasks)
- [1.8 Management Commands](#18-management-commands)

---

## 1.1 Why Split Models Across Multiple Files?

### Django Concept: Models

A Django **model** is a Python class that represents a database table. Each class attribute defines a column:

```python
class WildlifeObservation(models.Model):
    species = models.CharField(max_length=255)      # → VARCHAR(255) column
    species_count = models.IntegerField(null=True)   # → INTEGER column, nullable
    fauna_species = models.ForeignKey(               # → Foreign key to another table
        FaunaChecklist, null=True, on_delete=models.PROTECT
    )
```

When you run `python manage.py makemigrations` and `python manage.py migrate`, Django auto-generates SQL to create these tables.

### Django Concept: Split Models (`models/` package)

By default, Django expects models in a single `models.py` file. But PRISM-Matutum has **15+ model files** because the `api` app has a lot of models. To split them:

1. Create a `models/` directory with an `__init__.py`
2. Put model classes in separate files (e.g., `submissions.py`, `records_wildlife.py`)
3. Import everything in `__init__.py`:

```python
# api/models/__init__.py
from .choices import *
from .submissions import *
from .processed_base import *
from .records_wildlife import *
from .records_resourceuse import *
from .records_disturbance import *
from .records_landscape import *
# ... and so on
```

Django doesn't care which file a model lives in — it only needs to find it via the `__init__.py` imports.

### The Model Files

| File | What It Contains |
|------|-----------------|
| `submissions.py` | `DataSubmission`, `DataRetrievalLog` — raw KoboToolbox data |
| `processed_base.py` | `BaseProcessedSubmission` (abstract), `ProcessedSubmission` (log) |
| `records_wildlife.py` | `WildlifeObservation` — wildlife sighting records |
| `records_resourceuse.py` | `ResourceUseIncident` — resource harvesting incidents |
| `records_disturbance.py` | `DisturbanceRecord` — environmental disturbances |
| `records_landscape.py` | `LandscapeMonitoring` — landscape monitoring records |
| `choices.py` | `ChoiceList`, `Choice` — dynamic form choice options |
| `forms.py` | `FormChoiceMapping`, `FormUpdateLog` — Kobo form mappings |
| `monitoring.py` | `MonitoringLocation`, `TransectRoute` — spatial/GIS entities |
| `transect_segments.py` | `TransectSegment` — path segments between stations |
| `schedules.py` | `ScheduledSurvey` — survey scheduling and assignments |
| `fgd.py` | `FGDSession`, `FGDParticipant`, `FGDFinding`, `FGDAttachment` |
| `observation_media.py` | `ObservationMedia`, `ObservationType`, `ObservationRelatedArticle` |
| `validation_history.py` | `ValidationHistory`, `RecordValidationHistory` — audit trails |
| `dashboards.py` | `Dashboard`, `DashboardWidget`, `DashboardTemplate`, `WidgetDataCache` |

---

## 1.2 Models Walkthrough

### DataSubmission — The Starting Point

**File:** `api/models/submissions.py`

Every piece of field data enters the system as a `DataSubmission`. When a PA Staff member fills out a survey on their phone using KoboToolbox, the data is synced here.

```
KoboToolbox Mobile App  ───►  KoboToolbox Server  ───►  DataSubmission
```

**Key fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `asset_uid` | CharField | Identifies which KoboToolbox form this came from |
| `submission_id` | IntegerField | KoboToolbox's ID for this submission |
| `submission_uuid` | CharField (unique) | Globally unique identifier — prevents duplicates |
| `form_data` | JSONField | The **complete raw submission** data from KoboToolbox — this is the original payload that never gets modified |
| `username` | CharField | Who submitted it in KoboToolbox |
| `validation_status` | CharField | Current stage in the validation pipeline (see below) |
| `validated_by` | ForeignKey → User | Which validator reviewed it |
| `is_processed` | BooleanField | Has it been converted to structured records? |
| `database_committed` | BooleanField | Have the processed records been saved to the database? |
| `assigned_staff` | ManyToMany → User | PA Staff assigned via scheduled surveys |

**The six validation statuses:**

```
pending_pa_review  →  ---  →  approved
                             →  not_approved
                             →  on_hold
                             →  flagged
```

1. `pending_pa_review` — Just synced from KoboToolbox, waiting for PA Staff self-review
2. `---` — PA Staff reviewed, now waiting for Validator review
3. `approved` — Validator approved — this triggers auto-processing via signals
4. `not_approved` — Validator rejected
5. `on_hold` — Validator needs more information
6. `flagged` — Marked for special attention

**Related model:** `DataRetrievalLog` tracks every sync operation (how many submissions found, new, updated, errors).

---

### BaseProcessedSubmission — The Abstract Base

**File:** `api/models/processed_base.py`

### Django Concept: Abstract Base Classes

An **abstract model** is a model that is never saved to the database directly. Instead, other models **inherit** from it to share common fields:

```python
class BaseProcessedSubmission(models.Model):
    # These fields appear in EVERY processed record type
    submission = models.OneToOneField(DataSubmission, ...)
    latitude = models.FloatField(null=True)
    longitude = models.FloatField(null=True)
    username = models.CharField(max_length=255)
    processed_at = models.DateTimeField(default=timezone.now)
    # ... 20+ common fields

    class Meta:
        abstract = True  # ← This makes it abstract
```

When `WildlifeObservation` or `DisturbanceRecord` inherits from `BaseProcessedSubmission`, they get all those fields automatically — no copy-pasting needed.

**Key fields shared by all processed records:**

| Field | Type | Purpose |
|-------|------|---------|
| `submission` | OneToOneField → DataSubmission | Links back to the raw submission |
| `latitude` / `longitude` | FloatField | GPS coordinates from the field |
| `location_accuracy` | FloatField | GPS accuracy in meters |
| `transect_segment` | ForeignKey → TransectSegment | Which path segment this observation is near |
| `protected_area` | ForeignKey → ProtectedArea | Which protected area contains this point |
| `collected_by` | ForeignKey → User | PA Staff who collected the data |
| `validated_by` | ForeignKey → User | Validator who approved it |
| `kobo_id` | CharField | Original KoboToolbox submission ID |
| `image` | URLField | Link to field photo in KoboToolbox |
| `remarks` | TextField | Field notes |
| `observation_media` | GenericRelation | Links to species media (via ContentType) |
| `observation_articles` | GenericRelation | Links to related research articles |

**Important methods:**
- `assign_transect_segment(buffer_meters=50)` — Uses PostGIS spatial queries to find the nearest transect segment within a buffer distance
- `sync_to_kobo(validation_status)` — Pushes validation status changes back to KoboToolbox
- `get_location_display()` — Returns human-readable location (Barangay, Municipality, Province)

---

### Processed Record Types

These are the **four types of field observations** that PRISM-Matutum tracks. Each inherits from `BaseProcessedSubmission`:

#### WildlifeObservation (`records_wildlife.py`)

Wildlife sighting records. This is the most complex record type because it links to the species taxonomy system.

| Field | Type | Purpose |
|-------|------|---------|
| `species` | CharField | Species name as written in the field |
| `species_type` | CharField | Category (bird, mammal, reptile, etc.) |
| `species_count` | IntegerField | Number of individuals observed |
| `fauna_species` | ForeignKey → FaunaChecklist | Linked fauna species (auto-matched) |
| `flora_species` | ForeignKey → FloraChecklist | Linked flora species (auto-matched) |
| `observation_types` | ManyToMany → ObservationType | How was it observed? (seen, heard, traces, etc.) |
| `species_details` | TextField | Additional details from the field |

**Database constraint:** Only one of `fauna_species` or `flora_species` can be set (never both). This is enforced with a `CheckConstraint`.

**Auto-linking on save:** When a `WildlifeObservation` is saved, its `save()` method automatically:
1. Tries to match the `species` name to a `FaunaChecklist` or `FloraChecklist` entry
2. Links any matching field media (photos)
3. Links any related research articles

#### ResourceUseIncident (`records_resourceuse.py`)

Records of resource harvesting, hunting, or other human use of natural resources.

| Field | Type | Purpose |
|-------|------|---------|
| `resource_use_type` | CharField | Type of resource use (hunting, harvesting, etc.) |
| `resource_use_details` | TextField | Description of the incident |
| `action_taken` | TextField | What was done about it |

#### DisturbanceRecord (`records_disturbance.py`)

Environmental disturbances like traps, litter, illegal activities, etc.

| Field | Type | Purpose |
|-------|------|---------|
| `disturbance_type` | CharField | Type of disturbance |
| `disturbance_details` | TextField | Description |
| `action_taken` | TextField | Response taken |
| `follow_up_required` | BooleanField | Does this need follow-up? |

#### LandscapeMonitoring (`records_landscape.py`)

General landscape monitoring records tied to specific monitoring locations.

| Field | Type | Purpose |
|-------|------|---------|
| `monitoring_location` | ForeignKey → MonitoringLocation | The station being monitored |
| `monitoring_details` | TextField | Monitoring observations |

---

### Choice System

**Files:** `api/models/choices.py`, `api/models/forms.py`

The choice system is a **bidirectional sync** between Django and KoboToolbox. When you add a new species to the Django database, that species name can automatically appear as a dropdown option in the KoboToolbox mobile form.

#### How It Works

```
Django (ChoiceList + Choice models)
        │
        │  sync_form_choices_to_database()   ← Kobo → Django
        │  process_form_with_choice_updates() ← Django → Kobo
        ▼
KoboToolbox (XLSForm select_one/select_multiple options)
```

#### ChoiceList

A named collection of options (like "species_list" or "disturbance_types").

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField (unique) | Internal identifier (e.g., `"fauna_species"`) |
| `description` | TextField | Human-readable description |

#### Choice

A single option within a choice list.

| Field | Type | Purpose |
|-------|------|---------|
| `choice_list` | ForeignKey → ChoiceList | Which list this belongs to |
| `name` | CharField | Value stored in form data (e.g., `"philippine_eagle"`) |
| `label` | CharField | Display text (e.g., `"Philippine Eagle"`) |
| `order` | IntegerField | Sort position |
| `active` | BooleanField | Can be deactivated without deleting |
| `fauna_species` | ForeignKey → FaunaChecklist | Optional link to actual species record |
| `flora_species` | ForeignKey → FloraChecklist | Optional link to actual species record |
| `monitoring_location` | ForeignKey → MonitoringLocation | Optional link to location |

#### FormChoiceMapping

Maps a specific question in a KoboToolbox form to a Django choice list.

| Field | Type | Purpose |
|-------|------|---------|
| `asset_uid` | CharField | Which Kobo form |
| `question_name` | CharField | Which question in that form |
| `choice_list` | ForeignKey → ChoiceList | Which Django choice list provides the options |
| `auto_update` | BooleanField | Should this sync automatically? |

#### FormUpdateLog

Tracks every time a form is updated — for auditing.

---

### Monitoring & Spatial Models

**File:** `api/models/monitoring.py`, `api/models/transect_segments.py`

### Django Concept: PostGIS and Spatial Fields

PRISM-Matutum uses **PostGIS** (the spatial extension for PostgreSQL) and **GeoDjango** (`django.contrib.gis`). This lets you store geographic coordinates as proper spatial objects and do queries like "find all observations within 50 meters of this transect."

```python
from django.contrib.gis.db import models as gis_models

class MonitoringLocation(models.Model):
    geometry = gis_models.PointField(srid=4326, null=True)  # Lat/Lng point
```

`SRID 4326` means WGS 84 — the standard coordinate system used by GPS.

#### MonitoringLocation

Physical locations used for monitoring — transect stations, facilities, etc.

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField (unique) | Human-readable name |
| `location_type` | CharField | `facility`, `transect_station`, or `other` |
| `geometry` | PointField (SRID 4326) | PostGIS point geometry |
| `is_active` | BooleanField | Active/inactive status |

#### TransectRoute

A monitoring route that connects multiple stations in sequence. PA Staff walk these routes during surveys.

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField | Route name |
| `stations` | ManyToMany → MonitoringLocation | Stations along this route (limited to `transect_station` type) |
| `route_geometry` | LineStringField (SRID 4326) | The path connecting stations |
| `protected_area` | ForeignKey → ProtectedArea | Which protected area this route is in |

**Methods:**
- `update_geometry_from_stations()` — Auto-generates the `LineString` geometry by connecting station points in order
- `generate_segments()` — Creates `TransectSegment` records between consecutive stations

#### TransectSegment

An individual path segment between two consecutive stations on a route.

| Field | Type | Purpose |
|-------|------|---------|
| `transect_route` | ForeignKey → TransectRoute | Parent route |
| `start_station` / `end_station` | ForeignKey → MonitoringLocation | The two endpoints |
| `segment_name` | CharField | Auto-generated (e.g., "Station 1 to Station 2") |
| `sequence_order` | IntegerField | Position in the route |
| `segment_geometry` | LineStringField (SRID 4326) | The line between start and end |
| `length_meters` | FloatField | Calculated distance |

**Key method:** `contains_point(lat, lon, buffer_meters=50)` — Uses PostGIS to check if a GPS coordinate falls within a buffer zone around this segment. This is how observations are automatically assigned to transect segments.

---

### Scheduling

**File:** `api/models/schedules.py`

#### ScheduledSurvey

Assigns PA Staff to walk transect routes at specific times.

| Field | Type | Purpose |
|-------|------|---------|
| `transect_route` | ForeignKey → TransectRoute | Which route to walk |
| `assigned_staff` | ManyToMany → User | Who is assigned |
| `start_time` / `end_time` | DateTimeField | When the survey should happen |
| `status` | CharField | `scheduled`, `in_progress`, `completed`, `overdue`, `cancelled` |
| `related_submissions` | ManyToMany → DataSubmission | Submissions made during this survey |

**Methods:**
- `check_and_update_status()` — Auto-transitions between statuses based on current time
- `assign_submissions_within_time_range()` — Finds and links submissions that occurred during the survey window

---

### Focus Group Discussions (FGD)

**File:** `api/models/fgd.py`

FGDs are structured community meetings where stakeholders discuss biodiversity issues. The system tracks sessions, participants, and findings.

#### FGDSession

| Field | Type | Purpose |
|-------|------|---------|
| `title` | CharField | Session title |
| `date_conducted` | DateField | When it happened |
| `location` | CharField | Venue name |
| `protected_area` | ForeignKey → ProtectedArea | Which PA this relates to |
| `facilitator` | ForeignKey → User | Session leader |
| `co_facilitators` | ManyToMany → User | Additional facilitators |
| `status` | CharField | `scheduled`, `in_progress`, `completed`, `cancelled` |
| `objectives` / `methodology` / `notes` / `summary` | TextField | Documentation |
| `audio_recording` / `consent_form` | FileField | Uploaded files |

#### FGDParticipant

Tracks participant demographics (with privacy options — name is optional).

| Field | Type | Purpose |
|-------|------|---------|
| `gender` | CharField | `male`, `female`, `other`, `prefer_not_to_say` |
| `age_group` | CharField | 7 age brackets |
| `stakeholder_type` | CharField | 10 types: farmer, fisher, indigenous community, teacher, etc. |
| `occupation` | CharField | Job description |

#### FGDFinding

Key findings/themes from the discussion, with optional species links and actionability tracking.

| Field | Type | Purpose |
|-------|------|---------|
| `category` | CharField | 9 categories: resource_use, threat, recommendation, biodiversity, livelihood, etc. |
| `priority` | CharField | `high`, `medium`, `low` |
| `is_actionable` | BooleanField | Can something be done about this? |
| `recommended_action` | TextField | What should be done |
| `related_fauna_species` / `related_flora_species` | ManyToMany | Species mentioned in the finding |
| `follow_up_required` / `follow_up_completed` | BooleanField | Tracking follow-up actions |

---

### Observation Media — Generic Relations

**File:** `api/models/observation_media.py`

### Django Concept: Generic Relations (ContentType Framework)

Sometimes you need to link one model to **any of several** other models. Instead of creating separate junction tables for `WildlifeObservationMedia`, `DisturbanceRecordMedia`, etc., Django's ContentType framework lets you create **one** table that can point to any model:

```python
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType

class ObservationMedia(models.Model):
    content_type = models.ForeignKey(ContentType, ...)  # "Which model?"
    object_id = models.PositiveIntegerField()            # "Which row?"
    observation = GenericForeignKey('content_type', 'object_id')  # Combined
    field_media = models.ForeignKey(FieldMedia, ...)     # The actual media
```

This means `ObservationMedia` can link a `FieldMedia` (photo, video) to a `WildlifeObservation`, `DisturbanceRecord`, or any other model — without separate tables.

#### ObservationType

Lookup table for how wildlife was observed:

| Code | Name |
|------|------|
| `seen` | Directly Seen |
| `heard` | Heard (vocalization) |
| `traces` | Traces/Signs |
| `captured` | Captured |
| ... | etc. |

#### ObservationRelatedArticle

Same pattern as `ObservationMedia` but links observations to research articles instead of media.

---

### Validation History

**File:** `api/models/validation_history.py`

Two audit trail models:

#### ValidationHistory
Tracks status changes on `DataSubmission` (the raw submission):

| Field | Type | Purpose |
|-------|------|---------|
| `submission` | ForeignKey → DataSubmission | Which submission |
| `previous_status` / `new_status` | CharField | The status change |
| `changed_by` | ForeignKey → User | Who made the change |
| `change_reason` | TextField | Why |
| `remarks` | TextField | Additional notes |

#### RecordValidationHistory
Tracks validation changes on **processed records** (using generic relations — can point to any record type):

| Field | Type | Purpose |
|-------|------|---------|
| `content_type` / `object_id` | GenericFK fields | Points to any record type |
| `previous_validator` / `new_validator` | ForeignKey → User | Validator change |
| `changes` | JSONField | What fields were changed |

---

### Dashboard Models (Deprecated)

**File:** `api/models/dashboards.py`

These models (`Dashboard`, `DashboardWidget`, `DashboardTemplate`, `WidgetDataCache`) were part of a **custom dashboard builder** that allowed users to create their own widget-based dashboards. The CRUD routes for these have been **deprecated** and replaced with static, role-based dashboards. The models remain in the codebase for migration integrity — they still exist in the database but the UI routes to manage them are disabled.

---

## 1.3 The Service Layer

### Django Concept: Service Layer Pattern

In a typical Django project, you might put business logic directly in views:

```python
# ❌ Anti-pattern: logic in the view
def approve_submission(request, pk):
    submission = DataSubmission.objects.get(pk=pk)
    submission.validation_status = 'approved'
    submission.save()
    # Process it...
    # Send notifications...
    # 50 more lines of logic...
```

PRISM-Matutum uses a **service layer** instead. Business logic lives in dedicated `*_services.py` files, and views just call service methods:

```python
# ✅ Service pattern: logic in a service class
class BMSWorkflowService:
    def validate_submission(self, submission_id, validator_user, decision, remarks):
        # All the complex logic is here
        ...

# View just calls the service
def approve_submission(request, pk):
    service = BMSWorkflowService()
    success, message, data = service.validate_submission(pk, request.user, 'approved', remarks)
```

**Why?** The same logic can be called from views, Celery tasks, signals, management commands, or tests — without duplicating code.

### DataSubmissionService

**File:** `api/submission_services.py` (~800 lines)

Handles all communication with the KoboToolbox API and manages submissions in the local database.

| Method | What It Does |
|--------|-------------|
| `retrieve_submissions(asset_uid)` | Calls the KoboToolbox API to fetch submissions for a form |
| `sync_submissions_to_database(asset_uid)` | Syncs all submissions from Kobo → local DB, creating new `DataSubmission` records and updating existing ones. Returns counts (new, updated, skipped) |
| `update_validation_status(asset_uid, submission_id, status, user, remarks)` | Validates a submission — updates both the local database and KoboToolbox via API |
| `bulk_update_validation_status(...)` | Same as above but for multiple submissions at once |
| `get_pa_staff_submissions(username, status)` | Get submissions filtered by PA Staff user and status |
| `get_validation_statistics(asset_uid)` | Counts submissions by status (pending, approved, rejected, etc.) |
| `sync_processed_submission_status(...)` | After processing, syncs the status back to KoboToolbox with retry logic |

### BMSDataProcessor

**File:** `api/data_processing_services.py` (~1,800 lines)

Converts raw KoboToolbox JSON data into structured Django model records.

| Method | What It Does |
|--------|-------------|
| `process_validated_submission(submission)` | The main entry point. Takes a `DataSubmission`, reads its `form_data` JSON, determines the record type (wildlife, resource use, disturbance, landscape), and extracts all fields |
| `_flatten_form_data(form_data)` | KoboToolbox groups questions with prefixes like `group_wildlife/species_name`. This flattens them to just `species_name` |
| `_extract_common_data(submission, form_data)` | Extracts fields shared by all record types (GPS, dates, personnel) |
| `_process_wildlife_data(form_data, common_data)` | Extracts wildlife-specific fields |
| `_process_resource_use_data(...)` | Resource use fields |
| `_process_disturbance_data(...)` | Disturbance fields |
| `_process_landscape_data(...)` | Landscape monitoring fields |
| `_attempt_species_link(species_name, species_type)` | Tries to match a text species name to a `FaunaChecklist` or `FloraChecklist` entry |

### BMSDataCommitService

**File:** `api/data_processing_services.py` (same file, second class)

Takes the processed data from `BMSDataProcessor` and saves it to the database.

| Method | What It Does |
|--------|-------------|
| `commit_validated_submissions(submission_ids, user)` | Process + commit multiple submissions in a single database transaction (`@transaction.atomic`) |
| `_commit_to_database(submission, processed_data)` | Creates the appropriate model instance (`WildlifeObservation`, etc.) and saves it. Also auto-links species if possible |

### BMSWorkflowService

**File:** `api/workflow_services.py` (~340 lines)

Orchestrates the complete data workflow. This is the "conductor" that calls the other services in sequence.

| Method | What It Does |
|--------|-------------|
| `retrieve_new_submissions(asset_uid)` | Step 1: Pull new data from KoboToolbox |
| `validate_submission(submission_id, validator, decision, remarks)` | Step 2: Approve/reject/hold a submission |
| `notify_pa_staff_on_hold(submission_id)` | Step 3a: Notify PA Staff about on-hold submissions |
| `resubmit_edited_submission(submission_id, updated_data)` | Step 3b: PA Staff edits and resubmits |
| `commit_validated_data_to_database(submission_id)` | Step 4: Commit approved data to DB |
| `get_workflow_status(asset_uid)` | Get submission counts by stage |
| `bulk_process_workflow(asset_uid)` | Process all pending submissions at once |

### Form Services

**File:** `api/form_services.py` (~270 lines)

Manages KoboToolbox forms and choice synchronization.

| Function | What It Does |
|----------|-------------|
| `list_assets()` | List all KoboToolbox forms (assets) |
| `get_asset_details(asset_uid)` | Get details about a specific form |
| `download_asset_xls(asset_uid)` | Download the XLS form file for a Kobo asset |
| `upload_new_form(asset_uid, file_content, filename)` | Replace a Kobo form with a new XLSForm |
| `health_check()` | Test KoboToolbox API connectivity |
| `process_form_with_choice_updates(asset_uid)` | Full workflow: download form → update choices from Django database → re-upload → redeploy |
| `sync_form_choices_to_database(asset_uid)` | Reverse direction: download form → extract choices → save to Django database |

---

## 1.4 Signals — Auto-Processing Pipeline

**File:** `api/signals.py`

### Django Concept: Signals

Django **signals** let you run code automatically when certain events happen. The two most common:

- **`pre_save`** — Runs before a model instance is saved to the database
- **`post_save`** — Runs after a model instance is saved

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=DataSubmission)
def auto_process_validated_submission(sender, instance, **kwargs):
    # This runs every time a DataSubmission is saved
    if instance.validation_status == 'approved' and not instance.is_processed:
        # Automatically process it!
        ...
```

### The Auto-Processing Pipeline

This is one of the most important pieces of the system. When a Validator approves a submission, **three signal handlers fire in sequence**:

#### Signal 1: `track_validation_status_change` (pre_save)

Before the submission is saved, this signal checks if the `validation_status` field is about to change. If so, it stores the old value on the instance for later use.

#### Signal 2: `auto_process_validated_submission` (post_save)

After the submission is saved with `validation_status = 'approved'`, this signal:

1. Creates a `BMSDataProcessor` instance
2. Calls `process_validated_submission()` to extract structured data
3. Creates a `BMSDataCommitService` instance
4. Calls `_commit_to_database()` to save the processed records
5. Marks the submission as `is_processed = True` and `database_committed = True`

**This is the "magic" that makes approved submissions automatically appear as `WildlifeObservation`, `ResourceUseIncident`, etc. records — no manual step needed.**

#### Signal 3: `log_validation_status_change` (post_save)

After any status change, creates a `ValidationHistory` record for the audit trail.

#### Other Signals

- **Record validation handlers** — Log changes when processed records (WildlifeObservation etc.) are re-validated
- **`schedule_post_save_handler`** — Auto-updates schedule status when ScheduledSurvey is modified
- **`processed_submission_post_save_handler`** — Auto-assigns transect segments to new records and syncs status back to KoboToolbox

---

## 1.5 REST API Endpoints

**File:** `api/urls.py`

### Django Concept: Django REST Framework (DRF)

DRF extends Django to easily build JSON APIs. The key components:

- **Serializers** — Define what data goes in/out of the API (like forms but for JSON)
- **API Views** — Handle HTTP methods (GET, POST, PUT, DELETE) and return JSON responses
- **URL patterns** — Map URLs like `/api/submissions/` to view classes

### Endpoint Reference

All endpoints are under the `/api/` prefix:

#### KoboToolbox Integration

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/api/kobo/assets/` | List all KoboToolbox forms |
| GET | `/api/kobo/assets/<uid>/` | Get details of a specific form |
| GET | `/api/kobo/health/` | Check KoboToolbox API connectivity |
| POST | `/api/kobo/replace-form/` | Upload a replacement XLS form |
| GET | `/api/kobo/download-form/` | Download form as XLS file |
| POST | `/api/kobo/update-choices/` | Push Django choices → KoboToolbox form |
| POST | `/api/kobo/sync-choices/` | Pull KoboToolbox choices → Django database |
| GET | `/api/kobo/update-logs/` | View form update history |
| GET | `/api/kobo-image/` | Proxy for KoboToolbox media URLs (avoids CORS) |

#### Choice Management

| Method | URL | Purpose |
|--------|-----|---------|
| GET/POST | `/api/choices/lists/` | List/create choice lists |
| GET/PUT/DELETE | `/api/choices/lists/<pk>/` | Get/update/delete a choice list |
| GET/POST | `/api/choices/mappings/` | List/create form-choice mappings |

#### Submissions

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/api/submissions/` | List all submissions (with filters) |
| GET | `/api/submissions/<pk>/` | Get a specific submission |
| GET | `/api/submissions/stats/` | Get validation statistics |
| POST | `/api/submissions/sync/` | Trigger sync from KoboToolbox |

#### Validation

| Method | URL | Purpose |
|--------|-----|---------|
| POST | `/api/submissions/validate/` | Update validation status for a submission |
| POST | `/api/submissions/validate/bulk/` | Bulk update validation status |
| GET | `/api/submissions/unvalidated/` | Get all unvalidated submissions |
| GET | `/api/submissions/on-hold/` | Get all on-hold submissions |
| GET | `/api/submissions/validation-history/` | Get validation audit trail |

#### PA Staff

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/api/submissions/pa-staff/` | Get PA Staff's own submissions |
| POST | `/api/submissions/notify-staff/` | Notify PA Staff about on-hold submissions |

#### Data Processing

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/api/processing/stats/` | Processing statistics |
| GET | `/api/processing/pending/` | Submissions pending processing |
| GET | `/api/processing/processed/` | Already processed submissions |
| POST | `/api/processing/process-single/` | Process one submission |
| POST | `/api/processing/process-batch/` | Process multiple submissions |
| POST | `/api/processing/commit/` | Commit processed data to database |
| POST | `/api/processing/reprocess/` | Reprocess a submission |

#### Scheduling

| Method | URL | Purpose |
|--------|-----|---------|
| GET/POST | `/api/routes/` | List/create transect routes |
| GET/POST | `/api/surveys/` | List/create scheduled surveys |
| GET | `/api/staff/<id>/schedule/` | Get a staff member's schedule |

---

## 1.6 Serializers

**Files:** `api/form_serializers.py`, `api/submission_serializers.py`

### Django Concept: Serializers

Serializers convert complex Django objects (model instances, querysets) to and from JSON. Think of them as "forms for APIs":

```python
class DataSubmissionSerializer(serializers.ModelSerializer):
    class Meta:
        model = DataSubmission
        fields = '__all__'  # All model fields → JSON keys
```

**Key serializers in the API app:**

| Serializer | Model | Purpose |
|-----------|-------|---------|
| `DataSubmissionSerializer` | DataSubmission | Full submission detail for API responses |
| `DataSubmissionListSerializer` | DataSubmission | Simplified version for list views (fewer fields) |
| `UpdateValidationStatusSerializer` | — | Validates input when changing validation status |
| `BulkValidationUpdateSerializer` | — | Validates input for bulk status changes |
| `ChoiceListSerializer` | ChoiceList | Read-only: includes nested choices + choice count |
| `ChoiceListCreateSerializer` | ChoiceList | Write: accepts nested choices for creation |
| `FormChoiceMappingSerializer` | FormChoiceMapping | With computed `choice_list_name` + `choice_count` |
| `TransectRouteSerializer` | TransectRoute | Includes `scheduled_surveys_count` |
| `ScheduledSurveySerializer` | ScheduledSurvey | Includes `submission_count` |
| `SubmissionStatsSerializer` | — | Computed: `validation_rate`, `approval_rate` |

---

## 1.7 Celery Tasks

**File:** `api/tasks.py` (~327 lines)

### Django Concept: Celery and Background Tasks

Some operations take too long for a web request (syncing hundreds of submissions from KoboToolbox, generating reports). **Celery** runs these in the background:

1. A **task** is a Python function decorated with `@shared_task`
2. A **worker** is a separate process that picks up and runs tasks
3. A **broker** (Redis, in our case) is the message queue between Django and workers
4. **Celery Beat** is a scheduler that runs tasks periodically (like a cron job)

```python
from celery import shared_task

@shared_task
def sync_all_kobotoolbox_data():
    """Runs every 10 minutes via Celery Beat"""
    # Sync data from all active forms
    ...
```

### Task Reference

| Task | Schedule | Queue | Purpose |
|------|----------|-------|---------|
| `sync_all_kobotoolbox_data` | Every 10 minutes | `data_processing` | Syncs all active KoboToolbox forms to the local database |
| `sync_specific_asset_data` | On-demand | `data_processing` | Syncs a single KoboToolbox form |
| `check_overdue_surveys` | Every hour | `data_processing` | Marks scheduled surveys as `overdue` if past their end time |
| `sync_validation_status_with_kobo` | Every 2 hours | `data_processing` | Ensures local and KoboToolbox validation statuses match |
| `cleanup_old_logs` | Weekly | `data_processing` | Deletes `DataRetrievalLog` entries older than 30 days |
| `notify_validators_of_new_submissions` | On-demand | `data_processing` | Creates `UserNotification` for validators about unreviewed submissions |
| `generate_daily_submission_report` | On-demand | `data_processing` | Generates a daily summary of submission activity |

**Task routing:** All `api.tasks.*` are routed to the `data_processing` queue (configured in `prism/celery.py`). This separates data processing work from validation and media download work.

---

## 1.8 Management Commands

**Directory:** `api/management/commands/`

### Django Concept: Management Commands

Custom commands you run with `python manage.py <command_name>`. They're useful for one-time operations, data seeding, maintenance, and debugging.

### Command Reference

#### Data Seeding & Setup
| Command | Purpose |
|---------|---------|
| `initialize_choice_lists` | Seeds the initial choice lists for KoboToolbox integration. **Run this after fresh database setup.** |
| `populate_observation_types` | Seeds the `ObservationType` lookup table (seen, heard, traces, etc.) |

#### KoboToolbox Sync
| Command | Purpose |
|---------|---------|
| `sync_kobotoolbox_data` | Manually trigger a full sync from KoboToolbox |
| `sync_kobo_submissions` | Alternative sync command |

#### Data Processing
| Command | Purpose |
|---------|---------|
| `process_bms_data` | Manually process all pending BMS submissions |
| `process_validated_submissions` | Process only approved submissions |

#### Spatial Data
| Command | Purpose |
|---------|---------|
| `generate_transect_segments` | Auto-generates `TransectSegment` records from existing `TransectRoute` + `MonitoringLocation` data |
| `assign_observations_to_segments` | Spatial query: assigns each observation to its nearest transect segment |

#### Data Migration & Maintenance
| Command | Purpose |
|---------|---------|
| `update_survey_statuses` | Recalculates all `ScheduledSurvey` statuses |
| `backfill_assigned_staff` | Backfills the `assigned_staff` field on older submissions |
| `migrate_observation_modes` | One-time data migration utility |
| `debug_timezone_issues` | Diagnostic tool for timezone-related bugs |

---

**Next:** [Chapter 2 — The `dashboard` App →](02-dashboard-app.md)

**Previous:** [← Project Overview](README.md)
