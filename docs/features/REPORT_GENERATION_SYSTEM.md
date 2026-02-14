# Report Generation System Documentation

## Overview

The Report Generation System provides comprehensive reporting capabilities for biodiversity monitoring data, enabling authorized users to create customized reports with multiple output formats. The system supports five report types ranging from spatial GIS reports to comprehensive data backup exports, with asynchronous generation and robust archiving features.

**Key Features:**
- **Interactive Report Builder** - Visual form for configuring report parameters
- **Asynchronous Generation** - Background processing using Celery for large reports
- **Multiple Output Formats** - PDF, Excel, GeoJSON, Shapefile (ZIP), Markdown
- **Template System** - Reusable report configurations for recurring needs
- **Archive Management** - Version control and download tracking
- **Geographic Filtering** - Barangay and monitoring location-based scoping
- **Access Control** - Role-based permissions with public/private reports

---

## Architecture

### Three-Layer System

```
┌─────────────────────────────────────────────────────────────────┐
│                       PRESENTATION LAYER                         │
│  - Report Builder Interface (Interactive Form)                  │
│  - Data Export Center (Quick Exports)                           │
│  - Report Archives (List, Preview, Download)                    │
│  - Template Gallery (Browse Saved Configurations)               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     ORCHESTRATION LAYER                          │
│  - Report Views (views_reports.py - 12 views)                   │
│  - Async Task Queue (Celery - generate_report_task)            │
│  - Factory Pattern (get_report_generator)                       │
│  - Permission Decorators (@can_build_reports, @can_view_reports)│
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SERVICE LAYER                                │
│  - BaseReportGenerator (Abstract base class)                    │
│    ├─ GISReportGenerator (Spatial analysis + maps)              │
│    ├─ SemestralReportGenerator (Periodic BMS reports)           │
│    ├─ StandardBMSReportGenerator (Standard format)              │
│    ├─ DigitalBackupExporter (Full data backups)                 │
│    └─ FinalizedReportGenerator (Official submissions)           │
│                                                                  │
│  - Data Collection Methods (collect_wildlife_data, etc.)        │
│  - Aggregation Functions (species richness, endemic counts)     │
│  - Export Generators (Excel, GeoJSON, CSV, Markdown)            │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Backend Framework:** Django 5.2 Function-Based Views
- **Async Processing:** Celery + Redis task queue
- **Data Export:** openpyxl (Excel), json (GeoJSON), csv, markdown
- **Spatial Export:** Custom Shapefile export utility
- **File Storage:** Django FileField with organized upload paths
- **Database:** PostgreSQL 16 with JSONField for configuration storage

---

## Models & Database Schema

### 1. ReportTemplate

Reusable report configurations that users can save and reuse.

```python
class ReportTemplate(models.Model):
    """Reusable report templates with predefined configurations"""
    
    # Identification
    name = CharField(max_length=255)
    report_type = CharField(max_length=20, choices=REPORT_TYPE_CHOICES)
    description = TextField(blank=True)
    
    # Configuration (stored as JSON)
    configuration = JSONField(default=dict)
    # Stores: date_ranges, geographic_scope, sections, output_formats
    
    # Metadata
    created_by = ForeignKey(User)
    created_at = DateTimeField(auto_now_add=True)
    updated_at = DateTimeField(auto_now=True)
    is_active = BooleanField(default=True)
```

**REPORT_TYPE_CHOICES:**
- `GIS` - GIS-Enhanced Report (maps, spatial analysis, GeoJSON)
- `SEMESTRAL` - Semestral BMS Report (periodic biodiversity monitoring)
- `STANDARD_BMS` - Standard BMS Report (similar to semestral)
- `DIGITAL_BACKUP` - Digital Backup Export (comprehensive data dumps)
- `FINALIZED` - Finalized Report Copy (official submissions with archiving)
- `CUSTOM` - Custom Report (user-defined configurations)

**Indexes:**
- Primary index on `created_at` (descending)

---

### 2. GeneratedReport

Represents a generated report instance with output files and metadata.

```python
class GeneratedReport(models.Model):
    """Generated reports with multiple format outputs and metadata"""
    
    # Report Identification
    report_type = CharField(max_length=20, choices=REPORT_TYPE_CHOICES)
    title = CharField(max_length=255)
    description = TextField(blank=True)
    
    # Report Parameters
    date_range_start = DateField()
    date_range_end = DateField()
    geographic_scope = JSONField(default=dict)
    # Stores: {'barangays': [...], 'monitoring_locations': [...]}
    
    # Configuration
    template = ForeignKey(ReportTemplate, null=True, blank=True)
    configuration = JSONField(default=dict)
    # Stores: {'include_maps': bool, 'include_charts': bool, 
    #          'include_photos': bool, 'output_formats': [...]}
    
    # Status Tracking
    status = CharField(max_length=20, choices=STATUS_CHOICES, default='PENDING')
    # PENDING → PROCESSING → COMPLETED or FAILED → ARCHIVED
    progress = IntegerField(default=0)  # 0-100 percentage
    error_message = TextField(blank=True)
    
    # File Outputs (all optional based on configuration)
    pdf_file = FileField(upload_to='reports/pdf/%Y/%m/', null=True, blank=True)
    excel_file = FileField(upload_to='reports/excel/%Y/%m/', null=True, blank=True)
    geojson_file = FileField(upload_to='reports/geojson/%Y/%m/', null=True, blank=True)
    shapefile_archive = FileField(upload_to='reports/shapefiles/%Y/%m/', null=True, blank=True)
    markdown_file = FileField(upload_to='reports/markdown/%Y/%m/', null=True, blank=True)
    
    # Metadata (stored as JSON)
    metadata = JSONField(default=dict)
    # Stores: species_richness, endemic_count, threatened_count, 
    #         observation_count, generation_timestamp
    
    # File Information
    total_file_size = BigIntegerField(default=0)  # In bytes
    
    # User Tracking
    requested_by = ForeignKey(User, related_name='requested_reports')
    
    # Timestamps
    requested_at = DateTimeField(auto_now_add=True)
    generation_started_at = DateTimeField(null=True, blank=True)
    generation_completed_at = DateTimeField(null=True, blank=True)
    
    # Access Control
    is_public = BooleanField(default=False)
    allowed_users = ManyToManyField(User, related_name='accessible_reports', blank=True)
    
    # Methods
    def get_file_count(self) -> int
    def calculate_total_file_size(self) -> int
    def get_available_formats(self) -> List[str]
    def can_access(self, user) -> bool
