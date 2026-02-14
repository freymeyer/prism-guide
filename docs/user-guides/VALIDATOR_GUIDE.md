# Validator Workflow Documentation

## 📋 Overview

This document provides comprehensive documentation for Validator users in the PRISM-Matutum system. Validators are responsible for reviewing, approving, rejecting, or flagging field data submissions from PA Staff members to ensure data quality and accuracy.

**Role Code:** `validator`  
**Primary Function:** Data validation and quality assurance

---

## 🎯 Role Definition & Capabilities

### What Validators Can Do:
- ✅ Access the validation queue with all pending submissions
- ✅ Review detailed submission data in the validation workspace
- ✅ Approve, reject, hold, or flag submissions
- ✅ Add validation remarks and feedback
- ✅ Assign submissions to themselves or other validators
- ✅ View validation history and performance metrics
- ✅ Export validation history data
- ✅ Access species verification tools
- ✅ View data quality assessments
- ✅ Navigate between submissions efficiently

### What Validators Cannot Do:
- ❌ Edit raw submission data (data integrity protection)
- ❌ Delete submissions
- ❌ Access system-wide admin functions
- ❌ Manage users or organizations (unless also System Admin)
- ❌ Modify other validators' validation decisions (audit trail protection)

---

## 🏠 1. Validator Dashboard

**URL:** `/dashboard/validator/dashboard/`  
**Template:** Custom validator dashboard with widgets

### Dashboard Widgets

Default widgets include:
- **Pending Validations Counter** - Real-time count of submissions needing review
- **Assigned to Me Counter** - Submissions assigned to current validator
- **Approval Rate** - Percentage of approved submissions
- **Validation Stats** - Recent validation activity
- **Quick Access** - Links to queue and history

Validators can customize their dashboard with additional widgets.

---

## 📋 2. Validation Queue

