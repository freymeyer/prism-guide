# PRISM-Matutum Test Scenarios - UTAUT Framework

## 📋 Document Overview

This document provides comprehensive test scenarios for all user roles in PRISM-Matutum, structured around the **UTAUT (Unified Theory of Acceptance and Use of Technology)** framework. Each test scenario evaluates not only functional correctness but also technology acceptance factors.

---

## 🎯 UTAUT Framework Overview

The UTAUT model evaluates technology acceptance through four key constructs:

| Construct | Definition | Measurement in Tests |
|-----------|------------|---------------------|
| **Performance Expectancy (PE)** | The degree to which using the system will help users accomplish tasks better | Task completion time, accuracy, efficiency gains |
| **Effort Expectancy (EE)** | The ease of use associated with the system | Learning curve, error rates, help requests |
| **Social Influence (SI)** | The extent to which users believe others think they should use the system | Collaboration features, role clarity, perceived legitimacy |
| **Facilitating Conditions (FC)** | The degree to which users believe organizational and technical infrastructure exists to support use | System availability, help resources, error handling |

Additionally, we measure:
- **Behavioral Intention (BI)**: Willingness to continue using the system
- **Use Behavior (UB)**: Actual system usage patterns

---

## 🧪 Test Scenario Structure

Each test scenario includes:

1. **Functional Tests** - Does it work correctly?
2. **UTAUT Evaluation** - How well does it support user acceptance?
3. **Success Criteria** - Both functional and acceptance metrics
4. **Issues to Watch** - Common problems that affect adoption

---

## 👤 Role 1: PA Staff Test Scenarios

### Test Scenario PA-01: Initial Login & Dashboard Access

#### Functional Tests

**Test Steps:**
1. Navigate to login page
2. Enter valid PA Staff credentials
3. Submit login form
4. Verify redirect to PA Staff dashboard

**Expected Results:**
- ✅ Login succeeds without errors
- ✅ User redirected to `/dashboard/pa-staff/home/`
- ✅ Dashboard displays personal statistics
- ✅ Navigation menu shows PA Staff options only

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Dashboard loads within 2 seconds
- [ ] Statistics are immediately visible and relevant
- [ ] Clear value proposition displayed (e.g., "Track your submissions")
- [ ] Quick access to primary tasks (view submissions, observations)

**Effort Expectancy (EE):**
- [ ] Login requires only username/password (minimal steps)
- [ ] Dashboard layout is intuitive without training
- [ ] Navigation labels are clear and role-specific
- [ ] No confusing terminology or jargon

**Social Influence (SI):**
- [ ] Role badge clearly displays "PA Staff" status
- [ ] Dashboard shows organizational affiliation (agency/department)
- [ ] Professional appearance builds legitimacy

**Facilitating Conditions (FC):**
- [ ] Error messages are clear if login fails
- [ ] "Forgot Password" option is available
- [ ] System provides feedback on account status (pending/active)

**Acceptance Metrics:**
```python
# Test in dashboard/tests/test_utaut_pa_staff.py
def test_pa_staff_dashboard_load_time():
    """PE: Dashboard should load quickly"""
    start_time = time.time()
    response = self.client.get('/dashboard/pa-staff/home/')
    load_time = time.time() - start_time
    assert load_time < 2.0, f"Dashboard load time: {load_time}s exceeds 2s"
    
def test_pa_staff_dashboard_clarity():
    """EE: Dashboard should be self-explanatory"""
    response = self.client.get('/dashboard/pa-staff/home/')
    content = response.content.decode()
    # Check for clear section headers
    assert "My Submissions" in content
    assert "Recent Observations" in content
    assert "Statistics" in content or "Overview" in content
```

**Issues to Watch:**
- ⚠️ Slow dashboard load (>3s) → Low PE
- ⚠️ Confusing navigation → Low EE
- ⚠️ Missing role indicators → Low SI

---

### Test Scenario PA-02: View My Submissions

#### Functional Tests

**Test Steps:**
1. Login as PA Staff
2. Navigate to "My Submissions"
3. Verify submission list displays
4. Apply filters (date range, status)
5. Search for specific submission
6. Click on a submission to view details

**Expected Results:**
- ✅ All user's submissions are listed
- ✅ Pagination works for large lists
- ✅ Filters reduce results correctly
- ✅ Search finds submissions by ID or content
- ✅ Status badges display correctly (Pending, Approved, Rejected)

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Users can quickly find their submissions
- [ ] Status information is immediately clear
- [ ] Validation feedback is visible and actionable
- [ ] Export/download options available
- [ ] Time saved vs. paper tracking is evident

**Effort Expectancy (EE):**
- [ ] Filters are easy to use without documentation
- [ ] Search works intuitively (by any field)
- [ ] No more than 3 clicks to find a submission
- [ ] Status meanings are clear without explanation
- [ ] Mobile-responsive design for field use

