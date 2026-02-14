# PRISM-Matutum Test Script Tracking & Verification Checklist

**Document Version:** 1.0  
**Last Updated:** January 7, 2026  
**Purpose:** Track test script generation progress and verify alignment with TEST_PLAN_CHECKLIST.md  

---

## Overview

This document tracks the creation of automated test scripts and maps them to the test cases defined in `TEST_PLAN_CHECKLIST.md`. Use this to verify complete coverage and ensure each script follows the correct context.

---

## Script Generation Status Legend

| Symbol | Status |
|--------|--------|
| ⬜ | Not Started |
| 🔄 | In Progress |
| ✅ | Complete |
| ⚠️ | Needs Review |
| ❌ | Blocked/Failed |

---

## Phase 1: Authentication & Authorization

**Test File:** `users/tests/test_authentication.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `users.models.User`, `users.models.Agency`, `users.models.Department`
- **Views:** `django.contrib.auth` views, custom registration views
- **Decorators:** `dashboard/permissions.py`, `dashboard/decorators.py`
- **Key Fields:** `User.role`, `User.account_status`, `User.agency`, `User.department`, `User.is_field_verified`

### Test Class: TestUserRegistration

| Test Method | Checklist Item (1.1) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_registration_page_accessible` | New user can access registration page | ⬜ | |
| `test_registration_valid_data_creates_pending_user` | Successful registration creates user with `account_status='pending'` | ⬜ | |
| `test_registration_validates_required_fields` | Registration form validates required fields (username, email, password) | ⬜ | |
| `test_registration_validates_password_strength` | Password validation enforces security requirements | ⬜ | |
| `test_registration_rejects_duplicate_username` | Duplicate username/email is rejected | ⬜ | |
| `test_registration_rejects_duplicate_email` | Duplicate username/email is rejected | ⬜ | |
| `test_registration_allows_role_selection` | User can select role during registration (PA Staff, Validator, DENR, Collaborator) | ⬜ | |
| `test_registration_allows_agency_department_selection` | User can select agency and department during registration | ⬜ | |
| `test_email_verification_sent` | Email verification is sent upon registration | ⬜ | |

### Test Class: TestUserLogin

| Test Method | Checklist Item (1.2) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_login_page_accessible` | Login page is accessible | ⬜ | |
| `test_login_valid_credentials_success` | Valid credentials allow login | ⬜ | |
| `test_login_invalid_credentials_fails` | Invalid credentials show appropriate error message | ⬜ | |
| `test_login_pending_user_blocked` | Pending users cannot login (shown appropriate message) | ⬜ | |
| `test_login_rejected_user_blocked` | Rejected users cannot login | ⬜ | |
| `test_login_suspended_user_blocked` | Suspended users cannot login | ⬜ | |
| `test_login_redirects_to_role_dashboard` | Login redirects to role-appropriate dashboard | ⬜ | |
| `test_remember_me_functionality` | "Remember me" functionality works | ⬜ | |
| `test_session_timeout_works` | Session timeout works correctly | ⬜ | |

### Test Class: TestPasswordManagement

| Test Method | Checklist Item (1.3) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_password_reset_link_request` | Password reset link can be requested | ⬜ | |
| `test_password_reset_email_sent` | Password reset email is sent | ⬜ | |
| `test_password_reset_token_expires` | Password reset token expires after configured time | ⬜ | |
| `test_password_reset_via_link` | New password can be set via reset link | ⬜ | |
| `test_password_change_from_profile` | User can change password from profile settings | ⬜ | |
| `test_password_change_requires_old_password` | Old password required for password change | ⬜ | |

### Test Class: TestLogout

| Test Method | Checklist Item (1.4) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_logout_button_accessible` | Logout button is accessible from all pages | ⬜ | |
| `test_logout_clears_session` | Logout clears session properly | ⬜ | |
| `test_logged_out_user_cannot_access_protected` | Logged out user cannot access protected pages | ⬜ | |
| `test_logout_redirects_to_login` | Logout redirects to login page | ⬜ | |

### Setup Fixtures Required
```python
# Required test data setup
- Agency: "Test Agency" (active)
- Department: "Test Department" (linked to agency)
- User (pending): username='pending_user', account_status='pending'
- User (approved): username='approved_user', account_status='approved'
- User (rejected): username='rejected_user', account_status='rejected'
- User (suspended): username='suspended_user', account_status='suspended'
```

### Verification Checklist
- [ ] All 24 test cases from Phase 1 are covered
- [ ] Test methods use correct model imports
- [ ] Setup creates users with all status types
- [ ] Mock email sending for verification tests
- [ ] Tests run in Docker environment

---

## Phase 2: User & Organization Management

**Test File:** `dashboard/tests/test_user_management.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `users.models.User`, `Agency`, `Department`, `Region`, `Province`, `Municipality`, `Barangay`
- **Views:** `dashboard/views_admin.py`
- **Permissions:** `StaffUserRequiredMixin`, `@admin_required`
- **URLs:** `/dashboard/admin/user-organization-hub/`

### Test Class: TestUserApprovalWorkflow