```

**STATUS_CHOICES:**
- `PENDING` - Queued for generation
- `PROCESSING` - Currently generating
- `COMPLETED` - Successfully generated
- `FAILED` - Generation error occurred
- `ARCHIVED` - Moved to archive storage

**Indexes:**
- `requested_at` (descending) - for timeline display
- `status` - for filtering by state
- `report_type` - for filtering by type

**Key Methods:**
- `can_access(user)`: Checks if user has permission (public, requester, allowed_users, or staff)
- `calculate_total_file_size()`: Sums file sizes from all output files
- `get_available_formats()`: Returns list of formats that were successfully generated

---

### 3. ReportArchive

Archive management for generated reports with versioning and retention policies.

```python
class ReportArchive(models.Model):
    """Archive management for generated reports with versioning"""
    
    # Report Reference
    report = ForeignKey(GeneratedReport, related_name='archive_records')
    
    # Versioning
    version = IntegerField(default=1)
    is_latest_version = BooleanField(default=True)
    
    # Archive Metadata
    archive_notes = TextField(blank=True)
    archived_by = ForeignKey(User, related_name='archived_reports')
    archived_at = DateTimeField(auto_now_add=True)
    
    # Download Tracking
    download_count = IntegerField(default=0)
    last_downloaded_at = DateTimeField(null=True, blank=True)
    last_downloaded_by = ForeignKey(User, null=True, blank=True)
    
    # Retention Management
    retention_until = DateField(null=True, blank=True)
    is_permanent = BooleanField(default=False)
    
    # Methods
    def increment_download(self, user): ...
```

**Constraints:**
- `unique_together`: ['report', 'version']

**Key Logic:**
- Automatic versioning when saving with `is_latest_version=True` (sets others to False)
- `is_permanent` reports are never auto-deleted by retention policies
- Download tracking for audit and analytics

---

### 4. ReportDownloadLog

Detailed audit trail of all report downloads.

```python
class ReportDownloadLog(models.Model):
    """Detailed logging of report downloads for audit purposes"""
    
    # References
    report = ForeignKey(GeneratedReport, related_name='download_logs')
    archive = ForeignKey(ReportArchive, null=True, blank=True)
    
    # Download Details
    downloaded_by = ForeignKey(User)
    downloaded_at = DateTimeField(auto_now_add=True)
    file_format = CharField(max_length=20, choices=[
        ('PDF', 'PDF'),
        ('EXCEL', 'Excel'),
        ('GEOJSON', 'GeoJSON'),
        ('SHAPEFILE', 'Shapefile'),
        ('MARKDOWN', 'Markdown'),
    ])
    
    # Access Metadata
    ip_address = GenericIPAddressField(null=True, blank=True)
    user_agent = TextField(blank=True)
```

---

## Views & URL Patterns

### Report Generation Views

#### 1. `report_builder` - Interactive Report Configuration

```python
@can_build_reports
def report_builder(request)
```

**URL:** `/dashboard/reports/builder/`  
**Template:** `dashboard/reports/report_builder.html`  
**Permissions:** Validators, DENR staff, System Admins only  
**HTTP Methods:** GET, POST

**Features:**
- Visual form with 6 report type cards (radio selection)
- Date range pickers (start/end dates)
- Geographic scope multi-select (barangays, monitoring locations)
- Report options checkboxes (include maps, charts, photos)
- Output format multi-select (PDF, Excel, GeoJSON, Shapefile, Markdown)
- Template loading from saved configurations
- Title and description fields

**POST Processing:**
1. Extract form data (type, title, dates, geographic scope, configuration)
2. Create `GeneratedReport` with `status='PENDING'`
3. Queue async task: `generate_report_task.delay(report.id)`
4. Redirect to report archives with success message

**Context Data:**
- `templates`: Active ReportTemplate records
- `monitoring_locations`: All MonitoringLocation objects
- `barangays`: Unique barangay names from observations
- `report_types`: REPORT_TYPE_CHOICES list

---

#### 2. `data_export_center` - Quick Data Exports

```python
@can_view_reports
@can_build_reports
def data_export_center(request)
```

**URL:** `/dashboard/reports/export/`  
**Template:** `dashboard/reports/data_export_center.html`  
**Permissions:** Validators, DENR staff, System Admins  
**HTTP Methods:** GET, POST

**Features:**
- Quick export form without full report configuration
- Data type multi-select (wildlife, resource_use, disturbance, landscape, species, locations, transects, validation_history)
- Export format selection (excel, geojson, csv)
- Date range filters (defaults to last 365 days)
- Creates `DIGITAL_BACKUP` report type

**POST Processing:**
1. Create GeneratedReport with `report_type='DIGITAL_BACKUP'`
2. Set configuration: `{'data_types': [...], 'format': 'excel'}`
3. Queue async generation
4. Redirect to report archives

**Context Data:**
- `wildlife_count`: Total WildlifeObservation count
- `resource_use_count`: Total ResourceUseIncident count
- `disturbance_count`: Total DisturbanceRecord count
- `landscape_count`: Total LandscapeMonitoring count

---

### Report Access Views

#### 3. `report_archive_list` - Browse Generated Reports

```python
@can_view_reports
def report_archive_list(request)
```

**URL:** `/dashboard/reports/archive/`  
**Template:** `dashboard/reports/report_archive_list.html`  
**Permissions:** All authenticated users (with role-based filtering)  
**HTTP Methods:** GET

**Features:**
- Paginated list of generated reports (20 per page)
- Filter by report type, status, search query, date range
- Role-based visibility:
  - **PA Staff/Collaborators:** Only completed reports (public or assigned)
  - **Validators/DENR/Admins:** All reports
- Statistics cards (total, completed, pending, processing, failed)
- Action buttons: View preview, Download (dropdown for formats), Regenerate, Delete

**Query Logic:**
```python
# Role-based filtering
if user.is_pa_staff() or user.is_collaborator():
    reports = reports.filter(
        Q(status='COMPLETED') &
        (Q(is_public=True) | Q(requested_by=user) | Q(allowed_users=user))
    )