**Social Influence (SI):**
- [ ] Shows validator names (builds trust in process)
- [ ] Displays validation timestamps (transparency)
- [ ] Remarks from validators are visible (collaboration)

**Facilitating Conditions (FC):**
- [ ] Empty state message if no submissions exist
- [ ] Help text explains each status
- [ ] Error handling for network issues
- [ ] Offline access to cached data (future enhancement)

**Acceptance Metrics:**
```python
def test_submission_search_efficiency():
    """PE: Users should find submissions quickly"""
    # Create 50 test submissions
    submissions = [create_submission(user=self.pa_staff) for _ in range(50)]
    
    # Time how long it takes to filter
    start = time.time()
    response = self.client.get('/dashboard/my-submissions/', {
        'status': 'pending',
        'date_from': '2026-01-01'
    })
    search_time = time.time() - start
    
    assert search_time < 1.0, "Filter/search should be fast"
    assert response.status_code == 200
    
def test_submission_list_clarity():
    """EE: Submission list should be easy to scan"""
    response = self.client.get('/dashboard/my-submissions/')
    content = response.content.decode()
    
    # Check for visual clarity elements
    assert 'badge' in content.lower()  # Status badges
    assert 'table' in content or 'card' in content  # Structured layout
    assert 'data-table' in content  # Sortable table
```

**Issues to Watch:**
- ⚠️ Slow page load with many submissions → Low PE
- ⚠️ Unclear status indicators → Low EE
- ⚠️ No validator feedback visible → Low SI

---

### Test Scenario PA-03: View Submission Details & Validation Feedback

#### Functional Tests

**Test Steps:**
1. Navigate to "My Submissions"
2. Click on a validated submission (approved or rejected)
3. View submission details
4. Read validator remarks
5. Check validation history/audit trail

**Expected Results:**
- ✅ All submission data displays correctly
- ✅ Validator remarks are visible
- ✅ Validation status is clear
- ✅ Timestamp of validation shown
- ✅ Can access original data from Kobo

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Validation feedback helps improve future submissions
- [ ] Actionable guidance provided for rejections
- [ ] Approval confirmation provides closure
- [ ] Learning from feedback reduces rejection rate over time

**Effort Expectancy (EE):**
- [ ] Feedback is in plain language (not technical jargon)
- [ ] Rejection reasons are specific, not generic
- [ ] Contact validator option available if unclear
- [ ] Examples provided for correct data entry

**Social Influence (SI):**
- [ ] Validator name and role displayed (accountability)
- [ ] Feedback tone is professional and constructive
- [ ] Validation process feels fair and transparent
- [ ] Peer submission quality visible (benchmarking)

**Facilitating Conditions (FC):**
- [ ] Help documentation linked from rejection reasons
- [ ] FAQ addresses common issues
- [ ] Training resources available
- [ ] Support contact information visible

**Acceptance Metrics:**
```python
def test_feedback_quality():
    """PE/SI: Feedback should be helpful and professional"""
    submission = create_submission(
        user=self.pa_staff,
        validation_status='rejected',
        validation_remarks='Species identification unclear. Please refer to field guide page 42.'
    )
    
    response = self.client.get(f'/dashboard/submission/{submission.id}/')
    content = response.content.decode()
    
    # Check feedback is visible and detailed
    assert submission.validation_remarks in content
    assert len(submission.validation_remarks) > 20  # Not generic
    
def test_feedback_accessibility():
    """EE: Feedback should be easy to understand"""
    response = self.client.get('/dashboard/submission/1/')
    # Check for clear formatting, no technical jargon
    # This would involve content analysis or user testing
```

**Issues to Watch:**
- ⚠️ Generic/unhelpful feedback → Low PE & SI
- ⚠️ Technical jargon in remarks → Low EE
- ⚠️ No follow-up mechanism → Low FC

---

### Test Scenario PA-04: Access Restrictions Test

#### Functional Tests

**Test Steps:**
1. Login as PA Staff
2. Attempt to access validation queue URL directly
3. Attempt to access admin dashboard URL
4. Attempt to access other users' submission details

**Expected Results:**
- ✅ Validation queue returns 403 Forbidden
- ✅ Admin dashboard returns 403 Forbidden
- ✅ Other users' submissions return 403 or redirect
- ✅ Error page is user-friendly, not technical

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Security protects data integrity
- [ ] Users trust their data is protected
- [ ] Role clarity prevents confusion

**Effort Expectancy (EE):**
- [ ] Error messages explain what went wrong
- [ ] Redirect back to appropriate page
- [ ] No confusing technical errors

**Social Influence (SI):**
- [ ] Role-based access feels legitimate
- [ ] Security builds trust in system