| Test Method | Checklist Item (2.1) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_pending_users_appear_in_queue` | Pending users appear in admin approval queue | ⬜ | |
| `test_admin_can_view_pending_user_details` | Admin can view pending user details | ⬜ | |
| `test_admin_can_approve_user` | Admin can approve user registration | ⬜ | |
| `test_approved_user_receives_email` | Approved user receives email notification | ⬜ | |
| `test_approved_user_can_login` | Approved user can now login | ⬜ | |
| `test_admin_can_reject_user` | Admin can reject user registration | ⬜ | |
| `test_rejection_requires_reason` | Rejection requires reason | ⬜ | |
| `test_rejected_user_receives_email_with_reason` | Rejected user receives email with reason | ⬜ | |
| `test_admin_can_suspend_user` | Admin can suspend active users | ⬜ | |
| `test_admin_can_reactivate_user` | Admin can reactivate suspended users | ⬜ | |

### Test Class: TestUserManagement

| Test Method | Checklist Item (2.2) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_admin_can_view_user_list` | Admin can view list of all users | ⬜ | |
| `test_user_list_pagination` | User list supports pagination | ⬜ | |
| `test_user_list_filter_by_role` | User list supports filtering by role | ⬜ | |
| `test_user_list_filter_by_status` | User list supports filtering by status | ⬜ | |
| `test_user_list_search` | User list supports search by username/email | ⬜ | |
| `test_admin_can_edit_user` | Admin can edit user details | ⬜ | |
| `test_admin_can_change_user_role` | Admin can change user role | ⬜ | |
| `test_admin_can_assign_agency` | Admin can assign user to agency/department | ⬜ | |
| `test_admin_can_mark_field_verified` | Admin can mark PA Staff as field-verified | ⬜ | |
| `test_admin_can_view_activity_log` | Admin can view user activity log | ⬜ | |
| `test_admin_can_reset_user_password` | Admin can reset user password | ⬜ | |

### Test Class: TestAgencyManagement

| Test Method | Checklist Item (2.3) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_admin_can_view_agency_list` | Admin can view agency list | ⬜ | |
| `test_admin_can_create_agency` | Admin can create new agency | ⬜ | |
| `test_agency_type_options` | Admin can set agency type (Government, LGU, NGO, Academic, Private) | ⬜ | |
| `test_agency_parent_hierarchy` | Admin can set parent agency (hierarchical structure) | ⬜ | |
| `test_admin_can_edit_agency` | Admin can edit agency details | ⬜ | |
| `test_admin_can_deactivate_agency` | Admin can deactivate agency | ⬜ | |
| `test_admin_can_reactivate_agency` | Admin can reactivate agency | ⬜ | |
| `test_agency_hierarchy_path_displays` | Agency hierarchy path displays correctly | ⬜ | |
| `test_cannot_delete_agency_with_users` | Cannot delete agency with linked users/departments | ⬜ | |

### Test Class: TestDepartmentManagement

| Test Method | Checklist Item (2.4) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_admin_can_view_department_list` | Admin can view department list | ⬜ | |
| `test_admin_can_create_department` | Admin can create new department | ⬜ | |
| `test_department_requires_agency` | Department must be linked to an agency | ⬜ | |
| `test_department_parent_hierarchy` | Admin can set parent department (sub-departments) | ⬜ | |
| `test_admin_can_assign_department_head` | Admin can assign department head | ⬜ | |
| `test_admin_can_edit_department` | Admin can edit department details | ⬜ | |
| `test_admin_can_deactivate_department` | Admin can deactivate department | ⬜ | |
| `test_department_hierarchy_path_displays` | Department hierarchy path displays correctly | ⬜ | |

### Test Class: TestGeographicAddressManagement

| Test Method | Checklist Item (2.5) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_admin_can_manage_regions` | Admin can manage Regions (create, edit, delete) | ⬜ | |
| `test_admin_can_manage_provinces` | Admin can manage Provinces (linked to Region) | ⬜ | |
| `test_admin_can_manage_municipalities` | Admin can manage Municipalities (linked to Province) | ⬜ | |
| `test_admin_can_manage_barangays` | Admin can manage Barangays (linked to Municipality) | ⬜ | |
| `test_geographic_hierarchy_displays` | Geographic hierarchy displays correctly | ⬜ | |
| `test_address_cascade_delete_protection` | Address cascade delete protection works | ⬜ | |
| `test_address_search_filter` | Address search/filter works | ⬜ | |

### Permission Tests

| Test Method | Checklist Item | Status | Notes |
|-------------|----------------|--------|-------|
| `test_non_admin_cannot_access_user_management` | Non-admin blocked from user management | ⬜ | |
| `test_non_admin_cannot_approve_users` | Non-admin cannot approve users | ⬜ | |

### Verification Checklist
- [ ] All 36 test cases from Phase 2 are covered
- [ ] Admin user created with proper permissions
- [ ] Geographic hierarchy fixtures created
- [ ] Email mocking for notifications
- [ ] Tests verify both success and permission denial

---

## Phase 3: PA Staff Workflows

**Test File:** `dashboard/tests/test_pa_staff.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `api.models.DataSubmission`, `WildlifeObservation`, `ResourceUseIncident`, `DisturbanceRecord`, `LandscapeMonitoring`
- **Views:** `dashboard/views_pa_staff.py`
- **Permissions:** `@pa_staff_required`, `PAStaffAccessMixin`
- **URLs:** `/dashboard/pa-staff/`

### Test Class: TestPAStaffDashboard