elif not user.is_staff and not user.is_system_admin():
    reports = reports.filter(
        Q(requested_by=user) | Q(is_public=True) | Q(allowed_users=user)
    )
# Admins see all reports
```

**Context Data:**
- `page_obj`: Paginated queryset of GeneratedReport
- `stats`: Dictionary with counts by status
- `report_types`: REPORT_TYPE_CHOICES
- `status_choices`: STATUS_CHOICES
- `current_filters`: Active filter values

---

#### 4. `report_preview` - Detailed Report View

```python
@can_view_reports
def report_preview(request, report_id)
```

**URL:** `/dashboard/reports/<report_id>/preview/`  
**Template:** `dashboard/reports/report_preview.html`  
**Permissions:** Report requester, allowed users, or staff  
**HTTP Methods:** GET

**Features:**
- Full report metadata display
- Available formats with file sizes
- Archive version history
- Recent download logs (last 10)
- Generation timeline (requested → started → completed)
- Geographic scope details
- Configuration summary
- Error messages (if failed)

**Access Control:**
- Checks `report.can_access(request.user)`
- Returns 403 error if unauthorized

**Context Data:**
- `report`: GeneratedReport instance
- `archives`: Related ReportArchive records (ordered by version)
- `recent_downloads`: Last 10 ReportDownloadLog entries
- `available_formats`: List of generated formats

---

#### 5. `download_report` - File Download Handler

```python
@can_view_reports
def download_report(request, report_id, file_format)
```

**URL:** `/dashboard/reports/<report_id>/download/<file_format>/`  
**Permissions:** Report requester, allowed users, or staff  
**HTTP Methods:** GET

**Features:**
- Downloads specific format of generated report
- Content-Type mapping:
  - `pdf`: application/pdf
  - `excel`: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
  - `geojson`: application/geo+json
  - `shapefile`: application/zip
  - `markdown`: text/markdown
- Logs download in ReportDownloadLog (user, format, IP, user agent)
- Updates ReportArchive download tracking
- Sets Content-Disposition header for proper filename

**Response:**
- FileResponse with appropriate content type
- 404 if format not available
- 403 if access denied

---

### Report Management Views

#### 6. `report_template_list` - Browse Templates

```python
@can_view_reports
def report_template_list(request)
```

**URL:** `/dashboard/reports/templates/`  
**Template:** `dashboard/reports/report_template_list.html`  
**Permissions:** All authenticated users  
**HTTP Methods:** GET

**Features:**
- Card-based template gallery
- Shows template name, type, description, creator, creation date
- "Use Template" button → redirects to report_builder with `?template=<id>` param
- Pagination (20 templates per page)

---

#### 7. `create_report_template` - Save Configuration

```python
@can_view_reports
@require_http_methods(["POST"])
def create_report_template(request)
```

**URL:** `/dashboard/reports/templates/create/`  
**Permissions:** All authenticated users  
**HTTP Methods:** POST only

**Features:**
- Creates ReportTemplate from POST data
- Parses JSON configuration string
- Associates with current user as creator
- Sets `is_active=True` by default

**POST Parameters:**
- `name`: Template name
- `report_type`: Report type code
- `description`: Optional description
- `configuration`: JSON string with report config

---

#### 8. `delete_report` - Remove Report

```python
@can_view_reports
@require_http_methods(["POST"])
def delete_report(request, report_id)
```

**URL:** `/dashboard/reports/<report_id>/delete/`  
**Permissions:** Report requester or staff only  
**HTTP Methods:** POST only

**Features:**
- Deletes GeneratedReport and all associated files
- Removes files from storage: pdf_file, excel_file, geojson_file, shapefile_archive, markdown_file
- Cascades to ReportArchive and ReportDownloadLog (via foreign key ON DELETE)
- Success/error messages

**Access Control:**
- Only report requester or staff can delete
- Returns error message if unauthorized

---

#### 9. `regenerate_report` - Duplicate Configuration

```python
@can_view_reports
@require_http_methods(["POST"])
def regenerate_report(request, report_id)
```

**URL:** `/dashboard/reports/<report_id>/regenerate/`  
**Permissions:** Report requester, allowed users, or staff  
**HTTP Methods:** POST only

**Features:**
- Creates new GeneratedReport with same configuration as original
- Appends "(Regenerated)" to title
- Useful for updating data with same parameters
- Queues async generation
- Redirects to preview page of new report

**Cloned Fields:**
- report_type, description, date_range_start, date_range_end
- geographic_scope, template, configuration

---

#### 10. `report_status_api` - Progress Polling Endpoint

```python
@can_view_reports
def report_status_api(request, report_id)
```

**URL:** `/dashboard/reports/<report_id>/status/` (API endpoint)  
**Permissions:** Report requester, allowed users, or staff  
**HTTP Methods:** GET

**Features:**
- Returns JSON with current generation status
- Used for frontend polling during async generation
- Provides progress percentage, status, error messages

**Response JSON:**
```json
{
  "id": 123,
  "status": "PROCESSING",
  "progress": 65,
  "error_message": "",
  "completed": false,
  "available_formats": []
}
```

---

#### 11. `download_shapefile` - Direct Shapefile Export

```python
@can_view_reports
def download_shapefile(request)
```

**URL:** `/dashboard/reports/shapefile/export/`  
**Permissions:** All authenticated users with report viewing permission  
**HTTP Methods:** GET

**Features:**
- Direct shapefile export without creating GeneratedReport
- Filters WildlifeObservation queryset by URL parameters
- Generates ZIP archive with .shp, .shx, .dbf, .prj files
- Uses `export_shapefile()` utility from `dashboard.utils_geo`

**Query Parameters:**
- `date_start`: Start date filter
- `date_end`: End date filter
- `species`: Species name filter (common or scientific)
- `protected_area`: Protected area ID filter
- `validation_status`: Comma-separated status list (defaults to 'validated,approved')

**Processing:**
1. Build queryset with filters
2. Check for location data (`location__isnull=False`)
3. Generate shapefile ZIP archive
4. Log download in ReportDownloadLog (with `report=None`)
5. Return FileResponse with cleanup callback

---

### Report Template Views

#### 12. Additional Template Views

The system includes additional template-related views for managing and using saved report configurations. These follow similar patterns to the core views above.

---

## Service Layer: Report Generators

### BaseReportGenerator (Abstract Base Class)

All report generators inherit from this base class which provides common functionality.

```python
class BaseReportGenerator:
    """Base class for all report generators"""
    
    def __init__(self, report: GeneratedReport):
        self.report = report
        self.start_date = report.date_range_start
        self.end_date = report.date_range_end
        self.geographic_scope = report.geographic_scope or {}
        self.data = {}
    
    # Data Collection Methods
    def get_base_queryset(self, model) -> QuerySet
    def collect_wildlife_data(self) -> None
    def collect_resource_use_data(self) -> None
    def collect_disturbance_data(self) -> None
    def collect_landscape_data(self) -> None
    def collect_all_data(self) -> dict
    
    # Aggregation Methods
    def _calculate_species_richness(self, observations) -> int
    def _count_endemic_species(self, observations) -> int
    def _count_threatened_species(self, observations) -> int
    def _group_by_observation_type(self, observations) -> dict
    def _group_by_field(self, records, field_name) -> dict
    def _group_by_location(self, records) -> dict
    
    # Abstract Method (must be implemented by subclasses)
    def generate(self):
        raise NotImplementedError("Subclasses must implement generate()")
