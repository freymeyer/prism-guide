# PRISM-Matutum Automated Test Generation Prompts

**Document Version:** 1.0  
**Last Updated:** January 7, 2026  
**Purpose:** Prompts and context for generating Python test scripts  

---

## Overview

This document provides structured prompts and context for generating Python test scripts to automate the test cases defined in `TEST_PLAN_CHECKLIST.md`. Each section contains the necessary context, models, views, and specific prompts for test generation.

---

## Project Context

### Technology Stack
- **Framework:** Django 5.2 with Django REST Framework
- **Database:** PostgreSQL 16 + PostGIS 3.x
- **Task Queue:** Celery + Redis
- **Testing Framework:** Django TestCase, pytest-django (optional)
- **Test Client:** Django Test Client, RequestFactory

### Project Structure
```
PRISM-Matutum/
├── api/                    # Core data models and services
│   ├── models/             # Split models (submissions, records, choices, etc.)
│   ├── *_services.py       # Business logic services
│   ├── *_views.py          # API views
│   └── tests/              # API tests
├── dashboard/              # Main UI module
│   ├── views_*.py          # View controllers by feature
│   ├── permissions.py      # Access control decorators/mixins
│   ├── decorators.py       # Function-based view decorators
│   └── tests/              # Dashboard tests
├── users/                  # User and organization models
├── species/                # Species management
└── prism/                  # Django project settings
```

### Test File Locations
- `api/tests.py` or `api/tests/`
- `dashboard/tests.py` or `dashboard/tests/`
- `users/tests.py`
- `species/tests.py`

### Running Tests
```bash
# Run all tests
docker-compose exec web python manage.py test

# Run specific app tests
docker-compose exec web python manage.py test api
docker-compose exec web python manage.py test dashboard

# Run specific test class
docker-compose exec web python manage.py test dashboard.tests.TestValidationQueue

# Run with verbosity
docker-compose exec web python manage.py test -v 2
```

---

## Phase 1: Authentication & Authorization Tests

### Context
```
Models: users.models.User (extends AbstractUser)
Fields: role, account_status, agency, department, is_field_verified
Views: django.contrib.auth views, custom registration views
Decorators: dashboard/permissions.py, dashboard/decorators.py
```

### Prompt

```
Make a Python test script for Django to test the Authentication & Authorization system in PRISM-Matutum.

**Test File Location:** `users/tests/test_authentication.py`

**Models to Import:**
- from users.models import User, Agency, Department

**Test Cases to Implement:**

1. **TestUserRegistration**
   - test_registration_page_accessible
   - test_registration_valid_data_creates_pending_user
   - test_registration_validates_required_fields
   - test_registration_validates_password_strength
   - test_registration_rejects_duplicate_username
   - test_registration_rejects_duplicate_email
   - test_registration_allows_role_selection
   - test_new_user_has_pending_status

2. **TestUserLogin**
   - test_login_page_accessible
   - test_login_valid_credentials_success
   - test_login_invalid_credentials_fails
   - test_login_pending_user_blocked
   - test_login_rejected_user_blocked
   - test_login_suspended_user_blocked
   - test_login_redirects_to_role_dashboard
   - test_session_timeout_works

3. **TestPasswordManagement**
   - test_password_reset_request
   - test_password_reset_email_sent
   - test_password_reset_token_works
   - test_password_change_requires_old_password

4. **TestLogout**
   - test_logout_clears_session
   - test_logged_out_user_cannot_access_protected

**Setup Requirements:**
- Create test users with different statuses (pending, approved, rejected, suspended)
- Create test agency and department
- Use Django TestCase and Client

**Example Test Structure:**
```python
from django.test import TestCase, Client
from django.urls import reverse
from users.models import User, Agency

class TestUserRegistration(TestCase):
    def setUp(self):
        self.client = Client()
        self.agency = Agency.objects.create(name="Test Agency")
    
    def test_registration_page_accessible(self):
        response = self.client.get(reverse('register'))
        self.assertEqual(response.status_code, 200)
    
    def test_registration_creates_pending_user(self):
        data = {
            'username': 'newuser',
            'email': 'new@test.com',
            'password1': 'SecurePass123!',
            'password2': 'SecurePass123!',
            'role': 'pa_staff',
        }
        response = self.client.post(reverse('register'), data)
        user = User.objects.get(username='newuser')
        self.assertEqual(user.account_status, 'pending')
```
```

---

## Phase 2: User & Organization Management Tests

### Context
```
Models: 
- users.models.User, Agency, Department
- users.models.Region, Province, Municipality, Barangay
Views: dashboard/views_admin.py
Permissions: StaffUserRequiredMixin, @admin_required
URLs: /dashboard/admin/user-organization-hub/
```

### Prompt

```
Make a Python test script for Django to test User & Organization Management in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_user_management.py`

**Models to Import:**
- from users.models import User, Agency, Department, Region, Province, Municipality, Barangay

**Test Cases to Implement:**

1. **TestUserApprovalWorkflow**
   - test_pending_users_appear_in_queue
   - test_admin_can_view_pending_user_details
   - test_admin_can_approve_user
   - test_approved_user_can_login
   - test_admin_can_reject_user_with_reason
   - test_rejection_requires_reason
   - test_admin_can_suspend_user
   - test_admin_can_reactivate_user