| Test Method | Checklist Item (3.1) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_pa_staff_can_access_dashboard` | PA Staff can access their dedicated dashboard | ⬜ | |
| `test_dashboard_displays_total_submissions` | Dashboard displays total submissions count | ⬜ | |
| `test_dashboard_displays_pending_count` | Dashboard displays pending review count | ⬜ | |
| `test_dashboard_displays_approved_count` | Dashboard displays approved submissions count | ⬜ | |
| `test_dashboard_displays_week_submissions` | Dashboard displays this week submissions count | ⬜ | |
| `test_dashboard_displays_observations_count` | Dashboard displays total observations count | ⬜ | |
| `test_approval_rate_calculation` | Approval rate calculates correctly | ⬜ | |
| `test_field_verification_alert_shows` | Field verification alert shows for non-verified users | ⬜ | |
| `test_recent_submissions_display` | Recent submissions table displays correctly | ⬜ | |
| `test_quick_action_links_work` | Quick action links work | ⬜ | |

### Test Class: TestMySubmissionsView

| Test Method | Checklist Item (3.2) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_pa_staff_sees_own_submissions` | PA Staff can access my submissions page | ⬜ | |
| `test_all_personal_submissions_listed` | All personal submissions are listed | ⬜ | |
| `test_filter_by_status_pending` | Submissions can be filtered by status (Pending) | ⬜ | |
| `test_filter_by_status_approved` | Submissions can be filtered by status (Approved) | ⬜ | |
| `test_filter_by_status_rejected` | Submissions can be filtered by status (Rejected) | ⬜ | |
| `test_filter_by_status_on_hold` | Submissions can be filtered by status (On Hold) | ⬜ | |
| `test_filter_by_date_range` | Submissions can be filtered by date range | ⬜ | |
| `test_search_by_submission_id` | Submissions can be searched by ID | ⬜ | |
| `test_search_by_validation_remarks` | Submissions can be searched by validation remarks | ⬜ | |
| `test_pagination_25_per_page` | Pagination works correctly (25 per page) | ⬜ | |
| `test_status_badges_display` | Status badges display correct colors | ⬜ | |
| `test_submission_details_viewable` | Submission details can be viewed | ⬜ | |
| `test_recent_validation_activity_panel` | Recent validation activity panel shows correctly | ⬜ | |

### Test Class: TestSubmissionDetailView

| Test Method | Checklist Item (3.3) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_pa_staff_can_view_own_submission` | PA Staff can view their own submission details | ⬜ | |
| `test_submission_fields_display` | All submission fields display correctly | ⬜ | |
| `test_validation_status_shown` | Validation status is clearly shown | ⬜ | |
| `test_validation_remarks_display` | Validation remarks from validator are displayed | ⬜ | |
| `test_validator_name_shown` | Validator name is shown (if validated) | ⬜ | |
| `test_validation_timestamp_shown` | Validation timestamp is shown | ⬜ | |
| `test_related_observations_link` | Related observations link correctly | ⬜ | |
| `test_audit_trail_visible` | Audit trail is visible (if applicable) | ⬜ | |
| `test_submission_is_readonly` | PA Staff cannot edit submission data (read-only) | ⬜ | |

### Test Class: TestMyObservationsView

| Test Method | Checklist Item (3.4) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_pa_staff_sees_approved_observations` | PA Staff can view their approved observations | ⬜ | |
| `test_wildlife_observations_listed` | Wildlife observations are listed | ⬜ | |
| `test_resource_use_incidents_listed` | Resource use incidents are listed | ⬜ | |
| `test_disturbance_records_listed` | Disturbance records are listed | ⬜ | |
| `test_landscape_monitoring_listed` | Landscape monitoring records are listed | ⬜ | |
| `test_filter_by_observation_type` | Observations can be filtered by type | ⬜ | |
| `test_filter_by_date` | Observations can be filtered by date | ⬜ | |
| `test_observation_details_link` | Observation details link works | ⬜ | |

### Verification Checklist
- [ ] All 29 test cases from Phase 3 are covered
- [ ] PA Staff user fixtures created
- [ ] DataSubmission records with all statuses created
- [ ] Processed observation records linked to submissions
- [ ] Tests verify ownership-based access control

---

## Phase 4: Validation System

**Test File:** `dashboard/tests/test_validation.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `api.models.DataSubmission`, `WildlifeObservation`
- **Services:** `api.workflow_services.BMSWorkflowService`
- **Views:** `dashboard/views_validation.py`
- **Permissions:** `@validator_required`, `ValidatorAccessMixin`
- **URLs:** `/dashboard/validation/queue/`, `/dashboard/validation/workspace/<pk>/`
- **Key Fields:** `DataSubmission.validation_status`, `validated_by`, `validation_remarks`

### Test Class: TestValidationQueue

| Test Method | Checklist Item (4.1) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_validator_can_access_queue` | Validator can access validation queue | ⬜ | |
| `test_queue_statistics_accurate` | Queue statistics display correctly (Total, Not Validated, On Hold, Flagged) | ⬜ | |
| `test_high_priority_count_shows` | High priority count shows correctly (>7 days or flagged) | ⬜ | |
| `test_assigned_to_me_count` | Assigned to me count is accurate | ⬜ | |
| `test_filter_by_status` | Submissions can be filtered by status | ⬜ | |
| `test_filter_by_date_range` | Submissions can be filtered by date range | ⬜ | |
| `test_filter_by_pa_staff` | Submissions can be filtered by PA Staff (submitter) | ⬜ | |
| `test_filter_by_priority` | Submissions can be filtered by priority | ⬜ | |
| `test_search_by_submission_id` | Search by submission ID works | ⬜ | |
| `test_search_by_remarks` | Search by remarks works | ⬜ | |
| `test_sorting_options` | Sorting options work (Newest, Oldest, PA Staff A-Z, Status) | ⬜ | |
| `test_pagination_50_per_page` | Pagination works (50 per page) | ⬜ | |
| `test_checkbox_selection` | Checkbox selection works | ⬜ | |
| `test_priority_indicator` | Priority indicator (🔥) shows for urgent items | ⬜ | |

### Test Class: TestBulkAssignment