```

**Key Methods:**

**`get_base_queryset(model)`** - Builds filtered queryset with date and geographic constraints
```python
qs = model.objects.filter(today__gte=start_date, today__lte=end_date)

if geographic_scope.get('barangays'):
    qs = qs.filter(protected_area__address__barangay__name__in=barangays)

if geographic_scope.get('monitoring_locations'):
    # Only for models with monitoring_location field
    qs = qs.filter(monitoring_location_id__in=location_ids)
```

**`collect_wildlife_data()`** - Gathers WildlifeObservation records and calculates:
- Total observation count
- Species richness (unique species count)
- Endemic species count
- Threatened species count
- Groupings by observation type and location

**`collect_resource_use_data()`** - Gathers ResourceUseIncident records with:
- Total incident count
- Groupings by incident type and location

**`collect_disturbance_data()`** - Gathers DisturbanceRecord records with:
- Total disturbance count
- Groupings by disturbance type, severity, and location

**`collect_landscape_data()`** - Gathers LandscapeMonitoring records with:
- Total monitoring point count
- Groupings by location

---

### 1. GISReportGenerator

Generates spatial reports with GeoJSON and coordinate-based exports.

```python
class GISReportGenerator(BaseReportGenerator):
    """Generate GIS-Enhanced Reports with maps and spatial analysis"""
    
    def generate(self):
        """Generate GIS report with maps and spatial data"""
        self.collect_all_data()
        
        # Generate GeoJSON FeatureCollection
        geojson_data = self._create_geojson()
        geojson_file = self._save_geojson(geojson_data)
        
        # Generate Excel with coordinates
        excel_file = self._create_excel_export()
        
        # Update report
        self.report.geojson_file = geojson_file
        self.report.excel_file = excel_file
        self.report.status = 'COMPLETED'
        self.report.progress = 100
        self.report.generation_completed_at = timezone.now()
        self.report.calculate_total_file_size()
        self.report.save()
        
        return self.report
```

**Output Formats:**
- **GeoJSON:** FeatureCollection with Point geometries for all observations with coordinates
- **Excel:** Multiple sheets (Wildlife, Resource Use, Disturbance) with lat/lon columns

**GeoJSON Structure:**
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [longitude, latitude]
      },
      "properties": {
        "type": "wildlife",
        "species": "Tarictic Hornbill",
        "date": "2025-01-06",
        "location": "Transect A, Station 1",
        "observer": "Juan Dela Cruz"
      }
    }
  ],
  "metadata": {
    "generated": "2025-01-06T10:30:00Z",
    "date_range": {"start": "2024-01-01", "end": "2024-12-31"},
    "total_features": 1523
  }
}
```

---

### 2. SemestralReportGenerator

Generates comprehensive biodiversity monitoring reports for periodic submissions.

```python
class SemestralReportGenerator(BaseReportGenerator):
    """Generate Semestral BMS Reports"""
    
    def generate(self):
        """Generate semestral report with trend analysis"""
        self.collect_all_data()
        
        # Generate Excel with comprehensive data
        excel_file = self._create_semestral_excel()
        
        # Generate Markdown summary
        markdown_file = self._create_markdown_summary()
        
        # Update metadata
        self.report.metadata = {
            'wildlife_observations': self.data['wildlife']['total_count'],
            'species_richness': self.data['wildlife']['species_richness'],
            'endemic_species': self.data['wildlife']['endemic_count'],
            'threatened_species': self.data['wildlife']['threatened_count'],
            'resource_use_incidents': self.data['resource_use']['total_count'],
            'disturbance_records': self.data['disturbance']['total_count'],
            'landscape_monitoring': self.data['landscape']['total_count'],
            'generation_timestamp': timezone.now().isoformat(),
        }
        
        self.report.excel_file = excel_file
        self.report.markdown_file = markdown_file
        self.report.status = 'COMPLETED'
        self.report.progress = 100
        self.report.save()
        
        return self.report
```

**Output Formats:**
- **Excel:** Multi-sheet workbook with:
  - Summary sheet (metrics and statistics)
  - Wildlife Data sheet (detailed observations)
  - Issues & Concerns sheet (resource use + disturbance records)
- **Markdown:** Executive summary document with key findings and recommendations