2. **TestUserManagement**
   - test_admin_can_view_user_list
   - test_user_list_pagination
   - test_user_list_filter_by_role
   - test_user_list_filter_by_status
   - test_user_list_search
   - test_admin_can_edit_user
   - test_admin_can_change_user_role
   - test_admin_can_assign_agency
   - test_admin_can_mark_field_verified

3. **TestAgencyManagement**
   - test_admin_can_create_agency
   - test_agency_type_options
   - test_agency_hierarchy_parent
   - test_admin_can_edit_agency
   - test_admin_can_deactivate_agency
   - test_cannot_delete_agency_with_users

4. **TestDepartmentManagement**
   - test_admin_can_create_department
   - test_department_requires_agency
   - test_department_hierarchy
   - test_admin_can_assign_head

5. **TestGeographicManagement**
   - test_admin_can_create_region
   - test_province_requires_region
   - test_municipality_requires_province
   - test_barangay_requires_municipality
   - test_address_cascade_protection

**Permission Tests:**
- test_non_admin_cannot_access_user_management
- test_non_admin_cannot_approve_users

**Setup Requirements:**
- Create admin user (role='system_admin' or is_superuser=True)
- Create non-admin users for permission tests
- Create sample agency/department hierarchy
- Create geographic address hierarchy

**Key URLs to Test:**
- /dashboard/admin/user-organization-hub/
- /dashboard/admin/users/<pk>/approve/
- /dashboard/admin/users/<pk>/reject/
- /dashboard/admin/agencies/
- /dashboard/admin/departments/
```

---

## Phase 3: PA Staff Workflow Tests

### Context
```
Models:
- api.models.DataSubmission
- api.models.WildlifeObservation, ResourceUseIncident, DisturbanceRecord, LandscapeMonitoring
Views: dashboard/views_pa_staff.py
Permissions: @pa_staff_required, PAStaffAccessMixin
URLs: /dashboard/pa-staff/
```

### Prompt

```
Make a Python test script for Django to test PA Staff Workflows in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_pa_staff.py`

**Models to Import:**
- from users.models import User
- from api.models import DataSubmission, WildlifeObservation, ResourceUseIncident

**Test Cases to Implement:**

1. **TestPAStaffDashboard**
   - test_pa_staff_can_access_dashboard
   - test_dashboard_shows_total_submissions
   - test_dashboard_shows_pending_count
   - test_dashboard_shows_approved_count
   - test_dashboard_shows_week_submissions
   - test_dashboard_shows_observation_count
   - test_approval_rate_calculation
   - test_field_verification_alert_shows
   - test_recent_submissions_display

2. **TestMySubmissionsView**
   - test_pa_staff_sees_own_submissions_only
   - test_filter_by_status_pending
   - test_filter_by_status_approved
   - test_filter_by_status_rejected
   - test_filter_by_date_range
   - test_search_by_submission_id
   - test_search_by_remarks
   - test_pagination_25_per_page
   - test_status_badges_display

3. **TestSubmissionDetailView**
   - test_pa_staff_can_view_own_submission
   - test_pa_staff_cannot_view_others_submission
   - test_submission_fields_display
   - test_validation_remarks_display
   - test_validator_info_display
   - test_submission_is_readonly

4. **TestMyObservationsView**
   - test_pa_staff_sees_approved_observations
   - test_wildlife_observations_listed
   - test_resource_use_incidents_listed
   - test_disturbance_records_listed
   - test_filter_by_observation_type

**Setup Requirements:**
- Create PA Staff user with approved status
- Create sample DataSubmission records with various statuses
- Create processed observation records linked to submissions

**Example Test:**
```python
from django.test import TestCase, Client
from django.urls import reverse
from users.models import User
from api.models import DataSubmission

class TestPAStaffDashboard(TestCase):
    def setUp(self):
        self.client = Client()
        self.pa_staff = User.objects.create_user(
            username='pa_staff_test',
            password='testpass123',
            role='pa_staff',
            account_status='approved'
        )
        # Create submissions
        for i in range(5):
            DataSubmission.objects.create(
                username=self.pa_staff.username,
                validation_status='---' if i < 3 else 'validation_status_approved',
                form_name='Wildlife Observation'
            )
    
    def test_dashboard_shows_correct_counts(self):
        self.client.login(username='pa_staff_test', password='testpass123')
        response = self.client.get(reverse('dashboard:pa_staff_home'))
        self.assertEqual(response.context['stats']['total'], 5)
        self.assertEqual(response.context['stats']['pending'], 3)
        self.assertEqual(response.context['stats']['approved'], 2)
```
```

---

## Phase 4: Validation System Tests

### Context
```
Models:
- api.models.DataSubmission (validation_status, validated_by, validation_remarks)
- api.models.WildlifeObservation (created from approved submissions)
Views: dashboard/views_validation.py
Services: api/workflow_services.py (BMSWorkflowService)
Permissions: @validator_required, ValidatorAccessMixin
URLs: /dashboard/validation/queue/, /dashboard/validation/workspace/<pk>/
```

### Prompt

```
Make a Python test script for Django to test the Validation System in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_validation.py`

**Models to Import:**
- from users.models import User
- from api.models import DataSubmission, WildlifeObservation
- from api.workflow_services import BMSWorkflowService

**Test Cases to Implement:**

1. **TestValidationQueue**
   - test_validator_can_access_queue
   - test_queue_statistics_accurate
   - test_filter_by_status
   - test_filter_by_date_range
   - test_filter_by_pa_staff
   - test_filter_by_priority
   - test_search_by_submission_id
   - test_sorting_options
   - test_pagination_50_per_page
   - test_high_priority_indicator