**Facilitating Conditions (FC):**
- [ ] Error pages provide navigation options
- [ ] Help text explains permissions
- [ ] Contact admin option available

**Acceptance Metrics:**
```python
def test_permission_denial_ux():
    """EE/FC: Permission errors should be user-friendly"""
    response = self.client.get('/dashboard/validation/queue/')
    
    assert response.status_code == 403
    content = response.content.decode()
    
    # Should have user-friendly message
    assert "permission" in content.lower() or "access" in content.lower()
    assert "contact" in content.lower() or "admin" in content.lower()
    # Should NOT have technical stack traces
    assert "Traceback" not in content
```

---

### PA Staff - Complete Test Checklist

#### Functional Tests
- [ ] PA-01: Login and dashboard access
- [ ] PA-02: View submissions list with filters
- [ ] PA-03: View submission details and feedback
- [ ] PA-04: Access restrictions enforced
- [ ] PA-05: View my observations (wildlife, resource use, etc.)
- [ ] PA-06: Profile management
- [ ] PA-07: Password change
- [ ] PA-08: Notification viewing

#### UTAUT Acceptance Tests

**Performance Expectancy Tests:**
- [ ] Dashboard loads in <2 seconds
- [ ] Submission tracking saves time vs. manual methods
- [ ] Feedback helps improve data quality
- [ ] Statistics provide useful insights

**Effort Expectancy Tests:**
- [ ] First-time use without training possible
- [ ] Navigation is intuitive
- [ ] Search and filters work as expected
- [ ] Error messages are clear

**Social Influence Tests:**
- [ ] Role is clearly communicated
- [ ] Validation process feels fair
- [ ] Professional appearance
- [ ] Collaboration features work

**Facilitating Conditions Tests:**
- [ ] Help resources available
- [ ] Support contacts visible
- [ ] System availability high (>99%)
- [ ] Error recovery is smooth

---

## ✅ Role 2: Validator Test Scenarios

### Test Scenario VAL-01: Validation Queue Access & Overview

#### Functional Tests

**Test Steps:**
1. Login as Validator
2. Navigate to Validation Queue
3. View pending submissions
4. Apply filters (date, PA staff, priority)
5. Sort by different columns
6. Check queue statistics

**Expected Results:**
- ✅ Queue displays all pending submissions
- ✅ Filters reduce list correctly
- ✅ Sorting works on all columns
- ✅ Statistics show workload (pending count, assigned to me, etc.)
- ✅ Pagination works for large queues

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Queue helps prioritize work efficiently
- [ ] Statistics show productivity (validations per day)
- [ ] Bulk actions save time (assign multiple)
- [ ] Clear workload visibility prevents bottlenecks
- [ ] Faster than paper-based validation

**Effort Expectancy (EE):**
- [ ] Queue is scannable at a glance
- [ ] Filters are intuitive
- [ ] Status indicators are clear
- [ ] Workload is not overwhelming (good UX design)
- [ ] Can start validating with minimal clicks

**Social Influence (SI):**
- [ ] Shows other validators' assignments (teamwork)
- [ ] Performance metrics foster healthy competition
- [ ] Quality standards are clear
- [ ] Role authority is evident

**Facilitating Conditions (FC):**
- [ ] Queue never "breaks" (robust error handling)
- [ ] Empty queue state is encouraging, not blank
- [ ] Help text explains priority levels
- [ ] Training materials linked

**Acceptance Metrics:**
```python
def test_validation_queue_efficiency():
    """PE: Queue should help validators work faster"""
    # Create 20 pending submissions
    submissions = [create_pending_submission() for _ in range(20)]
    
    start = time.time()
    response = self.client.get('/dashboard/validation/queue/')
    load_time = time.time() - start
    
    assert load_time < 2.0, "Queue should load quickly"
    
    # Check for efficiency features
    content = response.content.decode()
    assert 'bulk-select' in content or 'checkbox' in content
    assert 'filter' in content.lower()
    
def test_queue_clarity():
    """EE: Queue should be easy to scan"""
    response = self.client.get('/dashboard/validation/queue/')
    content = response.content.decode()
    
    # Check for visual hierarchy
    assert 'priority' in content.lower()
    assert 'date' in content.lower()
    assert 'status' in content.lower()
    # Check for status badges (visual clarity)
    assert 'badge' in content
```

**Issues to Watch:**
- ⚠️ Queue overload (>100 pending) → Low PE/EE
- ⚠️ Slow filtering → Low PE
- ⚠️ No priority indicators → Low EE

---

### Test Scenario VAL-02: Validation Workspace - Approve Submission

#### Functional Tests

**Test Steps:**
1. From validation queue, click on a pending submission
2. Review all submission data (observations, location, media)
3. Check data quality and completeness
4. Add validation remarks (optional)
5. Click "Approve" button
6. Verify success message
7. Check submission status updated to "Approved"