**Excel Structure:**
- **Summary Sheet:** Key metrics (species richness, endemic/threatened counts, incident totals)
- **Wildlife Data:** Date, Species, Common Name, Endemic (Yes/No), Threatened (Yes/No), Location, Observer, Notes
- **Issues & Concerns:** Date, Type (Resource Use/Disturbance), Description, Location, Severity, Recommended Action

---

### 3. StandardBMSReportGenerator

Similar to SemestralReportGenerator but with standardized formatting for official BMS reports.

```python
class StandardBMSReportGenerator(SemestralReportGenerator):
    """Generate Standard BMS Reports (similar to semestral but different format)"""
    pass  # Currently inherits all functionality from Semestral
```

**Note:** This generator currently extends SemestralReportGenerator without additional customization. It serves as a placeholder for future standardized formatting requirements.

---

### 4. DigitalBackupExporter

Generates comprehensive data exports for backup and archiving purposes.

```python
class DigitalBackupExporter(BaseReportGenerator):
    """Generate comprehensive data exports for backup purposes"""
    
    def __init__(self, report: GeneratedReport):
        super().__init__(report)
        # Get selected data types from configuration
        self.data_types = report.configuration.get('data_types', [
            'wildlife', 'resource_use', 'disturbance', 'landscape'
        ])
        self.export_format = report.configuration.get('format', 'excel')
    
    def collect_all_data(self):
        """Override to collect only selected data types"""
        if 'wildlife' in self.data_types:
            self.collect_wildlife_data()
        if 'resource_use' in self.data_types:
            self.collect_resource_use_data()
        if 'disturbance' in self.data_types:
            self.collect_disturbance_data()
        if 'landscape' in self.data_types:
            self.collect_landscape_data()
        if 'species' in self.data_types:
            self.collect_species_data()
        if 'locations' in self.data_types:
            self.collect_locations_data()
        if 'transects' in self.data_types:
            self.collect_transects_data()
        if 'validation_history' in self.data_types:
            self.collect_validation_history()
        return self.data
    
    def generate(self):
        """Generate comprehensive backup export"""
        # Generate based on format
        if self.export_format == 'excel':
            excel_file = self._create_backup_excel()
            self.report.excel_file = excel_file
        elif self.export_format == 'geojson':
            geojson_file = self._save_geojson_backup(self._create_geojson_backup())
            self.report.geojson_file = geojson_file
        elif self.export_format == 'csv':
            csv_archive = self._create_csv_archive()
            self.report.shapefile_archive = csv_archive  # Reuse field for CSV
        
        # Update metadata and status
        self.report.status = 'COMPLETED'
        self.report.progress = 100
        self.report.save()
        
        return self.report
```

**Supported Data Types:**
- `wildlife` - WildlifeObservation records
- `resource_use` - ResourceUseIncident records
- `disturbance` - DisturbanceRecord records
- `landscape` - LandscapeMonitoring records
- `species` - FaunaChecklist + FloraChecklist
- `locations` - MonitoringLocation records
- `transects` - TransectRoute records
- `validation_history` - Validation action logs

**Output Formats:**
- **Excel:** Comprehensive workbook with ALL fields for each data type (15+ columns per sheet)
- **CSV:** ZIP archive with separate CSV files for each data type
- **GeoJSON:** FeatureCollection with all spatial records

**Additional Collection Methods:**

**`collect_species_data()`** - Exports entire species checklists
- FaunaChecklist with taxonomy, endemicity, threat category, common names
- FloraChecklist with taxonomy, endemicity, threat category, common names

**`collect_validation_history()`** - Exports validation audit trail
- Attempts to use RecordValidationHistory model if available
- Falls back to validated_at/validated_by fields from observation records
- Includes: record type, record ID, action, validator, timestamp, status changes, comments

---

### 5. FinalizedReportGenerator

Generates official reports with permanent archiving.

```python
class FinalizedReportGenerator(SemestralReportGenerator):
    """Generate finalized reports for official submission"""
    
    def generate(self):
        """Generate finalized report with all components"""
        result = super().generate()  # Call SemestralReportGenerator
        
        # Mark as finalized in metadata
        self.report.metadata['finalized'] = True
        self.report.metadata['finalized_at'] = timezone.now().isoformat()
        self.report.save()
        
        # Create permanent archive record
        ReportArchive.objects.create(
            report=self.report,
            version=1,
            is_latest_version=True,
            archived_by=self.report.requested_by,
            is_permanent=True  # Never auto-deleted
        )
        
        return result
```

**Key Features:**
- Extends SemestralReportGenerator (inherits Excel + Markdown output)
- Automatically creates ReportArchive with `is_permanent=True`
- Adds finalization metadata (timestamp, finalized flag)
- Used for official submissions that require retention

---

### Factory Function: get_report_generator()

The factory function selects the appropriate generator class based on report type.

```python
def get_report_generator(report: GeneratedReport) -> BaseReportGenerator:
    """Get the appropriate report generator based on report type"""
    generators = {
        'GIS': GISReportGenerator,
        'SEMESTRAL': SemestralReportGenerator,
        'STANDARD_BMS': StandardBMSReportGenerator,
        'DIGITAL_BACKUP': DigitalBackupExporter,
        'FINALIZED': FinalizedReportGenerator,
    }
    
    generator_class = generators.get(report.report_type, BaseReportGenerator)
    return generator_class(report)
```

**Usage in Celery Task:**
```python
generator = get_report_generator(report)
generator.collect_all_data()
generator.generate()
```

---

## Async Task Processing

### generate_report_task (Celery)

The report generation process runs asynchronously to avoid blocking the web server during long-running exports.