2. **TestBulkAssignment**
   - test_assign_to_self
   - test_assign_multiple_submissions
   - test_assign_to_other_validator
   - test_assignment_creates_log
   - test_assignment_notification_created

3. **TestValidationWorkspace**
   - test_workspace_displays_submission_data
   - test_workspace_shows_media
   - test_workspace_shows_location_map
   - test_validation_actions_visible
   - test_remarks_field_available
   - test_navigation_between_submissions

4. **TestApproveAction**
   - test_approve_changes_status
   - test_approve_creates_observation_record
   - test_approve_sets_validator
   - test_approve_sets_timestamp
   - test_approve_with_remarks
   - test_pa_staff_notified_on_approve
   - test_approved_removed_from_queue

5. **TestRejectAction**
   - test_reject_requires_remarks
   - test_reject_changes_status
   - test_reject_does_not_create_record
   - test_pa_staff_notified_with_reason
   - test_rejected_is_final

6. **TestHoldAction**
   - test_hold_changes_status
   - test_hold_requires_remarks
   - test_hold_remains_in_queue
   - test_hold_can_be_rereviewed

7. **TestFlagAction**
   - test_flag_changes_status
   - test_flag_requires_remarks
   - test_flag_shows_high_priority
   - test_supervisor_notified

8. **TestValidationHistory**
   - test_validator_sees_history
   - test_filter_by_decision
   - test_export_history
   - test_performance_metrics

**Permission Tests:**
- test_pa_staff_cannot_access_queue
- test_denr_cannot_access_queue
- test_collaborator_cannot_access_queue

**Setup Requirements:**
- Create validator user
- Create PA Staff user
- Create submissions with various statuses
- Mock notification system

**Key Service Methods to Test:**
```python
from api.workflow_services import BMSWorkflowService

# Test the service directly
service = BMSWorkflowService()
result = service.approve_submission(submission, validator, remarks="Approved")
```
```

---

## Phase 5: Species Management Tests

### Context
```
Models:
- species.models.FaunaChecklist, FloraChecklist
- species.models.Taxonomy, Kingdom, Phylum, TaxonomyClass, TaxonomyOrder, Family, Genus
- species.models.Endemicity, ThreatAssessment
- species.models.FaunaCommonName, FloraCommonName
Views: dashboard/views_species.py, dashboard/views_articles.py
Permissions: Admin for CRUD, All roles for view
```

### Prompt

```
Make a Python test script for Django to test Species Management in PRISM-Matutum.

**Test File Location:** `species/tests/test_species.py`

**Models to Import:**
- from species.models import (
    FaunaChecklist, FloraChecklist, Taxonomy,
    Kingdom, Phylum, TaxonomyClass, TaxonomyOrder, Family, Genus,
    Endemicity, ThreatAssessment, FaunaCommonName, FloraCommonName,
    SpeciesType, FaunalCharacteristics, FloralCharacteristics
)

**Test Cases to Implement:**

1. **TestSpeciesListView**
   - test_species_list_accessible
   - test_fauna_checklist_displays
   - test_flora_checklist_displays
   - test_filter_by_species_type
   - test_filter_by_conservation_status
   - test_filter_by_endemicity
   - test_search_by_scientific_name
   - test_search_by_common_name
   - test_pagination

2. **TestSpeciesDetailView**
   - test_species_detail_accessible
   - test_taxonomy_hierarchy_displays
   - test_conservation_status_displays
   - test_endemicity_displays
   - test_common_names_display
   - test_characteristics_display
   - test_media_gallery_displays
   - test_observation_history_displays

3. **TestSpeciesCreateView (Admin)**
   - test_admin_can_access_create
   - test_non_admin_cannot_create
   - test_create_fauna_species
   - test_create_flora_species
   - test_taxonomy_selection_works
   - test_auto_classification_fauna
   - test_auto_classification_flora

4. **TestSpeciesUpdateView (Admin)**
   - test_admin_can_edit_species
   - test_update_conservation_status
   - test_update_endemicity

5. **TestSpeciesDeleteView (Admin)**
   - test_admin_can_delete
   - test_cannot_delete_with_observations
   - test_delete_confirmation_required

6. **TestCommonNames**
   - test_add_common_name
   - test_multiple_common_names
   - test_edit_common_name
   - test_delete_common_name

7. **TestSpeciesArticles**
   - test_add_article_to_species
   - test_edit_article
   - test_delete_article
   - test_external_links_work

8. **TestSpeciesMedia**
   - test_upload_species_image
   - test_multiple_images
   - test_delete_image
   - test_thumbnail_generation

