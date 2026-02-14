# Chapter 2 — The `dashboard` App

> The `dashboard` app is the **presentation layer** — everything users see and interact with. It contains 20+ view modules, 80+ templates, 53 CSS files, report generation, event management, and the entire permission enforcement system. If the `api` app is the engine, the `dashboard` app is the dashboard of the car.

---

## Table of Contents

- [2.1 App Overview](#21-app-overview)
- [2.2 Models](#22-models)
- [2.3 Views — How Pages Are Built](#23-views--how-pages-are-built)
  - [Role-Based Dashboards](#role-based-dashboards)
  - [Validation Views](#validation-views)
  - [Species Management](#species-management)
  - [GIS & Mapping](#gis--mapping)
  - [Reports](#reports)
  - [Events & FGD](#events--fgd)
  - [Admin Panel](#admin-panel)
  - [PA Staff Views](#pa-staff-views)
  - [Observations (Universal)](#observations-universal)
  - [Profile & Notifications](#profile--notifications)
  - [Internal API Endpoints](#internal-api-endpoints)
  - [Other View Modules](#other-view-modules)
- [2.4 URL Routing](#24-url-routing)
- [2.5 Permissions & Access Control](#25-permissions--access-control)
- [2.6 Middleware](#26-middleware)
- [2.7 Context Processors](#27-context-processors)
- [2.8 Forms](#28-forms)
- [2.9 Filters](#29-filters)
- [2.10 Template Tags](#210-template-tags)
- [2.11 Services](#211-services)
- [2.12 Celery Tasks](#212-celery-tasks)
- [2.13 Health Endpoints](#213-health-endpoints)
- [2.14 Management Commands](#214-management-commands)

---

## 2.1 App Overview

The dashboard app is organized by **feature area**, with each area getting its own view file, templates, and often its own CSS:

| Feature Area | View File | Templates | CSS |
|-------------|-----------|-----------|-----|
| Role dashboards | `views_dashboard.py` | `dashboard_home.html`, `validator_dashboard.html`, etc. | `validator_dashboard.css`, `admin_dashboard.css` |
| Validation queue | `views_validation.py` | `validation_queue.html`, `validation_workspace.html` | `validation_queue.css`, `validation_workspace.css` |
| Species CRUD | `views_species.py` | `species_list.html`, `species_detail.html` | `species.css` |
| GIS/Maps | `views_gis.py` | `interactive_map.html`, `monitoring_location_*.html` | `map.css` |
| Reports | `views_reports.py` | `reports/report_hub.html`, `report_builder.html` | `reports.css` |
| Events | `views_events.py` | `event_list/detail/form.html` | `events.css` |
| FGD | `views_fgd.py` | `fgd/fgd_session_list.html`, etc. | — |
| Admin | `views_admin.py` | `admin/*.html` (24 templates) | `admin_dashboard.css` |
| PA Staff | `views_pa_staff.py` | `pa_staff_home.html`, `my_submissions.html` | `pa_staff_home.css` |
| Schedules | `views_schedule.py` | `schedule_calendar.html` | `calendar.css` |
| Profile | `views_profile.py` | `profile.html`, `notifications.html` | `profile.css`, `notifications.css` |
| Observations | `views_universal_observation.py` | `observation_list/detail_universal.html` | — |
| AJAX endpoints | `views_api.py` | — (returns JSON) | — |

---

## 2.2 Models

**File:** `dashboard/models.py`

The dashboard app has its own models for features that don't belong in the core `api` app:

### ReportTemplate

Reusable report configurations that define what data a report should include.

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField | Template name |
| `report_type` | CharField | One of 7 types: `GIS`, `SEMESTRAL`, `STANDARD_BMS`, `DIGITAL_BACKUP`, `FINALIZED`, `FGD`, `CUSTOM` |
| `configuration` | JSONField | What filters, date ranges, and data sources to include |
| `created_by` | ForeignKey → User | Who created this template |
| `is_active` | BooleanField | Can it be used? |

### GeneratedReport

An actual generated report with output files in multiple formats.

| Field | Type | Purpose |
|-------|------|---------|
| `report_type` | CharField | Same 7 types as above |
| `title` / `description` | CharField / TextField | Report metadata |
| `date_range_start` / `date_range_end` | DateField | Data time range |
| `geographic_scope` | JSONField | Which areas to include |
| `template` | ForeignKey → ReportTemplate | Based on which template |
| `status` | CharField | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`, `ARCHIVED` |
| `progress` | IntegerField | 0-100 completion percentage |
| `pdf_file` | FileField | Generated PDF output |
| `excel_file` | FileField | Generated Excel output |
| `geojson_file` | FileField | Generated GeoJSON output |
| `shapefile_archive` | FileField | Generated Shapefile (ZIP) |
| `markdown_file` | FileField | Generated Markdown summary |
| `requested_by` | ForeignKey → User | Who requested the report |
| `is_public` | BooleanField | Visible to all? |
| `allowed_users` | ManyToMany → User | Specific access control |

### ReportArchive

Versioned archive of generated reports with download tracking.

| Field | Type | Purpose |
|-------|------|---------|
| `report` | ForeignKey → GeneratedReport | Which report |
| `version` | IntegerField | Version number |
| `is_latest_version` | BooleanField | Is this the current version? |
| `download_count` | IntegerField | How many times downloaded |
| `retention_until` | DateField | Keep until this date |

### Event

Organizational events (meetings, workshops, field activities, inspections, etc.).

| Field | Type | Purpose |
|-------|------|---------|
| `title` | CharField | Event name |
| `event_type` | CharField | `meeting`, `workshop`, `announcement`, `field_activity`, `inspection`, `community`, `other` |
| `start_datetime` / `end_datetime` | DateTimeField | When |
| `location` | CharField | Where |
| `protected_area` | ForeignKey → ProtectedArea | Which PA |
| `organizer` | ForeignKey → User | Who organized it |
| `attendees` | ManyToMany → User | Who's attending |

### Other Models

- **SystemMessage** — Broadcast messages from admins to users/roles
- **MessageRead** — Tracks which users have read which messages
- **SecurityEvent** — Audit trail for security events (login, password change, permission changes, suspicious activity) with risk levels
- **ErrorLog** — Application error tracking with traceback, resolution status
- **ReportDownloadLog** — Audit log for report downloads

---

## 2.3 Views — How Pages Are Built

### Django Concept: Function-Based Views vs Class-Based Views

Django supports two approaches to writing views:

**Function-Based Views (FBV):**
```python
def report_hub(request):
    reports = GeneratedReport.objects.filter(requested_by=request.user)
    return render(request, 'dashboard/reports/report_hub.html', {'reports': reports})
```

**Class-Based Views (CBV):**
```python
class SpeciesListView(CollaboratorAccessMixin, TemplateView):
    template_name = 'dashboard/species_list.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['fauna_species'] = FaunaChecklist.objects.all()
        return context
```

PRISM-Matutum uses **both** — CBVs for CRUD operations (list, detail, create, update, delete) because they reduce boilerplate, and FBVs for complex custom logic (report building, AJAX endpoints).

### Role-Based Dashboards

**File:** `dashboard/views_dashboard.py` (~1,068 lines)

When a user visits `/dashboard/`, the `dashboard_dispatch` function checks their role and routes them:

| Role | Redirected To | View Class |
|------|---------------|-----------|
| System Admin | `/dashboard/admin-dashboard/` | `AdminDashboardView` (in views_admin.py) |
| PA Staff | `/dashboard/pa-staff-home/` | `PAStaffDashboardHome` |
| Validator | `/dashboard/validator/` | `ValidatorDashboardView` |
| DENR | `/dashboard/denr/` | `DENRDashboardView` |
| Collaborator | `/dashboard/collaborator/` | `CollaboratorDashboardView` |

Each dashboard shows different information:
- **Validators** see pending submissions count, approval rates, recent validations
- **PA Staff** see their own submissions, upcoming schedules, field activity summary
- **DENR** see report summaries, biodiversity statistics, protected area status
- **Admins** see user management stats, system health, activity logs

### Validation Views

**File:** `dashboard/views_validation.py` (~2,017 lines)

The validation system is the heart of the dashboard — it's where Validators review and approve/reject field data.

| View | URL | Purpose |
|------|-----|---------|
| `ValidationQueueView` | `/dashboard/queue/` | Shows all pending submissions in a filterable, sortable table. Only accessible to validators |
| `ValidationWorkspaceView` | `/dashboard/workspace/<pk>/` | Detailed review of a single submission — shows form data, GPS location on map, media attachments, species identification, quality scoring. This is where validators approve/reject |
| `ValidationHistoryView` | `/dashboard/history/` | Log of all validation decisions with filters |
| `SubmissionAuditTrailView` | `/dashboard/audit/<pk>/` | Complete timeline of all changes to a submission |
| `ValidationHistoryExportView` | — | Export validation history as CSV |
| `PerformanceMetricsView` | — | Validator productivity dashboard (approval rates, processing times) |

### Species Management

**File:** `dashboard/views_species.py`

Full CRUD for managing the species checklists (fauna and flora).

| View | URL | Purpose |
|------|-----|---------|
| `SpeciesListView` | `/dashboard/species/` | List all species with search and filters |
| `SpeciesDetailView` | `/dashboard/species/<type>/<pk>/` | Detailed species page — taxonomy, characteristics, photos, observations map |
| `SpeciesCreateView` | `/dashboard/species/<type>/add/` | Add new species to checklist |
| `SpeciesUpdateView` | `/dashboard/species/<type>/<pk>/edit/` | Edit species details |
| `SpeciesDeleteView` | `/dashboard/species/<type>/<pk>/delete/` | Remove from checklist |
| `SpeciesObservationsView` | `/dashboard/species/<type>/<pk>/observations/` | All observations of this species |
| `SpeciesMediaView` | `/dashboard/species/<type>/<pk>/media/` | Manage species photos/media |

### GIS & Mapping

**File:** `dashboard/views_gis.py`

Manages geographic features and the interactive map.

| View | URL | Purpose |
|------|-----|---------|
| `InteractiveMapView` | `/dashboard/map/` | Full-screen Leaflet map with layers for observations, routes, protected areas |
| `MonitoringLocationListView` | `/dashboard/monitoring-locations/` | List all monitoring stations |
| `MonitoringLocationCreateView` | `/dashboard/monitoring-locations/add/` | Add a new station (lat/lng picker on map) |
| `TransectRouteListView` | `/dashboard/transect-routes/` | List all transect routes |
| `TransectRouteCreateView` | `/dashboard/transect-routes/add/` | Create route by selecting stations |
| `TransectSegmentListView` | `/dashboard/transect-segments/` | Auto-generated segments between stations |
| `ProtectedAreaListView/DetailView` | `/dashboard/protected-areas/` | View/edit protected area boundaries |

### Reports

**File:** `dashboard/views_reports.py`

The report system supports 7 report types with multiple output formats.

| View/Function | URL | Purpose |
|--------------|-----|---------|
| `report_hub` | `/dashboard/reports/` | Central report management page — list generated reports, access builder |
| `report_builder` | `/dashboard/reports/builder/` | Interactive form to configure and generate reports (date range, geographic scope, filters) |
| `data_export_center` | `/dashboard/reports/export/` | Export raw data as Excel, CSV, GeoJSON, or Shapefile |
| `report_archive_list` | `/dashboard/reports/archive/` | Browse historical reports |
| `download_report` | `/dashboard/reports/download/<pk>/<format>/` | Download a specific report file (PDF, Excel, GeoJSON, etc.) |
| `report_preview` | `/dashboard/reports/preview/<pk>/` | Preview before downloading |
| `report_status_api` | `/dashboard/reports/api/status/<pk>/` | AJAX endpoint for checking async report generation progress |
| `regenerate_report` | `/dashboard/reports/regenerate/<pk>/` | Re-generate a report with updated data |

### Events & FGD

**Files:** `dashboard/views_events.py`, `dashboard/views_fgd.py`

Standard CRUD for organizational events and Focus Group Discussion sessions. Both use access mixins to control who can create vs. view:

- **Events:** Anyone with dashboard access can view events. Only admins, validators, and DENR can create/edit.
- **FGD:** Validators and admins can manage sessions. PA Staff can view completed sessions.

### Admin Panel

**File:** `dashboard/views_admin.py` (~51 view classes)

The admin panel at `/dashboard/admin/` is a **custom admin interface** (separate from Django's built-in admin at `/admin/`). It provides user-friendly management for:

| Feature | Views | URL Prefix |
|---------|-------|-----------|
| **User Management** | `UserManagementView`, `UserApproveView`, `UserRejectView`, `UserUpdateView`, `UserDeleteView` | `/dashboard/admin/users/` |
| **Organization Hub** | `UserOrganizationHubView` | `/dashboard/admin/organization-hub/` |
| **Agency CRUD** | `AgencyManagementView`, `AgencyCreateView`, `AgencyUpdateView`, `AgencyDeleteView` | `/dashboard/admin/agencies/` |
| **Department CRUD** | 4 view classes | `/dashboard/admin/departments/` |
| **Address Hierarchy** | 16 view classes (Region, Province, Municipality, Barangay × 4 operations each) | `/dashboard/admin/address/` |
| **Choice Management** | `ChoiceManagementView`, `ChoiceListSyncView`, choice CRUD, `KoboFormBrowseView` | `/dashboard/admin/choices/` |

### PA Staff Views

**File:** `dashboard/views_pa_staff.py`

PA Staff see their own submissions and observations — they can't see other staff members' data.

| View | URL | Purpose |
|------|-----|---------|
| `PAStaffDashboardHome` | `/dashboard/pa-staff-home/` | PA Staff's home dashboard — recent submissions, upcoming schedules, stats |
| `MySubmissionsView` | `/dashboard/my-submissions/` | List of all their KoboToolbox submissions with validation status |
| `MyObservationsView` | `/dashboard/my-observations/` | Their processed observations (wildlife, disturbance, etc.) |
| `MySubmissionDetailView` | `/dashboard/my-submissions/<pk>/` | Detailed view of a single submission |
| `MySubmissionEditView` | — | Edit submission (for on-hold submissions that need corrections) |

### Observations (Universal)

**Files:** `dashboard/views_universal_observation.py`, `dashboard/views_universal_detail.py`

### Django Concept: Generic Views and Inheritance

Instead of writing separate list views for wildlife, resource use, disturbance, and landscape observations, PRISM-Matutum uses a **base class** and **subclasses**:

```python
class UniversalObservationListView(StaffAndValidatorAccessMixin, RoleBasedQuerysetMixin, ListView):
    """Base list view — subclasses only need to set the model"""
    pass

class WildlifeObservationListView(UniversalObservationListView):
    model = WildlifeObservation
    template_name = 'dashboard/wildlife_observation_list.html'

class DisturbanceRecordListView(UniversalObservationListView):
    model = DisturbanceRecord
    template_name = 'dashboard/disturbance_record_list.html'
```

The `RoleBasedQuerysetMixin` automatically filters results based on the user's role — PA Staff only see their own data, validators see everything.

| View | URL | Purpose |
|------|-----|---------|
| `WildlifeObservationListView` | `/dashboard/observations/` | All wildlife observations |
| `ResourceUseIncidentListView` | `/dashboard/resource-use/` | All resource use incidents |
| `DisturbanceRecordListView` | `/dashboard/disturbances/` | All disturbance records |
| `LandscapeMonitoringListView` | `/dashboard/landscape/` | All landscape records |
| `ObservationDetailView` | `/dashboard/observations/<pk>/` | Detailed view with map, media, species info |
| `ObservationUpdateView` | — | Edit a processed observation |
| `ObservationExportView` | — | Export observations to Excel/CSV |

### Profile & Notifications

**File:** `dashboard/views_profile.py`

| View | URL | Purpose |
|------|-----|---------|
| `ProfileView` | `/dashboard/profile/` | User profile page |
| `ProfileUpdateView` | `/dashboard/profile/edit/` | Edit profile details |
| `ActivityHistoryView` | `/dashboard/profile/activity/` | Personal activity log |
| `NotificationsView` | `/dashboard/notifications/` | In-app notification center |
| `NotificationSettingsView` | — | Configure notification preferences |

### Internal API Endpoints

**File:** `dashboard/views_api.py`

These are AJAX endpoints called by JavaScript on the frontend. They return JSON, not HTML.

| Function | URL | Purpose |
|----------|-----|---------|
| `api_validate` | `/dashboard/api/validate/` | Submit validation decision from workspace |
| `api_pending_counts` | `/dashboard/api/pending-counts/` | Get pending submission counts (used to update badges) |
| `api_species_search` | `/dashboard/api/species-search/` | Autocomplete search for species names |
| `api_species_info` | `/dashboard/api/species-info/` | Get species details for validation workspace |
| `api_sync_submissions` | `/dashboard/api/sync-submissions/` | Trigger KoboToolbox sync from UI |
| `api_mark_notification_read` | `/dashboard/api/notifications/mark-read/` | Mark notification as read |
| `api_bulk_mark_notifications_read` | — | Mark multiple notifications as read |

### GeoJSON API Endpoints

**File:** `dashboard/api_geo.py`

These endpoints return GeoJSON (RFC 7946) for the Leaflet.js map. The JavaScript map loads these to render layers.

| Function | URL | Purpose |
|----------|-----|---------|
| `geojson_protected_areas` | `/dashboard/api/geo/protected-areas/` | Protected area boundaries |
| `geojson_transect_routes` | `/dashboard/api/geo/transect-routes/` | Transect route lines |
| `geojson_monitoring_locations` | `/dashboard/api/geo/monitoring-locations/` | Station point markers |
| `geojson_transect_segments` | `/dashboard/api/geo/transect-segments/` | Segment lines |
| `geojson_observations` | `/dashboard/api/geo/observations/` | Observation point markers with popup data |
| `geojson_statistics` | `/dashboard/api/geo/statistics/` | Aggregated stats by area |
| `geojson_export` | `/dashboard/api/geo/export/` | Download GeoJSON files |

### Other View Modules

| File | Purpose |
|------|---------|
| `views_schedule.py` | Survey calendar, scheduled survey CRUD |
| `views_analytics_widgets.py` | Analytics data functions (disturbance, wildlife, resource use, landscape charts) |
| `views_taxonomy.py` | Taxonomy hierarchy management (Kingdom through Genus) |
| `views_articles.py` | Research articles CRUD (linked to fauna/flora species) |
| `views_widgets.py` | Widget data fetching functions (counter, chart, map, table data) |

---

## 2.4 URL Routing

**File:** `dashboard/urls.py` (~130 URL patterns)

### Django Concept: URL Configuration

Django maps URLs to views using `path()` patterns in `urls.py`:

```python
from django.urls import path
from . import views_species

app_name = 'dashboard'  # Namespace — allows {% url 'dashboard:species_list' %}

urlpatterns = [
    path('species/', views_species.SpeciesListView.as_view(), name='species_list'),
    path('species/<str:species_type>/<int:pk>/',
         views_species.SpeciesDetailView.as_view(), name='species_detail'),
]
```

The `app_name = 'dashboard'` creates a **namespace** so you can refer to URLs as `dashboard:species_list` in templates, avoiding name collisions with other apps.

### URL Groups

The ~130 URLs are organized into these groups:

| Group | Prefix | # Routes | Notable Patterns |
|-------|--------|----------|-----------------|
| **Health** | `health/`, `ready/`, `alive/`, `metrics/` | 4 | Kubernetes-compatible health checks |
| **Dashboards** | `''`, `validator/`, `denr/`, `collaborator/`, `admin-dashboard/` | 6 | Role-based entry points |
| **Validation** | `queue/`, `history/`, `workspace/<pk>/`, `audit/<pk>/` | 5 | Core validation workflow |
| **PA Staff** | `my-submissions/`, `my-observations/`, `pa-staff-home/` | 6 | PA Staff personal views |
| **Species** | `species/`, `species/<type>/<pk>/` | 8 | Full CRUD + observations + media |
| **Observations** | `observations/`, `resource-use/`, `disturbances/`, `landscape/` | 8 | Universal list/detail/edit/export |
| **GIS** | `monitoring-locations/`, `transect-routes/`, `transect-segments/`, `map/` | 12 | CRUD + interactive map |
| **Schedules** | `schedules/` | 5 | Calendar, CRUD |
| **Events** | `events/` | 5 | Event CRUD |
| **FGD** | `fgd/` | 8 | Sessions, findings, participants |
| **Reports** | `reports/` | 12 | Hub, builder, export, archive, download |
| **Admin** | `admin/` | 30+ | Users, orgs, addresses, choices, agencies |
| **Profile** | `profile/`, `notifications/` | 6 | Profile, activity, notifications |
| **API** | `api/` | 15 | AJAX validation, species search, sync, GeoJSON |
| **Taxonomy** | `taxonomy/` | 4 | Hierarchy management |
| **Articles** | `articles/fauna/`, `articles/flora/` | 6 | Article CRUD |

---

## 2.5 Permissions & Access Control

**Files:** `dashboard/permissions.py`, `dashboard/decorators.py`

### Django Concept: Mixins

A **mixin** is a class that adds behavior to other classes via multiple inheritance. In Django views, mixins are used to add permission checks:

```python
class ValidatorAccessMixin(LoginRequiredMixin, UserPassesTestMixin):
    """Only validators, staff, and superusers can access"""
    def test_func(self):
        return self.request.user.is_validator() or self.request.user.is_staff

# Usage:
class ValidationQueueView(ValidatorAccessMixin, ListView):
    model = DataSubmission  # Only validators can see this
```

### Permission Mixins (Class-Based)

| Mixin | Who Can Access | Used By |
|-------|---------------|---------|
| `DashboardAccessMixin` | Any logged-in user with approved account | Base for most views |
| `PAStaffAccessMixin` | PA Staff with field verification | PA Staff views |
| `ValidatorAccessMixin` | Validators + staff/superusers | Validation, species management |
| `DENRAccessMixin` | DENR + validators | Report views |
| `CollaboratorAccessMixin` | Collaborators + validators + DENR + PA Staff | Species list (read-only) |
| `AdminAccessMixin` | System admins only | Admin panel |
| `StaffAndValidatorAccessMixin` | PA Staff + validators + DENR + collaborators | Observation views |
| `CanValidateMixin` | Users with `can_validate()` permission | Validation actions |
| `RoleBasedQuerysetMixin` | All — but auto-filters data by role | Observation lists |

### Permission Decorators (Function-Based)

For function-based views:

```python
@validator_required
def some_view(request):
    # Only validators can reach this
    pass

@rate_limit(requests_per_minute=30, requests_per_hour=500)
def api_endpoint(request):
    # Rate-limited API endpoint
    pass
```

| Decorator | Purpose |
|-----------|---------|
| `dashboard_access_required` | Approved account required |
| `validator_required` | Validator role required |
| `pa_staff_required` | PA Staff role required |
| `denr_required` | DENR role required |
| `admin_required` | System admin required |
| `can_validate_required` | Uses `User.can_validate()` method |
| `rate_limit(per_min, per_hour)` | Redis-based rate limiting |
| `rate_limit_by_ip(per_min)` | IP-based rate limiting |
| `api_rate_limit` | Default 30/min, 500/hour |

### The Three Layers of Permission Enforcement

```
Request → Middleware → Decorator/Mixin → View Logic
   │           │              │              │
   │     DashboardAccess   validator_     role-specific
   │     Middleware         required      query filtering
   │     (approved         decorator      (RoleBased
   │      accounts)        or mixin        QuerysetMixin)
```

1. **Middleware** (`DashboardAccessMiddleware`) — Checks that every request to `/dashboard/` is from an approved account
2. **Decorator/Mixin** — Checks role-specific permissions on individual views
3. **Queryset filtering** — Even with access, users only see data appropriate to their role

---

## 2.6 Middleware

**File:** `dashboard/middleware.py`

### Django Concept: Middleware

Middleware is code that runs on **every** request before it reaches a view, and on **every** response before it's sent back. Think of it as a pipeline:

```
Request → Middleware 1 → Middleware 2 → ... → View → ... → Middleware 2 → Middleware 1 → Response
```

### DashboardAccessMiddleware

Runs on all requests to `/dashboard/` URLs. It checks:

1. Is the user authenticated?
2. Is their `account_status == 'approved'`?
3. If not → redirect to login (regular requests) or return JSON 403 (AJAX requests)

**Exempt patterns:** Health check endpoints and Django admin views bypass this check.
**Bypass:** Staff and superusers always pass.

### DashboardActivityLogMiddleware

Logs user activities to `UserActivityLog`, categorizing them by type:
- **validation** — Requests that involve validation endpoints
- **report_generation** — Report building/downloading
- **data_submission** — Submission-related actions
- **user_management** — Admin management actions
- **settings_change** — Settings modifications

---

## 2.7 Context Processors

**File:** `dashboard/context_processors.py`

### Django Concept: Context Processors

A context processor is a function that adds variables to **every template** automatically. Instead of passing `user_role` in every single view, you define it once:

```python
def dashboard_context(request):
    if not request.user.is_authenticated:
        return {}
    return {
        'user_role': request.user.role,
        'can_validate': request.user.can_validate(),
        # ... more variables
    }
```

Then in any template: `{{ user_role }}`, `{% if can_validate %}...{% endif %}`

### What PRISM-Matutum Injects

The `dashboard_context` processor adds these variables to every template:

| Variable | Type | Purpose |
|----------|------|---------|
| `user_role` | str | Current user's role (e.g., `"validator"`) |
| `user_role_display` | str | Human-readable role (e.g., `"Validator"`) |
| `can_validate` | bool | Can this user validate submissions? |
| `can_manage_users` | bool | Can this user manage other users? |
| `can_view_reports` | bool | Can this user view reports? |
| `can_collect_data` | bool | Can this user collect field data? |
| `dashboard_home_url` | str | URL for this user's role-appropriate dashboard |
| `pending_validations_count` | int | Number of submissions awaiting validation (validators only) |
| `my_pending_submissions` | int | Number of user's own pending submissions (PA Staff only) |
| `unread_notifications_count` | int | Unread notifications count |
| `recent_notifications` | QuerySet | Last 5 notifications for the notification dropdown |

---

## 2.8 Forms

**File:** `dashboard/forms.py` (~2,126 lines)

### Django Concept: Forms

Django forms handle user input validation and rendering. There are two types:

- **`Form`** — Manual field definitions (not tied to a model)
- **`ModelForm`** — Auto-generates fields from a model (fewer lines to write)

```python
# ModelForm example — fields come from the model
class EventForm(forms.ModelForm):
    class Meta:
        model = Event
        fields = ['title', 'event_type', 'start_datetime', 'end_datetime', 'location']
        widgets = {
            'start_datetime': forms.DateTimeInput(attrs={'type': 'datetime-local'}),
        }

# Regular Form — manual fields
class ValidationActionForm(forms.Form):
    action = forms.ChoiceField(choices=[('approve', 'Approve'), ('reject', 'Reject')])
    remarks = forms.CharField(widget=forms.Textarea, required=False)
```

### Key Forms

| Form | Type | Purpose |
|------|------|---------|
| `ValidationActionForm` | Form | Approve/reject/hold/flag actions in validation workspace |
| `SpeciesCorrectionForm` | Form | Manually correct species identification during validation |
| `FaunaSpeciesForm` | ModelForm | Create/edit fauna checklist entries |
| `FloraSpeciesForm` | ModelForm | Create/edit flora checklist entries |
| `FaunalCharacteristicsForm` | ModelForm | Edit fauna morphological characteristics |
| `FloralCharacteristicsForm` | ModelForm | Edit flora morphological characteristics (leaf types, stem, flower, fruit) |
| `TaxonomyForm` | ModelForm | Create/edit taxonomy records |
| `MonitoringLocationForm` | ModelForm | Add/edit monitoring stations with lat/lng |
| `TransectRouteForm` | ModelForm | Create transect routes with station ordering |
| `ScheduledSurveyForm` | ModelForm | Schedule surveys with conflict detection |
| `EventForm` | ModelForm | Create/edit events with attendees |
| `FGDSessionForm` | ModelForm | Create FGD sessions |
| `FGDFindingForm` | ModelForm | Record FGD findings with species links |
| `ChoiceListForm` | ModelForm | Manage choice lists |
| `ChoiceBulkAddForm` | Form | Bulk import choices (format: `name\|label` per line) |
| `FormChoiceMappingForm` | ModelForm | Map Kobo form questions to choice lists |

---

## 2.9 Filters

**File:** `dashboard/filters.py`

### Django Concept: Django-Filter

The `django-filter` library provides declarative filter definitions for list views:

```python
import django_filters

class SubmissionFilter(django_filters.FilterSet):
    submission_time_after = django_filters.DateTimeFilter(
        field_name='submission_time', lookup_expr='gte'
    )
    class Meta:
        model = DataSubmission
        fields = ['validation_status', 'username']
```

### Defined Filters

| Filter | Model | Filterable Fields |
|--------|-------|-------------------|
| `SubmissionFilter` | DataSubmission | `validation_status`, `submission_time` (date range), `username`, `assigned_staff`, `validated_by`, `protected_area`, `observation_type`, `priority` |
| `FaunaSpeciesFilter` | FaunaChecklist | `fauna_type`, `conservation_status`, `endemicity_status`, `search` (scientific name + common name) |
| `FloraSpeciesFilter` | FloraChecklist | `conservation_status`, `endemicity_status`, `search` (scientific name + common name) |

---

## 2.10 Template Tags

**File:** `dashboard/templatetags/dashboard_tags.py`

### Django Concept: Custom Template Tags and Filters

Template tags extend Django's template language. **Filters** transform values, **tags** produce content:

```html
<!-- Filter: transforms a value -->
{{ field_name|field_display_name }}

<!-- Tag: produces HTML -->
{% issue_badge issue_type count %}
{% quality_score_display score %}
```

### Key Filters

| Filter | Input | Output |
|--------|-------|--------|
| `field_display_name` | Field name string | Human-readable display name |
| `format_coord` | Float | Formatted GPS coordinate |
| `severity_icon` | Severity level | Icon class for severity |
| `status_display` | Status code | Human-readable status |
| `species_display_name` | Species object | Formatted species name |
| `notification_icon`/`notification_color` | Notification type | Icon/color class |
| `activity_icon`/`activity_color` | Activity type | Icon/color class |
| `record_type_display` | Record type code | Display name |
| `percentage` | (value, total) | Calculated percentage |

### Inclusion Tags

These render reusable HTML components:

| Tag | Template | Purpose |
|-----|----------|---------|
| `{% issue_badge type count %}` | `components/issue_badge.html` | Renders a colored badge showing issue count |
| `{% quality_score_display score %}` | `components/quality_score.html` | Renders a quality score with progress bar and color coding |

---

## 2.11 Services

### Report Services

**File:** `dashboard/report_services.py` (~1,382 lines)

#### BaseReportGenerator

Base class for all report types. Provides:
- Date range and geographic scope filtering
- Data collection (wildlife, resource use, disturbance, landscape, FGD)
- Species richness, endemic species, and threatened species calculations
- Common fields for all report types

#### Report Types

| Generator Class | Report Type | Output Formats |
|----------------|-------------|----------------|
| `GISReportGenerator` | GIS | GeoJSON + Excel (with coordinates) |
| `SemestralReportGenerator` | Semestral | Excel + Markdown summary |
| `StandardBMSReportGenerator` | Standard BMS | Excel + Markdown (extends Semestral) |
| `DigitalBackupExporter` | Digital Backup | Excel + CSV archive + GeoJSON (full data backup with species, locations, transects, validation history) |
| `FinalizedReportGenerator` | Finalized | Excel + Markdown (extends Semestral) |
| `PDFReportMixin` | — | Mixin that adds HTML-to-PDF conversion via WeasyPrint |

### Event Services

**File:** `dashboard/event_services.py` (~396 lines)

`EventNotificationService` — Sends notifications when events are created, updated, cancelled, or when attendees are added.

---

## 2.12 Celery Tasks

**File:** `dashboard/tasks.py` (~1,009 lines)

All dashboard tasks route to the `validation` queue.

| Task | Retry | Purpose |
|------|-------|---------|
| `update_validation_statistics` | 3 | Caches validation stats for all validators (runs every 30 min via beat) |
| `process_bulk_validation` | 3 | Async bulk approve/reject/hold/flag for multiple submissions |
| `generate_validation_report` | 3 | Generate CSV report of validation activity |
| `generate_report_task` | 3 | Generate a `GeneratedReport` with progress tracking |
| `generate_report_with_progress` | 3 | Report generation with real-time progress updates |
| `export_map_data_async` | 3 | Export GeoJSON/Shapefile/KML data asynchronously |
| `send_validation_notifications` | 3 | Batch send validation-related notifications |
| `send_event_notifications` | 3 | Send event creation/update notifications |
| `send_event_cancellation_notifications` | 2 | Notify attendees of event cancellation |
| `send_upcoming_event_reminders` | — | Periodic reminders for upcoming events |
| `update_species_verification_cache` | — | Cache species verification data for faster lookups |
| `cleanup_expired_cache` | — | Clean up expired cache entries |
| `cleanup_old_reports` | — | Remove old reports based on retention policies |
| `database_maintenance` | — | Database optimization operations |

---

## 2.13 Health Endpoints

**File:** `dashboard/health.py`

Kubernetes-compatible health check endpoints:

| Endpoint | URL | Checks | Response |
|----------|-----|--------|----------|
| `health_check_view` | `/dashboard/health/` | Database, cache, Redis, disk space | 200 OK / 503 Unhealthy + JSON details |
| `readiness_check_view` | `/dashboard/ready/` | Apps loaded + database connection | 200 Ready / 503 Not Ready |
| `liveness_check_view` | `/dashboard/alive/` | Process is running | 200 Alive (always succeeds) |
| `metrics_view` | `/dashboard/metrics/` | — | JSON: total submissions, pending validations, active validators |

---

## 2.14 Management Commands

**Directory:** `dashboard/management/commands/`

| Command | Purpose |
|---------|---------|
| `setup_validator_dashboard` | Initialize default validator dashboard configuration |
| `generate_test_report` | Generate a test report for development/debugging |

---

**Next:** [Chapter 3 — The `users` App →](03-users-app.md)

**Previous:** [← Chapter 1: The `api` App](01-api-app.md)