```python
@shared_task(bind=True, max_retries=3)
def generate_report_task(self, report_id):
    """
    Background task to generate reports asynchronously.
    
    Args:
        report_id: ID of the GeneratedReport instance to process
    """
    try:
        # Get report instance
        report = GeneratedReport.objects.get(id=report_id)
        
        # Update status to processing
        report.status = 'PROCESSING'
        report.progress = 0
        report.generation_started_at = timezone.now()
        report.save()
        
        # Update progress: Initializing
        self.update_state(state='PROGRESS', meta={
            'current': 10, 'total': 100, 'status': 'Initializing...'
        })
        report.progress = 10
        report.save()
        
        # Get appropriate generator
        generator = get_report_generator(report)
        
        # Update progress: Collecting data
        self.update_state(state='PROGRESS', meta={
            'current': 30, 'total': 100, 'status': 'Collecting data...'
        })
        report.progress = 30
        report.save()
        
        # Collect all data
        generator.collect_all_data()
        
        # Update progress: Generating outputs
        self.update_state(state='PROGRESS', meta={
            'current': 60, 'total': 100, 'status': 'Generating outputs...'
        })
        report.progress = 60
        report.save()
        
        # Generate report files
        generator.generate()
        
        # Update progress: Complete
        report.status = 'COMPLETED'
        report.progress = 100
        report.generation_completed_at = timezone.now()
        report.save()
        
        logger.info(f"Report generation completed: {report.title} (ID: {report_id})")
        
    except Exception as exc:
        logger.error(f"Report generation failed: {str(exc)}", exc_info=True)
        
        # Update report status to failed
        report.status = 'FAILED'
        report.error_message = str(exc)
        report.save()
        
        # Retry task (up to max_retries)
        raise self.retry(exc=exc, countdown=60)  # Retry after 60 seconds
```

**Progress States:**
1. **0-10%:** Initializing generator
2. **10-30%:** Collecting data from database
3. **30-60%:** Processing and aggregating data
4. **60-100%:** Generating output files (Excel, GeoJSON, PDF, etc.)

**Error Handling:**
- Sets `status='FAILED'` and stores error message in `error_message` field
- Retries up to 3 times with 60-second delays
- Logs errors for debugging

**Task Configuration:**
- `bind=True`: Allows access to task instance (for `self.update_state()`)
- `max_retries=3`: Retry failed tasks up to 3 times
- Task ID can be used for client-side progress polling

---

## Security & Permissions

### Permission Decorators

#### @can_build_reports

Restricts report generation to authorized roles.

**Allowed Roles:**
- Validators
- DENR Staff
- System Admins

**Implementation:**
```python
@can_build_reports
def report_builder(request):
    # Only authorized users can access
    ...
```

**Purpose:** Prevents general users and PA staff from creating resource-intensive reports without approval.

---

#### @can_view_reports

Restricts report viewing to authenticated users with proper data access.

**Allowed Roles:**
- All authenticated users (with role-based filtering)

**Additional Checks:**
- Report access control via `report.can_access(user)`
- Public reports accessible to all authenticated users
- Private reports restricted to requester, allowed_users, and staff

**Implementation:**
```python
@can_view_reports
def report_archive_list(request):
    # All authenticated users can view their accessible reports
    ...
```

---

### Access Control Matrix

| View                  | PA Staff | Collaborator | Validator | DENR | System Admin |
|-----------------------|----------|--------------|-----------|------|--------------|
| report_builder        | ❌       | ❌           | ✅        | ✅   | ✅           |
| data_export_center    | ❌       | ❌           | ✅        | ✅   | ✅           |
| report_archive_list   | ✅*      | ✅*          | ✅        | ✅   | ✅           |
| report_preview        | ✅*      | ✅*          | ✅        | ✅   | ✅           |
| download_report       | ✅*      | ✅*          | ✅        | ✅   | ✅           |
| report_template_list  | ✅       | ✅           | ✅        | ✅   | ✅           |
| create_report_template| ✅       | ✅           | ✅        | ✅   | ✅           |
| delete_report         | ❌**     | ❌**         | ✅        | ✅   | ✅           |
| regenerate_report     | ✅*      | ✅*          | ✅        | ✅   | ✅           |
| report_status_api     | ✅*      | ✅*          | ✅        | ✅   | ✅           |
| download_shapefile    | ✅       | ✅           | ✅        | ✅   | ✅           |

**Legend:**
- ✅ Full access
- ✅* Restricted to completed reports and public/assigned reports only
- ❌ No access
- ❌** Can only delete own reports

---

### Report Access Logic

The `GeneratedReport.can_access(user)` method determines viewing permissions:

```python
def can_access(self, user) -> bool:
    """Check if a user can access this report"""
    if self.is_public:
        return True  # Public reports accessible to all authenticated users
    if self.requested_by == user:
        return True  # Report requester always has access
    if user in self.allowed_users.all():
        return True  # Explicitly granted access
    if user.is_staff or user.is_superuser:
        return True  # Staff and admins see all reports
    return False
```

**Key Concepts:**
- **Public Reports:** `is_public=True` makes report accessible to all authenticated users
- **Private Reports:** Only requester, allowed_users (many-to-many), and staff can access
- **Staff Override:** Staff and superusers bypass all access restrictions

---

## Testing Scenarios

### 1. GIS Report Generation

**Setup:**
1. Login as Validator
2. Navigate to Report Builder (`/dashboard/reports/builder/`)
3. Select "GIS-Enhanced Report" type
4. Set date range: 2024-01-01 to 2024-12-31
5. Select 2 barangays: "Poblacion", "Banga"
6. Select output formats: Excel, GeoJSON
7. Click "Generate Report"

**Expected Results:**
- Report queued with `status='PENDING'`
- Redirect to report archives
- Celery task processes in background
- After completion (30-60 seconds):
  - `status='COMPLETED'`
  - `excel_file` contains Wildlife, Resource Use, Disturbance sheets with lat/lon columns
  - `geojson_file` contains FeatureCollection with Point geometries
  - `metadata` includes total_features count
- Download buttons appear for Excel and GeoJSON formats

**Validation:**
- Excel sheets match selected date range
- GeoJSON features only include selected barangays
- Coordinate values are valid (latitude: -90 to 90, longitude: -180 to 180)
- Feature properties include species names and dates

---

### 2. Semestral BMS Report

**Setup:**
1. Login as DENR Staff
2. Navigate to Report Builder
3. Select "Semestral BMS Report" type
4. Set date range: 2024-07-01 to 2024-12-31 (semester)
5. Leave geographic scope empty (all areas)
6. Select output formats: Excel, Markdown
7. Submit