**Auto-Classification Test:**
```python
def test_fauna_auto_classification(self):
    # Taxonomy with class 'Aves' should auto-assign SpeciesType 'Bird'
    taxonomy = self._create_taxonomy(class_name='Aves')
    fauna = FaunaChecklist.objects.create(
        scientific_name='Test species',
        id_taxonomy=taxonomy,
        # ... other required fields
    )
    self.assertEqual(fauna.id_species_type.type, 'Bird')
```
```

---

## Phase 6: GIS & Spatial Features Tests

### Context
```
Models:
- users.models.ProtectedArea (boundary: MultiPolygonField)
- api.models.TransectRoute (route_geometry: LineStringField)
- api.models.MonitoringLocation (geometry: PointField)
- api.models.TransectSegment
Views: dashboard/views_gis.py, dashboard/api_geo.py
Database: PostgreSQL + PostGIS
```

### Prompt

```
Make a Python test script for Django to test GIS & Spatial Features in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_gis.py`

**Models to Import:**
- from users.models import ProtectedArea
- from api.models import TransectRoute, MonitoringLocation, TransectSegment
- from django.contrib.gis.geos import Point, LineString, MultiPolygon, Polygon

**Test Cases to Implement:**

1. **TestProtectedAreaManagement**
   - test_protected_area_list_displays
   - test_protected_area_detail_view
   - test_boundary_displays_on_map
   - test_associated_landscapes_display
   - test_transect_routes_display
   - test_protected_area_statistics
   - test_admin_can_edit_pa_name

2. **TestTransectRouteManagement**
   - test_transect_route_list
   - test_create_transect_route
   - test_assign_to_protected_area
   - test_add_stations_to_route
   - test_geometry_generates_from_stations
   - test_route_displays_on_map
   - test_edit_transect_route
   - test_deactivate_route
   - test_segments_auto_generate

3. **TestMonitoringLocationManagement**
   - test_location_list_displays
   - test_create_monitoring_location
   - test_location_type_options
   - test_gps_coordinates_saved
   - test_location_displays_on_map
   - test_edit_location
   - test_deactivate_location

4. **TestInteractiveMap**
   - test_map_page_loads
   - test_base_layer_options
   - test_protected_area_layer
   - test_observations_layer
   - test_transect_routes_layer
   - test_monitoring_locations_layer
   - test_layer_toggle
   - test_marker_popup
   - test_map_filtering

5. **TestGeoJSONAPI**
   - test_protected_areas_geojson
   - test_transect_routes_geojson
   - test_monitoring_locations_geojson
   - test_observations_geojson
   - test_geojson_rfc7946_format
   - test_api_respects_permissions
   - test_pagination_large_datasets

6. **TestShapefileExport**
   - test_shapefile_export_available
   - test_shapefile_generates_zip
   - test_shapefile_valid_geometry
   - test_attributes_mapped
   - test_download_works

7. **TestSpatialAssignment**
   - test_observation_assigns_to_nearest_segment
   - test_50m_buffer_matching
   - test_protected_area_assignment

**Spatial Test Examples:**
```python
from django.contrib.gis.geos import Point, LineString
from django.test import TestCase

class TestSpatialFeatures(TestCase):
    def test_monitoring_location_geometry(self):
        location = MonitoringLocation.objects.create(
            name='Test Station',
            geometry=Point(125.5, 6.5, srid=4326),  # Longitude, Latitude
            location_type='transect_station'
        )
        self.assertEqual(location.geometry.x, 125.5)
        self.assertEqual(location.geometry.y, 6.5)
    
    def test_transect_geometry_from_stations(self):
        route = TransectRoute.objects.create(
            name='Test Route',
            protected_area=self.protected_area
        )
        station1 = MonitoringLocation.objects.create(
            name='Station 1', geometry=Point(125.5, 6.5, srid=4326)
        )
        station2 = MonitoringLocation.objects.create(
            name='Station 2', geometry=Point(125.6, 6.6, srid=4326)
        )
        route.stations.add(station1, station2)
        route.update_geometry_from_stations()
        
        self.assertIsNotNone(route.route_geometry)
        self.assertEqual(route.route_geometry.num_points, 2)
```
```

---

## Phase 7: Dashboard & Widget System Tests

### Context
```
Models:
- api.models.Dashboard, DashboardWidget, DashboardTemplate, WidgetDataCache
Views: dashboard/views_dashboard.py, dashboard/views_widgets.py
Widget Types: counter, pie_chart, bar_chart, line_chart, data_table, map, text, etc.
```

### Prompt

```
Make a Python test script for Django to test the Dashboard & Widget System in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_dashboards.py`

**Models to Import:**
- from api.models import Dashboard, DashboardWidget, DashboardTemplate, WidgetDataCache
- from users.models import User

**Test Cases to Implement:**

1. **TestCustomDashboard**
   - test_user_can_access_dashboard
   - test_default_dashboard_auto_creates
   - test_role_specific_default_widgets_pa_staff
   - test_role_specific_default_widgets_validator
   - test_dashboard_displays_widgets
   - test_widget_data_loads
   - test_multiple_dashboards_supported
   - test_dashboard_switcher

2. **TestDashboardCRUD**
   - test_create_dashboard
   - test_set_dashboard_name
   - test_edit_dashboard
   - test_delete_dashboard
   - test_only_one_default
   - test_duplicate_dashboard

3. **TestWidgetManagement**
   - test_widget_library_accessible
   - test_add_widget_to_dashboard
   - test_widget_type_selection
   - test_widget_configuration
   - test_widget_position
   - test_widget_resize
   - test_edit_widget
   - test_delete_widget
   - test_widget_visibility_toggle

4. **TestWidgetTypes**
   - test_counter_widget_data
   - test_pie_chart_widget
   - test_bar_chart_widget
   - test_line_chart_widget
   - test_data_table_widget
   - test_map_widget
   - test_text_widget
   - test_recent_activity_widget
   - test_validation_stats_widget
   - test_species_summary_widget