**Expected Results:**
- ✅ All submission data is clearly displayed
- ✅ Map shows location accurately
- ✅ Media (photos) are viewable
- ✅ Approve button is prominent
- ✅ Success feedback shown immediately
- ✅ Status updates in database and UI

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] All information needed for decision is visible
- [ ] Validation decision takes <2 minutes for good submissions
- [ ] Immediate feedback confirms action
- [ ] Approved submissions move to next workflow stage
- [ ] Productivity metrics improve over time

**Effort Expectancy (EE):**
- [ ] Layout is logical (data → action → feedback)
- [ ] No scrolling required to see key info
- [ ] Approve button is easy to find
- [ ] Confirmation prevents accidental clicks
- [ ] Can use keyboard shortcuts (future)

**Social Influence (SI):**
- [ ] PA Staff info displayed (know who submitted)
- [ ] Previous validation history visible (consistency)
- [ ] Quality standards referenced in UI
- [ ] Approval feels official and legitimate

**Facilitating Conditions (FC):**
- [ ] Validation guidelines accessible from workspace
- [ ] Field guide/reference materials linked
- [ ] Undo option available (within timeframe)
- [ ] Technical issues don't block approval

**Acceptance Metrics:**
```python
def test_validation_workspace_efficiency():
    """PE: Validation should be quick for good data"""
    submission = create_pending_submission()
    
    # Time to load workspace
    start = time.time()
    response = self.client.get(f'/dashboard/validation/workspace/{submission.id}/')
    load_time = time.time() - start
    
    assert load_time < 1.5, "Workspace should load very quickly"
    
    # Time to approve
    start = time.time()
    response = self.client.post('/dashboard/api/validate/', {
        'submission_id': submission.id,
        'action': 'approve',
        'remarks': 'Data quality excellent'
    })
    approve_time = time.time() - start
    
    assert approve_time < 1.0, "Approval should be instant"
    assert response.json()['status'] == 'success'
    
def test_workspace_information_completeness():
    """EE: All needed info should be visible"""
    submission = create_pending_submission()
    response = self.client.get(f'/dashboard/validation/workspace/{submission.id}/')
    content = response.content.decode()
    
    # Check for essential information
    assert 'species' in content.lower() or 'observation' in content.lower()
    assert 'location' in content.lower() or 'coordinates' in content.lower()
    assert 'date' in content.lower()
    assert 'submitted by' in content.lower() or 'pa staff' in content.lower()
    # Check for action buttons
    assert 'approve' in content.lower()
    assert 'reject' in content.lower()
```

**Issues to Watch:**
- ⚠️ Missing key information → Low PE
- ⚠️ Confusing layout → Low EE
- ⚠️ Slow approval process → Low PE

---

### Test Scenario VAL-03: Validation Workspace - Reject Submission

#### Functional Tests

**Test Steps:**
1. Open validation workspace for a problematic submission
2. Identify data quality issues
3. Add detailed validation remarks explaining rejection
4. Click "Reject" button
5. Verify success message
6. Verify PA Staff receives notification (if applicable)

**Expected Results:**
- ✅ Remarks field is required for rejection
- ✅ Rejection saves successfully
- ✅ Status updates to "Rejected"
- ✅ Remarks are stored and visible to PA Staff
- ✅ Notification sent to submitter

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Rejection process is as easy as approval
- [ ] Detailed feedback improves future submissions
- [ ] System tracks common rejection reasons (analytics)
- [ ] Feedback loop improves overall data quality

**Effort Expectancy (EE):**
- [ ] Remarks field has helpful placeholder text
- [ ] Common rejection reasons available as templates
- [ ] No complex forms to fill out
- [ ] Can reject and move to next submission quickly

**Social Influence (SI):**
- [ ] Rejection feels constructive, not punitive
- [ ] Professional language in remarks
- [ ] PA Staff respect feedback (measured by re-submission quality)
- [ ] Validators feel supported by system in making decisions

**Facilitating Conditions (FC):**
- [ ] Validation guidelines specify rejection criteria
- [ ] Examples of good vs. bad submissions available
- [ ] Can consult with other validators if unsure
- [ ] Technical support for edge cases

**Acceptance Metrics:**
```python
def test_rejection_feedback_quality():
    """PE/SI: Rejection feedback should be constructive"""
    submission = create_pending_submission()
    
    # Reject with detailed remarks
    response = self.client.post('/dashboard/api/validate/', {
        'submission_id': submission.id,
        'action': 'reject',
        'remarks': 'Species identification is unclear. The photo does not show diagnostic features. Please refer to field guide page 42 and retake photos showing the dorsal view.'
    })
    
    assert response.json()['status'] == 'success'
    
    # Verify remarks are stored
    submission.refresh_from_db()
    assert submission.validation_status == 'rejected'
    assert len(submission.validation_remarks) > 50  # Detailed, not generic
    
def test_rejection_requires_remarks():
    """FC: System should prevent rejection without feedback"""
    submission = create_pending_submission()
    
    # Attempt to reject without remarks
    response = self.client.post('/dashboard/api/validate/', {
        'submission_id': submission.id,
        'action': 'reject',
        'remarks': ''
    })
    
    # Should fail or require remarks
    assert 'remarks' in response.json().get('errors', {}) or response.json()['status'] == 'error'
```