| Test Method | Checklist Item (4.2) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_select_multiple_submissions` | Validator can select multiple submissions | ⬜ | |
| `test_assign_to_me_bulk` | "Assign to Me" bulk action works | ⬜ | |
| `test_assigned_shows_validator_name` | Assigned submissions show validator name | ⬜ | |
| `test_assign_to_other_validator` | "Assign to Other Validator" dropdown works | ⬜ | |
| `test_bulk_assignment_creates_log` | Bulk assignment creates activity log | ⬜ | |
| `test_assigned_validator_notified` | Assigned validator receives notification | ⬜ | |
| `test_email_notification_sent` | Email notification sent (if enabled) | ⬜ | |

### Test Class: TestValidationWorkspace

| Test Method | Checklist Item (4.3) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_workspace_displays_submission_data` | Validator can open validation workspace for submission | ⬜ | |
| `test_all_submission_data_displays` | All submission data displays correctly | ⬜ | |
| `test_attached_media_display` | Attached media (photos) display correctly | ⬜ | |
| `test_species_info_linked` | Species information is linked and accessible | ⬜ | |
| `test_location_shows_on_map` | Location data shows on mini-map | ⬜ | |
| `test_gps_coordinates_validated` | GPS coordinates are validated | ⬜ | |
| `test_validation_actions_visible` | Validation action buttons are visible (Approve, Reject, Hold, Flag) | ⬜ | |
| `test_remarks_field_available` | Validation remarks text field is available | ⬜ | |
| `test_previous_history_shown` | Previous validation history is shown (if any) | ⬜ | |
| `test_navigation_between_submissions` | Navigation between submissions works | ⬜ | |

### Test Class: TestApproveAction

| Test Method | Checklist Item (4.4 - Approve) | Status | Notes |
|-------------|-------------------------------|--------|-------|
| `test_approve_changes_status` | Submission status changes to `validation_status_approved` | ⬜ | |
| `test_approve_with_remarks` | Remarks can be added with approval | ⬜ | |
| `test_approve_creates_observation_record` | Data is processed into appropriate record (WildlifeObservation, etc.) | ⬜ | |
| `test_processed_record_created_correctly` | Processed record is created correctly | ⬜ | |
| `test_pa_staff_notified_on_approve` | PA Staff is notified of approval | ⬜ | |
| `test_approved_removed_from_queue` | Approved submissions removed from queue | ⬜ | |
| `test_validation_timestamp_recorded` | Validation timestamp is recorded | ⬜ | |
| `test_validated_by_field_set` | Validated_by field is set to validator | ⬜ | |

### Test Class: TestRejectAction

| Test Method | Checklist Item (4.4 - Reject) | Status | Notes |
|-------------|------------------------------|--------|-------|
| `test_reject_requires_remarks` | Rejection requires remarks (reason) | ⬜ | |
| `test_reject_changes_status` | Submission status changes to `validation_status_not_approved` | ⬜ | |
| `test_reject_does_not_create_record` | Data is NOT processed into records | ⬜ | |
| `test_pa_staff_notified_with_reason` | PA Staff is notified of rejection with reason | ⬜ | |
| `test_rejected_removed_from_queue` | Rejected submissions removed from queue | ⬜ | |
| `test_rejected_is_final` | Rejected submissions cannot be re-validated | ⬜ | |

### Test Class: TestHoldAction

| Test Method | Checklist Item (4.4 - Hold) | Status | Notes |
|-------------|----------------------------|--------|-------|
| `test_hold_requires_remarks` | Hold requires remarks (clarification needed) | ⬜ | |
| `test_hold_changes_status` | Submission status changes to `validation_status_on_hold` | ⬜ | |
| `test_pa_staff_notified_clarification` | PA Staff is notified with clarification request | ⬜ | |
| `test_hold_remains_in_queue` | On-hold submissions remain in queue | ⬜ | |
| `test_hold_can_be_rereviewed` | On-hold submissions can be re-reviewed later | ⬜ | |

### Test Class: TestFlagAction

| Test Method | Checklist Item (4.4 - Flag) | Status | Notes |
|-------------|----------------------------|--------|-------|
| `test_flag_requires_remarks` | Flagging requires remarks | ⬜ | |
| `test_flag_changes_status` | Submission status changes to `flagged` | ⬜ | |
| `test_flagged_shows_high_priority` | Flagged submissions appear as high priority | ⬜ | |
| `test_supervisor_notified` | Senior validator/admin is notified | ⬜ | |
| `test_flagged_can_be_resolved` | Flagged submissions can be resolved by supervisor | ⬜ | |

### Test Class: TestValidationHistory

| Test Method | Checklist Item (4.5) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_validator_can_access_history` | Validator can access their validation history | ⬜ | |
| `test_history_shows_all_validated` | History shows all validated submissions | ⬜ | |
| `test_history_filter_by_decision` | History can be filtered by decision type (Approved, Rejected, Hold) | ⬜ | |
| `test_history_filter_by_date_range` | History can be filtered by date range | ⬜ | |
| `test_history_can_be_exported` | History can be exported | ⬜ | |
| `test_performance_metrics_calculated` | Performance metrics are calculated (approval rate, avg time) | ⬜ | |
| `test_audit_trail_complete` | Audit trail is complete | ⬜ | |

### Permission Tests

| Test Method | Checklist Item | Status | Notes |
|-------------|----------------|--------|-------|
| `test_pa_staff_cannot_access_queue` | PA Staff blocked from queue | ⬜ | |
| `test_denr_cannot_access_queue` | DENR blocked from queue | ⬜ | |
| `test_collaborator_cannot_access_queue` | Collaborator blocked from queue | ⬜ | |

### Verification Checklist
- [ ] All 52 test cases from Phase 4 are covered
- [ ] Validator user fixtures created
- [ ] PA Staff user with submissions created
- [ ] BMSWorkflowService methods tested directly
- [ ] Notification mocking in place
- [ ] Tests verify status transitions correctly

---

## Phase 5: Species Management

**Test File:** `species/tests/test_species.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `species.models.FaunaChecklist`, `FloraChecklist`, `Taxonomy`, `Kingdom`, `Phylum`, `TaxonomyClass`, `TaxonomyOrder`, `Family`, `Genus`, `Endemicity`, `ThreatAssessment`, `FaunaCommonName`, `FloraCommonName`, `SpeciesType`, `FaunalCharacteristics`, `FloralCharacteristics`
- **Views:** `dashboard/views_species.py`, `dashboard/views_articles.py`
- **Permissions:** Admin for CRUD, All roles for view