5. **TestDashboardTemplates**
   - test_template_gallery_accessible
   - test_system_templates_display
   - test_template_preview
   - test_instantiate_template
   - test_create_template_from_dashboard

6. **TestDashboardSharing**
   - test_mark_dashboard_shared
   - test_select_shared_users
   - test_shared_dashboard_visible
   - test_shared_users_cannot_edit

7. **TestWidgetAutoRefresh**
   - test_refresh_interval_config
   - test_manual_refresh
   - test_widget_cache_works
   - test_cache_expiration

**Widget Data Test Example:**
```python
def test_counter_widget_data(self):
    # Create submissions for stats
    for _ in range(10):
        DataSubmission.objects.create(
            username=self.pa_staff.username,
            validation_status='validation_status_approved'
        )
    
    widget = DashboardWidget.objects.create(
        dashboard=self.dashboard,
        widget_type='counter',
        title='Total Approved',
        configuration={'data_source': 'approved_submissions'}
    )
    
    data = widget.get_data()
    self.assertEqual(data['value'], 10)
```
```

---

## Phase 8: Report Generation System Tests

### Context
```
Models:
- dashboard.models.ReportTemplate, GeneratedReport, ReportArchive
Services: dashboard/report_services.py
Tasks: dashboard/tasks.py (generate_report_task)
Report Types: GIS, SEMESTRAL, STANDARD_BMS, DIGITAL_BACKUP, FINALIZED
Output Formats: PDF, Excel, GeoJSON, Shapefile, Markdown
```

### Prompt

```
Make a Python test script for Django to test the Report Generation System in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_reports.py`

**Models to Import:**
- from dashboard.models import ReportTemplate, GeneratedReport, ReportArchive
- from dashboard.report_services import (
    GISReportGenerator, SemestralReportGenerator,
    StandardBMSReportGenerator, DigitalBackupExporter
)

**Test Cases to Implement:**

1. **TestReportBuilder**
   - test_report_builder_accessible
   - test_report_type_selection
   - test_date_range_picker
   - test_geographic_scope_selection
   - test_monitoring_location_filter
   - test_species_filter
   - test_output_format_selection
   - test_include_options (maps, charts, photos)

2. **TestReportGeneration**
   - test_report_request_submitted
   - test_report_status_pending
   - test_celery_task_created
   - test_status_updates_processing
   - test_status_updates_completed
   - test_status_updates_failed
   - test_error_message_on_failure
   - test_notification_on_complete

3. **TestReportTypes**
   - test_gis_report_generation
   - test_gis_report_includes_maps
   - test_semestral_report_format
   - test_standard_bms_report
   - test_digital_backup_export
   - test_finalized_report

4. **TestReportOutputFormats**
   - test_pdf_generation
   - test_excel_generation
   - test_geojson_generation
   - test_shapefile_generation
   - test_markdown_generation
   - test_multiple_formats_single_report

5. **TestReportArchive**
   - test_archive_page_accessible
   - test_user_reports_listed
   - test_filter_by_type
   - test_filter_by_date
   - test_filter_by_status
   - test_view_report_details
   - test_download_report_file
   - test_pdf_preview

6. **TestReportTemplates**
   - test_save_template
   - test_template_list
   - test_create_from_template
   - test_edit_template
   - test_delete_template

7. **TestReportAccessControl**
   - test_creator_can_access
   - test_public_report_access
   - test_private_report_restricted
   - test_allowed_users_access
   - test_staff_full_access

8. **TestDataExportCenter**
   - test_quick_export_options
   - test_wildlife_export
   - test_species_checklist_export
   - test_spatial_data_export
   - test_export_respects_filters

**Async Task Test (using Celery test utilities):**
```python
from unittest.mock import patch
from dashboard.tasks import generate_report_task

class TestReportGeneration(TestCase):
    @patch('dashboard.tasks.generate_report_task.delay')
    def test_report_task_queued(self, mock_task):
        response = self.client.post(reverse('dashboard:report_builder'), {
            'report_type': 'GIS',
            'date_from': '2026-01-01',
            'date_to': '2026-01-31',
        })
        mock_task.assert_called_once()
```
```

---

## Phase 9: Event & Schedule Management Tests

### Context
```
Models:
- dashboard.models.Event
- api.models.ScheduledSurvey
- api.models.FGDSession, FGDParticipant, FGDFinding, FGDAttachment
Views: dashboard/views_events.py, dashboard/views_schedule.py, dashboard/views_fgd.py
```

### Prompt

```
Make a Python test script for Django to test Event & Schedule Management in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_events.py`

**Models to Import:**
- from dashboard.models import Event
- from api.models import ScheduledSurvey, FGDSession, FGDParticipant, FGDFinding

**Test Cases to Implement:**

1. **TestEventsSystem**
   - test_event_list_accessible
   - test_events_display
   - test_filter_by_type
   - test_filter_by_date
   - test_filter_by_protected_area
   - test_past_event_indicator
   - test_ongoing_event_indicator
   - test_calendar_view

2. **TestEventCRUD**
   - test_create_event
   - test_event_title_required
   - test_event_type_selection
   - test_datetime_picker
   - test_location_field
   - test_protected_area_link
   - test_attendees_selection
   - test_public_private_toggle
   - test_edit_event
   - test_delete_event

3. **TestScheduledSurveys**
   - test_survey_list_displays
   - test_status_filters
   - test_create_survey
   - test_transect_route_selection
   - test_staff_assignment
   - test_time_picker
   - test_edit_survey
   - test_cancel_survey