**Issues to Watch:**
- ⚠️ Generic rejection reasons → Low PE/SI
- ⚠️ No template options → Low EE
- ⚠️ PA Staff not receiving feedback → Low FC

---

### Test Scenario VAL-04: Validation History & Performance Metrics

#### Functional Tests

**Test Steps:**
1. Navigate to "Validation History"
2. View list of all validations performed
3. Filter by date, status, PA staff
4. View performance metrics (approval rate, avg time per validation)
5. Export validation history

**Expected Results:**
- ✅ All past validations are listed
- ✅ Filters work correctly
- ✅ Metrics display accurately
- ✅ Export downloads successfully

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Metrics help track personal productivity
- [ ] Approval rate shows quality consistency
- [ ] Historical data aids in training new validators
- [ ] Benchmarking against peers (optional)

**Effort Expectancy (EE):**
- [ ] Metrics are auto-calculated (no manual tracking)
- [ ] Visualizations (charts) make data easy to understand
- [ ] Export is one-click

**Social Influence (SI):**
- [ ] Performance transparency builds accountability
- [ ] Positive metrics boost morale
- [ ] Team metrics foster collaboration

**Facilitating Conditions (FC):**
- [ ] Metrics help identify training needs
- [ ] Historical reference for difficult cases
- [ ] System maintains complete audit trail

**Acceptance Metrics:**
```python
def test_validation_metrics_accuracy():
    """PE: Metrics should accurately reflect performance"""
    validator = self.validator_user
    
    # Create validation history
    for i in range(10):
        submission = create_pending_submission()
        submission.validate(
            validator=validator,
            action='approve' if i < 8 else 'reject',
            remarks='Test validation'
        )
    
    response = self.client.get('/dashboard/validation/history/')
    content = response.content.decode()
    
    # Check for metrics display
    assert '80' in content or '8/10' in content  # 80% approval rate
    assert 'approval rate' in content.lower()
    
def test_history_export_functionality():
    """EE: Export should be simple and fast"""
    response = self.client.get('/dashboard/validation/history/export/')
    
    assert response.status_code == 200
    assert 'application/vnd.ms-excel' in response['Content-Type'] or \
           'text/csv' in response['Content-Type']
```

---

### Validator - Complete Test Checklist

#### Functional Tests
- [ ] VAL-01: Access validation queue
- [ ] VAL-02: Approve submission
- [ ] VAL-03: Reject submission with remarks
- [ ] VAL-04: View validation history
- [ ] VAL-05: Bulk assign submissions
- [ ] VAL-06: Filter and sort queue
- [ ] VAL-07: View submission audit trail
- [ ] VAL-08: Access validator dashboard

#### UTAUT Acceptance Tests

**Performance Expectancy Tests:**
- [ ] Validation process is faster than paper-based
- [ ] Queue management improves workflow
- [ ] Metrics track productivity
- [ ] Feedback loop improves data quality

**Effort Expectancy Tests:**
- [ ] Workspace is intuitive
- [ ] All info is visible without scrolling excessively
- [ ] Actions (approve/reject) are simple
- [ ] Bulk operations save time

**Social Influence Tests:**
- [ ] Role authority is clear
- [ ] Feedback is constructive
- [ ] Team collaboration is supported
- [ ] Performance metrics are fair

**Facilitating Conditions Tests:**
- [ ] Guidelines are accessible
- [ ] Technical issues don't block work
- [ ] Training resources available
- [ ] Undo/correction mechanisms exist

---

## 🔧 Role 3: System Admin Test Scenarios

### Test Scenario ADM-01: User Approval Workflow

#### Functional Tests

**Test Steps:**
1. Login as System Admin
2. Navigate to "User Management" → "Pending Users"
3. Review a pending user account
4. Verify user details (name, email, organization, role requested)
5. Approve the user
6. Verify user can now login
7. Reject a different user with reason
8. Verify rejection notification sent

**Expected Results:**
- ✅ Pending users list displays all unapproved accounts
- ✅ Approval updates account_status to "approved"
- ✅ Approved user receives email notification
- ✅ Rejection stores reason
- ✅ Rejected user receives notification

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Approval process takes <1 minute per user
- [ ] Bulk approval option available (for multiple users)
- [ ] Automated checks reduce manual verification
- [ ] System prevents duplicate accounts