**Expected Results:**
- Report generates with Excel and Markdown files
- Excel contains:
  - Summary sheet with species richness, endemic counts, threatened counts
  - Wildlife Data sheet with endemic/threatened indicators
  - Issues & Concerns sheet combining resource use and disturbance records
- Markdown file includes:
  - Executive summary with key findings
  - Observations by location breakdown
  - Recommendations section
- `metadata` field populated with all key metrics

**Validation:**
- All observation dates within 2024-07-01 to 2024-12-31 range
- Species richness calculation accurate (unique species count)
- Endemic count matches species with `endemicity_status='Endemic'`
- Threatened count matches IUCN red list categories (CR, EN, VU, NT)

---

### 3. Digital Backup Export

**Setup:**
1. Login as System Admin
2. Navigate to Data Export Center (`/dashboard/reports/export/`)
3. Select data types: Wildlife, Resource Use, Disturbance, Species
4. Select format: Excel
5. Set date range: All time (leave empty for last 365 days)
6. Submit

**Expected Results:**
- Report type set to 'DIGITAL_BACKUP'
- Excel file generated with 4+ sheets:
  - Wildlife Observations (15+ columns including ID, Date, Time, Species, Count, Location, Lat, Lon, Submitted By, Validator, Status, Notes, Submission Time, Validated At)
  - Resource Use Incidents (similar comprehensive columns)
  - Disturbance Records (similar comprehensive columns)
  - Species Checklists (Type, ID, Scientific Name, Common Names, Family, Endemicity, Threat Category)
- All fields exported (no summary aggregation)

**Validation:**
- All observation records present (no filtering except date range)
- Species checklist includes both fauna and flora
- Validation history complete with all status changes

---

### 4. Report Template Reuse

**Setup:**
1. Login as Validator
2. Create GIS Report with specific configuration:
   - Date range: 2024-01-01 to 2024-03-31
   - Barangays: "Poblacion", "Banga"
   - Formats: Excel, GeoJSON
3. After generation, save as template:
   - Name: "Q1 2024 GIS Report"
   - Description: "Quarterly GIS report for Poblacion and Banga"
4. Navigate to Report Templates (`/dashboard/reports/templates/`)
5. Find saved template and click "Use Template"

**Expected Results:**
- Report Builder pre-fills with template configuration
- Date range, barangays, output formats match saved template
- User can modify any field before generating
- New report inherits template settings but is independent instance

**Validation:**
- Template appears in template list with creator name and date
- "Use Template" redirects to builder with `?template=<id>` parameter
- Template configuration correctly loaded into form fields

---

### 5. Report Access Control

**Setup:**
1. Login as Validator (User A)
2. Create Semestral Report
3. Set `is_public=False`
4. Do NOT add any allowed_users
5. Logout
6. Login as different Validator (User B)
7. Navigate to Report Archives

**Expected Results:**
- User B does NOT see User A's private report in archives list
- Attempting direct URL access (`/dashboard/reports/<report_id>/preview/`) returns error message
- User B can only see their own reports + public reports

**Additional Test:**
- Have User A edit report and add User B to `allowed_users`
- User B should now see and access the report

**Validation:**
- Role-based filtering enforced
- Access control method (`can_access()`) working correctly
- Public reports visible to all authenticated users
- Private reports restricted properly

---

### 6. Failed Report Handling

**Setup:**
1. Temporarily modify `BaseReportGenerator.collect_wildlife_data()` to raise exception
2. Login as Validator
3. Create any report type
4. Wait for Celery task to process

**Expected Results:**
- Report status transitions: PENDING → PROCESSING → FAILED
- `error_message` field populated with exception details
- Task retries up to 3 times (60-second intervals)
- After 3 failures, report remains in FAILED state
- User sees error message in report preview

**Validation:**
- Status accurately reflects failure
- Error message helpful for debugging
- Retry mechanism functional
- No partial file generation (files created only on success)

---

### 7. Shapefile Direct Export

**Setup:**
1. Login as PA Staff
2. Navigate to interactive map or observation list
3. Apply filters:
   - Date range: 2024-06-01 to 2024-06-30
   - Species: "Tarictic Hornbill"
   - Validation status: "approved"
4. Click "Export as Shapefile" button

**Expected Results:**
- Shapefile export endpoint called: `/dashboard/reports/shapefile/export/?date_start=2024-06-01&date_end=2024-06-30&species=Tarictic%20Hornbill&validation_status=approved`
- ZIP file downloaded containing:
  - observations.shp (shapefile)
  - observations.shx (shape index)
  - observations.dbf (attribute table)
  - observations.prj (projection file - WGS84)
- Download logged in ReportDownloadLog with `report=None`
- Temporary files cleaned up after download

**Validation:**
- Shapefile opens in QGIS/ArcGIS
- Only approved Tarictic Hornbill observations from June 2024 included
- Attribute table contains observation details
- Projection correctly set to EPSG:4326 (WGS84)

---

### 8. Report Regeneration

**Setup:**
1. Create GIS Report for January 2024
2. Wait for completion
3. Add new observations for January 2024 (backdated)
4. Click "Regenerate" button on original report

**Expected Results:**
- New GeneratedReport created with same configuration
- Title appended with "(Regenerated)"
- New report includes newly added observations
- Original report unchanged
- Both reports visible in archives

**Validation:**
- Original report observation count: X
- Regenerated report observation count: X + new observations
- Configuration identical between original and regenerated
- Both reports independently downloadable

---

### 9. Report Archive Download Tracking

**Setup:**
1. Create report and wait for completion
2. Download Excel format
3. Wait 1 minute
4. Download GeoJSON format
5. Navigate to report preview

**Expected Results:**
- Recent downloads section shows 2 entries:
  - Entry 1: Excel, timestamp, user, IP address
  - Entry 2: GeoJSON, timestamp, user, IP address
- ReportArchive `download_count` incremented to 2
- `last_downloaded_at` updated to latest download time
- `last_downloaded_by` set to current user