4. **TestSurveyStatusTracking**
   - test_auto_update_in_progress
   - test_auto_update_overdue
   - test_auto_update_completed
   - test_related_submissions_link

5. **TestFGDSessions**
   - test_fgd_list_displays
   - test_filter_by_status
   - test_filter_by_protected_area
   - test_filter_by_barangay
   - test_create_fgd_session
   - test_facilitator_assignment
   - test_co_facilitators
   - test_edit_session
   - test_cancel_session

6. **TestFGDParticipants**
   - test_add_participant
   - test_participant_demographics
   - test_participant_list
   - test_remove_participant
   - test_participant_count_updates

7. **TestFGDFindings**
   - test_add_finding
   - test_finding_categories
   - test_edit_finding
   - test_delete_finding
   - test_summary_field

8. **TestFGDAttachments**
   - test_upload_audio_recording
   - test_upload_consent_form
   - test_upload_photos
   - test_download_attachment
   - test_delete_attachment

9. **TestNotifications**
   - test_attendee_notification_event
   - test_staff_notification_survey
   - test_reminder_notifications
   - test_overdue_alerts
```

---

## Phase 10: KoboToolbox Integration Tests

### Context
```
Models:
- api.models.ChoiceList, Choice, FormChoiceMapping
- api.models.DataSubmission
Services: api/form_services.py, api/submission_services.py
Tasks: api/tasks.py (sync_kobotoolbox_data)
Management Commands: initialize_choice_lists, sync_kobotoolbox_data
```

### Prompt

```
Make a Python test script for Django to test KoboToolbox Integration in PRISM-Matutum.

**Test File Location:** `api/tests/test_kobo_integration.py`

**Models to Import:**
- from api.models import ChoiceList, Choice, FormChoiceMapping, DataSubmission

**Test Cases to Implement:**

1. **TestChoiceLists**
   - test_choice_list_management_accessible
   - test_all_choice_lists_display
   - test_create_choice_list
   - test_edit_choice_list
   - test_delete_choice_list_without_mappings
   - test_cannot_delete_with_mappings
   - test_default_lists_initialize

2. **TestIndividualChoices**
   - test_choices_display_in_list
   - test_add_choice
   - test_choice_name_value
   - test_choice_label_display
   - test_choice_order
   - test_activate_deactivate
   - test_link_to_fauna_species
   - test_link_to_flora_species
   - test_link_to_monitoring_location
   - test_edit_choice
   - test_delete_choice

3. **TestFormChoiceMappings**
   - test_mappings_page_accessible
   - test_create_mapping
   - test_kobo_asset_uid
   - test_question_name
   - test_link_choice_list
   - test_auto_update_toggle
   - test_edit_mapping
   - test_delete_mapping

4. **TestBidirectionalSync**
   - test_django_to_kobo_sync_initiates
   - test_xlsform_modified
   - test_form_upload_to_kobo
   - test_form_update_log_created
   - test_kobo_to_django_import
   - test_import_creates_choices
   - test_import_updates_labels

5. **TestDataSubmissionSync**
   - test_sync_task_runs
   - test_new_submissions_created
   - test_submission_data_stored
   - test_duplicate_handling
   - test_sync_errors_logged
   - test_manual_sync_trigger

**Mocking KoboToolbox API:**
```python
from unittest.mock import patch, Mock
from api.form_services import KoboToolboxService

class TestKoboIntegration(TestCase):
    @patch('api.form_services.requests.get')
    def test_fetch_kobo_forms(self, mock_get):
        mock_response = Mock()
        mock_response.json.return_value = {
            'results': [
                {'uid': 'abc123', 'name': 'Wildlife Form'},
                {'uid': 'def456', 'name': 'Resource Use Form'}
            ]
        }
        mock_response.status_code = 200
        mock_get.return_value = mock_response
        
        service = KoboToolboxService()
        forms = service.get_forms()
        
        self.assertEqual(len(forms), 2)
    
    @patch('api.form_services.requests.get')
    def test_sync_submissions(self, mock_get):
        mock_response = Mock()
        mock_response.json.return_value = {
            'results': [
                {
                    '_id': 12345,
                    'species_observed': 'Pithecophaga jefferyi',
                    '_submission_time': '2026-01-07T10:00:00'
                }
            ]
        }
        mock_response.status_code = 200
        mock_get.return_value = mock_response
        
        # Run sync
        from api.tasks import sync_kobotoolbox_data
        sync_kobotoolbox_data()
        
        # Verify submission created
        self.assertTrue(DataSubmission.objects.filter(kobo_id=12345).exists())
```
```

---

## Phase 11: Cross-Role Access Control Tests

### Context
```
Decorators: dashboard/decorators.py, dashboard/permissions.py
Mixins: ValidatorAccessMixin, PAStaffAccessMixin, DENRAccessMixin, etc.
Roles: pa_staff, validator, denr, collaborator, system_admin
```

### Prompt

```
Make a Python test script for Django to test Cross-Role Access Control in PRISM-Matutum.

**Test File Location:** `dashboard/tests/test_permissions.py`

**Models to Import:**
- from users.models import User
- from dashboard.permissions import (
    dashboard_access_required, multi_role_required,
    validator_required, pa_staff_required, denr_required, admin_required
)

**Test Cases to Implement:**