**Effort Expectancy (EE):**
- [ ] User details are clearly displayed
- [ ] Approve/Reject buttons are prominent
- [ ] Decision criteria provided
- [ ] Rejection reason templates available

**Social Influence (SI):**
- [ ] Admin role authority is clear
- [ ] Decision process feels fair and transparent
- [ ] Organizational hierarchy respected

**Facilitating Conditions (FC):**
- [ ] User approval guidelines documented
- [ ] Can verify organization affiliations
- [ ] Support for unusual cases
- [ ] Audit trail of all approval decisions

**Acceptance Metrics:**
```python
def test_user_approval_efficiency():
    """PE: User approval should be quick"""
    pending_user = create_user(account_status='pending', role='pa_staff')
    
    start = time.time()
    response = self.client.post(f'/dashboard/admin/users/{pending_user.id}/approve/', {
        'action': 'approve'
    })
    approval_time = time.time() - start
    
    assert approval_time < 1.0, "Approval should be near-instant"
    assert response.status_code == 302  # Redirect after success
    
    pending_user.refresh_from_db()
    assert pending_user.account_status == 'approved'
    
def test_approval_workflow_clarity():
    """EE: Approval interface should be clear"""
    pending_user = create_user(account_status='pending', role='validator')
    response = self.client.get(f'/dashboard/admin/users/{pending_user.id}/')
    content = response.content.decode()
    
    # Check for clear information display
    assert pending_user.email in content
    assert pending_user.get_role_display() in content
    assert 'approve' in content.lower()
    assert 'reject' in content.lower()
```

---

### Test Scenario ADM-02: Organization Management

#### Functional Tests

**Test Steps:**
1. Navigate to "User Organization Hub"
2. Create a new Agency (e.g., "DENR Region 12")
3. Create a Department under that agency
4. Assign users to the department
5. Edit agency details
6. Delete a department (with confirmation)

**Expected Results:**
- ✅ Agency created successfully
- ✅ Department hierarchy maintained
- ✅ Users can be assigned/reassigned
- ✅ Edit operations persist
- ✅ Delete confirmation prevents accidents

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Organizational structure mirrors real-world entities
- [ ] User management is easier with organization grouping
- [ ] Reports can be filtered by organization
- [ ] Hierarchy aids in permission management

**Effort Expectancy (EE):**
- [ ] Forms are simple (minimal required fields)
- [ ] Hierarchy is visually clear (tree view or indentation)
- [ ] Search helps find organizations quickly
- [ ] Bulk user assignment available

**Social Influence (SI):**
- [ ] Official organizational structure increases legitimacy
- [ ] Departmental grouping fosters team identity
- [ ] Clear chain of command

**Facilitating Conditions (FC):**
- [ ] Import option for bulk organization setup
- [ ] Templates for common organizational types
- [ ] Help text explains agency types
- [ ] System prevents orphaned users

**Acceptance Metrics:**
```python
def test_organization_crud_efficiency():
    """PE/EE: Organization management should be straightforward"""
    # Create agency
    response = self.client.post('/dashboard/admin/agencies/create/', {
        'name': 'DENR Region 12',
        'agency_type': 'government',
        'description': 'Department of Environment and Natural Resources'
    })
    assert response.status_code == 302
    
    agency = Agency.objects.get(name='DENR Region 12')
    
    # Create department
    response = self.client.post('/dashboard/admin/departments/create/', {
        'name': 'Biodiversity Management Bureau',
        'agency': agency.id
    })
    assert response.status_code == 302
    
    department = Department.objects.get(name='Biodiversity Management Bureau')
    assert department.agency == agency
```

---

### System Admin - Complete Test Checklist

#### Functional Tests
- [ ] ADM-01: Approve/reject users
- [ ] ADM-02: Manage agencies and departments
- [ ] ADM-03: Edit user roles
- [ ] ADM-04: Manage choice lists
- [ ] ADM-05: Sync with KoboToolbox
- [ ] ADM-06: View system health dashboard
- [ ] ADM-07: Manage geographic data (regions, provinces, etc.)
- [ ] ADM-08: Access all system features

#### UTAUT Acceptance Tests
- [ ] Admin tasks are efficient (not time-consuming)
- [ ] Interface is clear despite complexity
- [ ] Role authority is evident
- [ ] Comprehensive help resources available

---

## 📊 Role 4: DENR Personnel Test Scenarios

### Test Scenario DENR-01: Dashboard & Report Access

#### Functional Tests

**Test Steps:**
1. Login as DENR user
2. View DENR dashboard
3. Navigate to Report Archive
4. Download a semestral BMS report
5. Attempt to access validation queue (should fail)
6. Attempt to access admin functions (should fail)

