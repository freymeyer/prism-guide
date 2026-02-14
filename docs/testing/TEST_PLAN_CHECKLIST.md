# PRISM-Matutum Comprehensive Test Plan

**Document Version:** 1.0  
**Last Updated:** January 7, 2026  
**Purpose:** Feature Use Case Testing Checklist for System Verification  

---

## Table of Contents

1. [Phase 1: Authentication & Authorization](#phase-1-authentication--authorization)
2. [Phase 2: User & Organization Management](#phase-2-user--organization-management)
3. [Phase 3: PA Staff Workflows](#phase-3-pa-staff-workflows)
4. [Phase 4: Validation System](#phase-4-validation-system)
5. [Phase 5: Species Management](#phase-5-species-management)
6. [Phase 6: GIS & Spatial Features](#phase-6-gis--spatial-features)
7. [Phase 7: Dashboard & Widget System](#phase-7-dashboard--widget-system)
8. [Phase 8: Report Generation System](#phase-8-report-generation-system)
9. [Phase 9: Event & Schedule Management](#phase-9-event--schedule-management)
10. [Phase 10: KoboToolbox Integration](#phase-10-kobotoolbox-integration)
11. [Phase 11: Cross-Role Access Control](#phase-11-cross-role-access-control)
12. [Phase 12: System Integration & Performance](#phase-12-system-integration--performance)

---

## Phase 1: Authentication & Authorization

### 1.1 User Registration
- [ ] New user can access registration page
- [ ] Registration form validates required fields (username, email, password)
- [ ] Password validation enforces security requirements
- [ ] User can select role during registration (PA Staff, Validator, DENR, Collaborator)
- [ ] User can select agency and department during registration
- [ ] Successful registration creates user with `account_status='pending'`
- [ ] Email verification is sent upon registration
- [ ] Duplicate username/email is rejected

### 1.2 User Login
- [ ] Login page is accessible
- [ ] Valid credentials allow login
- [ ] Invalid credentials show appropriate error message
- [ ] Pending users cannot login (shown appropriate message)
- [ ] Rejected users cannot login
- [ ] Suspended users cannot login
- [ ] Login redirects to role-appropriate dashboard
- [ ] "Remember me" functionality works
- [ ] Session timeout works correctly

### 1.3 Password Management
- [ ] Password reset link can be requested
- [ ] Password reset email is sent
- [ ] Password reset token expires after configured time
- [ ] New password can be set via reset link
- [ ] User can change password from profile settings
- [ ] Old password required for password change

### 1.4 Logout
- [ ] Logout button is accessible from all pages
- [ ] Logout clears session properly
- [ ] Logged out user cannot access protected pages
- [ ] Logout redirects to login page

---

## Phase 2: User & Organization Management

### 2.1 User Approval Workflow (System Admin)
- [ ] Pending users appear in admin approval queue
- [ ] Admin can view pending user details
- [ ] Admin can approve user registration
- [ ] Approved user receives email notification
- [ ] Approved user can now login
- [ ] Admin can reject user registration
- [ ] Rejection requires reason
- [ ] Rejected user receives email with reason
- [ ] Admin can suspend active users
- [ ] Admin can reactivate suspended users

### 2.2 User Management (System Admin)
- [ ] Admin can view list of all users
- [ ] User list supports pagination
- [ ] User list supports filtering by role
- [ ] User list supports filtering by status
- [ ] User list supports search by username/email
- [ ] Admin can edit user details
- [ ] Admin can change user role
- [ ] Admin can assign user to agency/department
- [ ] Admin can mark PA Staff as field-verified
- [ ] Admin can view user activity log
- [ ] Admin can reset user password

### 2.3 Agency Management
- [ ] Admin can view agency list
- [ ] Admin can create new agency
- [ ] Admin can set agency type (Government, LGU, NGO, Academic, Private)
- [ ] Admin can set parent agency (hierarchical structure)
- [ ] Admin can edit agency details
- [ ] Admin can deactivate agency
- [ ] Admin can reactivate agency
- [ ] Agency hierarchy path displays correctly
- [ ] Cannot delete agency with linked users/departments

### 2.4 Department Management
- [ ] Admin can view department list
- [ ] Admin can create new department
- [ ] Department must be linked to an agency
- [ ] Admin can set parent department (sub-departments)
- [ ] Admin can assign department head
- [ ] Admin can edit department details
- [ ] Admin can deactivate department
- [ ] Department hierarchy path displays correctly

### 2.5 Geographic Address Management
- [ ] Admin can manage Regions (create, edit, delete)
- [ ] Admin can manage Provinces (linked to Region)
- [ ] Admin can manage Municipalities (linked to Province)
- [ ] Admin can manage Barangays (linked to Municipality)
- [ ] Geographic hierarchy displays correctly
- [ ] Address cascade delete protection works
- [ ] Address search/filter works

---

## Phase 3: PA Staff Workflows

### 3.1 PA Staff Dashboard
- [ ] PA Staff can access their dedicated dashboard
- [ ] Dashboard displays total submissions count
- [ ] Dashboard displays pending review count
- [ ] Dashboard displays approved submissions count
- [ ] Dashboard displays this week submissions count
- [ ] Dashboard displays total observations count
- [ ] Approval rate calculates correctly
- [ ] Field verification alert shows for non-verified users
- [ ] Recent submissions table displays correctly
- [ ] Quick action links work

### 3.2 My Submissions View
- [ ] PA Staff can access my submissions page
- [ ] All personal submissions are listed
- [ ] Submissions can be filtered by status (Pending, Approved, Rejected, On Hold)
- [ ] Submissions can be filtered by date range
- [ ] Submissions can be searched by ID
- [ ] Submissions can be searched by validation remarks
- [ ] Pagination works correctly (25 per page)
- [ ] Status badges display correct colors
- [ ] Submission details can be viewed
- [ ] Recent validation activity panel shows correctly

### 3.3 Submission Detail View
- [ ] PA Staff can view their own submission details
- [ ] All submission fields display correctly
- [ ] Validation status is clearly shown
- [ ] Validation remarks from validator are displayed
- [ ] Validator name is shown (if validated)
- [ ] Validation timestamp is shown
- [ ] Related observations link correctly
- [ ] Audit trail is visible (if applicable)
- [ ] PA Staff cannot edit submission data (read-only)

### 3.4 My Observations View
- [ ] PA Staff can view their approved observations
- [ ] Wildlife observations are listed
- [ ] Resource use incidents are listed
- [ ] Disturbance records are listed
- [ ] Landscape monitoring records are listed
- [ ] Observations can be filtered by type
- [ ] Observations can be filtered by date
- [ ] Observation details link works

---

## Phase 4: Validation System

### 4.1 Validation Queue (Validator)
- [ ] Validator can access validation queue
- [ ] Queue statistics display correctly (Total, Not Validated, On Hold, Flagged)
- [ ] High priority count shows correctly (>7 days or flagged)
- [ ] Assigned to me count is accurate
- [ ] Submissions can be filtered by status
- [ ] Submissions can be filtered by date range
- [ ] Submissions can be filtered by PA Staff (submitter)
- [ ] Submissions can be filtered by priority
- [ ] Search by submission ID works
- [ ] Search by remarks works
- [ ] Sorting options work (Newest, Oldest, PA Staff A-Z, Status)
- [ ] Pagination works (50 per page)
- [ ] Checkbox selection works
- [ ] Priority indicator (🔥) shows for urgent items

### 4.2 Bulk Assignment Actions
- [ ] Validator can select multiple submissions
- [ ] "Assign to Me" bulk action works
- [ ] Assigned submissions show validator name
- [ ] "Assign to Other Validator" dropdown works
- [ ] Bulk assignment creates activity log
- [ ] Assigned validator receives notification
- [ ] Email notification sent (if enabled)

### 4.3 Validation Workspace
- [ ] Validator can open validation workspace for submission
- [ ] All submission data displays correctly
- [ ] Attached media (photos) display correctly
- [ ] Species information is linked and accessible
- [ ] Location data shows on mini-map
- [ ] GPS coordinates are validated
- [ ] Validation action buttons are visible (Approve, Reject, Hold, Flag)
- [ ] Validation remarks text field is available
- [ ] Previous validation history is shown (if any)
- [ ] Navigation between submissions works

### 4.4 Validation Actions
#### Approve Action
- [ ] Validator can approve submission
- [ ] Remarks can be added with approval
- [ ] Submission status changes to `validation_status_approved`
- [ ] Data is processed into appropriate record (WildlifeObservation, etc.)
- [ ] Processed record is created correctly
- [ ] PA Staff is notified of approval
- [ ] Approved submissions removed from queue
- [ ] Validation timestamp is recorded
- [ ] Validated_by field is set to validator

#### Reject Action
- [ ] Validator can reject submission
- [ ] Rejection requires remarks (reason)
- [ ] Submission status changes to `validation_status_not_approved`
- [ ] Data is NOT processed into records
- [ ] PA Staff is notified of rejection with reason
- [ ] Rejected submissions removed from queue
- [ ] Rejected submissions cannot be re-validated

#### Hold Action
- [ ] Validator can place submission on hold
- [ ] Hold requires remarks (clarification needed)
- [ ] Submission status changes to `validation_status_on_hold`
- [ ] PA Staff is notified with clarification request
- [ ] On-hold submissions remain in queue
- [ ] On-hold submissions can be re-reviewed later

#### Flag Action
- [ ] Validator can flag submission for supervisor review
- [ ] Flagging requires remarks
- [ ] Submission status changes to `flagged`
- [ ] Flagged submissions appear as high priority
- [ ] Senior validator/admin is notified
- [ ] Flagged submissions can be resolved by supervisor

### 4.5 Validation History
- [ ] Validator can access their validation history
- [ ] History shows all validated submissions
- [ ] History can be filtered by decision type (Approved, Rejected, Hold)
- [ ] History can be filtered by date range
- [ ] History can be exported
- [ ] Performance metrics are calculated (approval rate, avg time)
- [ ] Audit trail is complete

---

## Phase 5: Species Management

### 5.1 Species List View
- [ ] Species list is accessible
- [ ] Fauna checklist displays correctly
- [ ] Flora checklist displays correctly
- [ ] Species can be filtered by type (Bird, Mammal, Reptile, etc.)
- [ ] Species can be filtered by conservation status
- [ ] Species can be filtered by endemicity
- [ ] Species can be searched by scientific name
- [ ] Species can be searched by common name
- [ ] Pagination works correctly
- [ ] Species count statistics display

### 5.2 Species Detail View
- [ ] Species detail page is accessible
- [ ] Scientific name displays correctly
- [ ] Complete taxonomy hierarchy displays (Kingdom → Genus)
- [ ] Conservation status (IUCN) displays
- [ ] Endemicity status displays
- [ ] Common names list displays
- [ ] Physical characteristics display (fauna/flora specific)
- [ ] Species media gallery displays
- [ ] Related research/articles section displays
- [ ] Observation history for species displays
- [ ] Observation count is accurate
- [ ] Link to species observations works

### 5.3 Species CRUD Operations (Admin Only)
- [ ] Admin can create new fauna species
- [ ] Admin can create new flora species
- [ ] Taxonomy selection works (cascading dropdowns)
- [ ] Auto-classification assigns correct species type
- [ ] Conservation status can be set
- [ ] Endemicity can be set
- [ ] Characteristics can be added
- [ ] Admin can edit species information
- [ ] Admin can delete species (with confirmation)
- [ ] Cannot delete species with linked observations

### 5.4 Species Common Names
- [ ] Multiple common names can be added per species
- [ ] Common names can be edited
- [ ] Common names can be deleted
- [ ] Search includes common names

### 5.5 Species Articles & Research
- [ ] Admin can add articles to species
- [ ] Article details can be edited
- [ ] Articles display on species detail page
- [ ] External links work correctly
- [ ] Related research can be linked

### 5.6 Species Media
- [ ] Media (photos) can be uploaded for species
- [ ] Multiple images are supported
- [ ] Image gallery displays correctly
- [ ] Images can be deleted
- [ ] Thumbnail generation works

---

## Phase 6: GIS & Spatial Features

### 6.1 Protected Area Management
- [ ] Protected area list displays correctly
- [ ] Protected area count and statistics show
- [ ] Protected area detail view works
- [ ] Boundary polygon displays on map (if set)
- [ ] Associated landscapes list displays
- [ ] Transect routes for PA display
- [ ] Monitoring locations display
- [ ] Recent observations (last 10) show
- [ ] Protected area statistics are accurate
- [ ] Admin can edit protected area name

### 6.2 Transect Route Management
- [ ] Transect route list displays
- [ ] New transect route can be created
- [ ] Route can be assigned to protected area
- [ ] Stations can be added to route
- [ ] Route geometry generates from stations
- [ ] Route displays on map
- [ ] Route can be edited
- [ ] Route can be deactivated
- [ ] Observations linked to route display
- [ ] Transect segments auto-generate between stations

### 6.3 Monitoring Location Management
- [ ] Monitoring location list displays
- [ ] New location can be created
- [ ] Location type can be set (Facility, Transect Station, Other)
- [ ] GPS coordinates can be entered
- [ ] Location displays on map
- [ ] Location can be edited
- [ ] Location can be deactivated
- [ ] Linked transect routes display

### 6.4 Interactive Map Features
- [ ] Interactive map page loads correctly
- [ ] Base layer options work (Satellite, Topo, OSM)
- [ ] Protected area boundaries layer displays
- [ ] Observation points layer displays
- [ ] Transect routes layer displays
- [ ] Monitoring locations layer displays
- [ ] Layer toggle (on/off) works
- [ ] Map clustering works for dense points
- [ ] Click on marker shows popup with details
- [ ] Popup links to detail pages work
- [ ] Map filtering by species works
- [ ] Map filtering by date range works
- [ ] Zoom and pan controls work

### 6.5 GeoJSON API
- [ ] Protected areas GeoJSON endpoint works
- [ ] Transect routes GeoJSON endpoint works
- [ ] Monitoring locations GeoJSON endpoint works
- [ ] Observations GeoJSON endpoint works
- [ ] GeoJSON follows RFC 7946 format
- [ ] API respects user permissions
- [ ] Pagination works for large datasets

### 6.6 Shapefile Export
- [ ] Shapefile export option is available
- [ ] Export generates valid shapefile (ZIP)
- [ ] Shapefile contains correct geometry
- [ ] Attributes are properly mapped
- [ ] Export handles large datasets
- [ ] Download works correctly

### 6.7 Spatial Assignment
- [ ] Observations with GPS auto-assign to nearest transect segment
- [ ] 50m buffer for segment matching works
- [ ] Protected area assignment works
- [ ] Spatial indexing provides acceptable performance

---

## Phase 7: Dashboard & Widget System

### 7.1 Custom Dashboard
- [ ] User can access their custom dashboard
- [ ] Default dashboard auto-creates for new users
- [ ] Role-specific default widgets are applied
- [ ] Dashboard displays all visible widgets
- [ ] Widget data loads correctly
- [ ] Multiple dashboards per user supported
- [ ] Dashboard switcher works

### 7.2 Dashboard CRUD
- [ ] User can create new dashboard
- [ ] Dashboard name and description can be set
- [ ] User can edit dashboard properties
- [ ] User can delete dashboard (with confirmation)
- [ ] Only one dashboard can be default
- [ ] Dashboard can be duplicated

### 7.3 Widget Management
- [ ] Widget library is accessible
- [ ] User can add widget to dashboard
- [ ] Widget type selection works
- [ ] Widget configuration options display
- [ ] Widget title can be customized
- [ ] Widget position (row, column) can be set
- [ ] Widget size (width, height) can be set
- [ ] Widget can be edited
- [ ] Widget can be deleted
- [ ] Widget visibility toggle works

### 7.4 Widget Types
#### Counter Widget
- [ ] Counter widget displays single metric
- [ ] Value updates correctly
- [ ] Configuration for data source works

#### Chart Widgets
- [ ] Pie chart displays correctly
- [ ] Bar chart displays correctly
- [ ] Line chart displays correctly
- [ ] Chart data is accurate
- [ ] Chart legends display
- [ ] Chart interactivity works

#### Data Table Widget
- [ ] Table widget displays data rows
- [ ] Table pagination works
- [ ] Table sorting works

#### Map Widget
- [ ] Mini-map displays correctly
- [ ] Map shows relevant data layer
- [ ] Map is interactive

#### Other Widgets
- [ ] Text/Notes widget works
- [ ] Image widget works
- [ ] Recent activity feed works
- [ ] Assigned submissions widget works (Validator)
- [ ] Validation stats widget works (Validator)
- [ ] Species summary widget works
- [ ] Disturbance alerts widget works
- [ ] Schedule calendar widget works

### 7.5 Dashboard Templates
- [ ] Template gallery is accessible
- [ ] System templates display
- [ ] Template preview works
- [ ] User can instantiate template
- [ ] Instantiated dashboard has all template widgets
- [ ] User can create template from dashboard

### 7.6 Dashboard Sharing
- [ ] Dashboard can be marked as shared
- [ ] Specific users can be selected for sharing
- [ ] Shared dashboard visible to selected users
- [ ] Shared users can view but not edit

### 7.7 Widget Auto-Refresh
- [ ] Refresh interval can be configured
- [ ] Widget data auto-refreshes
- [ ] Manual refresh button works

---

## Phase 8: Report Generation System

### 8.1 Report Builder Interface
- [ ] Report builder page is accessible
- [ ] Report type selection works (GIS, Semestral, Standard, Backup, Finalized)
- [ ] Date range picker works
- [ ] Geographic scope selection works (Region, Province, Municipality, Barangay)
- [ ] Monitoring location filter works
- [ ] Species filter works
- [ ] Output format selection works (PDF, Excel, GeoJSON, Shapefile, Markdown)
- [ ] Include options work (maps, charts, photos)
- [ ] Report title can be customized
- [ ] Report description can be added

### 8.2 Report Generation
- [ ] Report request is submitted successfully
- [ ] Report shows as "Pending" status
- [ ] Celery task is created
- [ ] Report status updates to "Processing"
- [ ] Progress percentage updates
- [ ] Report status updates to "Completed" on success
- [ ] Report status updates to "Failed" on error
- [ ] Error message displays for failed reports
- [ ] User is notified when report is ready

### 8.3 Report Types
#### GIS-Enhanced Report
- [ ] Report includes spatial analysis
- [ ] Maps are generated
- [ ] GeoJSON output works
- [ ] Shapefile output works

#### Semestral BMS Report
- [ ] Report follows periodic format
- [ ] Date range is validated (6-month periods)
- [ ] Summary statistics are correct

#### Standard BMS Report
- [ ] Standard format is applied
- [ ] All required sections are included

#### Digital Backup Export
- [ ] Complete data export generated
- [ ] Multiple CSV files in ZIP
- [ ] All data tables included
- [ ] Shapefiles included

#### Finalized Report
- [ ] Official format applied
- [ ] Archiving works

### 8.4 Report Output Formats
- [ ] PDF generation works
- [ ] Excel (XLSX) generation works
- [ ] GeoJSON generation works
- [ ] Shapefile (ZIP) generation works
- [ ] Markdown generation works
- [ ] Multiple formats in single report work

### 8.5 Report Archive
- [ ] Report archive page is accessible
- [ ] All user's reports are listed
- [ ] Reports can be filtered by type
- [ ] Reports can be filtered by date
- [ ] Reports can be filtered by status
- [ ] Report details can be viewed
- [ ] Report files can be downloaded
- [ ] PDF preview in browser works
- [ ] File size displays correctly
- [ ] Download count is tracked

### 8.6 Report Templates
- [ ] User can save report configuration as template
- [ ] Template name and description can be set
- [ ] Template list displays
- [ ] User can create report from template
- [ ] Template can be edited
- [ ] Template can be deleted

### 8.7 Report Access Control
- [ ] Report creator can access their reports
- [ ] Public reports visible to appropriate roles
- [ ] Private reports only visible to creator
- [ ] Allowed users list works
- [ ] Staff/Admin can access all reports

### 8.8 Data Export Center
- [ ] Quick export options available
- [ ] Wildlife observations export works
- [ ] Species checklist export works
- [ ] Spatial data export works
- [ ] Export respects filters
- [ ] Export file downloads correctly

---

## Phase 9: Event & Schedule Management

### 9.1 Events System
#### Event List
- [ ] Event list page is accessible
- [ ] Events display with relevant details
- [ ] Events can be filtered by type
- [ ] Events can be filtered by date
- [ ] Events can be filtered by protected area
- [ ] Past events indicator works
- [ ] Ongoing events indicator works
- [ ] Calendar view option works (Month/Week/Day)

#### Event CRUD
- [ ] New event can be created
- [ ] Event title is required
- [ ] Event type selection works
- [ ] Start date/time picker works
- [ ] End date/time picker works
- [ ] Location field works
- [ ] Protected area link works
- [ ] Organizer is auto-set
- [ ] Attendees can be selected
- [ ] Description can be added
- [ ] Public/Private toggle works
- [ ] Event can be edited
- [ ] Event can be deleted (with confirmation)

#### Event Notifications
- [ ] Attendees receive notification for new events
- [ ] Notification sent for event updates
- [ ] Reminder notification before event

### 9.2 Scheduled Surveys
#### Survey Schedule List
- [ ] Scheduled surveys list displays
- [ ] Status filters work (Scheduled, In Progress, Completed, Overdue, Cancelled)
- [ ] Date filtering works
- [ ] Transect route filtering works

#### Survey CRUD
- [ ] New survey can be scheduled
- [ ] Transect route selection works
- [ ] Staff assignment (multiple) works
- [ ] Start time picker works
- [ ] End time picker works
- [ ] Notes field works
- [ ] Survey can be edited
- [ ] Survey can be cancelled

#### Survey Status Tracking
- [ ] Status auto-updates to "In Progress" when start time reached
- [ ] Status auto-updates based on submissions
- [ ] Status shows "Overdue" if no submissions after end time
- [ ] Status shows "Completed" if submissions received
- [ ] Related submissions link correctly

#### Survey Notifications
- [ ] Assigned staff receive notification
- [ ] Reminder notification before survey
- [ ] Overdue alert notification

### 9.3 Focus Group Discussions (FGD)
#### FGD Session List
- [ ] FGD sessions list displays
- [ ] Sessions can be filtered by status
- [ ] Sessions can be filtered by protected area
- [ ] Sessions can be filtered by barangay
- [ ] Sessions can be filtered by date

#### FGD Session CRUD
- [ ] New FGD session can be created
- [ ] Title and date are required
- [ ] Start/end time works
- [ ] Location field works
- [ ] Protected area link works
- [ ] Barangay selection works
- [ ] Facilitator is assigned
- [ ] Co-facilitators can be added
- [ ] Participant count is tracked
- [ ] Objectives can be added
- [ ] Methodology can be described
- [ ] Session can be edited
- [ ] Session can be cancelled

#### FGD Participants
- [ ] Participants can be added to session
- [ ] Participant demographics can be recorded
- [ ] Participant list displays
- [ ] Participant can be removed

#### FGD Findings
- [ ] Key findings can be added
- [ ] Findings can be categorized/themed
- [ ] Findings can be edited
- [ ] Findings can be deleted
- [ ] Summary can be written

#### FGD Attachments
- [ ] Audio recording can be uploaded
- [ ] Consent form can be uploaded
- [ ] Photos can be uploaded
- [ ] Attachments can be downloaded
- [ ] Attachments can be deleted

---

## Phase 10: KoboToolbox Integration

### 10.1 Choice Lists
- [ ] Choice list management page is accessible
- [ ] All choice lists display
- [ ] Choice count per list shows
- [ ] New choice list can be created
- [ ] Choice list can be edited
- [ ] Choice list can be deleted (if no mappings)
- [ ] Default choice lists initialize correctly

### 10.2 Individual Choices
- [ ] Choices within list display
- [ ] New choice can be added to list
- [ ] Choice name (value) is set
- [ ] Choice label (display text) is set
- [ ] Choice order can be set
- [ ] Choice can be activated/deactivated
- [ ] Choice can be linked to fauna species
- [ ] Choice can be linked to flora species
- [ ] Choice can be linked to monitoring location
- [ ] Choice can be edited
- [ ] Choice can be deleted

### 10.3 Form Choice Mappings
- [ ] Form mappings page is accessible
- [ ] All mappings display
- [ ] New mapping can be created
- [ ] Kobo form (asset_uid) can be specified
- [ ] Question name can be specified
- [ ] Choice list can be linked
- [ ] Auto-update toggle works
- [ ] Mapping can be edited
- [ ] Mapping can be deleted

### 10.4 KoboToolbox Form Browser
- [ ] Kobo forms list fetches successfully
- [ ] Form details display
- [ ] Form questions can be viewed
- [ ] Select/choice questions are identified

### 10.5 Bidirectional Sync
#### Django to Kobo Sync
- [ ] Choice list sync to Kobo initiates
- [ ] XLSForm is modified correctly
- [ ] Updated form is uploaded to Kobo
- [ ] Sync completion is confirmed
- [ ] FormUpdateLog is created

#### Kobo to Django Sync
- [ ] Kobo choices can be imported
- [ ] Imported choices create/update Django choices
- [ ] Sync handles new choices
- [ ] Sync handles updated labels
- [ ] Conflict resolution works

### 10.6 Data Submission Sync
- [ ] KoboToolbox submissions sync via Celery task
- [ ] New submissions create DataSubmission records
- [ ] Submission data is stored correctly (JSONField)
- [ ] Duplicate submissions are handled
- [ ] Sync errors are logged
- [ ] Manual sync trigger works

---

## Phase 11: Cross-Role Access Control

### 11.1 PA Staff Access Restrictions
- [ ] PA Staff can access PA Staff dashboard
- [ ] PA Staff CANNOT access validation queue
- [ ] PA Staff CANNOT access validation workspace
- [ ] PA Staff CANNOT access admin functions
- [ ] PA Staff CANNOT access other users' submissions
- [ ] PA Staff CANNOT manage users
- [ ] PA Staff can only see own submissions

### 11.2 Validator Access Permissions
- [ ] Validator can access validator dashboard
- [ ] Validator can access validation queue
- [ ] Validator can access validation workspace
- [ ] Validator can access species verification
- [ ] Validator can access reports
- [ ] Validator CANNOT access admin user management
- [ ] Validator CANNOT modify other validators' decisions

### 11.3 DENR Personnel Access
- [ ] DENR can access DENR dashboard
- [ ] DENR can view approved observations
- [ ] DENR can access reports archive
- [ ] DENR can export data
- [ ] DENR can access interactive maps
- [ ] DENR CANNOT access validation queue
- [ ] DENR CANNOT validate submissions
- [ ] DENR CANNOT access admin functions
- [ ] DENR only sees validated data

### 11.4 Collaborator Access
- [ ] Collaborator can access collaborator dashboard
- [ ] Collaborator can view approved observations
- [ ] Collaborator can access species explorer
- [ ] Collaborator can access reports (public only)
- [ ] Collaborator can export limited data
- [ ] Collaborator can access maps (with privacy protection)
- [ ] Collaborator CANNOT access pending data
- [ ] Collaborator CANNOT submit data
- [ ] Collaborator CANNOT validate
- [ ] Collaborator sees anonymized PA Staff info
- [ ] Sensitive species coordinates are obscured

### 11.5 System Admin Full Access
- [ ] Admin can access all dashboards
- [ ] Admin can access validation queue
- [ ] Admin can validate submissions
- [ ] Admin can manage all users
- [ ] Admin can manage organizations
- [ ] Admin can manage choice lists
- [ ] Admin can access Django admin
- [ ] Admin can manage species
- [ ] Admin can manage geographic data
- [ ] Admin bypasses most permission checks

### 11.6 Decorator/Mixin Enforcement
- [ ] `@dashboard_access_required` blocks unapproved users
- [ ] `@multi_role_required` blocks unauthorized roles
- [ ] `@validator_required` works correctly
- [ ] `@pa_staff_required` works correctly
- [ ] `@denr_required` works correctly
- [ ] `@admin_required` works correctly
- [ ] AJAX requests return JSON 403 for unauthorized
- [ ] Non-AJAX requests redirect with message
- [ ] Rate limiting works (if enabled)

---

## Phase 12: System Integration & Performance

### 12.1 Celery Task Processing
- [ ] Celery worker starts correctly
- [ ] Redis connection works
- [ ] KoboToolbox sync task runs
- [ ] Report generation task runs
- [ ] Task status can be monitored
- [ ] Failed tasks are logged
- [ ] Task retry mechanism works

### 12.2 Database Operations
- [ ] PostgreSQL connection is stable
- [ ] PostGIS extension is functional
- [ ] Spatial queries execute correctly
- [ ] Migrations apply cleanly
- [ ] Database queries are optimized (no N+1)
- [ ] Complex queries complete in acceptable time

### 12.3 File Upload/Storage
- [ ] Image uploads work
- [ ] Audio uploads work (FGD)
- [ ] Document uploads work
- [ ] File size limits are enforced
- [ ] File type validation works
- [ ] Files are stored in correct directories
- [ ] Files can be retrieved/downloaded
- [ ] Thumbnail generation works

### 12.4 API Endpoints
- [ ] REST API authentication works
- [ ] GeoJSON endpoints return valid data
- [ ] API pagination works
- [ ] API filtering works
- [ ] API error responses are appropriate
- [ ] API rate limiting works (if enabled)

### 12.5 Frontend Functionality
- [ ] Pages load without JavaScript errors
- [ ] Bootstrap responsive design works
- [ ] Leaflet maps render correctly
- [ ] Chart.js visualizations render
- [ ] Form validation works (client-side)
- [ ] AJAX requests complete successfully
- [ ] Loading indicators display
- [ ] Error messages display appropriately
- [ ] Success messages display

### 12.6 Email Notifications
- [ ] Email configuration works
- [ ] Registration confirmation email sends
- [ ] Approval notification email sends
- [ ] Rejection notification email sends
- [ ] Password reset email sends
- [ ] Event notification emails send
- [ ] Validation notification emails send

### 12.7 Error Handling
- [ ] 404 page displays for not found
- [ ] 403 page displays for forbidden
- [ ] 500 page displays for server errors
- [ ] Error logging works
- [ ] Errors do not expose sensitive information

### 12.8 Performance
- [ ] Dashboard pages load in <3 seconds
- [ ] List pages with 1000+ records perform well
- [ ] Map with 1000+ points performs well
- [ ] Report generation handles large datasets
- [ ] Export handles large datasets
- [ ] Concurrent users don't cause issues
- [ ] Memory usage stays reasonable

### 12.9 Docker Environment
- [ ] Docker containers start successfully
- [ ] Web container runs Django
- [ ] Database container runs PostgreSQL
- [ ] Redis container runs
- [ ] Celery worker container runs
- [ ] Containers communicate correctly
- [ ] Volume mounts work (data persistence)
- [ ] Environment variables load correctly

---

## Test Execution Summary

### Phase Completion Tracking

| Phase | Total Tests | Passed | Failed | Blocked | % Complete |
|-------|-------------|--------|--------|---------|------------|
| Phase 1: Auth | 24 | | | | |
| Phase 2: User/Org | 36 | | | | |
| Phase 3: PA Staff | 29 | | | | |
| Phase 4: Validation | 52 | | | | |
| Phase 5: Species | 30 | | | | |
| Phase 6: GIS | 42 | | | | |
| Phase 7: Dashboard | 47 | | | | |
| Phase 8: Reports | 45 | | | | |
| Phase 9: Events | 48 | | | | |
| Phase 10: Kobo | 27 | | | | |
| Phase 11: Access | 36 | | | | |
| Phase 12: Integration | 34 | | | | |
| **TOTAL** | **450** | | | | |

---

## Test Environment Requirements

### Prerequisites
- [ ] Docker and Docker Compose installed
- [ ] Test database seeded with sample data
- [ ] Test KoboToolbox account configured (or mocked)
- [ ] Email server configured (or mocked)
- [ ] Test users for each role created

### Test Data Requirements
- [ ] At least 5 users per role
- [ ] At least 3 agencies with departments
- [ ] Complete geographic hierarchy (Region → Barangay)
- [ ] At least 50 species (fauna and flora)
- [ ] At least 100 data submissions
- [ ] At least 2 protected areas with transects

### Test User Accounts
| Role | Username | Purpose |
|------|----------|---------|
| PA Staff | `test_pa_staff` | PA Staff workflow testing |
| Validator | `test_validator` | Validation workflow testing |
| DENR | `test_denr` | DENR access testing |
| Collaborator | `test_collaborator` | Collaborator access testing |
| System Admin | `test_admin` | Admin workflow testing |
| Pending User | `test_pending` | Approval workflow testing |

---

## Notes

- Mark items with ✅ when test passes
- Mark items with ❌ when test fails (add bug ticket reference)
- Mark items with ⏸️ when blocked by dependency
- Add comments for any deviations or issues discovered
- Run regression tests after bug fixes
- Document any environment-specific issues

---

**Document Maintained By:** QA Team  
**Last Test Run:** [DATE]  
**Next Scheduled Run:** [DATE]