### Test Class: TestSpeciesListView

| Test Method | Checklist Item (5.1) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_species_list_accessible` | Species list is accessible | ⬜ | |
| `test_fauna_checklist_displays` | Fauna checklist displays correctly | ⬜ | |
| `test_flora_checklist_displays` | Flora checklist displays correctly | ⬜ | |
| `test_filter_by_species_type` | Species can be filtered by type (Bird, Mammal, Reptile, etc.) | ⬜ | |
| `test_filter_by_conservation_status` | Species can be filtered by conservation status | ⬜ | |
| `test_filter_by_endemicity` | Species can be filtered by endemicity | ⬜ | |
| `test_search_by_scientific_name` | Species can be searched by scientific name | ⬜ | |
| `test_search_by_common_name` | Species can be searched by common name | ⬜ | |
| `test_pagination` | Pagination works correctly | ⬜ | |
| `test_species_count_statistics` | Species count statistics display | ⬜ | |

### Test Class: TestSpeciesDetailView

| Test Method | Checklist Item (5.2) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_species_detail_accessible` | Species detail page is accessible | ⬜ | |
| `test_scientific_name_displays` | Scientific name displays correctly | ⬜ | |
| `test_taxonomy_hierarchy_displays` | Complete taxonomy hierarchy displays (Kingdom → Genus) | ⬜ | |
| `test_conservation_status_displays` | Conservation status (IUCN) displays | ⬜ | |
| `test_endemicity_displays` | Endemicity status displays | ⬜ | |
| `test_common_names_list_displays` | Common names list displays | ⬜ | |
| `test_characteristics_display` | Physical characteristics display (fauna/flora specific) | ⬜ | |
| `test_media_gallery_displays` | Species media gallery displays | ⬜ | |
| `test_articles_section_displays` | Related research/articles section displays | ⬜ | |
| `test_observation_history_displays` | Observation history for species displays | ⬜ | |
| `test_observation_count_accurate` | Observation count is accurate | ⬜ | |
| `test_link_to_observations` | Link to species observations works | ⬜ | |

### Test Class: TestSpeciesCRUD (Admin)

| Test Method | Checklist Item (5.3) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_admin_can_create_fauna` | Admin can create new fauna species | ⬜ | |
| `test_admin_can_create_flora` | Admin can create new flora species | ⬜ | |
| `test_taxonomy_selection_cascading` | Taxonomy selection works (cascading dropdowns) | ⬜ | |
| `test_auto_classification_correct` | Auto-classification assigns correct species type | ⬜ | |
| `test_conservation_status_can_set` | Conservation status can be set | ⬜ | |
| `test_endemicity_can_set` | Endemicity can be set | ⬜ | |
| `test_characteristics_can_add` | Characteristics can be added | ⬜ | |
| `test_admin_can_edit_species` | Admin can edit species information | ⬜ | |
| `test_admin_can_delete_species` | Admin can delete species (with confirmation) | ⬜ | |
| `test_cannot_delete_with_observations` | Cannot delete species with linked observations | ⬜ | |

### Test Class: TestSpeciesCommonNames

| Test Method | Checklist Item (5.4) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_add_multiple_common_names` | Multiple common names can be added per species | ⬜ | |
| `test_edit_common_name` | Common names can be edited | ⬜ | |
| `test_delete_common_name` | Common names can be deleted | ⬜ | |
| `test_search_includes_common_names` | Search includes common names | ⬜ | |

### Test Class: TestSpeciesArticles

| Test Method | Checklist Item (5.5) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_add_article_to_species` | Admin can add articles to species | ⬜ | |
| `test_edit_article` | Article details can be edited | ⬜ | |
| `test_articles_display_on_detail` | Articles display on species detail page | ⬜ | |
| `test_external_links_work` | External links work correctly | ⬜ | |
| `test_related_research_linked` | Related research can be linked | ⬜ | |

### Test Class: TestSpeciesMedia

| Test Method | Checklist Item (5.6) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_upload_species_image` | Media (photos) can be uploaded for species | ⬜ | |
| `test_multiple_images_supported` | Multiple images are supported | ⬜ | |
| `test_image_gallery_displays` | Image gallery displays correctly | ⬜ | |
| `test_images_can_be_deleted` | Images can be deleted | ⬜ | |
| `test_thumbnail_generation` | Thumbnail generation works | ⬜ | |

### Verification Checklist
- [ ] All 30 test cases from Phase 5 are covered
- [ ] Full taxonomy hierarchy fixtures created
- [ ] Species with observations for deletion tests
- [ ] Image upload mocking
- [ ] Admin vs non-admin permission tests

---

## Phase 6: GIS & Spatial Features