**Expected Results:**
- ✅ DENR dashboard shows overview statistics
- ✅ Report archive is accessible
- ✅ Reports download successfully
- ✅ Validation queue returns 403 Forbidden
- ✅ Admin functions return 403 Forbidden

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Quick access to reports (primary need)
- [ ] Reports are comprehensive and useful
- [ ] Dashboard provides at-a-glance insights
- [ ] Easier than requesting reports via email

**Effort Expectancy (EE):**
- [ ] Login and navigation are simple
- [ ] Report download is one-click
- [ ] No unnecessary features cluttering interface
- [ ] Clear labeling of report types

**Social Influence (SI):**
- [ ] DENR role is clearly indicated
- [ ] Read-only access feels appropriate for oversight role
- [ ] Professional presentation suitable for government use

**Facilitating Conditions (FC):**
- [ ] Reports are always available (no "pending generation")
- [ ] Archive is organized chronologically
- [ ] Help documentation for report interpretation
- [ ] Technical support contact available

---

## 👥 Role 5: Collaborator Test Scenarios

### Test Scenario COL-01: Limited Access Verification

#### Functional Tests

**Test Steps:**
1. Login as Collaborator
2. View available reports
3. Attempt to access raw data (should be limited)
4. Attempt to validate submissions (should fail)
5. Attempt to edit species data (should fail depending on permissions)

**Expected Results:**
- ✅ Collaborator dashboard shows approved content
- ✅ Report downloads work
- ✅ Data editing returns 403 if not permitted
- ✅ Validation queue returns 403

#### UTAUT Evaluation

**Performance Expectancy (PE):**
- [ ] Collaborators can access needed research data
- [ ] Easier than formal data requests
- [ ] Download formats support research (CSV, Excel, Shapefiles)

**Effort Expectancy (EE):**
- [ ] Interface is not confusing despite limited access
- [ ] Clear what is and isn't accessible
- [ ] No frustrating permission errors

**Social Influence (SI):**
- [ ] External researcher role is respected
- [ ] Data sharing builds research partnerships
- [ ] Appropriate restrictions maintain data integrity

---

## 📈 UTAUT Measurement Framework

### Quantitative Metrics

| Metric | Measurement Method | Target |
|--------|-------------------|--------|
| **Task Completion Time** | Time from login to task completion | <3 min for common tasks |
| **Error Rate** | User errors per session | <5% |
| **System Uptime** | Availability monitoring | >99% |
| **Page Load Time** | Browser performance API | <2s for 90th percentile |
| **Search Efficiency** | Keystrokes to find item | <10 interactions |

### Qualitative Metrics

Post-implementation user surveys (7-point Likert scale):

**Performance Expectancy:**
- "Using PRISM-Matutum improves my work performance"
- "Using PRISM-Matutum saves me time"
- "Using PRISM-Matutum helps me accomplish tasks more quickly"

**Effort Expectancy:**
- "Learning to use PRISM-Matutum is easy"
- "I find PRISM-Matutum easy to use"
- "It is easy to become skillful at using PRISM-Matutum"

**Social Influence:**
- "People whose opinions I value use PRISM-Matutum"
- "Management supports the use of PRISM-Matutum"
- "The organization encourages use of PRISM-Matutum"

**Facilitating Conditions:**
- "I have the resources necessary to use PRISM-Matutum"
- "Technical support is available when I encounter difficulties"
- "PRISM-Matutum is compatible with our current systems"

**Behavioral Intention:**
- "I intend to continue using PRISM-Matutum"
- "I would recommend PRISM-Matutum to colleagues"

---

## 🔍 Testing Implementation

### Automated UTAUT Tests