1. **TestPAStaffRestrictions**
   - test_pa_staff_accesses_own_dashboard
   - test_pa_staff_blocked_from_validation_queue
   - test_pa_staff_blocked_from_validation_workspace
   - test_pa_staff_blocked_from_admin
   - test_pa_staff_blocked_from_others_submissions
   - test_pa_staff_blocked_from_user_management
   - test_pa_staff_sees_only_own_submissions

2. **TestValidatorPermissions**
   - test_validator_accesses_validator_dashboard
   - test_validator_accesses_validation_queue
   - test_validator_accesses_validation_workspace
   - test_validator_accesses_species_verification
   - test_validator_accesses_reports
   - test_validator_blocked_from_user_management
   - test_validator_cannot_modify_others_decisions

3. **TestDENRAccess**
   - test_denr_accesses_denr_dashboard
   - test_denr_views_approved_observations
   - test_denr_accesses_reports_archive
   - test_denr_can_export_data
   - test_denr_accesses_maps
   - test_denr_blocked_from_validation_queue
   - test_denr_blocked_from_admin
   - test_denr_sees_only_validated_data

4. **TestCollaboratorAccess**
   - test_collaborator_accesses_dashboard
   - test_collaborator_views_approved_observations
   - test_collaborator_accesses_species_explorer
   - test_collaborator_views_public_reports
   - test_collaborator_exports_limited_data
   - test_collaborator_blocked_from_pending_data
   - test_collaborator_blocked_from_submitting
   - test_collaborator_blocked_from_validation
   - test_collaborator_sees_anonymized_staff
   - test_sensitive_species_coordinates_obscured

5. **TestSystemAdminFullAccess**
   - test_admin_accesses_all_dashboards
   - test_admin_accesses_validation_queue
   - test_admin_can_validate
   - test_admin_manages_users
   - test_admin_manages_organizations
   - test_admin_manages_choice_lists
   - test_admin_accesses_django_admin
   - test_admin_manages_species
   - test_admin_manages_geography
   - test_admin_bypasses_permission_checks

6. **TestDecoratorEnforcement**
   - test_dashboard_access_blocks_unapproved
   - test_multi_role_blocks_unauthorized
   - test_validator_required_works
   - test_pa_staff_required_works
   - test_denr_required_works
   - test_admin_required_works
   - test_ajax_returns_json_403
   - test_non_ajax_redirects_with_message
   - test_rate_limiting

**Permission Test Pattern:**
```python
class TestCrossRoleAccess(TestCase):
    def setUp(self):
        self.client = Client()
        self.pa_staff = User.objects.create_user(
            username='pa_staff', password='test', 
            role='pa_staff', account_status='approved'
        )
        self.validator = User.objects.create_user(
            username='validator', password='test',
            role='validator', account_status='approved'
        )
        self.denr = User.objects.create_user(
            username='denr', password='test',
            role='denr', account_status='approved'
        )
        self.admin = User.objects.create_user(
            username='admin', password='test',
            role='system_admin', account_status='approved',
            is_superuser=True
        )
    
    def test_pa_staff_blocked_from_validation_queue(self):
        self.client.login(username='pa_staff', password='test')
        response = self.client.get(reverse('dashboard:validation_queue'))
        self.assertEqual(response.status_code, 403)
    
    def test_validator_accesses_validation_queue(self):
        self.client.login(username='validator', password='test')
        response = self.client.get(reverse('dashboard:validation_queue'))
        self.assertEqual(response.status_code, 200)
    
    def test_ajax_returns_json_403(self):
        self.client.login(username='pa_staff', password='test')
        response = self.client.get(
            reverse('dashboard:validation_queue'),
            HTTP_X_REQUESTED_WITH='XMLHttpRequest'
        )
        self.assertEqual(response.status_code, 403)
        self.assertEqual(response['Content-Type'], 'application/json')
```
```

---

## Phase 12: System Integration & Performance Tests

### Context
```
Services: Celery, Redis, PostgreSQL/PostGIS
Configuration: prism/settings.py, docker-compose.yml
```

### Prompt

```
Make a Python test script for Django to test System Integration & Performance in PRISM-Matutum.

**Test File Location:** `api/tests/test_integration.py`

**Test Cases to Implement:**

1. **TestCeleryTaskProcessing**
   - test_celery_worker_runs
   - test_redis_connection
   - test_kobo_sync_task_runs
   - test_report_generation_task
   - test_task_status_monitoring
   - test_failed_task_logging
   - test_task_retry_mechanism

2. **TestDatabaseOperations**
   - test_postgresql_connection
   - test_postgis_extension_functional
   - test_spatial_queries_execute
   - test_migrations_clean
   - test_no_n_plus_1_queries
   - test_complex_query_performance

3. **TestFileUploadStorage**
   - test_image_upload
   - test_audio_upload
   - test_document_upload
   - test_file_size_limits
   - test_file_type_validation
   - test_correct_storage_directory
   - test_file_retrieval
   - test_thumbnail_generation

4. **TestAPIEndpoints**
   - test_api_authentication
   - test_geojson_endpoints
   - test_api_pagination
   - test_api_filtering
   - test_api_error_responses
   - test_api_rate_limiting

5. **TestFrontend**
   - test_pages_no_js_errors
   - test_responsive_design
   - test_leaflet_maps_render
   - test_charts_render
   - test_form_validation
   - test_ajax_requests
   - test_loading_indicators
   - test_messages_display