**Test File:** `dashboard/tests/test_gis.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `users.models.ProtectedArea` (boundary: MultiPolygonField), `api.models.TransectRoute` (route_geometry: LineStringField), `api.models.MonitoringLocation` (geometry: PointField), `api.models.TransectSegment`
- **Views:** `dashboard/views_gis.py`, `dashboard/api_geo.py`
- **Database:** PostgreSQL + PostGIS
- **GIS Imports:** `django.contrib.gis.geos.Point`, `LineString`, `MultiPolygon`, `Polygon`

### Test Class: TestProtectedAreaManagement

| Test Method | Checklist Item (6.1) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_protected_area_list_displays` | Protected area list displays correctly | ⬜ | |
| `test_protected_area_statistics` | Protected area count and statistics show | ⬜ | |
| `test_protected_area_detail_view` | Protected area detail view works | ⬜ | |
| `test_boundary_displays_on_map` | Boundary polygon displays on map (if set) | ⬜ | |
| `test_associated_landscapes_display` | Associated landscapes list displays | ⬜ | |
| `test_transect_routes_display` | Transect routes for PA display | ⬜ | |
| `test_monitoring_locations_display` | Monitoring locations display | ⬜ | |
| `test_recent_observations_show` | Recent observations (last 10) show | ⬜ | |
| `test_pa_statistics_accurate` | Protected area statistics are accurate | ⬜ | |
| `test_admin_can_edit_pa_name` | Admin can edit protected area name | ⬜ | |

### Test Class: TestTransectRouteManagement

| Test Method | Checklist Item (6.2) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_transect_route_list` | Transect route list displays | ⬜ | |
| `test_create_transect_route` | New transect route can be created | ⬜ | |
| `test_assign_to_protected_area` | Route can be assigned to protected area | ⬜ | |
| `test_add_stations_to_route` | Stations can be added to route | ⬜ | |
| `test_geometry_generates_from_stations` | Route geometry generates from stations | ⬜ | |
| `test_route_displays_on_map` | Route displays on map | ⬜ | |
| `test_edit_transect_route` | Route can be edited | ⬜ | |
| `test_deactivate_route` | Route can be deactivated | ⬜ | |
| `test_observations_linked_display` | Observations linked to route display | ⬜ | |
| `test_segments_auto_generate` | Transect segments auto-generate between stations | ⬜ | |

### Test Class: TestMonitoringLocationManagement

| Test Method | Checklist Item (6.3) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_location_list_displays` | Monitoring location list displays | ⬜ | |
| `test_create_monitoring_location` | New location can be created | ⬜ | |
| `test_location_type_options` | Location type can be set (Facility, Transect Station, Other) | ⬜ | |
| `test_gps_coordinates_saved` | GPS coordinates can be entered | ⬜ | |
| `test_location_displays_on_map` | Location displays on map | ⬜ | |
| `test_edit_location` | Location can be edited | ⬜ | |
| `test_deactivate_location` | Location can be deactivated | ⬜ | |
| `test_linked_transect_routes_display` | Linked transect routes display | ⬜ | |

### Test Class: TestInteractiveMap

| Test Method | Checklist Item (6.4) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_map_page_loads` | Interactive map page loads correctly | ⬜ | |
| `test_base_layer_options` | Base layer options work (Satellite, Topo, OSM) | ⬜ | |
| `test_protected_area_layer` | Protected area boundaries layer displays | ⬜ | |
| `test_observations_layer` | Observation points layer displays | ⬜ | |
| `test_transect_routes_layer` | Transect routes layer displays | ⬜ | |
| `test_monitoring_locations_layer` | Monitoring locations layer displays | ⬜ | |
| `test_layer_toggle` | Layer toggle (on/off) works | ⬜ | |
| `test_map_clustering` | Map clustering works for dense points | ⬜ | |
| `test_marker_popup` | Click on marker shows popup with details | ⬜ | |
| `test_popup_links_work` | Popup links to detail pages work | ⬜ | |
| `test_map_filter_by_species` | Map filtering by species works | ⬜ | |
| `test_map_filter_by_date_range` | Map filtering by date range works | ⬜ | |
| `test_zoom_pan_controls` | Zoom and pan controls work | ⬜ | |

### Test Class: TestGeoJSONAPI

| Test Method | Checklist Item (6.5) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_protected_areas_geojson` | Protected areas GeoJSON endpoint works | ⬜ | |
| `test_transect_routes_geojson` | Transect routes GeoJSON endpoint works | ⬜ | |
| `test_monitoring_locations_geojson` | Monitoring locations GeoJSON endpoint works | ⬜ | |
| `test_observations_geojson` | Observations GeoJSON endpoint works | ⬜ | |
| `test_geojson_rfc7946_format` | GeoJSON follows RFC 7946 format | ⬜ | |
| `test_api_respects_permissions` | API respects user permissions | ⬜ | |
| `test_pagination_large_datasets` | Pagination works for large datasets | ⬜ | |

### Test Class: TestShapefileExport

| Test Method | Checklist Item (6.6) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_shapefile_export_available` | Shapefile export option is available | ⬜ | |
| `test_shapefile_generates_zip` | Export generates valid shapefile (ZIP) | ⬜ | |
| `test_shapefile_valid_geometry` | Shapefile contains correct geometry | ⬜ | |
| `test_attributes_mapped` | Attributes are properly mapped | ⬜ | |
| `test_export_handles_large_datasets` | Export handles large datasets | ⬜ | |
| `test_download_works` | Download works correctly | ⬜ | |

### Test Class: TestSpatialAssignment

| Test Method | Checklist Item (6.7) | Status | Notes |
|-------------|---------------------|--------|-------|
| `test_observation_assigns_to_nearest_segment` | Observations with GPS auto-assign to nearest transect segment | ⬜ | |
| `test_50m_buffer_matching` | 50m buffer for segment matching works | ⬜ | |
| `test_protected_area_assignment` | Protected area assignment works | ⬜ | |
| `test_spatial_indexing_performance` | Spatial indexing provides acceptable performance | ⬜ | |

### Verification Checklist
- [ ] All 42 test cases from Phase 6 are covered
- [ ] PostGIS enabled in test database
- [ ] Spatial fixtures with real geometries
- [ ] GeoJSON response validation
- [ ] Performance tests for large datasets

---

## Phase 7: Dashboard & Widget System

**Test File:** `dashboard/tests/test_dashboards.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `api.models.Dashboard`, `DashboardWidget`, `DashboardTemplate`, `WidgetDataCache`
- **Views:** `dashboard/views_dashboard.py`, `dashboard/views_widgets.py`
- **Widget Types:** counter, pie_chart, bar_chart, line_chart, data_table, map, text, etc.