```python
# dashboard/tests/test_utaut_framework.py

from django.test import TestCase, Client
from django.contrib.auth import get_user_model
from django.utils import timezone
import time

User = get_user_model()

class UTAUTPerformanceExpectancyTests(TestCase):
    """Tests for Performance Expectancy construct"""
    
    def setUp(self):
        self.client = Client()
        self.pa_staff = User.objects.create_user(
            username='pa_staff_test',
            password='testpass123',
            role='pa_staff',
            account_status='approved'
        )
        
    def test_dashboard_load_time_pe(self):
        """PE: Dashboard should load quickly to be useful"""
        self.client.login(username='pa_staff_test', password='testpass123')
        
        start = time.time()
        response = self.client.get('/dashboard/pa-staff/home/')
        load_time = time.time() - start
        
        self.assertLess(load_time, 2.0, 
            f"Dashboard load time {load_time:.2f}s exceeds 2s target")
        self.assertEqual(response.status_code, 200)
        
    def test_task_completion_efficiency_pe(self):
        """PE: Common tasks should be fast"""
        self.client.login(username='pa_staff_test', password='testpass123')
        
        # Scenario: PA Staff checks submission status
        start = time.time()
        
        # Step 1: Navigate to submissions
        self.client.get('/dashboard/my-submissions/')
        
        # Step 2: Search for submission
        self.client.get('/dashboard/my-submissions/?search=TEST-001')
        
        # Step 3: View details
        self.client.get('/dashboard/submission/1/')
        
        total_time = time.time() - start
        
        self.assertLess(total_time, 3.0,
            f"Task completion time {total_time:.2f}s exceeds 3s target")


class UTAUTEffortExpectancyTests(TestCase):
    """Tests for Effort Expectancy construct"""
    
    def test_login_simplicity_ee(self):
        """EE: Login should require minimal steps"""
        response = self.client.get('/login/')
        self.assertEqual(response.status_code, 200)
        
        # Check form has only essential fields
        content = response.content.decode()
        self.assertIn('username', content.lower())
        self.assertIn('password', content.lower())
        # Should NOT have excessive fields
        self.assertNotIn('captcha', content.lower())
        
    def test_error_message_clarity_ee(self):
        """EE: Error messages should be user-friendly"""
        response = self.client.post('/login/', {
            'username': 'nonexistent',
            'password': 'wrong'
        })
        
        content = response.content.decode()
        # Should have clear message, not technical jargon
        self.assertTrue(
            'incorrect' in content.lower() or 
            'invalid' in content.lower() or
            'check' in content.lower()
        )
        # Should NOT have technical details
        self.assertNotIn('Traceback', content)
        self.assertNotIn('Exception', content)


class UTAUTSocialInfluenceTests(TestCase):
    """Tests for Social Influence construct"""
    
    def test_role_clarity_si(self):
        """SI: User role should be clearly communicated"""
        self.client.login(username='validator_test', password='testpass123')
        response = self.client.get('/dashboard/')
        
        content = response.content.decode()
        # Role should be visible
        self.assertTrue(
            'validator' in content.lower() or
            'role' in content.lower()
        )
        
    def test_professional_appearance_si(self):
        """SI: System should look legitimate and professional"""
        response = self.client.get('/dashboard/')
        content = response.content.decode()
        
        # Check for professional elements
        self.assertIn('PRISM', content)  # System branding
        # Check CSS is loaded (not a blank page)
        self.assertIn('.css', content)


class UTAUTFacilitatingConditionsTests(TestCase):
    """Tests for Facilitating Conditions construct"""
    
    def test_help_resources_availability_fc(self):
        """FC: Help resources should be accessible"""
        response = self.client.get('/dashboard/')
        content = response.content.decode()
        
        # Check for help links
        self.assertTrue(
            'help' in content.lower() or
            'support' in content.lower() or
            'guide' in content.lower()
        )
        
    def test_error_recovery_fc(self):
        """FC: System should handle errors gracefully"""
        # Attempt invalid operation
        response = self.client.get('/dashboard/submission/999999/')
        
        # Should return 404, not 500
        self.assertEqual(response.status_code, 404)
        
        # Should have user-friendly 404 page
        content = response.content.decode()
        self.assertIn('not found', content.lower())
        self.assertIn('home', content.lower() or 'dashboard' in content.lower())
```

---

## 📋 Test Execution Plan

### Phase 1: Functional Testing (Week 1)
- Run all functional tests for each role
- Document failures and bugs
- Fix critical issues

### Phase 2: UTAUT Quantitative Testing (Week 2)
- Measure page load times
- Track task completion times
- Calculate error rates
- Monitor system performance

### Phase 3: UTAUT Qualitative Testing (Week 3)
- Conduct user acceptance testing with real users
- Administer UTAUT survey questionnaires
- Conduct interviews for detailed feedback
- Observe user sessions

### Phase 4: Analysis & Improvements (Week 4)
- Analyze quantitative and qualitative data
- Identify areas below acceptance threshold
- Prioritize improvements
- Implement changes
- Re-test

---

## ✅ Success Criteria

### Functional Success
- [ ] All role-based tests pass (100%)
- [ ] No critical security issues
- [ ] Performance benchmarks met

### UTAUT Acceptance Success

| Construct | Target Score | Minimum Acceptable |
|-----------|-------------|-------------------|
| Performance Expectancy | >5.5/7 | >5.0/7 |
| Effort Expectancy | >6.0/7 | >5.5/7 |
| Social Influence | >5.0/7 | >4.5/7 |
| Facilitating Conditions | >5.5/7 | >5.0/7 |
| Behavioral Intention | >5.5/7 | >5.0/7 |

### Adoption Metrics
- [ ] >80% of PA Staff actively use system
- [ ] >90% of Validators use system daily
- [ ] <10% of users request return to paper-based system
- [ ] User satisfaction >75%

---

*Document Version: 1.0*  
*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Testing Documentation*