**URL:** `/dashboard/validation/queue/`  
**View:** `ValidationQueueView` ([views_validation.py](dashboard/views_validation.py#L15-L165))  
**Template:** [validation_queue.html](dashboard/templates/dashboard/validation_queue.html)

### 2.1 Queue Statistics

Six statistics cards displayed at the top:

| Stat | Description | Color | Calculation |
|------|-------------|-------|-------------|
| **Total Pending** | All submissions needing validation | Primary | Count of `validation_status` in [`---`, `validation_status_on_hold`, `flagged`] |
| **Not Validated** | Fresh submissions | Warning | `validation_status='---'` |
| **On Hold** | Needs more information | Secondary | `validation_status='validation_status_on_hold'` |
| **Flagged** | Quality concerns | Info | `validation_status='flagged'` |
| **High Priority** | Urgent submissions | Red | Over 7 days old OR flagged |
| **Assigned to Me** | Your assignments | Primary | `validated_by=current_user` |

```python
stats = {
    'total': queryset.count(),
    'not_validated': queryset.filter(validation_status='---').count(),
    'on_hold': queryset.filter(validation_status='validation_status_on_hold').count(),
    'flagged': queryset.filter(validation_status='flagged').count(),
    'high_priority': queryset.filter(
        Q(submission_time__lt=now - timedelta(days=7)) |
        Q(validation_status='flagged')
    ).count(),
    'assigned_to_me': queryset.filter(validated_by=request.user).count(),
}
```

### 2.2 Advanced Filtering

**Filter Sidebar includes:**

| Filter Type | Options | Description |
|-------------|---------|-------------|
| **Status** | All, Pending (---), On Hold, Flagged | Filter by validation status |
| **From Date** | Date picker | Submissions from this date |
| **To Date** | Date picker | Submissions up to this date |
| **PA Staff (Submitter)** | Dropdown of PA Staff users | Filter by who submitted |
| **Assigned PA Staff** | Dropdown of assigned staff | Filter by assignment |
| **Priority** | All, High, Normal | Based on submission age |
| **Search** | Text input | Search by Submission ID or remarks |

**Implementation:**
```python
from .filters import SubmissionFilter

queryset = DataSubmission.objects.filter(
    validation_status__in=['---', 'validation_status_on_hold', 'flagged']
)

self.filterset = SubmissionFilter(self.request.GET, queryset=queryset)
queryset = self.filterset.qs
```

### 2.3 Sorting Options

| Sort Option | Value | Description |
|-------------|-------|-------------|
| Newest First | `-submission_time` | **Default** - Most recent submissions first |
| Oldest First | `submission_time` | Oldest submissions (high priority) first |
| PA Staff A-Z | `username` | Alphabetical by collector name |
| PA Staff Z-A | `-username` | Reverse alphabetical |
| Status A-Z | `validation_status` | By status code |
| Status Z-A | `-validation_status` | Reverse status order |

**Query parameter:** `?sort=-submission_time`

### 2.4 Submissions Table

**Columns:**
1. **Checkbox** - For bulk actions (select/deselect)
2. **Submission ID** - Unique identifier, clickable to open workspace
3. **Form Name** - Type of record (Wildlife, Resource Use, etc.)
4. **PA Staff** - Data collector username
5. **Submission Date** - When data was collected
6. **Status Badge** - Color-coded validation status
7. **Priority Indicator** - 🔥 High priority icon if urgent
8. **Assigned To** - Current validator (if assigned)
9. **Actions** - "Review" button to open workspace

**Pagination:** 50 submissions per page

### 2.5 Bulk Actions

**Available Actions:**

#### A. Assign to Self
```html
<form method="post">
    <input type="hidden" name="action" value="assign_to_self">
    <!-- Selected submission IDs from checkboxes -->
    <button type="submit">Assign to Me</button>
</form>
```

**Process:**
1. Select submissions using checkboxes
2. Click "Assign to Me" button
3. System updates `validated_by` field
4. Activity logged
5. Dashboard stats updated

#### B. Assign to Other Validator
```html
<form method="post">
    <input type="hidden" name="action" value="assign_to_other">
    <select name="validator">
        <option value="[validator_id]">Validator Name</option>
    </select>
    <button type="submit">Assign</button>
</form>
```

**Process:**
1. Select submissions
2. Choose target validator from dropdown
3. Submit form
4. System assigns and creates notification for assigned validator
5. Email notification sent (if enabled)

**Code Reference:**
```python
def assign_submissions(self, submission_ids, validator):
    assignable_submissions = DataSubmission.objects.filter(
        id__in=submission_ids,
        validation_status__in=['---', 'validation_status_on_hold', 'flagged']
    )
    
    updated_count = assignable_submissions.update(
        validated_by=validator,
        last_updated=timezone.now()
    )
    
    # Log activity
    UserActivityLog.objects.create(
        user=self.request.user,
        activity_type='validation',
        description=f"Assigned {updated_count} submissions to {validator.get_full_name()}",
        metadata={'submission_ids': list(submission_ids), ...}
    )
    
    # Create notification
    if validator != self.request.user:
        UserNotification.objects.create(
            user=validator,
            notification_type='assignment',
            title='New Validation Assignment',
            message=f'{self.request.user.get_full_name()} assigned {updated_count} submissions to you.'
        )
```

### 2.6 Quick Actions

- **Sync from KoboToolbox** - Manually trigger data sync
- **Refresh Queue** - Reload queue data

---

## 🔬 3. Validation Workspace

**URL:** `/dashboard/validation/workspace/<int:pk>/`  
**View:** `ValidationWorkspaceView` ([views_validation.py](dashboard/views_validation.py#L168-L500))  
**Template:** [validation_workspace.html](dashboard/templates/dashboard/validation_workspace.html)

### 3.1 Workspace Layout

#### Header Navigation
- **Previous Button** - Go to previous submission in queue
- **Back to Queue** - Return to validation queue
- **Next Button** - Go to next submission in queue
- **Position Indicator** - "X of Y submissions"

```python
context['navigation'] = {
    'previous_id': previous_submission.id if exists else None,
    'next_id': next_submission.id if exists else None,
    'current_position': position,
    'total_submissions': total_count
}
```

### 3.2 Main Content Sections

#### A. Collection Metadata (Collapsible)

**Left Column:**
- Submission ID
- Collector Name
- Submission Time
- Start Time

**Right Column:**
- End Time
- Collection Duration
- Device Info
- Form Version

**Implementation:**
```python
metadata = {
    'submission_id': submission.submission_id,
    'collector': submission.username,
    'submission_time': submission.submission_time,
    'start_time': submission.start_time,
    'end_time': submission.end_time,
    'device_info': form_data.get('deviceid', 'Unknown'),
    'form_version': form_data.get('__version__', 'Unknown'),
}

if submission.start_time and submission.end_time:
    metadata['collection_duration'] = str(submission.end_time - submission.start_time)
```

#### B. Data Quality Assessment

**Quality Score Card:**
- Overall quality score (0-100)
- Color-coded badge (green/yellow/red)
- Quality factors breakdown:
  - ✅ GPS accuracy
  - ✅ Completeness
  - ✅ Consistency
  - ✅ Timeliness

**Quality Issues List:**
- ⚠️ Warning level issues
- ❌ Error level issues
- ℹ️ Info level suggestions

```python
from .utils import DataQualityAssessor

quality = DataQualityAssessor.assess_submission_quality(submission)
# Returns: {
#     'score': 85,
#     'level': 'good',  # excellent/good/fair/poor
#     'issues': [
#         {'type': 'warning', 'message': 'GPS accuracy exceeds 50m'},
#         ...
#     ],
#     'factors': {
#         'gps_accuracy': True,
#         'completeness': True,
#         'consistency': False,
#         'timeliness': True
#     }
# }
```

#### C. Record-Specific Data

Parsed and formatted based on record type:

**Wildlife Observation:**
- Species (scientific name)
- Species Count
- Observation Modes (Seen, Heard, Scent, Traces)
- Species Details/Remarks

**Resource Use Incident:**
- Resource Use Type
- Details/Description
- Evidence (if any)

**Disturbance Record:**
- Disturbance Type
- Details/Description
- Severity assessment

**Landscape Monitoring:**
- Monitoring Location
- Habitat Type
- Monitoring Details

#### D. Species Verification (Wildlife only)

**Verification Results:**
- ✅ **Matched** - Species found in master checklist
  - Scientific name
  - Common names
  - Conservation status
  - Endemicity status
  - Taxonomy hierarchy
- ❌ **Not Matched** - Species not in checklist
  - Suggestion to review spelling
  - Similar species suggestions
  - Option to flag for taxonomist review

**Code:**
```python
def verify_species(self, form_data):
    species_name = self._normalize_species_name(form_data.get('species'))
    
    fauna_match = FaunaChecklist.objects.filter(
        scientific_name__iexact=species_name
    ).select_related('id_taxonomy', 'id_threat_category', 'id_endemicity').first()
    
    if fauna_match:
        return {
            'match': True,
            'species': fauna_match,
            'conservation_status': fauna_match.id_threat_category.threat_category_status,
            'endemicity': fauna_match.id_endemicity.endemicity_status,
            ...
        }
    else:
        return {'match': False, 'raw_name': species_name}
```

#### E. Spatial Analysis

**Location Data:**
- Coordinates (Latitude, Longitude)
- Altitude
- GPS Accuracy
- Interactive map with marker

**Spatial Checks:**
- ✅ Inside protected area boundaries
- ⚠️ Near protected area (within buffer)
- ❌ Outside protected area
- Distance to nearest protected area

```python
def analyze_spatial_data(self, submission):
    if submission.location:
        from .utils_geo import check_protected_area_coverage
        
        return {
            'coordinates': (submission.location.y, submission.location.x),
            'accuracy': submission.location_accuracy,
            'inside_pa': check_protected_area_coverage(submission.location),
            'nearest_pa': get_nearest_protected_area(submission.location),
            ...
        }
```

#### F. Media Attachments

**Display:**
- Thumbnail gallery
- Full-size image viewer (modal)
- Image metadata (timestamp, GPS coordinates embedded in photo)
- Download links

#### G. Validation History

**Timeline of Actions:**
- Status change (from → to)
- Validator name
- Timestamp
- Remarks/Reason

**Example:**
```
🟢 Approved by Jane Validator - Jan 6, 2026 14:30
   "Species identification verified. GPS coordinates within park boundaries."

🟡 On Hold by John Validator - Jan 5, 2026 10:15
   "Need clearer photo of species. Please resubmit with better quality image."

⚪ Pending - Submitted - Jan 4, 2026 08:45
```

### 3.3 Validation Actions Panel (Right Sidebar)

#### Action Buttons:

**1. Approve ✅**
```html
<button class="btn btn-success" onclick="validateSubmission('approve')">
    <i class="fas fa-check"></i> Approve
</button>
```

**Requirements:**
- All quality checks passed (or acknowledged)
- Species verified (if wildlife)
- Location within reasonable bounds

**Effect:**
- Sets `validation_status='validation_status_approved'`
- Sets `validated_by=current_user`
- Sets `validated_at=now`
- Records validation history entry
- Triggers processing: Creates/updates observation records
- Sends notification to PA Staff

---

**2. Reject ❌**
```html
<button class="btn btn-danger" onclick="validateSubmission('reject')">
    <i class="fas fa-times"></i> Reject
</button>
```

**Requirements:**
- **Must** provide rejection reason (validation remarks)

**Common Rejection Reasons:**
- Incorrect species identification
- GPS coordinates clearly wrong (e.g., in ocean)
- Duplicate submission
- Incomplete data
- Photos do not match description

**Effect:**
- Sets `validation_status='validation_status_not_approved'`
- Records validation remarks
- Sends notification to PA Staff with feedback
- Submission excluded from reports/analysis

---

**3. Put on Hold ⏸️**
```html
<button class="btn btn-warning" onclick="validateSubmission('on_hold')">
    <i class="fas fa-pause"></i> On Hold
</button>
```

**Use Cases:**
- Need more information from PA Staff
- Unclear species identification - need expert consultation
- Photo quality insufficient
- GPS accuracy concerns - need field verification

**Requirements:**
- Should provide remarks explaining what's needed

**Effect:**
- Sets `validation_status='validation_status_on_hold'`
- Stays in validation queue
- Can be resumed later

---

**4. Flag for Review 🚩**
```html
<button class="btn btn-info" onclick="validateSubmission('flag')">
    <i class="fas fa-flag"></i> Flag
</button>
```

**Use Cases:**
- Exceptional/rare species sighting - needs expert verification
- Data anomaly requiring senior validator review
- Potential data quality issue
- Training case for other validators

**Options:**
- Assign to specific validator/taxonomist
- Add detailed flag reason

**Effect:**
- Sets `validation_status='flagged'`
- High priority in queue
- Notification sent to flagged validator

---

#### Validation Remarks Field

**Purpose:** Provide feedback to PA Staff

**Guidelines:**
- Be specific and constructive
- Explain why submission was rejected/held
- Provide guidance for resubmission
- Note any corrections needed

**Examples of Good Remarks:**

✅ **Approval:**
"Species identification confirmed. Excellent photo quality. GPS coordinates verified within park boundaries."

✅ **Rejection:**
"Species misidentification. Photo shows *Macaca fascicularis* (Long-tailed Macaque), not *Macaca nemestrina* as indicated. Please review species identification guide."

✅ **On Hold:**
"GPS accuracy is 150m, which is too high for precise location. Please resubmit with GPS accuracy < 50m if possible."

✅ **Flag:**
"Potential first recorded sighting of *Pithecophaga jefferyi* (Philippine Eagle) in this area. Flagging for Senior Taxonomist verification."

### 3.4 AJAX Validation Submission

**JavaScript Function:**
```javascript
function validateSubmission(action) {
    const remarks = $('#validation-remarks').val();
    
    if (action === 'reject' && !remarks) {
        alert('Please provide rejection reason.');
        return;
    }
    
    $.ajax({
        url: `/api/validate/${submissionId}/`,
        method: 'POST',
        data: {
            action: action,  // 'approve', 'reject', 'on_hold', 'flag'
            remarks: remarks,
            csrfmiddlewaretoken: getCookie('csrftoken')
        },
        success: function(response) {
            if (response.success) {
                showSuccessMessage(response.message);
                // Navigate to next submission or return to queue
                if (nextSubmissionId) {
                    window.location.href = `/dashboard/validation/workspace/${nextSubmissionId}/`;
                } else {
                    window.location.href = '/dashboard/validation/queue/';
                }
            } else {
                showErrorMessage(response.error);
            }
        },
        error: function(xhr) {
            showErrorMessage('Validation failed. Please try again.');
        }
    });
}
```

**API Endpoint:** `/api/validate/<int:pk>/`  
**Method:** POST  
**Parameters:**
- `action` (required): `approve`, `reject`, `on_hold`, `flag`
- `remarks` (optional but recommended): Validation feedback
- `assign_to` (optional, for flag action): Validator user ID

---

## 📊 4. Validation History

**URL:** `/dashboard/validation/history/`  
**View:** `ValidationHistoryView`  
**Template:** [validation_history.html](dashboard/templates/dashboard/validation_history.html)

### 4.1 History Statistics

Six statistics cards:

| Stat | Color | Description |
|------|-------|-------------|
| **Total Actions** | Primary | All validation decisions made by this validator |
| **Approved** | Success Green | Count of approved submissions |
| **Rejected** | Danger Red | Count of rejected submissions |
| **On Hold** | Warning Yellow | Count of submissions put on hold |
| **Flagged** | Info Blue | Count of flagged submissions |
| **Approval Rate** | Primary | Percentage of approvals vs total validations |

**Calculation:**
```python
history_stats = ValidationHistory.objects.filter(
    changed_by=request.user
).aggregate(
    total=Count('id'),
    approved=Count('id', filter=Q(new_status='validation_status_approved')),
    rejected=Count('id', filter=Q(new_status='validation_status_not_approved')),
    on_hold=Count('id', filter=Q(new_status='validation_status_on_hold')),
    flagged=Count('id', filter=Q(new_status='flagged'))
)

if history_stats['total'] > 0:
    history_stats['approval_rate'] = round(
        (history_stats['approved'] / history_stats['total']) * 100, 1
    )
```

### 4.2 History Filters

**Available Filters:**
- **From Date** - Start of date range
- **To Date** - End of date range
- **Action** - Filter by validation action (approve/reject/on hold/flag/all)
- **PA Staff** - Filter by data collector
- **Search** - Search by submission ID or remarks

### 4.3 History Table

**Columns:**
1. **Timestamp** - When validation occurred
2. **Submission ID** - Link to submission detail
3. **PA Staff** - Data collector
4. **Action** - Color-coded badge (Approved/Rejected/On Hold/Flagged)
5. **Previous Status** - Status before validation
6. **Remarks** - Validation feedback provided
7. **View** - Link to view full submission

**Pagination:** 50 records per page

### 4.4 Export Functionality

**Export to CSV:**
```python
def validation_history_export(request):
    history = ValidationHistory.objects.filter(
        changed_by=request.user
    ).select_related('submission', 'changed_by').order_by('-changed_at')
    
    # Apply same filters as history view
    ...
    
    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="validation_history.csv"'
    
    writer = csv.writer(response)
    writer.writerow(['Date', 'Submission ID', 'PA Staff', 'Action', 'Previous Status', 'Remarks'])
    
    for entry in history:
        writer.writerow([
            entry.changed_at.strftime('%Y-%m-%d %H:%M:%S'),
            entry.submission.submission_id,
            entry.submission.username,
            entry.new_status,
            entry.old_status,
            entry.remarks
        ])
    
    return response
```

---

## 🔄 5. Validation Status Workflow

### Status Codes & Meanings

| Code | Display | Meaning | Can Transition To |
|------|---------|---------|-------------------|
| `---` | Pending Review | Fresh submission, not yet reviewed | Approved, Rejected, On Hold, Flagged |
| `validation_status_approved` | Approved | Validated and accepted | *Final State* (can be flagged for re-review) |
| `validation_status_not_approved` | Rejected | Not acceptable, excluded from analysis | *Final State* |
| `validation_status_on_hold` | On Hold | Needs more information | Approved, Rejected, Flagged |
| `flagged` | Flagged for Review | Needs expert/senior review | Approved, Rejected, On Hold |

### Status Transition Diagram

```
    [Submitted from Kobo]
            ↓
     [Pending (---)]
            │
            ├───→ [Approved] ──────────→ (Processing → Observation Records)
            │
            ├───→ [Rejected] ──────────→ (Excluded from analysis)
            │
            ├───→ [On Hold] ───┐
            │                  │
            └───→ [Flagged] ───┴───→ (Back to queue for re-review)
                                    ↓
                            [Approved/Rejected]
```

### Validation Decision Tree

```
┌─────────────────────┐
│ Open Submission     │
│ in Workspace        │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Review Data Quality │
└──────────┬──────────┘
           ↓
    ┌──────────┐
    │ Quality  │
    │   OK?    │
    └─┬────┬───┘
  NO  │    │ YES
      ↓    ↓
┌──────┐   ┌───────────────┐
│REJECT│   │Species Correct?│
└──────┘   └─┬─────────┬───┘
         NO  │         │ YES
             ↓         ↓
       ┌─────────┐   ┌────────────────┐
       │ON HOLD /│   │Location Valid? │
       │  FLAG   │   └─┬──────────┬───┘
       └─────────┘ NO  │          │ YES
                       ↓          ↓
                  ┌────────┐   ┌────────┐
                  │REJECT /│   │APPROVE │
                  │ON HOLD │   └────────┘
                  └────────┘
```

---

## 🧪 6. Testing Checklist for Validators

### 6.1 Access Control

- [ ] **Test:** Validator can login and access validation queue
- [ ] **Test:** Validator cannot access admin-only pages
- [ ] **Test:** Validator cannot access PA Staff-only pages
- [ ] **Test:** Validator dashboard loads correctly

### 6.2 Validation Queue

- [ ] **Test:** Queue displays submissions with correct statuses (pending, on hold, flagged)
- [ ] **Test:** Queue statistics are accurate
- [ ] **Test:** Status filter works (all/pending/on hold/flagged)
- [ ] **Test:** Date range filter works
- [ ] **Test:** PA Staff filter works
- [ ] **Test:** Search by submission ID works
- [ ] **Test:** Sorting works (newest/oldest/PA staff/status)
- [ ] **Test:** Pagination displays 50 items per page
- [ ] **Test:** High priority indicator shows for submissions over 7 days old
- [ ] **Test:** "Assigned to Me" filter displays correctly

### 6.3 Bulk Actions

- [ ] **Test:** Can select multiple submissions using checkboxes
- [ ] **Test:** "Assign to Self" assigns submissions correctly
- [ ] **Test:** "Assign to Other" shows validator dropdown
- [ ] **Test:** Assigning to other validator creates notification
- [ ] **Test:** Activity log records assignment
- [ ] **Test:** Cannot assign already-processed submissions
- [ ] **Test:** Success/error messages display correctly

### 6.4 Validation Workspace

- [ ] **Test:** Workspace loads with all data sections
- [ ] **Test:** Collection metadata displays correctly
- [ ] **Test:** Data quality score is calculated
- [ ] **Test:** Quality issues are listed
- [ ] **Test:** Record-specific data is parsed correctly (wildlife/resource use/disturbance/landscape)
- [ ] **Test:** Species verification works (matched/not matched)
- [ ] **Test:** Spatial analysis shows location on map
- [ ] **Test:** GPS accuracy is displayed
- [ ] **Test:** Media attachments display in gallery
- [ ] **Test:** Validation history timeline is shown
- [ ] **Test:** Previous/Next navigation works
- [ ] **Test:** Navigation shows correct position (X of Y)

### 6.5 Validation Actions

- [ ] **Test:** Can approve submission with remarks
- [ ] **Test:** Can reject submission (remarks required)
- [ ] **Test:** Cannot reject without remarks
- [ ] **Test:** Can put submission on hold with remarks
- [ ] **Test:** Can flag submission for review
- [ ] **Test:** Validation status updates in database
- [ ] **Test:** Validation history entry is created
- [ ] **Test:** PA Staff receives notification of validation decision
- [ ] **Test:** Approved submissions trigger observation record creation
- [ ] **Test:** Rejected submissions are excluded from reports
- [ ] **Test:** After validation, redirects to next submission or queue

### 6.6 Validation History

- [ ] **Test:** History displays all validator's past actions
- [ ] **Test:** Statistics cards show correct counts
- [ ] **Test:** Approval rate is calculated correctly
- [ ] **Test:** Date filter works
- [ ] **Test:** Action filter works (approved/rejected/on hold/flagged)
- [ ] **Test:** Search works
- [ ] **Test:** Export to CSV downloads correctly
- [ ] **Test:** CSV contains all filtered records
- [ ] **Test:** Pagination works (50 per page)

### 6.7 Performance

- [ ] **Test:** Queue loads quickly with 100+ submissions
- [ ] **Test:** Workspace loads within 2 seconds
- [ ] **Test:** Species verification is fast
- [ ] **Test:** Map renders correctly
- [ ] **Test:** Image loading is optimized

---

## 🛠️ 7. Troubleshooting

### Common Issues

#### Issue: "Not authorized to validate"

**Cause:** User role is not `validator` or account is pending  
**Solution:** Check user role and account approval status

```bash
docker-compose exec web python manage.py shell
>>> from users.models import User
>>> user = User.objects.get(username='validator_name')
>>> user.role
'validator'  # Should be 'validator'
>>> user.account_status
'approved'   # Should be 'approved'
```

#### Issue: Submission not appearing in queue

**Possible Causes:**
1. Submission already validated
2. Filters hiding the submission
3. Submission status is not pending/on hold/flagged

**Solution:**
- Clear all filters
- Check submission status in Django admin
- Verify submission exists: `DataSubmission.objects.filter(submission_id='...')`

#### Issue: Validation action fails (AJAX error)

**Causes:**
- CSRF token missing
- Network error
- Permission issue

**Solution:**
```javascript
// Check browser console for errors
// Ensure CSRF token is included in AJAX request
$.ajaxSetup({
    headers: { "X-CSRFToken": getCookie('csrftoken') }
});
```

#### Issue: Species verification not working

**Cause:** Species name format mismatch

**Solution:**
- Check spelling of species name
- Ensure name is in `Genus species` format
- Verify species exists in `FaunaChecklist` or `FloraChecklist`

```python
# Check if species exists
from species.models import FaunaChecklist
FaunaChecklist.objects.filter(scientific_name__icontains='macaca').values('scientific_name')
```

#### Issue: Map not displaying

**Cause:** Missing GPS coordinates or invalid data

**Solution:**
- Check if `_geolocation` field exists in form_data
- Verify coordinate format: `[latitude, longitude, altitude, accuracy]`
- Ensure Leaflet.js library is loaded

---

## 📚 8. Best Practices

### 8.1 Validation Guidelines

**Quality Standards:**
1. **GPS Accuracy:** Prefer < 50m accuracy. Flag if > 100m.
2. **Species ID:** Cross-reference with photos and field notes. When uncertain, flag for expert review.
3. **Data Completeness:** Check that required fields are filled.
4. **Consistency:** Verify that observations make sense (e.g., marine species shouldn't be far from water).

**Communication:**
- Always provide clear, constructive feedback in remarks
- Be specific about what needs correction
- Suggest resources or references when helpful
- Acknowledge good quality submissions

**Efficiency:**
- Use bulk assignment to organize workload
- Process similar submission types together
- Use keyboard shortcuts (if available)
- Leverage previous/next navigation to work through queue quickly

### 8.2 Prioritization

**High Priority:**
1. Submissions > 7 days old
2. Flagged submissions
3. Rare/endangered species sightings
4. Submissions from high-value monitoring areas

**Normal Priority:**
5. Recent submissions (< 7 days)
6. Routine observations

### 8.3 Decision-Making Framework

**Approve when:**
- ✅ Data quality score > 70
- ✅ Species identified correctly (if wildlife)
- ✅ GPS coordinates reasonable
- ✅ No obvious errors or inconsistencies

**Reject when:**
- ❌ Clear species misidentification
- ❌ GPS coordinates impossible (ocean, different country)
- ❌ Duplicate submission
- ❌ Incomplete critical data

**Put On Hold when:**
- ⏸️ Need clarification from PA Staff
- ⏸️ GPS accuracy poor but otherwise good
- ⏸️ Photo quality insufficient for species verification

**Flag when:**
- 🚩 Rare/exceptional sighting needs expert confirmation
- 🚩 Data anomaly requiring investigation
- 🚩 Potential new species record
- 🚩 Training example for other validators

---

## 🔗 9. Related Documentation

- **Access Control:** [RBAC_DOCUMENTATION.md](../architecture/RBAC_DOCUMENTATION.md)
- **Permission System:** [PERMISSION_SYSTEM.md](../architecture/PERMISSION_SYSTEM.md)
- **Validation Workflow:** [VALIDATION_WORKFLOW.md](../../dashboard/docs/VALIDATION_WORKFLOW.md)
- **Testing:** [VALIDATOR_TEST_CHECKLIST.md](../../dashboard/docs/VALIDATOR_TEST_CHECKLIST.md)
- **Data Quality:** [DATA_QUALITY_STANDARDS.md](../../dashboard/docs/DATA_QUALITY_STANDARDS.md)

---

## 📞 Support

**For Validators:**
- Contact Senior Validator or System Administrator for complex cases
- Use flag feature to escalate difficult decisions
- Refer to species identification guides in system

**For Developers:**
- Reference: [views_validation.py](dashboard/views_validation.py)
- API: [views_api.py](dashboard/views_api.py) (`api_validate` function)
- Tests: [test_validator_functionality.py](dashboard/tests/test_validator_functionality.py)
- Templates: `dashboard/templates/dashboard/validation_*.html`

---

*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Capstone Documentation Project*