6. **TestEmailNotifications**
   - test_email_configuration
   - test_registration_email
   - test_approval_email
   - test_password_reset_email
   - test_validation_notification_email

7. **TestErrorHandling**
   - test_404_page
   - test_403_page
   - test_500_page
   - test_error_logging
   - test_no_sensitive_info_exposed

8. **TestPerformance**
   - test_dashboard_load_time
   - test_list_1000_records
   - test_map_1000_points
   - test_report_large_dataset
   - test_export_large_dataset
   - test_concurrent_users

9. **TestDockerEnvironment**
   - test_containers_running
   - test_container_communication
   - test_volume_persistence
   - test_environment_variables

**Performance Test Example:**
```python
import time
from django.test import TestCase

class TestPerformance(TestCase):
    def test_dashboard_load_time(self):
        self.client.login(username='validator', password='test')
        
        start = time.time()
        response = self.client.get(reverse('dashboard:validator_dashboard'))
        elapsed = time.time() - start
        
        self.assertEqual(response.status_code, 200)
        self.assertLess(elapsed, 3.0, "Dashboard should load in under 3 seconds")
    
    def test_list_1000_records(self):
        # Create 1000 submissions
        submissions = [
            DataSubmission(username='test', form_name='Test', validation_status='---')
            for _ in range(1000)
        ]
        DataSubmission.objects.bulk_create(submissions)
        
        self.client.login(username='validator', password='test')
        
        start = time.time()
        response = self.client.get(reverse('dashboard:validation_queue'))
        elapsed = time.time() - start
        
        self.assertEqual(response.status_code, 200)
        self.assertLess(elapsed, 5.0, "Queue should load in under 5 seconds")
```

**N+1 Query Detection:**
```python
from django.test.utils import CaptureQueriesContext
from django.db import connection

class TestQueryOptimization(TestCase):
    def test_no_n_plus_1_queries(self):
        # Create test data
        for i in range(20):
            DataSubmission.objects.create(...)
        
        self.client.login(username='validator', password='test')
        
        with CaptureQueriesContext(connection) as context:
            response = self.client.get(reverse('dashboard:validation_queue'))
        
        # Should have consistent query count regardless of record count
        # Typically: auth queries + main query + prefetch queries
        self.assertLess(len(context), 10, 
            f"Too many queries: {len(context)}. Possible N+1 issue.")
```
```

---

## Running All Tests

### Full Test Suite Command

```bash
# Run all tests with coverage
docker-compose exec web coverage run --source='.' manage.py test

# Generate coverage report
docker-compose exec web coverage report

# Generate HTML coverage report
docker-compose exec web coverage html
```

### Test by Phase

```bash
# Phase 1: Authentication
docker-compose exec web python manage.py test users.tests.test_authentication

# Phase 2: User Management
docker-compose exec web python manage.py test dashboard.tests.test_user_management

# Phase 3: PA Staff
docker-compose exec web python manage.py test dashboard.tests.test_pa_staff

# Phase 4: Validation
docker-compose exec web python manage.py test dashboard.tests.test_validation

# Phase 5: Species
docker-compose exec web python manage.py test species.tests.test_species

# Phase 6: GIS
docker-compose exec web python manage.py test dashboard.tests.test_gis

# Phase 7: Dashboards
docker-compose exec web python manage.py test dashboard.tests.test_dashboards

# Phase 8: Reports
docker-compose exec web python manage.py test dashboard.tests.test_reports

# Phase 9: Events
docker-compose exec web python manage.py test dashboard.tests.test_events

# Phase 10: Kobo Integration
docker-compose exec web python manage.py test api.tests.test_kobo_integration

# Phase 11: Permissions
docker-compose exec web python manage.py test dashboard.tests.test_permissions

# Phase 12: Integration
docker-compose exec web python manage.py test api.tests.test_integration
```

### Continuous Integration Example

```yaml
# .github/workflows/tests.yml
name: PRISM-Matutum Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_DB: prism_test
          POSTGRES_USER: prism
          POSTGRES_PASSWORD: prism
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests
        run: python manage.py test --verbosity=2
        env:
          DATABASE_URL: postgres://prism:prism@localhost:5432/prism_test
          REDIS_URL: redis://localhost:6379/0
```

---

## Test Data Fixtures

### Create Test Fixtures

```bash
# Export existing data as fixtures
docker-compose exec web python manage.py dumpdata users --indent 2 > fixtures/users.json
docker-compose exec web python manage.py dumpdata species --indent 2 > fixtures/species.json
docker-compose exec web python manage.py dumpdata api --indent 2 > fixtures/api.json
```

### Load Fixtures in Tests

```python
class TestWithFixtures(TestCase):
    fixtures = ['users.json', 'species.json', 'api.json']
    
    def test_with_preloaded_data(self):
        # Data from fixtures is available
        users = User.objects.all()
        self.assertGreater(users.count(), 0)
```

---

## Summary

Use these prompts to generate comprehensive Python test scripts for each phase of the PRISM-Matutum test plan. Each prompt includes:

1. **Context** - Relevant models, views, and services
2. **Test Cases** - Specific test methods to implement
3. **Setup Requirements** - Test data and fixtures needed
4. **Code Examples** - Sample test implementations
5. **Mocking Guidance** - For external services like KoboToolbox

The generated tests should achieve >80% code coverage and verify all features work correctly across all user roles.
