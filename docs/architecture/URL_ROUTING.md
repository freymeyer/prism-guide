# PRISM-Matutum URL Routing Structure

**Document Version:** 1.0  
**Last Updated:** January 6, 2026  
**Author:** Documentation for Capstone Project  

---

## Table of Contents

1. [Overview](#overview)
2. [Health Check Endpoints](#health-check-endpoints)
3. [Dashboard & Home Views](#dashboard--home-views)
4. [PA Staff URLs](#pa-staff-urls)
5. [Validation URLs](#validation-urls)
6. [Species Management URLs](#species-management-urls)
7. [GIS & Spatial URLs](#gis--spatial-urls)
8. [Event & Schedule URLs](#event--schedule-urls)
9. [Report Generation URLs](#report-generation-urls)
10. [Admin Management URLs](#admin-management-urls)
11. [API Endpoints](#api-endpoints)
12. [URL Pattern Reference Table](#url-pattern-reference-table)

---

## 1. Overview

The PRISM-Matutum Dashboard uses a well-organized URL structure with role-based access control. All URLs are prefixed with `/dashboard/` at the project level.

### URL Structure Principles

1. **Feature-Based Grouping**: URLs are organized by feature area
2. **RESTful Patterns**: CRUD operations follow REST conventions where appropriate
3. **Namespacing**: App name is `'dashboard'`, so URLs are referenced as `dashboard:url_name`
4. **Role-Based Access**: Each URL is protected by appropriate permission mixins

### URL Format

```
/dashboard/{feature}/{action}/{pk}/
```

**Examples**:
- `/dashboard/species/fauna/123/` - View fauna species #123
- `/dashboard/queue/` - Validation queue
- `/dashboard/my-submissions/` - PA Staff submissions

---

## 2. Health Check Endpoints

**Purpose**: System health monitoring and load balancer checks.

| URL Pattern | View | Name | Required Role | Purpose |
|-------------|------|------|---------------|---------|
| `/health/` | `health_check_view` | `health_check` | None | Basic health check |
| `/ready/` | `readiness_check_view` | `readiness_check` | None | Readiness probe (K8s) |
| `/alive/` | `liveness_check_view` | `liveness_check` | None | Liveness probe (K8s) |
| `/metrics/` | `metrics_view` | `metrics` | None | Prometheus metrics |

**Access**: Public (no authentication required)

**Response Format**: JSON

```json
{
    "status": "ok",
    "database": "connected",
    "cache": "available",
    "timestamp": "2026-01-06T10:30:00Z"
}
```

---

## 3. Dashboard & Home Views

### Main Dashboard

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/` | `CustomDashboardView` | `dashboard_home` | All (approved) | Custom dashboard (default view) |
| `/denr/` | `DENRDashboardView` | `denr_dashboard` | DENR, Validator | DENR-specific dashboard |
| `/collaborator/` | `CollaboratorDashboardView` | `collaborator_dashboard` | Collaborator | Researcher dashboard |
| `/admin-dashboard/` | `AdminDashboardView` | `admin_dashboard` | System Admin | Admin control panel |
| `/pa-staff-home/` | `PAStaffDashboardHome` | `pa_staff_home` | PA Staff | PA Staff home page |

**Permission Mixins**:
- `CustomDashboardView`: `DashboardAccessMixin`
- `DENRDashboardView`: `DENRAccessMixin`
- `CollaboratorDashboardView`: `CollaboratorAccessMixin`
- `AdminDashboardView`: `StaffUserRequiredMixin`
- `PAStaffDashboardHome`: `PAStaffAccessMixin`

---

### Custom Dashboard Management

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/dashboards/` | `DashboardListView` | `dashboard_list` | All | List user's dashboards |
| `/dashboards/create/` | `DashboardCreateView` | `dashboard_create` | All | Create new dashboard |
| `/dashboards/<pk>/` | `CustomDashboardView` | `custom_dashboard` | All | View custom dashboard |
| `/dashboards/<pk>/edit/` | `DashboardUpdateView` | `dashboard_edit` | Owner | Edit dashboard |
| `/dashboards/<pk>/delete/` | `DashboardDeleteView` | `dashboard_delete` | Owner | Delete dashboard |

---

### Widget Management

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/dashboards/<dashboard_pk>/widgets/add/` | `WidgetCreateView` | `widget_add` | Dashboard Owner | Add widget to dashboard |
| `/widgets/<pk>/edit/` | `WidgetUpdateView` | `widget_edit` | Dashboard Owner | Edit widget |
| `/widgets/<pk>/delete/` | `WidgetDeleteView` | `widget_delete` | Dashboard Owner | Delete widget |

---

### Dashboard Templates

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/templates/` | `TemplateGalleryView` | `template_gallery` | All | Browse dashboard templates |
| `/templates/<template_id>/instantiate/` | `api_template_instantiate` | `template_instantiate` | All | Create dashboard from template |

---

## 4. PA Staff URLs

**Purpose**: PA Staff-specific features for viewing their own submissions and observations.

**Access**: `PAStaffAccessMixin` (PA Staff role + field verified)

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/my-submissions/` | `MySubmissionsView` | `my_submissions` | List all submissions by current PA Staff |
| `/my-submissions/<pk>/` | `MySubmissionDetailView` | `my_submission_detail` | View detailed submission |
| `/my-observations/` | `MyObservationsView` | `my_observations` | View processed observations |

**Features**:
- **Filtering**: By date, status, location
- **Search**: Full-text search on submission data
- **Statistics**: Pending, approved, rejected counts
- **Pagination**: 20 items per page

**Template Files**:
- `dashboard/my_submissions.html`
- `dashboard/my_submission_detail.html`
- `dashboard/my_observations.html`

---

## 5. Validation URLs

**Purpose**: Data validation workflow for Validators.

**Access**: `ValidatorAccessMixin` (Validator role + approved account)

### Main Validation Views

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/queue/` | `ValidationQueueView` | `validation_queue` | List pending submissions for validation |
| `/workspace/<pk>/` | `ValidationWorkspaceView` | `validation_workspace` | Validate a specific submission |
| `/history/` | `ValidationHistoryView` | `validation_history` | View validation history |
| `/history/export/` | `ValidationHistoryExportView` | `validation_history_export` | Export validation history (Excel) |
| `/audit/<pk>/` | `SubmissionAuditTrailView` | `submission_audit_trail` | View complete audit trail |

**Queue Features**:
- Filter by status, PA Staff, date range
- Sort by submission date, priority
- Bulk assignment to validators
- Statistics dashboard

**Workspace Features**:
- View submission data and media
- Approve, reject, or hold submission
- Add validation remarks
- View related observations

---

### Performance & Quality Control

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/performance/` | `PerformanceMetricsView` | `performance_metrics` | Validator performance metrics |

---

## 6. Species Management URLs

**Purpose**: Manage fauna and flora species checklists.

**Access**: 
- **View**: All authenticated users
- **Add/Edit**: `ValidatorOrStaffRequiredMixin`
- **Delete**: `AdminAccessMixin`

### Main Species Views

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/species/` | `SpeciesListView` | `species_list` | All | List all species (fauna/flora) |
| `/species/<type>/<pk>/` | `SpeciesDetailView` | `species_detail` | All | View species details |
| `/species/<type>/add/` | `SpeciesCreateView` | `species_add` | Validator, Admin | Add new species |
| `/species/<type>/<pk>/edit/` | `SpeciesUpdateView` | `species_edit` | Validator, Admin | Edit species |
| `/species/<type>/<pk>/delete/` | `SpeciesDeleteView` | `species_delete` | Admin only | Delete species |
| `/species/<type>/<pk>/observations/` | `SpeciesObservationsView` | `species_observations` | All | View species observations |
| `/species/<type>/<pk>/media/` | `SpeciesMediaView` | `species_media` | All | View/add species media |
| `/species/<type>/<pk>/media/<media_pk>/delete/` | `SpeciesMediaDeleteView` | `species_media_delete` | Validator, Admin | Delete media |

**Species Types**: `fauna` or `flora`

---

### Species Articles

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/species/fauna/<pk>/add-article/` | `FaunaArticleCreateView` | `fauna_article_add` | Validator, Admin | Add article to fauna species |
| `/article/fauna/<pk>/edit/` | `FaunaArticleUpdateView` | `fauna_article_edit` | Validator, Admin | Edit fauna article |
| `/article/fauna/<pk>/delete/` | `FaunaArticleDeleteView` | `fauna_article_delete` | Validator, Admin | Delete fauna article |
| `/species/flora/<pk>/add-article/` | `FloraArticleCreateView` | `flora_article_add` | Validator, Admin | Add article to flora species |
| `/article/flora/<pk>/edit/` | `FloraArticleUpdateView` | `flora_article_edit` | Validator, Admin | Edit flora article |
| `/article/flora/<pk>/delete/` | `FloraArticleDeleteView` | `flora_article_delete` | Validator, Admin | Delete flora article |

---

### Taxonomy Management

**Purpose**: Manage taxonomic hierarchy (Kingdom → Phylum → Class → Order → Family → Genus).

**Access**: `AdminAccessMixin`

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/taxonomy/` | `TaxonomyManagementView` | `taxonomy_management` | Taxonomy overview |
| `/taxonomy/<rank>/add/` | `TaxonomyCreateView` | `taxonomy_add` | Add taxon at rank |
| `/taxonomy/<rank>/<pk>/edit/` | `TaxonomyUpdateView` | `taxonomy_edit` | Edit taxon |
| `/taxonomy/<rank>/<pk>/delete/` | `TaxonomyDeleteView` | `taxonomy_delete` | Delete taxon |

**Ranks**: `kingdom`, `phylum`, `class`, `order`, `family`, `genus`

---

## 7. GIS & Spatial URLs

**Purpose**: Geographic information system features for mapping and spatial analysis.

**Access**: 
- **View**: All authenticated users
- **Create/Edit**: `ValidatorAccessMixin`

### Monitoring Locations

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/monitoring-locations/` | `MonitoringLocationListView` | `monitoring_location_list` | All | List monitoring locations |
| `/monitoring-locations/create/` | `MonitoringLocationCreateView` | `monitoring_location_create` | Validator, Admin | Add location |
| `/monitoring-locations/<pk>/` | `MonitoringLocationDetailView` | `monitoring_location_detail` | All | View location details |

---

### Transect Routes & Segments

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/transect-routes/` | `TransectRouteListView` | `transect_route_list` | All | List transect routes |
| `/transect-routes/create/` | `TransectRouteCreateView` | `transect_route_create` | Validator, Admin | Create route |
| `/transect-routes/<pk>/` | `TransectRouteDetailView` | `transect_route_detail` | All | View route |
| `/transect-segments/` | `TransectSegmentListView` | `transect_segment_list` | All | List segments |
| `/transect-segments/<pk>/` | `TransectSegmentDetailView` | `transect_segment_detail` | All | View segment |

---

### Interactive Map

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/map/` | `InteractiveMapView` | `interactive_map` | All | Interactive Leaflet map |

**Map Features**:
- Observation points
- Protected area boundaries
- Transect routes
- Monitoring locations
- Heat maps
- Filtering by species, date, location

---

### Observation Management (Universal Views)

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/observations/` | `UniversalWildlifeListView` | `wildlife_observations` | All | List wildlife observations |
| `/observations/<pk>/` | `UniversalWildlifeDetailView` | `observation_detail` | All | View observation |
| `/observations/<pk>/edit/` | `ObservationUpdateView` | `observation_edit` | Validator, Admin | Edit observation |
| `/observations/export/` | `ObservationExportView` | `observation_export` | All | Export observations (Excel) |

---

### Resource Use Incidents

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/resource-use/` | `UniversalResourceUseListView` | `resource_use_incidents` | All | List incidents |
| `/resource-use/<pk>/` | `UniversalResourceUseDetailView` | `resource_use_incident_detail` | All | View incident |

---

### Disturbance Records

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/disturbances/` | `UniversalDisturbanceListView` | `disturbance_records` | All | List disturbances |
| `/disturbances/<pk>/` | `UniversalDisturbanceDetailView` | `disturbance_record_detail` | All | View disturbance |

---

### Landscape Monitoring

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/landscape/` | `UniversalLandscapeListView` | `landscape_monitoring` | All | List landscape data |
| `/landscape/<pk>/` | `UniversalLandscapeDetailView` | `landscape_monitoring_detail` | All | View landscape record |

---

## 8. Event & Schedule URLs

### Survey Schedule Management

**Purpose**: Plan and manage scheduled surveys.

**Access**: `ValidatorAccessMixin`

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/schedules/` | `ScheduledSurveyCalendarView` | `schedule_calendar` | Validator, Admin | Calendar view |
| `/schedules/create/` | `ScheduledSurveyCreateView` | `schedule_create` | Validator, Admin | Create schedule |
| `/schedules/<pk>/` | `ScheduledSurveyDetailView` | `schedule_detail` | All | View schedule |
| `/schedules/<pk>/edit/` | `ScheduledSurveyUpdateView` | `schedule_edit` | Validator, Admin | Edit schedule |
| `/schedules/<pk>/delete/` | `ScheduledSurveyDeleteView` | `schedule_delete` | Validator, Admin | Delete schedule |

---

### Event Management

**Purpose**: Manage meetings, workshops, and announcements.

**Access**: `ValidatorAccessMixin` (view: all)

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/events/` | `EventListView` | `event_list` | All | List events |
| `/events/create/` | `EventCreateView` | `event_create` | Validator, Admin | Create event |
| `/events/<pk>/` | `EventDetailView` | `event_detail` | All | View event |
| `/events/<pk>/edit/` | `EventUpdateView` | `event_edit` | Validator, Admin | Edit event |
| `/events/<pk>/delete/` | `EventDeleteView` | `event_delete` | Validator, Admin | Delete event |

**Event Types**: Meeting, Workshop, Training, Announcement

---

### Focus Group Discussion (FGD)

**Purpose**: Document FGD sessions with communities.

**Access**: `FGDAccessMixin` (Validator, DENR, Admin)

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/fgd/` | `FGDSessionListView` | `fgd_session_list` | Validator, DENR | List FGD sessions |
| `/fgd/create/` | `FGDSessionCreateView` | `fgd_session_create` | Validator, Admin | Create session |
| `/fgd/<pk>/` | `FGDSessionDetailView` | `fgd_session_detail` | Validator, DENR | View session |
| `/fgd/<pk>/edit/` | `FGDSessionUpdateView` | `fgd_session_edit` | Validator, Admin | Edit session |
| `/fgd/<pk>/delete/` | `FGDSessionDeleteView` | `fgd_session_delete` | Admin | Delete session |
| `/fgd/<session_pk>/findings/add/` | `FGDFindingCreateView` | `fgd_finding_create` | Validator, Admin | Add finding |
| `/fgd/findings/<pk>/edit/` | `FGDFindingUpdateView` | `fgd_finding_edit` | Validator, Admin | Edit finding |
| `/fgd/findings/<pk>/delete/` | `FGDFindingDeleteView` | `fgd_finding_delete` | Admin | Delete finding |
| `/fgd/<session_pk>/participants/add/` | `FGDParticipantCreateView` | `fgd_participant_create` | Validator, Admin | Add participant |

---

## 9. Report Generation URLs

**Purpose**: Generate and manage biodiversity reports.

**Access**:
- **View Reports**: `CanViewReportsMixin` (PA Staff, Collaborator, DENR, Validator)
- **Build Reports**: `CanBuildReportsMixin` (DENR, Validator only)

### Report Builder & Archives

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/reports/builder/` | `report_builder` | `report_builder` | DENR, Validator | Report builder interface |
| `/reports/archive/` | `report_archive_list` | `report_archive_list` | All (approved) | List generated reports |
| `/reports/<report_id>/preview/` | `report_preview` | `report_preview` | All | Preview report |
| `/reports/<report_id>/download/<format>/` | `download_report` | `download_report` | All | Download (PDF, XLSX, etc.) |
| `/reports/<report_id>/delete/` | `delete_report` | `delete_report` | DENR, Validator | Delete report |
| `/reports/<report_id>/regenerate/` | `regenerate_report` | `regenerate_report` | DENR, Validator | Regenerate report |
| `/reports/<report_id>/status/` | `report_status_api` | `report_status_api` | All | Check generation status (API) |

**Report Types**:
- GIS-Enhanced Report
- Semestral BMS Report
- Standard BMS Report
- Digital Backup Export
- Custom Report

---

### Data Export Center

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/reports/export/` | `data_export_center` | `data_export_center` | DENR, Validator | Export data center |
| `/reports/export/shapefile/` | `download_shapefile` | `download_shapefile` | DENR, Validator | Download shapefile |

---

### Report Templates

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/reports/templates/` | `report_template_list` | `report_template_list` | DENR, Validator | List templates |
| `/reports/templates/create/` | `create_report_template` | `create_report_template` | Validator, Admin | Create template |

---

## 10. Admin Management URLs

**Purpose**: System administration and configuration.

**Access**: `StaffUserRequiredMixin` or `AdminAccessMixin` (System Admin only)

### User Management

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/organization-hub/` | `UserOrganizationHubView` | `user_organization_hub` | Unified user/org management |
| `/admin/users/` | `UserManagementView` | `user_management` | User list with tabs |
| `/admin/users/<pk>/approve/` | `UserApproveView` | `user_approve` | Approve user account |
| `/admin/users/<pk>/reject/` | `UserRejectView` | `user_reject` | Reject user account |
| `/admin/users/<pk>/edit/` | `UserUpdateView` | `user_update` | Edit user details |
| `/admin/users/<pk>/delete/` | `UserDeleteView` | `user_delete` | Delete user |

**User Tabs**:
- Pending Approval
- Active Users
- Staff Users
- All Users

---

### Agency Management

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/agencies/` | `AgencyManagementView` | `agency_management` | List agencies |
| `/admin/agencies/create/` | `AgencyCreateView` | `agency_create` | Create agency |
| `/admin/agencies/<pk>/edit/` | `AgencyUpdateView` | `agency_edit` | Edit agency |
| `/admin/agencies/<pk>/delete/` | `AgencyDeleteView` | `agency_delete` | Delete agency |

**Agency Types**: Government, LGU, NGO, Academic, Private

---

### Department Management

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/departments/` | `DepartmentManagementView` | `department_management` | List departments |
| `/admin/departments/create/` | `DepartmentCreateView` | `department_create` | Create department |
| `/admin/departments/<pk>/edit/` | `DepartmentUpdateView` | `department_edit` | Edit department |
| `/admin/departments/<pk>/delete/` | `DepartmentDeleteView` | `department_delete` | Delete department |

---

### Address Management

**Purpose**: Manage geographic administrative divisions.

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/addresses/` | `AddressManagementView` | `address_management` | Address hub |
| `/addresses/region/add/` | `RegionCreateView` | `region_create` | Add region |
| `/addresses/region/<pk>/edit/` | `RegionUpdateView` | `region_update` | Edit region |
| `/addresses/region/<pk>/delete/` | `RegionDeleteView` | `region_delete` | Delete region |
| `/addresses/province/add/` | `ProvinceCreateView` | `province_create` | Add province |
| `/addresses/province/<pk>/edit/` | `ProvinceUpdateView` | `province_update` | Edit province |
| `/addresses/province/<pk>/delete/` | `ProvinceDeleteView` | `province_delete` | Delete province |
| `/addresses/municipality/add/` | `MunicipalityCreateView` | `municipality_create` | Add municipality |
| `/addresses/municipality/<pk>/edit/` | `MunicipalityUpdateView` | `municipality_update` | Edit municipality |
| `/addresses/municipality/<pk>/delete/` | `MunicipalityDeleteView` | `municipality_delete` | Delete municipality |
| `/addresses/barangay/add/` | `BarangayCreateView` | `barangay_create` | Add barangay |
| `/addresses/barangay/<pk>/edit/` | `BarangayUpdateView` | `barangay_update` | Edit barangay |
| `/addresses/barangay/<pk>/delete/` | `BarangayDeleteView` | `barangay_delete` | Delete barangay |

**Hierarchy**: Region → Province → Municipality → Barangay

---

### Choice List Management

**Purpose**: Manage dynamic choice lists for KoboToolbox forms.

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/choice-management/` | `ChoiceManagementView` | `choice_management` | Choice management hub |
| `/admin/choices/` | `ChoiceListListView` | `choice_list_list` | List choice lists |
| `/admin/choices/create/` | `ChoiceListCreateView` | `choice_list_create` | Create choice list |
| `/admin/choices/<pk>/` | `ChoiceListDetailView` | `choice_list_detail` | View choice list |
| `/admin/choices/<pk>/edit/` | `ChoiceListUpdateView` | `choice_list_update` | Edit choice list |
| `/admin/choices/<pk>/delete/` | `ChoiceListDeleteView` | `choice_list_delete` | Delete choice list |
| `/admin/choices/<pk>/sync/` | `ChoiceListSyncView` | `choice_list_sync` | Sync with KoboToolbox |

---

### Individual Choice Management

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/choices/<choice_list_pk>/add-choice/` | `ChoiceCreateView` | `choice_create` | Add choice to list |
| `/admin/choices/<choice_list_pk>/bulk-add/` | `ChoiceBulkAddView` | `choice_bulk_add` | Bulk add choices |
| `/admin/choice/<pk>/edit/` | `ChoiceUpdateView` | `choice_update` | Edit choice |
| `/admin/choice/<pk>/delete/` | `ChoiceDeleteView` | `choice_delete` | Delete choice |
| `/admin/choice/<pk>/toggle/` | `ChoiceToggleActiveView` | `choice_toggle_active` | Activate/deactivate choice |

---

### Form Choice Mapping

**Purpose**: Map KoboToolbox form questions to Django choice lists.

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/mappings/` | `FormChoiceMappingListView` | `form_mapping_list` | List mappings |
| `/admin/mappings/create/` | `FormChoiceMappingCreateView` | `form_mapping_create` | Create mapping |
| `/admin/mappings/<pk>/edit/` | `FormChoiceMappingUpdateView` | `form_mapping_update` | Edit mapping |
| `/admin/mappings/<pk>/delete/` | `FormChoiceMappingDeleteView` | `form_mapping_delete` | Delete mapping |
| `/admin/mappings/<pk>/sync/` | `FormChoiceMappingSyncView` | `form_mapping_sync` | Sync mapping |

---

### KoboToolbox Integration

| URL | View | Name | Description |
|-----|------|------|-------------|
| `/admin/kobo/browse/` | `KoboFormBrowseView` | `kobo_form_browse` | Browse Kobo forms |

---

## 11. API Endpoints

**Purpose**: AJAX and API endpoints for dynamic functionality.

**Access**: Varies by endpoint (usually requires authentication)

### Validation API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/validate/` | `api_validate` | `api_validate` | Validator | Validate submission (AJAX) |
| `/api/pending-counts/` | `api_pending_counts` | `api_pending_counts` | Validator | Get pending submission counts |

---

### Species API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/species-info/<type>/<id>/` | `api_species_info` | `api_species_info` | All | Get species information (JSON) |
| `/api/species-search/<type>/` | `api_species_search` | `api_species_search` | All | Search species (autocomplete) |
| `/api/update-expertise/` | `api_update_expertise` | `update_expertise` | Validator | Update validator expertise |

---

### Notification API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/notifications/<id>/mark-read/` | `api_mark_notification_read` | `api_mark_notification_read` | All | Mark notification as read |
| `/api/notifications/mark-all-read/` | `api_mark_all_notifications_read` | `api_mark_all_notifications_read` | All | Mark all as read |
| `/api/notifications/<id>/delete/` | `api_delete_notification` | `api_delete_notification` | All | Delete notification |
| `/api/notifications/bulk-mark-read/` | `api_bulk_mark_notifications_read` | `api_bulk_mark_notifications_read` | All | Bulk mark as read |
| `/api/notifications/bulk-delete/` | `api_bulk_delete_notifications` | `api_bulk_delete_notifications` | All | Bulk delete |

---

### Data Sync API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/sync-submissions/` | `api_sync_submissions` | `api_sync_submissions` | Admin | Sync single form from Kobo |
| `/api/sync-all-submissions/` | `api_sync_all_submissions` | `api_sync_all_submissions` | Admin | Sync all forms from Kobo |

---

### GeoJSON API

**Purpose**: Provide spatial data in GeoJSON format for map display.

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/geo/protected-areas/` | `api_protected_areas_geojson` | `api_protected_areas_geojson` | All | Protected area boundaries |
| `/api/geo/transect-routes/` | `api_transect_routes_geojson` | `api_transect_routes_geojson` | All | Transect routes |
| `/api/geo/monitoring-locations/` | `api_monitoring_locations_geojson` | `api_monitoring_locations_geojson` | All | Monitoring locations |
| `/api/geo/transect-segments/` | `api_transect_segments_geojson` | `api_transect_segments_geojson` | All | Transect segments |
| `/api/geo/observations/` | `api_observations_geojson` | `api_observations_geojson` | All | Observation points |
| `/api/geo/statistics/` | `api_map_statistics` | `api_map_statistics` | All | Map statistics |
| `/api/geo/export/` | `api_export_map_data` | `api_export_map_data` | DENR, Validator | Export map data |

---

### Dashboard Widget API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/widgets/<id>/move/` | `api_widget_move` | `api_widget_move` | Dashboard Owner | Move widget position |
| `/api/widgets/<id>/resize/` | `api_widget_resize` | `api_widget_resize` | Dashboard Owner | Resize widget |
| `/api/widgets/<id>/toggle/` | `api_widget_toggle_visibility` | `api_widget_toggle` | Dashboard Owner | Show/hide widget |
| `/api/widgets/<id>/refresh/` | `api_widget_refresh` | `api_widget_refresh` | Dashboard Owner | Refresh widget data |
| `/api/widgets/<id>/configure/` | `api_widget_configure` | `api_widget_configure` | Dashboard Owner | Configure widget |
| `/api/widgets/<id>/configure-form/` | `api_widget_configure_form` | `api_widget_configure_form` | Dashboard Owner | Get config form HTML |

---

### Dashboard Management API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/dashboards/<pk>/duplicate/` | `api_dashboard_duplicate` | `api_dashboard_duplicate` | All | Duplicate dashboard |
| `/api/dashboards/<pk>/export/` | `api_dashboard_export` | `api_dashboard_export` | All | Export dashboard config |
| `/api/dashboards/<pk>/set-default/` | `api_set_default_dashboard` | `api_set_default_dashboard` | All | Set as default |
| `/api/dashboards/<pk>/save-layout/` | `api_save_dashboard_layout` | `api_save_dashboard_layout` | Dashboard Owner | Save widget layout |

---

### FGD API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/api/fgd/stats/` | `api_fgd_stats` | `api_fgd_stats` | Validator, DENR | Get FGD statistics |

---

### Profile API

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/profile/` | `ProfileView` | `profile` | All | View profile |
| `/profile/edit/` | `ProfileUpdateView` | `profile_edit` | All | Edit profile |
| `/profile/activity/` | `ActivityHistoryView` | `activity_history` | All | View activity log |
| `/profile/activity/export/` | `ActivityExportView` | `activity_export` | All | Export activity (CSV) |
| `/notifications/` | `NotificationsView` | `notifications` | All | View notifications |
| `/notifications/settings/` | `NotificationSettingsView` | `notification_settings` | All | Configure notifications |

---

### Authentication

| URL | View | Name | Required Role | Description |
|-----|------|------|---------------|-------------|
| `/logout/` | `logout_view` | `logout` | All | Logout user |

---

## 12. URL Pattern Reference Table

### Complete URL Mapping

| Feature Area | URL Prefix | View Count | Primary Role Access |
|--------------|-----------|------------|-------------------|
| **Health Checks** | `/health/`, `/ready/`, `/alive/`, `/metrics/` | 4 | Public |
| **Dashboards** | `/`, `/dashboards/`, `/templates/` | 12 | All (approved) |
| **PA Staff** | `/my-submissions/`, `/my-observations/`, `/pa-staff-home/` | 3 | PA Staff |
| **Validation** | `/queue/`, `/workspace/`, `/history/`, `/audit/` | 5 | Validator |
| **Species** | `/species/`, `/article/`, `/taxonomy/` | 15 | All (view), Validator (edit) |
| **GIS** | `/monitoring-locations/`, `/transect-routes/`, `/map/`, `/observations/` | 15 | All (view), Validator (edit) |
| **Events** | `/events/`, `/schedules/`, `/fgd/` | 19 | Validator, DENR |
| **Reports** | `/reports/builder/`, `/reports/archive/`, `/reports/export/` | 11 | DENR, Validator |
| **Admin** | `/admin/users/`, `/admin/agencies/`, `/admin/choices/`, `/addresses/` | 45+ | System Admin |
| **API** | `/api/*` | 35+ | Varies |

**Total URLs**: ~150+

---

## Summary

The PRISM-Matutum URL structure is organized into logical feature areas with consistent naming conventions and role-based access control. Each URL is protected by appropriate permission mixins or decorators to ensure data security and proper workflow enforcement.

### Key Characteristics

1. **Feature-Based Organization**: URLs grouped by functionality
2. **RESTful Patterns**: CRUD operations follow REST conventions
3. **Role-Based Access**: Every URL has defined access requirements
4. **API Endpoints**: Comprehensive AJAX/API support for dynamic features
5. **Namespacing**: All URLs use `dashboard:` namespace

### URL Naming Conventions

- **List Views**: `{feature}_list` (e.g., `species_list`)
- **Detail Views**: `{feature}_detail` (e.g., `species_detail`)
- **Create Views**: `{feature}_create` (e.g., `species_add`)
- **Update Views**: `{feature}_edit` or `{feature}_update`
- **Delete Views**: `{feature}_delete`
- **API Endpoints**: `api_{action}` (e.g., `api_validate`)

---

**Related Documentation**:
- [RBAC_DOCUMENTATION.md](RBAC_DOCUMENTATION.md) - User roles and permissions
- [PERMISSION_SYSTEM.md](PERMISSION_SYSTEM.md) - Permission mixins and decorators
- [SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md) - Overall architecture

---

*Document created as part of PRISM-Matutum Capstone Project Documentation*  
*For questions or updates, refer to the project repository.*
