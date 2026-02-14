# Chapter 6 — Frontend Architecture

> This chapter explains how the browser-facing side of PRISM-Matutum works: the template system that generates HTML, the CSS architecture that styles it, the JavaScript modules that add interactivity, and the map/chart integrations that visualize data.

---

## Table of Contents

- [6.1 Technology Stack](#61-technology-stack)
- [6.2 Template System (Django Templates)](#62-template-system-django-templates)
- [6.3 CSS Architecture](#63-css-architecture)
- [6.4 JavaScript Architecture](#64-javascript-architecture)
- [6.5 Map Integration (Leaflet.js)](#65-map-integration-leafletjs)
- [6.6 Chart Integration (Chart.js)](#66-chart-integration-chartjs)
- [6.7 Custom Template Tags & Filters](#67-custom-template-tags--filters)
- [6.8 Template Directory Reference](#68-template-directory-reference)

---

## 6.1 Technology Stack

| Technology | Version | Purpose | Loaded Via |
|-----------|---------|---------|-----------|
| **Bootstrap 5** | 5.3.8 | Layout grid, components, utilities | CDN |
| **Bootstrap Icons** | 1.10.0 | Icon library (1,800+ icons) | CDN |
| **Leaflet.js** | 1.9.4 | Interactive maps | CDN (unpkg) |
| **Chart.js** | Latest | Data visualization (bar, line, doughnut, radar) | CDN |
| **Google Fonts** | — | Crimson Pro (display), DM Sans (body), JetBrains Mono (code) | CDN |
| **Django Templates** | — | Server-side HTML generation | Built-in |
| **crispy-forms** | — | Form rendering with Bootstrap styling | pip |

> **Note:** This project uses **server-side rendering** (Django generates full HTML pages), NOT a JavaScript framework like React or Vue. JavaScript is used only for interactive features (maps, charts, AJAX validation actions, etc.).

---

## 6.2 Template System (Django Templates)

### What Are Django Templates?

Django templates are HTML files with special tags that allow you to insert dynamic data, loop over lists, and conditionally show/hide sections. The key syntax elements:

| Syntax | Purpose | Example |
|--------|---------|---------|
| `{{ variable }}` | Output a value | `{{ user.username }}` |
| `{% tag %}` | Logic (loops, conditions, includes) | `{% if user.is_authenticated %}` |
| `{{ value\|filter }}` | Transform a value | `{{ name\|title }}` |
| `{# comment #}` | Comment (not rendered) | `{# TODO: fix this #}` |

### Template Inheritance

Django templates use a **block-based inheritance** system, like object-oriented programming for HTML.

#### The Base Template (`base.html`)

Every page extends `base.html`, which provides the shell:

```
┌──────────────────────────────────────────────┐
│  <head>                                      │
│    Bootstrap CSS, Leaflet CSS, Google Fonts   │
│    main.css (→ 50+ component CSS files)       │
│    {% block extra_css %}  ← per-page CSS      │
│  </head>                                     │
│                                              │
│  <body>                                      │
│  ┌───────────────────────────────────────┐   │
│  │  NAVBAR (fixed top)                   │   │
│  │  Logo │ Brand │ Notifications │ User  │   │
│  └───────────────────────────────────────┘   │
│  ┌──────────┬────────────────────────────┐   │
│  │ SIDEBAR  │  MAIN CONTENT              │   │
│  │          │                            │   │
│  │ Nav      │  {% block page_actions %}   │   │
│  │ links    │  ← toolbar/buttons          │   │
│  │ (role-   │                            │   │
│  │  based)  │  {% block content %}        │   │
│  │          │  ← the actual page          │   │
│  │          │                            │   │
│  └──────────┴────────────────────────────┘   │
│                                              │
│  Bootstrap JS, Leaflet JS                    │
│  validation.js, map.js, charts.js            │
│  {% block extra_js %}  ← per-page JS         │
│                                              │
└──────────────────────────────────────────────┘
```

#### How a Child Template Works

Every page template follows this pattern:

```html
{% extends "dashboard/base.html" %}
{% load static %}
{% load dashboard_tags %}

{% block title %}Wildlife Observations{% endblock %}

{% block extra_css %}
<link rel="stylesheet" href="{% static 'dashboard/css/my_observations.css' %}">
{% endblock %}

{% block page_actions %}
<a href="{% url 'dashboard:export' %}" class="btn-action-link">
    <i class="bi bi-download"></i> Export Data
</a>
{% endblock %}

{% block content %}
<div class="page-header">
    <h1 class="page-title">Wildlife Observations</h1>
</div>

<table class="data-table data-table--striped">
    <thead>
        <tr>
            <th>Species</th>
            <th>Date</th>
            <th>Status</th>
        </tr>
    </thead>
    <tbody>
        {% for obs in observations %}
        <tr>
            <td>{{ obs|species_display_name }}</td>
            <td>{{ obs.submission_time|date:"M d, Y" }}</td>
            <td>
                <span class="badge-status--{{ obs.validation_status }}">
                    {{ obs.validation_status|status_display }}
                </span>
            </td>
        </tr>
        {% empty %}
        <tr>
            <td colspan="3">No observations found.</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}

{% block extra_js %}
<script>
    // Page-specific JavaScript here
</script>
{% endblock %}
```

### Blocks Defined in `base.html`

| Block | Where | Purpose |
|-------|-------|---------|
| `{% block title %}` | `<head>` | Page title (default: "Dashboard") |
| `{% block extra_css %}` | Bottom of `<head>` | Additional CSS files for this page |
| `{% block page_actions %}` | Above main content | Action buttons toolbar |
| `{% block content %}` | Main area | The actual page content |
| `{% block extra_js %}` | Bottom of `<body>` | Additional JavaScript for this page |

### Role-Based Sidebar Navigation

The sidebar dynamically shows/hides sections based on the logged-in user's role:

```html
<!-- In base.html: -->
{% if user.role == 'pa_staff' %}
    <a href="{% url 'dashboard:my_submissions' %}">My Submissions</a>
    <a href="{% url 'dashboard:my_observations' %}">My Observations</a>
{% endif %}

{% if user.role == 'validator' %}
    <a href="{% url 'dashboard:validation_queue' %}">
        Validation Queue
        {% if pending_validation_count %}
            <span class="badge-count">{{ pending_validation_count }}</span>
        {% endif %}
    </a>
{% endif %}
```

The `pending_validation_count` comes from a **context processor** (`dashboard/context_processors.py`) that runs on every request and injects the count into every template.

### Sidebar Sections

| Section | Links | Who Sees It |
|---------|-------|-------------|
| **Role Dashboard** | Admin/DENR/Researcher/PA Staff/Validator Dashboard | Role-specific |
| **PA Staff Items** | My Submissions, My Observations | PA Staff only |
| **Validator Items** | Validation Queue (with badge), Validation History | Validators only |
| **Observations** | Wildlife, Resource Use, Disturbance, Landscape | All users |
| **Species Database** | Species Checklists | Validators, DENR, Collaborators, Admins |
| **Spatial Tools** | Interactive Map, Monitoring Locations, Transect Routes | All users |
| **Schedule Management** | Survey Calendar, Events & Meetings | All users |
| **Reports & Archives** | Report Hub, Builder, Exports, Archives, Templates | Permission-based |
| **System Administration** | User Hub, Taxonomy, Choice Hub, Admin Panel | Admins |

---

## 6.3 CSS Architecture

### Component-Based Design System

Rather than writing CSS per-page, PRISM-Matutum has a **component-based CSS architecture**. Reusable CSS components (cards, tables, badges, buttons) are defined once and used everywhere.

### File Structure

```
dashboard/static/dashboard/css/
│
├── main.css                ← Master file — imports everything in order
│
├── variables.css           ← Design tokens (colors, fonts, shadows)
│
├── base.css                ← Reset, body styles, HTML defaults
├── utilities.css           ← Utility classes (.mt-2, .text-center, etc.)
├── typography.css          ← .page-title, .section-title, .text-highlight
├── layouts.css             ← .page-header, .dashboard-grid--4col, .filter-panel
│
├── navigation.css          ← Sidebar + navbar styles
├── cards.css               ← .stat-card, .stat-card--compact, .stat-card--success
├── buttons.css             ← .btn-action-link, .btn-icon, .btn-group-actions
├── forms.css               ← Form inputs, labels, crispy-forms overrides
├── tables.css              ← .data-table, .data-table--striped, .action-icon
├── alerts.css              ← .alert--verification, .alert--success-subtle
├── badges.css              ← .badge-status--pending/approved/rejected, .badge-count
├── progress.css            ← Progress bars
├── dropdowns.css           ← Dropdown menus
├── pagination.css          ← Page navigation
├── breadcrumb.css          ← Breadcrumb trails
├── empty_states.css        ← Empty state placeholders
│
├── dashboard.css           ← Dashboard home layout
├── validation_queue.css    ← Validation queue unique styles
├── validation_workspace.css ← Workspace page unique styles
├── species.css             ← Species pages
├── events.css              ← Events/calendar pages
├── reports.css             ← Report pages
├── ... (28 feature-specific files)
│
├── map.css                 ← Leaflet map customizations
├── chart.css               ← Chart.js/Chart container styles
├── calendar.css            ← Calendar widget styles
├── media.css               ← Image gallery styles
│
├── footer.css              ← Footer styles
├── animation.css           ← CSS animations/transitions
└── responsive.css          ← Media queries for mobile/tablet
```

### How `main.css` Works

`main.css` is the only CSS file loaded in `base.html`. It uses `@import` to pull in all other files **in dependency order**:

```css
/* main.css */

/* 1. Design tokens first — everything else references these */
@import url('variables.css');

/* 2. Foundation — resets and base typography */
@import url('base.css');
@import url('utilities.css');
@import url('typography.css');
@import url('layouts.css');

/* 3. Core components — reusable across all pages */
@import url('navigation.css');
@import url('cards.css');
@import url('buttons.css');
@import url('forms.css');
@import url('tables.css');
@import url('alerts.css');
@import url('badges.css');
/* ... */

/* 4. Feature modules — page-specific unique styles */
@import url('dashboard.css');
@import url('validation_queue.css');
/* ... */

/* 5. Integrations */
@import url('map.css');
@import url('chart.css');

/* 6. Layout finishing */
@import url('responsive.css');
```

### Design Tokens (`variables.css`)

All colors, fonts, shadows, and transitions are defined as CSS custom properties (variables) in `variables.css`. **Never hardcode colors** — always use these tokens:

```css
:root {
    /* Forest palette — primary brand colors */
    --forest-deep: #0d1f1a;
    --forest-dark: #1a3329;
    --forest-mid:  #2d5246;
    --forest-light: #4a7c6b;
    --moss-green:  #5e9c7f;
    --sage:        #8db596;

    /* Earth tones — secondary palette */
    --earth-dark:  ...;
    --earth-mid:   ...;
    --earth-light: ...;
    --sand:        #d4b896;

    /* Accent colors */
    --ember:  #e67e50;    /* Action, CTA */
    --sunset: #f4a261;    /* Highlights */
    --dawn:   #ffd89b;    /* Warm accent */
    --sky:    #89cff0;    /* Cool accent */
    --mist:   #b8d4d8;    /* Soft cool */

    /* Functional colors */
    --warning: ...;
    --danger:  ...;
    --success: ...;
    --info:    ...;

    /* Neutral gray scale (10 stops) */
    --ash-50:  ...;       /* Lightest */
    --ash-100: ...;
    /* ... */
    --ash-900: ...;       /* Darkest */

    /* Typography */
    --font-display: 'Crimson Pro', serif;     /* Headings */
    --font-body:    'DM Sans', sans-serif;    /* Body text */
    --font-mono:    'JetBrains Mono', monospace; /* Code */

    /* Shadows */
    --shadow-sm:  0 1px 2px rgba(0,0,0,.05);
    --shadow-md:  0 4px 6px rgba(0,0,0,.1);
    --shadow-lg:  0 10px 15px rgba(0,0,0,.1);
    --shadow-glow: 0 0 15px rgba(94,156,127,.3);

    /* Transitions */
    --transition-fast: 150ms ease;
    --transition-base: 250ms ease;
    --transition-slow: 400ms ease;

    /* Gradients */
    --gradient-primary: linear-gradient(135deg, var(--forest-dark), var(--forest-mid));
    --gradient-header:  linear-gradient(...);
    --gradient-button:  linear-gradient(...);
}
```

### Key CSS Components & Their Classes

#### Cards (`cards.css`)

```html
<!-- Basic stat card -->
<div class="stat-card">
    <div class="stat-card__value">142</div>
    <div class="stat-card__label">Total Observations</div>
</div>

<!-- Compact variant -->
<div class="stat-card stat-card--compact">...</div>

<!-- Color variants -->
<div class="stat-card stat-card--success">...</div>
<div class="stat-card stat-card--warning">...</div>
```

#### Tables (`tables.css`)

```html
<table class="data-table data-table--striped">
    <thead>...</thead>
    <tbody>
        <tr>
            <td>Data</td>
            <td>
                <a href="..." class="action-icon" title="View">
                    <i class="bi bi-eye"></i>
                </a>
            </td>
        </tr>
    </tbody>
</table>
```

#### Badges (`badges.css`)

```html
<span class="badge-status--approved">Approved</span>
<span class="badge-status--pending">Pending</span>
<span class="badge-status--rejected">Rejected</span>
<span class="badge-count">5</span>
<span class="badge-role">Validator</span>
```

#### Layouts (`layouts.css`)

```html
<div class="page-header">
    <h1 class="page-title">Page Title</h1>
</div>

<div class="dashboard-grid--4col">
    <!-- Cards auto-arrange in 4-column grid -->
</div>

<div class="filter-panel">
    <!-- Filter form fields -->
</div>
```

### Rules for Adding New CSS

1. **Check existing components first** — Cards, tables, badges, and buttons are already built. Use them.
2. **Never hardcode colors** — Use `var(--moss-green)`, not `#5e9c7f`
3. **No inline styles** — Use a CSS file
4. **Page-specific files are for unique layouts only** — Don't recreate table or card styles in a page file
5. **Import new files in `main.css`** — In the correct section

---

## 6.4 JavaScript Architecture

The project uses **four JavaScript files**, each containing one or more class-based modules. No build tools (webpack, vite) are used — files are loaded directly via `<script>` tags.

### `validation.js` (1,107 lines)

Two classes for the validation workflow:

#### `ValidationManager` — Validation Queue Interactivity

| Feature | How It Works |
|---------|-------------|
| **AJAX Validation Actions** | POST to `/dashboard/api/validate/` with CSRF token. Approve/reject/hold submissions without page reload |
| **Batch Operations** | Select-all checkbox + individual row checkboxes. Batch action bar appears when items are selected |
| **Quick Filters** | `<select>` dropdowns that modify URL query params and reload |
| **Debounced Search** | 300ms debounce on keystroke → GET `/dashboard/api/search/` |
| **Pending Count Polling** | Every 30 seconds, fetches `/dashboard/api/pending-counts/` and updates sidebar badge + notification count |
| **Toast Notifications** | Creates auto-dismissing alert banners (5 seconds) |

#### `ValidationWorkspace` — Submission Detail Review

| Feature | How It Works |
|---------|-------------|
| **Collapsible Sections** | Bootstrap collapse with chevron icon toggle |
| **JSON Viewer** | Interactive expandable/collapsible tree with syntax highlighting, field statistics (count, size, depth), copy-to-clipboard, expand/collapse all |
| **Validation Form** | Conditional required fields — notes are mandatory for reject/hold actions, minimum 10 characters. Confirmation dialogs per action type |
| **Media Gallery** | Modal viewer for photos/videos/audio with type filtering and URL clipboard copy |
| **Map** | Per-submission map with the observation's GPS location |

### `map.js` (356 lines) — `MapManager`

Loaded globally. Auto-discovers `<div class="map-container">` elements on any page:

```html
<!-- In a template: -->
<div class="map-container"
     data-lat="6.357"
     data-lng="125.073"
     data-zoom="12"
     data-data-url="/dashboard/api/geo/observations/">
</div>
```

The `MapManager` reads these `data-*` attributes and initializes a Leaflet map with:
- **Two base layers**: OpenStreetMap + Esri Satellite (with layer switcher)
- **Marker clustering**: Groups nearby markers into clusters
- **Color-coded markers**: Green (approved), red (rejected), orange (pending)
- **Rich popups**: Species name, date, observer, validation buttons

### `map-enhanced.js` (1,016 lines) — `EnhancedMapManager`

Used exclusively on the Interactive Map page (`/dashboard/map/`). Extends the basic map with:

| Feature | Description |
|---------|-------------|
| **Three base layers** | OSM, Esri Satellite, OpenTopoMap |
| **Five overlay groups** | Protected Areas, Transect Routes, Monitoring Locations, Transect Segments, Observations |
| **GeoJSON API** | Fetches from `/dashboard/api/geo/protected-areas/`, `transect-routes/`, `monitoring-locations/`, `observations/` |
| **Custom Controls** | Search bar, filter panel (date range, status, PA), export dialog |
| **Styled geometries** | PA boundaries (green polygons), transect routes (blue lines), locations (type-specific icons) |
| **Scale bar** | Shows distance scale |

### `charts.js` (415 lines) — `ChartManager`

Auto-discovers `<div class="chart-container">` elements:

```html
<!-- In a template: -->
<div class="chart-container"
     data-chart-type="species-distribution"
     data-data-url="/dashboard/api/widgets/species-distribution/">
</div>
```

Supported chart types:

| `data-chart-type` | Chart.js Type | What It Shows |
|-------------------|---------------|---------------|
| `validation-status` | Doughnut | Approved / Rejected / Pending / In Review |
| `validation-timeline` | Filled Line | Validations per day over time |
| `species-distribution` | Bar | Observation counts per species |
| `validator-performance` | Radar | 5 metrics (Accuracy, Speed, Thoroughness, Consistency, Communication) |
| `monthly-trends` | Stacked Bar | Approved vs Rejected by month |
| Default | Line | Any numeric data series |

Features: AJAX data loading, PNG export/download, window resize handling, tab-change re-render.

---

## 6.5 Map Integration (Leaflet.js)

### Map API Endpoints

The interactive map consumes a GeoJSON REST API served by `dashboard/api_geo.py`:

| Endpoint | Returns | Geometry Type |
|----------|---------|--------------|
| `/dashboard/api/geo/protected-areas/` | PA boundaries | MultiPolygon |
| `/dashboard/api/geo/transect-routes/` | Transect route lines | LineString |
| `/dashboard/api/geo/monitoring-locations/` | Station points | Point |
| `/dashboard/api/geo/transect-segments/` | Segment lines | LineString |
| `/dashboard/api/geo/observations/` | Observation points | Point |

All responses follow [RFC 7946 GeoJSON](https://tools.ietf.org/html/rfc7946) format:

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": { "type": "Point", "coordinates": [125.073, 6.357] },
      "properties": {
        "name": "Philippine Eagle",
        "date": "2026-01-15",
        "status": "approved"
      }
    }
  ]
}
```

### How Maps Are Added to Pages

1. **Any page** — Add a `<div class="map-container" data-*>` and `MapManager` auto-initializes it
2. **Interactive Map page** — Include `map-enhanced.js` in `{% block extra_js %}`
3. **Validation Workspace** — `ValidationWorkspace.initializeValidationMap()` creates a focused map

### PostGIS & Spatial Queries

The backend uses PostGIS for all geographic operations:

```python
# Example from api_geo.py — finding observations within a protected area:
from django.contrib.gis.geos import Point
from django.contrib.gis.db.models.functions import Distance

observations = WildlifeObservation.objects.filter(
    location__within=protected_area.boundary
).annotate(
    distance=Distance('location', protected_area.centroid)
)
```

---

## 6.6 Chart Integration (Chart.js)

### Widget-Based Chart System

Dashboard pages use a **widget system** where chart containers are discovered automatically:

```html
<!-- In dashboard_home.html: -->
<div class="dashboard-grid--4col">

    <div class="chart-container"
         data-chart-type="validation-status"
         data-data-url="{% url 'dashboard:widget_validation_status' %}">
    </div>

    <div class="chart-container"
         data-chart-type="species-distribution"
         data-data-url="{% url 'dashboard:widget_species_distribution' %}">
    </div>

</div>
```

### Widget API Endpoints

Chart data is served by `dashboard/views_analytics_widgets.py`:

```python
# Simplified widget view
class SpeciesDistributionWidget(LoginRequiredMixin, View):
    def get(self, request):
        data = WildlifeObservation.objects.values('species') \
            .annotate(count=Count('id')) \
            .order_by('-count')[:10]
        
        return JsonResponse({
            'labels': [d['species'] for d in data],
            'datasets': [{
                'data': [d['count'] for d in data],
                'backgroundColor': [...],
            }]
        })
```

The `ChartManager` on the frontend receives this JSON and passes it directly to Chart.js.

---

## 6.7 Custom Template Tags & Filters

Defined in `dashboard/templatetags/dashboard_tags.py` (401 lines). Load them with `{% load dashboard_tags %}`.

### Most-Used Filters

| Filter | Example | Output |
|--------|---------|--------|
| `status_display` | `{{ "---"\|status_display }}` | "Pending Validation" |
| `species_display_name` | `{{ observation\|species_display_name }}` | Best species name (linked DB name → raw text fallback) |
| `field_display_name` | `{{ "species_count"\|field_display_name }}` | "Species Count" |
| `format_coord` | `{{ 6.35723\|format_coord:4 }}` | "6.3572" |
| `choice_label` | `{{ "BU"\|choice_label:"observation_type" }}` | "Burrow" (looks up from DB) |
| `percentage` | `{{ 42\|percentage:200 }}` | "21.0" |
| `get_item` | `{{ mydict\|get_item:"nested.key" }}` | Value at dict["nested"]["key"] |
| `record_type_display` | `{{ "wildlife_observation"\|record_type_display }}` | "Wildlife Observation" |

### Inclusion Tags

These render entire template fragments:

```html
<!-- Issue severity badge -->
{% issue_badge issue %}
<!-- Renders components/issue_badge.html with icon + color -->

<!-- Quality score gauge -->
{% quality_score_display score "large" %}
<!-- Renders components/quality_score.html with progress ring -->
```

### Utility Tags

```html
<!-- Replace URL parameter (for pagination + filters) -->
<a href="?{% url_replace request 'page' 3 %}">Page 3</a>
<!-- Output: ?status=approved&page=3 (preserves other query params) -->
```

---

## 6.8 Template Directory Reference

**145 templates** in `dashboard/templates/dashboard/`, organized by function:

### Core Pages

| Template | URL | Purpose |
|----------|-----|---------|
| `dashboard_home.html` | `/dashboard/` | Main dashboard with widgets |
| `admin/admin_dashboard.html` | `/dashboard/admin/` | Admin overview |
| `pa_staff_home.html` | `/dashboard/pa-staff-home/` | PA Staff personalized home |
| `validator_dashboard.html` | `/dashboard/validator/` | Validator overview |
| `denr_dashboard.html` | `/dashboard/denr/` | DENR dashboard |
| `collaborator_dashboard.html` | `/dashboard/collaborator/` | Researcher dashboard |

### Observations & Records

| Template | Purpose |
|----------|---------|
| `wildlife_observation_list.html` | Wildlife observation table |
| `observation_detail_universal.html` | Detail page (any record type) |
| `resource_use_incident_list.html` | Resource use table |
| `disturbance_record_list.html` | Disturbance record table |
| `landscape_monitoring_list.html` | Landscape monitoring table |

### Submissions & Validation

| Template | Purpose |
|----------|---------|
| `my_submissions.html` | PA Staff's own submissions |
| `my_submission_detail.html` | Single submission detail for PA Staff |
| `validation_queue.html` | Validator's pending submissions |
| `validation_workspace.html` | Full submission review page |
| `validation_history.html` | Past validation actions |

### Species & Taxonomy

| Template | Purpose |
|----------|---------|
| `species_list.html` | Searchable species checklist |
| `species_detail.html` | Species profile with info & media |
| `species_observations.html` | All sightings of one species |
| `taxonomy_management.html` | Admin taxonomy browser |

### Maps & Spatial

| Template | Purpose |
|----------|---------|
| `interactive_map.html` | Full-page interactive map (loads `map-enhanced.js`) |
| `monitoring_location_list/detail/form.html` | Monitoring station CRUD |
| `transect_route_list/detail/form.html` | Transect route CRUD |

### Reports

| Template | Purpose |
|----------|---------|
| `reports/report_hub.html` | Report center with generation controls |
| `reports/report_builder.html` | Custom report builder |
| `reports/data_export_center.html` | Data export (Excel, CSV, GeoJSON) |
| `reports/pdf/report_base.html` | Base HTML template for PDF generation |

### Reusable Components (`components/`)

| Template | Rendered By | Purpose |
|----------|------------|---------|
| `issue_badge.html` | `{% issue_badge %}` | Severity badge with icon |
| `quality_score.html` | `{% quality_score_display %}` | Quality score gauge |
| `workflow_stepper.html` | Include in pages | Multi-step progress indicator |
| `widgets.html` | Include in dashboards | Generic widget container |

### Dashboard Widgets (`widgets/`)

Small template fragments for dashboard cards:

`assigned_submissions.html`, `bar_chart.html`, `counter.html`, `data_table.html`, `disturbance_alerts.html`, `line_chart.html`, `map.html`, `pie_chart.html`, `recent_activity.html`, `schedule_calendar.html`, `species_summary.html`, `text.html`, `validation_stats.html`

### Admin Pages (`admin/`)

User management, agency/department management, address hierarchy management, choice list management, form mapping management.

### Error Pages (`errors/`)

`403.html`, `404.html`, `500.html` — all extending `errors/base_error.html`

---

**Next:** [Chapter 7 — Configuration & Deployment →](07-configuration-deployment.md)

**Previous:** [← Chapter 5: Data Flow Pipeline](05-data-flow-pipeline.md)