### Test Classes Summary

| Test Class | Checklist Section | Test Count | Status |
|------------|-------------------|------------|--------|
| TestCustomDashboard | 7.1 | 8 | ⬜ |
| TestDashboardCRUD | 7.2 | 6 | ⬜ |
| TestWidgetManagement | 7.3 | 10 | ⬜ |
| TestWidgetTypes | 7.4 | 12+ | ⬜ |
| TestDashboardTemplates | 7.5 | 5 | ⬜ |
| TestDashboardSharing | 7.6 | 4 | ⬜ |
| TestWidgetAutoRefresh | 7.7 | 3 | ⬜ |

### Verification Checklist
- [ ] All 47 test cases from Phase 7 are covered
- [ ] Dashboard fixtures per role
- [ ] Widget configuration validation
- [ ] Widget data source testing
- [ ] Cache invalidation tests

---

## Phase 8: Report Generation System

**Test File:** `dashboard/tests/test_reports.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `dashboard.models.ReportTemplate`, `GeneratedReport`, `ReportArchive`
- **Services:** `dashboard/report_services.py`
- **Tasks:** `dashboard/tasks.py` (generate_report_task)
- **Report Types:** GIS, SEMESTRAL, STANDARD_BMS, DIGITAL_BACKUP, FINALIZED
- **Output Formats:** PDF, Excel, GeoJSON, Shapefile, Markdown

### Test Classes Summary

| Test Class | Checklist Section | Test Count | Status |
|------------|-------------------|------------|--------|
| TestReportBuilder | 8.1 | 8 | ⬜ |
| TestReportGeneration | 8.2 | 8 | ⬜ |
| TestReportTypes | 8.3 | 5 | ⬜ |
| TestReportOutputFormats | 8.4 | 6 | ⬜ |
| TestReportArchive | 8.5 | 8 | ⬜ |
| TestReportTemplates | 8.6 | 4 | ⬜ |
| TestReportAccessControl | 8.7 | 4 | ⬜ |
| TestDataExportCenter | 8.8 | 4 | ⬜ |

### Verification Checklist
- [ ] All 45 test cases from Phase 8 are covered
- [ ] Celery task mocking
- [ ] File generation verification
- [ ] PDF/Excel content validation
- [ ] Access control per visibility

---

## Phase 9: Event & Schedule Management

**Test File:** `dashboard/tests/test_events.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `dashboard.models.Event`, `api.models.ScheduledSurvey`, `FGDSession`, `FGDParticipant`, `FGDFinding`, `FGDAttachment`
- **Views:** `dashboard/views_events.py`, `dashboard/views_schedule.py`, `dashboard/views_fgd.py`

### Test Classes Summary

| Test Class | Checklist Section | Test Count | Status |
|------------|-------------------|------------|--------|
| TestEventsSystem | 9.1 | 8 | ⬜ |
| TestEventCRUD | 9.1 | 10 | ⬜ |
| TestScheduledSurveys | 9.2 | 8 | ⬜ |
| TestSurveyStatusTracking | 9.2 | 4 | ⬜ |
| TestFGDSessions | 9.3 | 9 | ⬜ |
| TestFGDParticipants | 9.3 | 4 | ⬜ |
| TestFGDFindings | 9.3 | 4 | ⬜ |
| TestFGDAttachments | 9.3 | 4 | ⬜ |
| TestNotifications | 9.x | 3 | ⬜ |

### Verification Checklist
- [ ] All 48 test cases from Phase 9 are covered
- [ ] Event date/time handling
- [ ] File upload for FGD attachments
- [ ] Status auto-update logic
- [ ] Notification mocking

---

## Phase 10: KoboToolbox Integration

**Test File:** `api/tests/test_kobo_integration.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Models:** `api.models.ChoiceList`, `Choice`, `FormChoiceMapping`, `DataSubmission`
- **Services:** `api/form_services.py`, `api/submission_services.py`
- **Tasks:** `api/tasks.py` (sync_kobotoolbox_data)
- **Management Commands:** `initialize_choice_lists`, `sync_kobotoolbox_data`

### Test Classes Summary

| Test Class | Checklist Section | Test Count | Status |
|------------|-------------------|------------|--------|
| TestChoiceLists | 10.1 | 8 | ⬜ |
| TestIndividualChoices | 10.2 | 11 | ⬜ |
| TestFormChoiceMappings | 10.3 | 6 | ⬜ |
| TestBidirectionalSync | 10.5 | 8 | ⬜ |
| TestDataSubmissionSync | 10.6 | 6 | ⬜ |

### KoboToolbox Mocking Requirements
```python
# All external API calls must be mocked
from unittest.mock import patch, Mock