**Validation:**
- Download logs created in ReportDownloadLog table
- Archive tracking updated correctly
- User agent strings captured
- IP addresses recorded (if available)

---

### 10. Multi-Format Generation

**Setup:**
1. Create Standard BMS Report
2. Select ALL output formats: PDF, Excel, GeoJSON, Shapefile, Markdown
3. Submit

**Expected Results:**
- Report generates all 5 formats (if generator supports PDF)
- `get_file_count()` returns 5
- `get_available_formats()` returns ['PDF', 'Excel', 'GeoJSON', 'Shapefile', 'Markdown']
- Download dropdown shows all 5 options
- `total_file_size` accurately sums all file sizes

**Note:** Currently most generators do NOT support PDF. This test may result in 4 formats (Excel, GeoJSON, Shapefile/CSV, Markdown).

**Validation:**
- All selected formats generated
- File size calculation accurate
- Each format downloadable independently
- Content differs appropriately by format (Excel has tables, GeoJSON has geometries, etc.)

---

## Integration Points

### 1. KoboToolbox Data Integration

Reports pull from validated observations that originated from KoboToolbox submissions.

**Data Flow:**
```
KoboToolbox Form Submission
  → DataSubmission (raw Kobo JSON)
  → Validation Workflow
  → WildlifeObservation (validated record)
  → Report Generation (aggregated in reports)
```

**Integration:**
- Reports use `validation_status__in=['validated', 'approved']` filter
- Only validated records included in official reports
- Submission metadata preserved (submission_time, submitted_by)

---

### 2. Species Checklist Integration

Reports aggregate species data from centralized checklists.

**Related Models:**
- `FaunaChecklist` - Fauna species with taxonomy, endemicity, threat category
- `FloraChecklist` - Flora species with taxonomy, endemicity, threat category

**Integration:**
- Wildlife observations link to species checklists via foreign keys
- Endemicity status from `id_endemicity.endemicity_status`
- Threat category from `id_threat_category.threat_category_status`
- Common names from many-to-many relationship

---

### 3. GIS/Spatial Features Integration

Reports leverage spatial data for geographic analysis and exports.

**Integration:**
- GeoJSON export uses `latitude` and `longitude` fields from observations
- Shapefile export calls `export_shapefile()` utility from `dashboard.utils_geo`
- Geographic filtering uses `protected_area__address__barangay` relationships
- TransectSegment locations included in spatial reports

---

### 4. Validation System Integration

Reports reflect validation workflow status and validator assignments.

**Integration:**
- Validated records include `validated_by`, `validated_at`, `validation_status`
- Validation history (if RecordValidationHistory exists) can be exported
- Reports filtered by validation status (defaults to validated/approved)
- Validator names appear in export columns

---

### 5. User & Organization Integration

Reports respect organization boundaries and user roles.

**Integration:**
- PA Staff restricted to their assigned protected area's reports
- Report access controlled by organization membership
- Validator assignments tracked in reports
- Agency/department context preserved in metadata

---

### 6. Notification System Integration

Report generation completion can trigger notifications (not currently implemented but planned).

**Potential Integration:**
- Send notification when report status changes to 'COMPLETED'
- Email download links to report requester
- Alert administrators if report generation fails
- Weekly digest of generated reports for DENR staff

---

## Key Takeaways

1. **Asynchronous Architecture** - All report generation runs in background via Celery, preventing web server blocking and enabling progress tracking.

2. **Factory Pattern** - The `get_report_generator()` function provides clean extensibility for adding new report types without modifying existing code.

3. **Multiple Output Formats** - Single report request can generate up to 5 different formats simultaneously, accommodating various use cases (GIS analysis, spreadsheet review, backup archiving).

4. **Comprehensive Data Coverage** - Digital Backup exporter can export 8+ data types with full field sets, ensuring no data loss in exports.

5. **Role-Based Access Control** - Reports enforce strict permission checks with different visibility rules for PA Staff, Validators, DENR, and Admins.

6. **Template Reusability** - Users can save report configurations as templates for recurring reports, reducing repetitive form filling.

7. **Archive Management** - Built-in versioning and retention policies with download tracking for audit compliance.

8. **Geographic Filtering** - Reports can be scoped to specific barangays and monitoring locations, enabling localized analysis.

9. **Validation Integration** - Reports automatically filter by validation status, ensuring only approved data appears in official reports.

10. **Error Recovery** - Automatic retry mechanism (up to 3 attempts) handles transient failures gracefully, with detailed error logging for debugging.

---

## Future Enhancements

### Planned Features

1. **PDF Generation**
   - Add ReportLab or WeasyPrint integration for formatted PDF reports
   - Include charts (matplotlib/plotly) and maps (Folium) in PDF output
   - Template-based PDF layouts for professional formatting

2. **Scheduled Reports**
   - Celery Beat integration for automatic periodic report generation
   - Weekly, monthly, quarterly, annual schedules
   - Email delivery of scheduled reports

3. **Chart/Graph Generation**
   - Species occurrence trends (line charts)
   - Distribution maps (heatmaps)
   - Threat category pie charts
   - Endemic species bar charts

4. **Report Comparison**
   - Compare two reports side-by-side
   - Trend analysis between time periods
   - Species composition changes

5. **Dashboard Widgets**
   - Recent reports widget for homepage
   - Report generation activity timeline
   - Download statistics charts

6. **Advanced Filtering**
   - Species-specific reports
   - Transect route filtering
   - Observer/validator filtering
   - Threat category filtering

7. **Notification Integration**
   - Email notifications on report completion
   - Failure alerts to administrators
   - Weekly report generation digest

8. **Report Caching**
   - Cache frequently requested reports
   - Invalidation on new data submission
   - Reduced generation time for duplicate requests

---

**Last Updated:** January 6, 2026  
**Document Version:** 1.0  
**Related Documentation:**
- [Validation System](VALIDATION_QUEUE_WORKSPACE.md)
- [GIS/Spatial Features](GIS_SPATIAL_FEATURES.md)
- [Species Management](SPECIES_MANAGEMENT.md)
- [Dashboard & Widget System](DASHBOARD_WIDGET_SYSTEM.md)
