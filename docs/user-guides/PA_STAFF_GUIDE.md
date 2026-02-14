# PA Staff Workflow Documentation

## 📋 Overview

This document provides comprehensive documentation for Protected Area (PA) Staff users in the PRISM-Matutum system. PA Staff are field data collectors who submit wildlife observations and monitor protected areas.

**Role Code:** `pa_staff`  
**Primary Function:** Field data collection and submission tracking

---

## 🎯 Role Definition & Capabilities

### What PA Staff Can Do:
- ✅ View personalized dashboard with submission statistics
- ✅ Track all personal field data submissions
- ✅ View detailed submission status and validation feedback
- ✅ Monitor observations across all types (wildlife, resource use, disturbance, landscape)
- ✅ Filter and search their own submissions
- ✅ View validation history and remarks
- ✅ Access their own profile information

### What PA Staff Cannot Do:
- ❌ Validate or approve submissions from other users
- ❌ Access the validation queue
- ❌ Manage users or organizational data
- ❌ Create system-wide reports
- ❌ Access admin functions

---

## 🏠 1. PA Staff Home Dashboard

**URL:** `/dashboard/pa-staff/home/`  
**View:** `PAStaffDashboardHome` ([views_pa_staff.py](dashboard/views_pa_staff.py#L200-L280))  
**Template:** [pa_staff_home.html](dashboard/templates/dashboard/pa_staff_home.html)

### Dashboard Components

#### 1.1 Quick Statistics Cards

The dashboard displays four primary stat cards:

| Stat Card | Description | Value Source |
|-----------|-------------|--------------|
| **Total Submissions** | All submissions by this PA Staff | `DataSubmission.objects.filter(username=user.username).count()` |
| **Pending Review** | Submissions awaiting validation | `validation_status='---'` |
| **Approved** | Successfully validated submissions | `validation_status='validation_status_approved'` |
| **Total Observations** | Combined count across all observation types | Wildlife + Resource Use + Disturbance + Landscape |

**Additional Metrics:**
- **This Week:** Submissions from the last 7 days
- **Approval Rate:** Percentage of approved submissions

#### 1.2 Field Verification Alert

If the PA Staff user is not field-verified (`is_field_verified=False`), a warning banner appears:

```html
<div class="alert alert-warning">
    <strong>Field Verification Required:</strong> Your account is not yet verified for field work.
</div>
```

#### 1.3 Recent Submissions Table

Displays the 10 most recent submissions with:
- Submission ID
- Submission date/time
- Validation status (color-coded badge)
- Quick action link to view details

#### 1.4 Quick Actions

- **View All Submissions:** Links to the full submissions list
- **View My Observations:** Links to aggregated observations view

### Code Reference

```python
# Key statistics calculation
context['stats'] = {
    'total': all_submissions.count(),
    'pending': all_submissions.filter(validation_status='---').count(),
    'approved': all_submissions.filter(validation_status='validation_status_approved').count(),
    'this_week': all_submissions.filter(submission_time__gte=timezone.now() - timedelta(days=7)).count(),
    'total_observations': (
        WildlifeObservation.objects.filter(submission__username=username).count() +
        ResourceUseIncident.objects.filter(submission__username=username).count() +
        DisturbanceRecord.objects.filter(submission__username=username).count() +
        LandscapeMonitoring.objects.filter(submission__username=username).count()
    ),
}
```

---

## 📝 2. My Submissions View

**URL:** `/dashboard/pa-staff/my-submissions/`  
**View:** `MySubmissionsView` ([views_pa_staff.py](dashboard/views_pa_staff.py#L17-L106))  
**Template:** [my_submissions.html](dashboard/templates/dashboard/my_submissions.html)

### Features

#### 2.1 Statistics Dashboard

Six statistics cards displayed at the top:

| Stat | Badge Color | Icon | Description |
|------|-------------|------|-------------|
| Total Submissions | Primary | `bi-file-earmark-text` | All-time submission count |
| Pending Review | Warning | `bi-clock-history` | Awaiting validation |
| Approved | Success | `bi-check-circle` | Successfully validated |
| Rejected | Danger | `bi-x-circle` | Not approved by validators |
| This Week | Info | `bi-calendar-week` | Last 7 days |
| This Month | Primary | `bi-calendar-month` | Last 30 days |

#### 2.2 Advanced Filtering

**Status Filter:**
- All
- Pending (`validation_status='---'`)
- Approved (`validation_status='validation_status_approved'`)
- Rejected (`validation_status='validation_status_not_approved'`)
- On Hold (`validation_status='validation_status_on_hold'`)

**Date Range Filter:**
- Date From: Filter submissions from a specific date
- Date To: Filter submissions up to a specific date

**Search Filter:**
- Search by Submission ID
- Search by Validation Remarks (keywords in feedback)

```python
# Filter implementation
if status_filter:
    if status_filter == 'pending':
        queryset = queryset.filter(validation_status='---')
    elif status_filter == 'approved':
        queryset = queryset.filter(validation_status='validation_status_approved')
    # ... etc

# Date range
if date_from:
    queryset = queryset.filter(submission_time__gte=date_from)
if date_to:
    queryset = queryset.filter(submission_time__lte=date_to)

# Search
if search:
    queryset = queryset.filter(
        Q(submission_id__icontains=search) |
        Q(validation_remarks__icontains=search)
    )
```

#### 2.3 Submissions Table

**Columns:**
1. **Submission ID** - Unique identifier
2. **Form Name** - Type of submission
3. **Submission Date** - When the data was collected
4. **Status** - Color-coded validation status badge
5. **Validator** - Who validated (if applicable)
6. **Validated At** - Validation timestamp
7. **Actions** - View details button

**Pagination:** 25 submissions per page

#### 2.4 Status Badge Color Codes

```html
Pending (---) → <span class="badge bg-warning">Pending Review</span>
Approved → <span class="badge bg-success">Approved</span>
Rejected → <span class="badge bg-danger">Rejected</span>
On Hold → <span class="badge bg-secondary">On Hold</span>
Flagged → <span class="badge bg-info">Flagged for Review</span>
```

#### 2.5 Recent Validation Activity

Displays the 5 most recent validated submissions (non-pending) with:
- Submission ID
- Validation status
- Validation date
- Validator name

### User Flow Diagram

```
[Login] → [PA Staff Home Dashboard]
           ↓
    [Click "View All Submissions"]
           ↓
    [My Submissions Page]
           ↓
    [Apply Filters (Status/Date/Search)]
           ↓
    [Browse Paginated Results]
           ↓
    [Click "View Details" on Submission]
           ↓
    [My Submission Detail Page]
```

---

## 🔍 3. My Submission Detail View

**URL:** `/dashboard/pa-staff/submission/<int:pk>/`  
**View:** `MySubmissionDetailView` ([views_pa_staff.py](dashboard/views_pa_staff.py#L283-L351))  
**Template:** [my_submission_detail.html](dashboard/templates/dashboard/my_submission_detail.html)

### Security

**Access Control:**
```python
def get_object(self, queryset=None):
    """Ensure PA Staff can only view their own submissions"""
    obj = super().get_object(queryset)
    if obj.username != self.request.user.username:
        raise Http404("Submission not found")
    return obj
```

PA Staff can **only** view submissions where `submission.username == current_user.username`.

### Detail Sections

#### 3.1 Submission Header
- Submission ID
- Form Name
- Submission Date/Time
- Current Validation Status (large color-coded badge)

#### 3.2 Validation Information
- **Validator:** Name of the person who validated
- **Validated At:** Date and time of validation
- **Validation Remarks:** Feedback from validator
- **Validation History:** Timeline of status changes

**Validation History Table:**
```python
context['validation_history'] = ValidationHistory.objects.filter(
    submission=submission
).select_related('changed_by').order_by('-changed_at')
```

Displays:
- Status change (from → to)
- Changed by (validator name)
- Changed at (timestamp)
- Remarks/reason for change

#### 3.3 Location Data

If GPS coordinates are available:
```python
geolocation = form_data.get('_geolocation', [])
if geolocation and len(geolocation) >= 2:
    context['latitude'] = geolocation[0]
    context['longitude'] = geolocation[1]
```

Displays:
- Interactive map with marker at submission location
- Latitude/Longitude coordinates
- Accuracy information

#### 3.4 Related Observations

**Wildlife Observations:**
```python
context['wildlife_observations'] = WildlifeObservation.objects.filter(
    submission=submission
).select_related('fauna_species', 'flora_species')
```

**Resource Use Incidents:**
```python
context['resource_use_incidents'] = ResourceUseIncident.objects.filter(submission=submission)
```

**Disturbance Records:**
```python
context['disturbance_records'] = DisturbanceRecord.objects.filter(submission=submission)
```

**Landscape Monitoring:**
```python
context['landscape_monitoring'] = LandscapeMonitoring.objects.filter(
    submission=submission
).select_related('monitoring_location')
```

Each observation type displays:
- Species/Type
- Count/Details
- Observation-specific fields
- Link to detailed observation view

#### 3.5 Form Data (Raw)

Accordion section showing the raw KoboToolbox JSON form data:
- All form fields and values
- Metadata (device info, submission time, etc.)
- Attachments/media references

---

## 👁️ 4. My Observations View

**URL:** `/dashboard/pa-staff/my-observations/`  
**View:** `MyObservationsView` ([views_pa_staff.py](dashboard/views_pa_staff.py#L109-L197))  
**Template:** [my_observations.html](dashboard/templates/dashboard/my_observations.html)

### Purpose

Aggregates **all observation types** from all submissions into a unified view:
- Wildlife Observations
- Resource Use Incidents
- Disturbance Records
- Landscape Monitoring

### Statistics Cards

Four cards showing counts by observation type:

| Card | Icon | Color | Count Source |
|------|------|-------|--------------|
| Wildlife Observations | `bi-binoculars` | Primary | `WildlifeObservation.objects.filter(submission__username=username).count()` |
| Resource Use | `bi-exclamation-triangle` | Warning | `ResourceUseIncident.objects.filter(submission__username=username).count()` |
| Disturbances | `bi-shield-exclamation` | Danger | `DisturbanceRecord.objects.filter(submission__username=username).count()` |
| Landscape | `bi-tree` | Success | `LandscapeMonitoring.objects.filter(submission__username=username).count()` |

**Total Observations:** Sum of all four types

### Observations List

#### Data Aggregation Logic

```python
def get_queryset(self):
    """Combine all observation types for this PA Staff"""
    username = self.request.user.username
    
    # Get all observation types
    wildlife = list(WildlifeObservation.objects.filter(
        submission__username=username
    ).values(...).annotate(obs_type=Value('wildlife', output_field=CharField())))
    
    resource_use = list(ResourceUseIncident.objects.filter(
        submission__username=username
    ).values(...).annotate(obs_type=Value('resource_use', output_field=CharField())))
    
    disturbance = list(DisturbanceRecord.objects.filter(
        submission__username=username
    ).values(...).annotate(obs_type=Value('disturbance', output_field=CharField())))
    
    landscape = list(LandscapeMonitoring.objects.filter(
        submission__username=username
    ).values(...).annotate(obs_type=Value('landscape', output_field=CharField())))
    
    # Combine and sort by date
    all_observations = wildlife + resource_use + disturbance + landscape
    all_observations.sort(key=lambda x: x.get('submission_time') or min_date, reverse=True)
    
    return all_observations
```

#### List Display

Each observation shows:
1. **Icon** - Type-specific icon with color
2. **Title/Description:**
   - **Wildlife:** Species name + count
   - **Resource Use:** Type + brief details
   - **Disturbance:** Type + brief details
   - **Landscape:** Location name + monitoring details
3. **Type Badge** - Wildlife, Resource Use, Disturbance, or Landscape
4. **Date** - Observation submission date
5. **Validation Status** - Badge showing if parent submission is approved/pending/rejected

**Pagination:** 20 observations per page

### Icon & Color Mapping

```html
{% if obs.obs_type == 'wildlife' %}
    <i class="bi bi-binoculars fs-3 text-primary"></i>
{% elif obs.obs_type == 'resource_use' %}
    <i class="bi bi-exclamation-triangle fs-3 text-warning"></i>
{% elif obs.obs_type == 'disturbance' %}
    <i class="bi bi-shield-exclamation fs-3 text-danger"></i>
{% elif obs.obs_type == 'landscape' %}
    <i class="bi bi-tree fs-3 text-success"></i>
{% endif %}
```

---

## 🔄 5. Data Submission Process

### 5.1 External Data Collection (KoboToolbox)

PA Staff collect data using KoboToolbox mobile app or web forms:

1. **In the Field:**
   - Open KoboToolbox Collect app
   - Select appropriate form (Wildlife, Resource Use, etc.)
   - Fill out form fields
   - Capture GPS coordinates automatically
   - Take photos/media if needed
   - Submit form

2. **Data Sync:**
   - KoboToolbox data syncs to PRISM backend via API
   - Celery task processes new submissions
   - Creates `DataSubmission` record with `validation_status='---'` (pending)

### 5.2 Submission Status Workflow

```
[Submitted via Kobo]
        ↓
[Pending Review (---)]  ← PA Staff can view here
        ↓
[Assigned to Validator]
        ↓
[Under Review]
        ↓
    ┌───────────┬───────────────┬──────────┐
    ↓           ↓               ↓          ↓
[Approved] [Rejected] [On Hold] [Flagged]
```

**Status Codes:**
- `---` = Pending Review (awaiting validation)
- `validation_status_approved` = Approved
- `validation_status_not_approved` = Rejected
- `validation_status_on_hold` = On Hold (needs more info)
- `flagged` = Flagged for Review (quality concerns)

### 5.3 Viewing Submission Status

PA Staff can check status in three ways:

1. **Home Dashboard** - Quick overview of pending/approved counts
2. **My Submissions** - Filtered list with status badges
3. **Submission Detail** - Full validation history and remarks

### 5.4 Understanding Validation Feedback

When a validator reviews a submission, they can:
- Approve it ✅
- Reject it ❌ (with reason)
- Put it on hold ⏸️ (requesting more info)
- Flag it 🚩 (quality concerns)

**Validation Remarks** are visible to PA Staff and explain:
- Why submission was rejected
- What information is missing
- What needs to be corrected
- Next steps (if any)

---

## 🧭 6. User Journey Flowcharts

### 6.1 Daily Check-In Workflow

```
┌─────────────────┐
│   PA Staff      │
│   Login         │
└────────┬────────┘
         ↓
┌─────────────────┐
│  Dashboard      │
│  - View Stats   │
│  - Check        │
│    Pending      │
└────────┬────────┘
         ↓
    ┌────────┐
    │ Any    │  NO → Continue with other tasks
    │ New    │
    │Updates?│
    └───┬────┘
        │ YES
        ↓
┌─────────────────┐
│ View Recent     │
│ Validations     │
└────────┬────────┘
         ↓
┌─────────────────┐
│ Read Validator  │
│ Feedback        │
└────────┬────────┘
         ↓
┌─────────────────┐
│ Take Action     │
│ (if needed)     │
└─────────────────┘
```

### 6.2 Checking Specific Submission

```
[Home Dashboard]
        ↓
[Click "View All Submissions"]
        ↓
[My Submissions Page]
        ↓
[Search/Filter for Submission]
        ↓
[Click "View Details"]
        ↓
[Submission Detail Page]
        ↓
[Review Information]
   - Status
   - Validation Remarks
   - Observations
   - Location Data
        ↓
[Done]
```

### 6.3 Tracking Observations

```
[Home Dashboard]
        ↓
[Click "View My Observations"]
        ↓
[My Observations Page]
        ↓
[View Statistics by Type]
   - Wildlife
   - Resource Use
   - Disturbance
   - Landscape
        ↓
[Browse Observation List]
        ↓
[Click on Specific Observation]
        ↓
[View Observation Details]
```

---

## 🧪 7. Testing Checklist for PA Staff

### 7.1 Authentication & Access

- [ ] **Test:** PA Staff can login with valid credentials
- [ ] **Test:** PA Staff redirects to correct dashboard after login
- [ ] **Test:** PA Staff cannot access validator URLs directly
- [ ] **Test:** PA Staff cannot access admin URLs
- [ ] **Test:** PA Staff cannot access other users' submissions

### 7.2 Dashboard Functionality

- [ ] **Test:** Dashboard displays correct total submission count
- [ ] **Test:** Pending count matches submissions with `validation_status='---'`
- [ ] **Test:** Approved count is accurate
- [ ] **Test:** "This Week" count shows submissions from last 7 days
- [ ] **Test:** Recent submissions table shows 10 most recent items
- [ ] **Test:** Field verification alert shows when `is_field_verified=False`

### 7.3 My Submissions View

- [ ] **Test:** Only displays current user's submissions
- [ ] **Test:** Statistics cards show correct counts
- [ ] **Test:** Status filter works (pending/approved/rejected/on_hold)
- [ ] **Test:** Date range filter works correctly
- [ ] **Test:** Search by submission ID works
- [ ] **Test:** Search by validation remarks works
- [ ] **Test:** Pagination displays 25 items per page
- [ ] **Test:** Status badges have correct colors
- [ ] **Test:** Recent validations section displays correctly

### 7.4 Submission Detail View

- [ ] **Test:** Can view own submission details
- [ ] **Test:** Cannot view other users' submissions (404 error)
- [ ] **Test:** Validation status is displayed correctly
- [ ] **Test:** Validation remarks are visible
- [ ] **Test:** Validation history shows all status changes
- [ ] **Test:** GPS location displays on map (if available)
- [ ] **Test:** Related observations are listed correctly
- [ ] **Test:** Wildlife observations link to correct species
- [ ] **Test:** Form data accordion displays raw JSON correctly

### 7.5 My Observations View

- [ ] **Test:** Displays combined observations from all submissions
- [ ] **Test:** Statistics cards show correct counts per type
- [ ] **Test:** Total observations = sum of all types
- [ ] **Test:** Observations are sorted by date (newest first)
- [ ] **Test:** Wildlife icon and color are correct (blue)
- [ ] **Test:** Resource use icon and color are correct (yellow)
- [ ] **Test:** Disturbance icon and color are correct (red)
- [ ] **Test:** Landscape icon and color are correct (green)
- [ ] **Test:** Pagination displays 20 items per page
- [ ] **Test:** Observation types are labeled correctly

### 7.6 Permissions & Restrictions

- [ ] **Test:** Cannot access `/dashboard/validation/queue/`
- [ ] **Test:** Cannot access `/dashboard/admin/`
- [ ] **Test:** Cannot validate submissions via API
- [ ] **Test:** Cannot edit other users' data
- [ ] **Test:** Cannot create system reports
- [ ] **Test:** Cannot manage users or agencies

### 7.7 Performance

- [ ] **Test:** Dashboard loads within 2 seconds
- [ ] **Test:** Submissions list with 100+ records loads efficiently
- [ ] **Test:** Observations aggregation performs well
- [ ] **Test:** Filtering/search responds quickly

---

## 🛠️ 8. Troubleshooting

### Common Issues

#### Issue: "Submission not found" (404 error)

**Cause:** Trying to access another user's submission  
**Solution:** PA Staff can only view their own submissions. Ensure the submission belongs to the logged-in user.

#### Issue: No submissions appearing

**Possible Causes:**
1. No data has been submitted via KoboToolbox
2. Username mismatch between Kobo and PRISM
3. Synchronization task hasn't run yet

**Solution:**
```bash
# Check if submissions exist in database
docker-compose exec web python manage.py shell
>>> from api.models import DataSubmission
>>> DataSubmission.objects.filter(username='your_username').count()

# Manually trigger KoboToolbox sync
docker-compose exec web python manage.py sync_kobotoolbox_data
```

#### Issue: Validation status not updating

**Cause:** Validator hasn't processed the submission yet  
**Solution:** Wait for validator to review. Status will update automatically.

#### Issue: Observations not showing in "My Observations"

**Cause:** Submissions haven't been processed into observation records  
**Solution:** This happens automatically after validation approval. Check if submissions are approved first.

---

## 📚 9. Related Documentation

- **Access Control:** See [RBAC_DOCUMENTATION.md](../architecture/RBAC_DOCUMENTATION.md)
- **Permission System:** See [PERMISSION_SYSTEM.md](../architecture/PERMISSION_SYSTEM.md)
- **Validation Workflow:** See [dashboard/docs/VALIDATION_WORKFLOW.md](../../dashboard/docs/VALIDATION_WORKFLOW.md)
- **Testing:** See [PA_STAFF_TEST_CHECKLIST.md](../../dashboard/docs/PA_STAFF_TEST_CHECKLIST.md)

---

## 📞 Support

**For PA Staff Users:**
- Contact your supervisor or system administrator
- Check validation remarks for specific feedback
- Review training materials for data collection best practices

**For Developers:**
- Reference: [views_pa_staff.py](dashboard/views_pa_staff.py)
- Tests: [test_pa_staff_functionality.py](dashboard/tests/test_pa_staff_functionality.py)
- Templates: `dashboard/templates/dashboard/pa_staff_*.html` and `my_*.html`

---

*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Capstone Documentation Project*