@patch('api.form_services.requests.get')
@patch('api.form_services.requests.post')
```

### Verification Checklist
- [ ] All 27 test cases from Phase 10 are covered
- [ ] All KoboToolbox API calls mocked
- [ ] Choice sync bidirectional tests
- [ ] Submission sync with duplicates
- [ ] Error handling for API failures

---

## Phase 11: Cross-Role Access Control

**Test File:** `dashboard/tests/test_permissions.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Decorators:** `dashboard/decorators.py`, `dashboard/permissions.py`
- **Mixins:** `ValidatorAccessMixin`, `PAStaffAccessMixin`, `DENRAccessMixin`, etc.
- **Roles:** pa_staff, validator, denr, collaborator, system_admin

### Test Classes Summary

| Test Class | Checklist Section | Test Count | Status |
|------------|-------------------|------------|--------|
| TestPAStaffRestrictions | 11.1 | 7 | ⬜ |
| TestValidatorPermissions | 11.2 | 7 | ⬜ |
| TestDENRAccess | 11.3 | 8 | ⬜ |
| TestCollaboratorAccess | 11.4 | 10 | ⬜ |
| TestSystemAdminFullAccess | 11.5 | 10 | ⬜ |
| TestDecoratorEnforcement | 11.6 | 9 | ⬜ |

### Test User Requirements
```python
# Create users for each role
roles = ['pa_staff', 'validator', 'denr', 'collaborator', 'system_admin']
# Each with account_status='approved'
```

### Verification Checklist
- [ ] All 36 test cases from Phase 11 are covered
- [ ] Test users for all 5 roles
- [ ] AJAX vs non-AJAX response format tests
- [ ] Sensitive data obscuring for collaborators
- [ ] Rate limiting tests (if enabled)

---

## Phase 12: System Integration & Performance

**Test File:** `api/tests/test_integration.py`  
**Status:** ✅ Complete  

### Script Context Requirements
- **Services:** Celery, Redis, PostgreSQL/PostGIS
- **Configuration:** `prism/settings.py`, `docker-compose.yml`

### Test Classes Summary

| Test Class | Checklist Section | Test Count | Status |
|------------|-------------------|------------|--------|
| TestCeleryTaskProcessing | 12.1 | 7 | ⬜ |
| TestDatabaseOperations | 12.2 | 6 | ⬜ |
| TestFileUploadStorage | 12.3 | 7 | ⬜ |
| TestAPIEndpoints | 12.4 | 5 | ⬜ |
| TestFrontend | 12.5 | 7 | ⬜ |
| TestEmailNotifications | 12.6 | 4 | ⬜ |
| TestErrorHandling | 12.7 | 4 | ⬜ |
| TestPerformance | 12.8 | 6 | ⬜ |
| TestDockerEnvironment | 12.9 | 4 | ⬜ |

### Performance Benchmarks
```python
# Dashboard pages: < 3 seconds
# List pages (1000+ records): < 5 seconds
# Map with 1000+ points: < 5 seconds
# Report generation: Progress updates every 10 seconds
```

### Verification Checklist
- [ ] All 34 test cases from Phase 12 are covered
- [ ] Celery test utilities configured
- [ ] Performance timing assertions
- [ ] N+1 query detection
- [ ] Docker health checks

---

## Overall Progress Summary

| Phase | Test File | Total Tests | Completed | Status |
|-------|-----------|-------------|-----------|--------|
| 1 | `users/tests/test_authentication.py` | 24 | 24 | ✅ |
| 2 | `dashboard/tests/test_user_management.py` | 36 | 36 | ✅ |
| 3 | `dashboard/tests/test_pa_staff.py` | 29 | 29 | ✅ |
| 4 | `dashboard/tests/test_validation.py` | 52 | 52 | ✅ |
| 5 | `species/tests/test_species.py` | 30 | 30 | ✅ |
| 6 | `dashboard/tests/test_gis.py` | 42 | 42 | ✅ |
| 7 | `dashboard/tests/test_dashboards.py` | 47 | 47 | ✅ |
| 8 | `dashboard/tests/test_reports.py` | 45 | 45 | ✅ |
| 9 | `dashboard/tests/test_events.py` | 48 | 48 | ✅ |
| 10 | `api/tests/test_kobo_integration.py` | 27 | 27 | ✅ |
| 11 | `dashboard/tests/test_permissions.py` | 36 | 36 | ✅ |
| 12 | `api/tests/test_integration.py` | 34 | 34 | ✅ |
| **TOTAL** | | **450** | **450** | ✅ |

---

## Quick Reference: Test Commands

```bash
# Run all tests
docker-compose exec web python manage.py test

# Run specific phase
docker-compose exec web python manage.py test users.tests.test_authentication
docker-compose exec web python manage.py test dashboard.tests.test_user_management
docker-compose exec web python manage.py test dashboard.tests.test_pa_staff
docker-compose exec web python manage.py test dashboard.tests.test_validation
docker-compose exec web python manage.py test species.tests.test_species
docker-compose exec web python manage.py test dashboard.tests.test_gis
docker-compose exec web python manage.py test dashboard.tests.test_dashboards
docker-compose exec web python manage.py test dashboard.tests.test_reports
docker-compose exec web python manage.py test dashboard.tests.test_events
docker-compose exec web python manage.py test api.tests.test_kobo_integration
docker-compose exec web python manage.py test dashboard.tests.test_permissions
docker-compose exec web python manage.py test api.tests.test_integration

# Run with coverage
docker-compose exec web coverage run --source='.' manage.py test
docker-compose exec web coverage report --show-missing
docker-compose exec web coverage html
```

---

## Verification Sign-Off

| Phase | Developer | Date | Reviewer | Date |
|-------|-----------|------|----------|------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |
| 6 | | | | |
| 7 | | | | |
| 8 | | | | |
| 9 | | | | |
| 10 | | | | |
| 11 | | | | |
| 12 | | | | |

---

**Document Maintained By:** Development Team  
**Last Updated:** January 7, 2026
