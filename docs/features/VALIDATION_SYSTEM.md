# Validation Queue & Workspace Documentation

## 📋 Overview

The Validation Queue & Workspace is the core feature of PRISM-Matutum where validators review, approve, reject, or request clarification on biodiversity data submissions from PA Staff field collectors.

### Key Components

- **Validation Queue**: List view with filtering, sorting, and bulk assignment
- **Validation Workspace**: Detailed review interface for individual submissions
- **Validation History**: Complete audit trail of validation decisions
- **Validation Actions**: Approve, Reject, Hold, Flag

---

## 🏗️ Architecture

### Core Components

```
dashboard/views_validation.py
├── ValidationQueueView         # Main queue listing
├── ValidationWorkspaceView      # Detailed review workspace
├── ValidationHistoryView        # Validator's history
└── SubmissionAuditTrailView    # Submission audit trail

api/workflow_services.py
└── BMSWorkflowService          # Validation workflow logic

dashboard/utils.py
└── ValidationActionProcessor    # Action processing utility
```

### Data Flow

```
Field Data (KoboToolbox)
    ↓
DataSubmission (status='---')
    ↓
Validation Queue → Filter/Sort → Select Submission
    ↓
Validation Workspace → Review Data → Validation Action
    ↓
├─ Approve → BMSDataProcessor → WildlifeObservation/DisturbanceRecord/etc.
├─ Reject → Notify PA Staff → Stays in DataSubmission
├─ Hold → Notify PA Staff → Remains in Queue
└─ Flag → Assign to Supervisor → High Priority

ValidationHistory → Complete Audit Trail
```

---

## 🎯 Validation Status Workflow

### Status Progression

```
┌──────────────────────────────────────────────────────────┐
│                    INITIAL STATE                          │
│                   status = '---'                          │
│                (Not Yet Validated)                        │
└────────────┬─────────────────────────────────────────────┘
             │
        ┌────┴────┐
        ▼         ▼         ▼         ▼
   [APPROVE] [REJECT] [HOLD]  [FLAG]
        │         │         │         │
        ▼         │         │         │
validation_status_approved    │         │
  (PROCESSED)                 │         │
                              ▼         │
              validation_status_not_approved
                (REJECTED - FINAL)      │
                                        ▼
                         validation_status_on_hold
                        (AWAITING CLARIFICATION)
                                        │
                        Can be re-reviewed ─┐
                                           ↓
                                      [APPROVE/REJECT]
                                           │
                                           ▼
                                    (FINAL STATUS)
                                    
                                        │
                                        ▼
                                    flagged
                              (NEEDS SUPERVISOR REVIEW)
                                        │
                        Assigned validator reviews ─┐
                                                    ↓
                                            [APPROVE/REJECT]
                                                    │
                                                    ▼
                                            (FINAL STATUS)
```

### Status Codes

| Code | Display Name | Description | Can Change? |
|------|-------------|-------------|-------------|
| `---` | Not Validated | Initial state after submission | Yes |
| `validation_status_approved` | Approved | Passed validation, processed into records | **Immutable** |
| `validation_status_not_approved` | Rejected | Failed validation, not processed | **Immutable** |
| `validation_status_on_hold` | On Hold | Awaiting PA Staff clarification | Yes (re-review) |
| `flagged` | Flagged for Review | Needs supervisor attention | Yes (supervisor reviews) |

---

## 📊 Validation Queue

### Purpose

Centralized interface for validators to:
- View all submissions requiring validation
- Filter and sort by multiple criteria
- Assign submissions (bulk or individual)
- Track queue statistics

### URL

`/dashboard/validation/queue/`

### Access Control

**Required Permission**: `is_validator()` or `is_staff`

**Implemented By**: `ValidatorAccessMixin`

### Features

#### 1. Queue Statistics Panel

Displays real-time metrics:

```python
{
    'total': 156,              # Total pending submissions
    'not_validated': 120,      # Not yet reviewed
    'on_hold': 24,             # Awaiting clarification
    'flagged': 12,             # Flagged for review
    'high_priority': 45,       # Older than 7 days or flagged
    'assigned_to_me': 32       # Assigned to current validator
}
```

#### 2. Filtering System

**Filter Options**:

| Filter | Type | Purpose |
|--------|------|---------|
| `validation_status` | Dropdown | Filter by status (Not Validated, On Hold, Flagged) |
| `submission_time_after` | DatePicker | From date |
| `submission_time_before` | DatePicker | To date |
| `username` | Text | PA Staff (submitter) |
| `assigned_staff` | Dropdown | Assigned PA Staff |
| `validated_by` | Dropdown | Assigned validator |
| `protected_area` | Dropdown | Protected Area |
| `observation_type` | Dropdown | Wildlife, Disturbance, Resource Use, Landscape |
| `priority` | Dropdown | High, Normal |

**Implementation**:

```python
from .filters import SubmissionFilter

queryset = DataSubmission.objects.filter(
    validation_status__in=['---', 'validation_status_on_hold', 'flagged']
).select_related('validated_by').prefetch_related('assigned_staff')

filterset = SubmissionFilter(request.GET, queryset=queryset)
filtered_queryset = filterset.qs
```

#### 3. Sorting Options

| Sort By | Field |
|---------|-------|
| Newest First | `-submission_time` |
| Oldest First | `submission_time` |
| PA Staff A-Z | `username` |
| PA Staff Z-A | `-username` |
| Status A-Z | `validation_status` |
| Status Z-A | `-validation_status` |

#### 4. Bulk Actions

**Available Actions**:
- **Assign to Self**: Assign selected submissions to current validator
- **Assign to Other Validator**: Assign selected submissions to another validator

**Process**:

```python
def assign_submissions(self, submission_ids, validator):
    # Get assignable submissions
    assignable = DataSubmission.objects.filter(
        id__in=submission_ids,
        validation_status__in=['---', 'validation_status_on_hold', 'flagged']
    )
    
    # Update assignments
    updated_count = assignable.update(
        validated_by=validator,
        last_updated=timezone.now()
    )
    
    # Log activity
    UserActivityLog.objects.create(
        user=request.user,
        activity_type='validation',
        description=f"Assigned {updated_count} submissions to {validator.username}",
        metadata={'submission_ids': list(submission_ids), 'assigned_to': validator.id}
    )
    
    # Notify assigned validator (if not self)
    if validator != request.user:
        UserNotification.objects.create(
            user=validator,
            notification_type='assignment',
            title='New Validation Assignment',
            message=f'{request.user.get_full_name()} assigned {updated_count} submissions to you.'
        )
```

#### 5. Pagination

- **Items per page**: 50 submissions
- **Navigation**: Previous/Next buttons with page numbers

---

## 🔍 Validation Workspace

### Purpose

Comprehensive review interface providing:
- Complete submission data
- Data quality assessment
- Species verification
- Spatial analysis
- Media review
- Validation actions

### URL

`/dashboard/validation/workspace/<submission_id>/`

### Access Control

**Required**: `is_validator()` or `is_staff`

### Interface Sections

#### 1. Collection Metadata

**Displays**:
- Data Collector (PA Staff username)
- Collection Date/Time (start/end)
- Device ID
- Protected Area
- Monitoring Location
- GPS Coordinates (with map)

**Example**:

```python
collection_metadata = {
    'username': 'pa_staff_001',
    'today': '2026-01-15',
    'start': '08:30:00',
    'end': '14:45:00',
    'deviceid': 'PRISM_TABLET_05',
    'protected_area': 'Mount Apo Natural Park',
    'monitoring_location': 'Transect Alpha',
    'gps_latitude': 7.0167,
    'gps_longitude': 125.2833,
    'gps_accuracy': 5.2  # meters
}
```

#### 2. Record-Specific Data

Structured display based on `record_type`:

**Wildlife Observation**:
- Species identification (scientific & common names)
- Observation method (Direct sighting, Call/Sound, Track/Sign, Camera trap)
- Individual count
- Sex/Age distribution
- Behavior notes
- Habitat type

**Disturbance Record**:
- Disturbance type
- Severity level
- Affected area (hectares)
- Description
- Immediate actions taken

**Resource Use Incident**:
- Resource type
- Use intensity
- Estimated quantity
- Actors involved
- Enforcement actions

**Landscape Monitoring**:
- Vegetation type
- Cover percentage
- Condition assessment
- Notable changes

#### 3. Data Quality Assessment

**Automated Checks**:

```python
data_quality = {
    'score': 85,  # 0-100 scale
    'issues': [
        {
            'severity': 'warning',
            'category': 'GPS Accuracy',
            'message': 'GPS accuracy is 15m (recommended: <10m)',
            'field': 'gps_accuracy'
        },
        {
            'severity': 'info',
            'category': 'Species Verification',
            'message': 'Species auto-matched with 95% confidence',
            'field': 'species'
        }
    ],
    'checks_passed': 12,
    'checks_total': 14
}
```

**Quality Score Breakdown**:
- 90-100: Excellent
- 75-89: Good
- 60-74: Fair (review recommended)
- <60: Poor (likely reject)

#### 4. Species Verification

**For Wildlife Observations**:

```python
species_verification = {
    'match_found': True,
    'species': {
        'id': 234,
        'scientific_name': 'Pithecophaga jefferyi',
        'common_names': ['Philippine Eagle', 'Haring Ibon'],
        'conservation_status': 'Critically Endangered',
        'endemicity': 'Endemic to Philippines',
        'confidence': 0.95  # If auto-matched
    },
    'taxonomy': {
        'kingdom': 'Animalia',
        'phylum': 'Chordata',
        'class': 'Aves',
        'order': 'Accipitriformes',
        'family': 'Accipitridae',
        'genus': 'Pithecophaga'
    },
    'media_available': True,
    'observation_count': 47  # Historical observations of this species
}
```

**Actions**:
- View species details modal
- View species observation history
- Flag species identification as questionable

#### 5. Spatial Analysis

**GPS Validation**:
- Coordinates plotted on interactive map
- Protected area boundary overlay
- Distance from monitoring location
- Elevation check
- Habitat suitability (for species observations)

**Spatial Quality Indicators**:
```python
spatial_analysis = {
    'within_protected_area': True,
    'distance_from_monitoring_location': 245,  # meters
    'elevation': 1850,  # meters
    'habitat_match': 'montane_forest',  # Matches species habitat
    'gps_accuracy': 5.2,  # meters
    'coordinate_precision': 'high',  # high/medium/low
    'suspicious': False  # Flagged if coordinates seem impossible
}
```

#### 6. Media Attachments

**Photo Gallery**:
- Thumbnail grid with lightbox
- EXIF data (if available)
- GPS from photo metadata
- Timestamp verification

**Media Validation**:
- Check if photos support observation
- Verify timestamp matches collection time
- Assess image quality

#### 7. Validation History

**Previous Actions on This Submission**:

```python
validation_history = [
    {
        'changed_by': 'validator_john',
        'action': 'Put on Hold',
        'previous_status': '---',
        'new_status': 'validation_status_on_hold',
        'remarks': 'Need clarification on GPS coordinates',
        'changed_at': '2026-01-10 14:30:22'
    },
    {
        'changed_by': 'pa_staff_001',
        'action': 'Updated',
        'remarks': 'Verified GPS using backup device',
        'changed_at': '2026-01-12 09:15:00'
    }
]
```

#### 8. Navigation Controls

**Quick Navigation**:
- Previous submission in queue
- Next submission in queue
- Back to queue
- Skip to next assigned to me

---

## ⚖️ Validation Actions

### Action Panel

**Location**: Bottom of Validation Workspace

**Form Elements**:
```html
<form method="post" id="validation-form">
    <select name="action">
        <option value="approve">✅ Approve & Process</option>
        <option value="reject">❌ Reject</option>
        <option value="hold">⏸️ Put on Hold</option>
        <option value="flag">🚩 Flag for Review</option>
    </select>
    
    <textarea name="notes" placeholder="Validation notes (required for reject/hold)"></textarea>
    
    <select name="flag_to_user" id="flag-user-select" style="display:none;">
        <!-- Populated with other validators -->
    </select>
    
    <button type="submit">Submit Validation</button>
</form>
```

### Action Details

#### 1. APPROVE & PROCESS

**When to Use**:
- All data quality checks passed
- Species identification verified
- GPS coordinates validated
- Media supports observations
- Data is complete and reliable

**Process**:

```python
def _process_approve(submission, notes, user, previous_status):
    # 1. Update status
    submission.validation_status = 'validation_status_approved'
    submission.validated_by = user
    submission.validated_at = timezone.now()
    submission.validation_remarks = notes
    submission.save()
    
    # 2. Sync to KoboToolbox
    sync_service.update_validation_status(
        submission.asset_uid,
        submission.submission_id,
        'validation_status_approved',
        user,
        notes
    )
    
    # 3. Create validation history
    ValidationHistory.objects.create(
        submission=submission,
        previous_status=previous_status,
        new_status='validation_status_approved',
        changed_by=user,
        remarks=notes
    )
    
    # 4. Process into observation records
    processor = BMSDataProcessor()
    success, message = processor.process_submission(submission)
    
    if success:
        # 5. Commit to database
        commit_service = BMSDataCommitService()
        commit_service.commit_submission(submission)
        
        submission.database_committed = True
        submission.committed_at = timezone.now()
        submission.committed_by = user
        submission.save()
        
        # 6. Notify PA Staff
        UserNotification.objects.create(
            user=pa_staff,
            notification_type='validation_approved',
            title='Submission Approved',
            message=f'Your submission has been validated and processed successfully.',
            related_submission_id=submission.id
        )
        
        # 7. Log activity
        UserActivityLog.objects.create(
            user=user,
            activity_type='validation',
            description=f'Approved submission {submission.submission_id}',
            metadata={'submission_id': submission.id, 'action': 'approve'}
        )
        
        return True, "Submission approved and processed successfully"
    else:
        return False, f"Approval failed: {message}"
```

**Result**:
- Creates `WildlifeObservation`, `DisturbanceRecord`, `ResourceUseIncident`, or `LandscapeMonitoring` record
- Status becomes **immutable**
- PA Staff receives approval notification
- Validator gains approval credit

#### 2. REJECT

**When to Use**:
- Critical data quality issues
- Clearly incorrect species identification
- Invalid GPS coordinates
- Media contradicts data
- Fundamentally unreliable data

**Required**: Detailed rejection notes (minimum 10 characters)

**Process**:

```python
def _process_reject(submission, notes, user, previous_status):
    # Validation: notes required
    if not notes or len(notes.strip()) < 10:
        raise ValueError("Rejection notes must be at least 10 characters")
    
    # 1. Update status
    submission.validation_status = 'validation_status_not_approved'
    submission.validated_by = user
    submission.validated_at = timezone.now()
    submission.validation_remarks = notes
    submission.rejection_reason = notes  # Specific field
    submission.save()
    
    # 2. Sync to KoboToolbox
    sync_service.update_validation_status(
        submission.asset_uid,
        submission.submission_id,
        'validation_status_not_approved',
        user,
        notes
    )
    
    # 3. Create validation history
    ValidationHistory.objects.create(
        submission=submission,
        previous_status=previous_status,
        new_status='validation_status_not_approved',
        changed_by=user,
        remarks=notes
    )
    
    # 4. Notify PA Staff with detailed rejection reason
    UserNotification.objects.create(
        user=pa_staff,
        notification_type='validation_rejected',
        title='Submission Rejected',
        message=f'Your submission has been rejected. Reason: {notes}',
        related_submission_id=submission.id
    )
    
    # 5. Log activity
    UserActivityLog.objects.create(
        user=user,
        activity_type='validation',
        description=f'Rejected submission {submission.submission_id}',
        metadata={'submission_id': submission.id, 'action': 'reject', 'reason': notes}
    )
    
    return True, "Submission rejected"
```

**Result**:
- Status becomes **immutable**
- Data NOT processed into records
- PA Staff notified with specific reasons
- Submission remains in `DataSubmission` table for audit

#### 3. PUT ON HOLD

**When to Use**:
- Minor issues need clarification
- GPS accuracy questionable but fixable
- Species identification needs confirmation
- Additional information would help
- PA Staff can reasonably address issues

**Required**: Clear notes explaining what clarification is needed

**Process**:

```python
def _process_hold(submission, notes, user, previous_status):
    # 1. Update status
    submission.validation_status = 'validation_status_on_hold'
    submission.validated_by = user  # Optionally cleared for re-assignment
    submission.validated_at = timezone.now()
    submission.validation_remarks = notes
    submission.save()
    
    # 2. Sync to KoboToolbox
    sync_service.update_validation_status(
        submission.asset_uid,
        submission.submission_id,
        'validation_status_on_hold',
        user,
        notes
    )
    
    # 3. Create validation history
    ValidationHistory.objects.create(
        submission=submission,
        previous_status=previous_status,
        new_status='validation_status_on_hold',
        changed_by=user,
        remarks=notes
    )
    
    # 4. Notify PA Staff for clarification
    UserNotification.objects.create(
        user=pa_staff,
        notification_type='validation_required',
        title='Clarification Needed',
        message=f'Your submission needs clarification: {notes}',
        related_submission_id=submission.id
    )
    
    # 5. Log activity
    UserActivityLog.objects.create(
        user=user,
        activity_type='validation',
        description=f'Put submission {submission.submission_id} on hold',
        metadata={'submission_id': submission.id, 'action': 'hold', 'reason': notes}
    )
    
    return True, "Submission put on hold and PA Staff notified"
```

**Result**:
- Submission remains in queue
- PA Staff receives specific clarification request
- Can be re-reviewed after PA Staff responds
- Status can transition to Approved or Rejected

#### 4. FLAG FOR REVIEW

**When to Use**:
- Validator needs supervisor input
- Complex technical issue
- Policy decision required
- Training/quality assurance check

**Required**: Notes explaining why flagging + Supervisor selection

**Process**:

```python
def _process_flag(submission, notes, user, flag_to, previous_status):
    # Validation: flag_to required
    if not flag_to:
        raise ValueError("Must select a validator to flag to")
    
    # 1. Update status and reassign
    submission.validation_status = 'flagged'
    submission.validated_by = flag_to  # Assign to supervisor
    submission.validated_at = timezone.now()
    submission.validation_remarks = notes
    submission.flag_reason = notes
    submission.flagged_by = user
    submission.save()
    
    # 2. Sync to KoboToolbox
    sync_service.update_validation_status(
        submission.asset_uid,
        submission.submission_id,
        'flagged',
        user,
        notes
    )
    
    # 3. Create validation history
    ValidationHistory.objects.create(
        submission=submission,
        previous_status=previous_status,
        new_status='flagged',
        changed_by=user,
        remarks=f"Flagged to {flag_to.get_full_name()}: {notes}"
    )
    
    # 4. Notify supervisor
    UserNotification.objects.create(
        user=flag_to,
        notification_type='flagged_submission',
        title='Submission Flagged for Your Review',
        message=f'{user.get_full_name()} flagged a submission: {notes}',
        related_submission_id=submission.id
    )
    
    # 5. Log activity
    UserActivityLog.objects.create(
        user=user,
        activity_type='validation',
        description=f'Flagged submission {submission.submission_id} to {flag_to.username}',
        metadata={'submission_id': submission.id, 'action': 'flag', 'flagged_to': flag_to.id}
    )
    
    return True, f"Submission flagged to {flag_to.get_full_name()}"
```

**Result**:
- Submission assigned to supervisor
- Appears as high priority in supervisor's queue
- Original validator can see flagged status
- Supervisor reviews and makes final decision

---

## 📜 Validation History

### Purpose

Complete audit trail of:
- All validation decisions made by a validator
- Performance metrics
- Approval/rejection statistics

### URL

`/dashboard/validation/history/`

### Display Columns

| Column | Data |
|--------|------|
| Submission ID | Link to workspace |
| Status Change | Previous → New status |
| Timestamp | Date and time of action |
| PA Staff | Data collector |
| Remarks | Validation notes |
| Actions | View details, Re-open (if on hold) |

### Filtering

- Date range
- Status changes (Approved, Rejected, On Hold, Flagged)
- PA Staff (submitter)

### Export

**Format**: CSV

**Columns**:
```csv
Submission ID, Previous Status, New Status, Changed By, Changed At, PA Staff, Remarks
12345, ---, validation_status_approved, validator_john, 2026-01-15 14:30:00, pa_staff_001, "All checks passed"
```

### Performance Metrics

Calculated from validation history:

```python
validator_metrics = {
    'total_validated': 487,
    'approved': 392,
    'rejected': 64,
    'on_hold': 31,
    'approval_rate': 80.5,  # (approved / total) * 100
    'avg_validation_time': timedelta(minutes=12, seconds=34),
    'validations_this_week': 23,
    'validations_this_month': 95
}
```

---

## 📊 Submission Audit Trail

### Purpose

Complete history for a **specific submission**, showing all status changes and actions taken by multiple users.

### URL

`/dashboard/submission-audit/<submission_id>/`

### Display

**Timeline View**:

```
┌─────────────────────────────────────────────────────────────┐
│ Current Status: APPROVED                                    │
│ Validated By: validator_john                                │
│ Validated At: 2026-01-15 14:30:00                          │
└─────────────────────────────────────────────────────────────┘

Timeline of Changes:

📅 2026-01-15 14:30:00 - APPROVED by validator_john
   Previous Status: On Hold
   Remarks: "GPS coordinates verified, all checks passed"
   
📅 2026-01-12 09:15:00 - UPDATED by pa_staff_001
   Remarks: "Added backup GPS reading from handheld device"
   
📅 2026-01-10 14:30:22 - ON HOLD by validator_john
   Previous Status: Not Validated
   Remarks: "GPS accuracy 15m, need verification"
   
📅 2026-01-10 08:45:00 - SUBMITTED by pa_staff_001
   Initial submission from KoboToolbox
```

---

## 🧪 Testing Scenarios

### Scenario 1: Standard Approval Workflow

**Steps**:
1. Login as validator
2. Navigate to validation queue
3. Filter for "Not Validated" status
4. Select submission with ID #12345
5. Review all sections in workspace
6. Verify data quality score > 75
7. Check species auto-match confidence > 90%
8. Review GPS on map
9. Select "Approve & Process" action
10. Add notes: "All checks passed, data quality excellent"
11. Submit validation

**Expected Results**:
- ✅ Submission status changes to `validation_status_approved`
- ✅ Observation record created (`WildlifeObservation` or other)
- ✅ PA Staff receives approval notification
- ✅ ValidationHistory entry created
- ✅ Submission disappears from queue
- ✅ Validator approval count increases

### Scenario 2: Rejection Workflow

**Steps**:
1. Select submission with poor data quality (score < 60)
2. Identify critical issues (e.g., GPS coordinates impossible)
3. Select "Reject" action
4. Enter detailed rejection notes: "GPS coordinates place observation in ocean, 50km from protected area. Photo timestamp doesn't match collection time."
5. Submit validation

**Expected Results**:
- ✅ Submission status changes to `validation_status_not_approved`
- ✅ Rejection reason stored
- ✅ PA Staff receives rejection notification with specific reasons
- ✅ ValidationHistory entry created
- ✅ Submission NOT processed into records
- ✅ Submission disappears from queue

### Scenario 3: On Hold Workflow

**Steps**:
1. Select submission with questionable GPS accuracy
2. Identify issue: GPS accuracy 18m (recommended < 10m)
3. Select "Put on Hold" action
4. Enter clarification notes: "GPS accuracy is 18m. Please verify coordinates using backup device or provide explanation."
5. Submit validation

**Expected Results**:
- ✅ Submission status changes to `validation_status_on_hold`
- ✅ PA Staff receives clarification request
- ✅ ValidationHistory entry created
- ✅ Submission remains in queue with "On Hold" badge
- ✅ PA Staff can view submission and respond
- ✅ Validator can re-review after PA Staff response

### Scenario 4: Flag for Review Workflow

**Steps**:
1. Select submission with uncertain species identification
2. Species auto-match confidence only 65%
3. Select "Flag for Review" action
4. Select supervisor from dropdown
5. Enter notes: "Species identification uncertain. Photo shows bird with characteristics of both X and Y species. Need expert confirmation."
6. Submit validation

**Expected Results**:
- ✅ Submission status changes to `flagged`
- ✅ Submission assigned to selected supervisor
- ✅ Supervisor receives flagged notification
- ✅ ValidationHistory entry created
- ✅ Submission appears in supervisor's queue as high priority
- ✅ Original validator can track flagged status

### Scenario 5: Bulk Assignment

**Steps**:
1. Navigate to validation queue
2. Filter submissions by PA Staff "pa_staff_001"
3. Select 5 submissions using checkboxes
4. Choose bulk action "Assign to Self"
5. Submit

**Expected Results**:
- ✅ All 5 submissions assigned to current validator
- ✅ `validated_by` field updated
- ✅ UserActivityLog entry created
- ✅ Success message: "Successfully assigned 5 submissions to yourself"
- ✅ Submissions appear with "Assigned to Me" badge

---

## 🔧 Configuration & Settings

### Queue Settings

**Pagination**: Configurable in `ValidationQueueView.paginate_by`

```python
class ValidationQueueView(ValidatorAccessMixin, ListView):
    paginate_by = 50  # Adjust as needed
```

### High Priority Threshold

**Default**: 7 days

```python
high_priority = queryset.filter(
    Q(submission_time__lt=now - timedelta(days=7)) |
    Q(validation_status='flagged')
).count()
```

### Data Quality Thresholds

**Configured in** `dashboard/utils.py`:

```python
QUALITY_THRESHOLDS = {
    'excellent': 90,
    'good': 75,
    'fair': 60,
    'poor': 0
}
```

### Validation Notes Minimum Length

```python
MIN_NOTES_LENGTH = 10  # characters

if action in ['reject', 'hold'] and len(notes.strip()) < MIN_NOTES_LENGTH:
    raise ValueError(f"Notes must be at least {MIN_NOTES_LENGTH} characters")
```

---

## 📚 Related Documentation

- [Validation Workflow Guide](../dashboard/docs/VALIDATION_WORKFLOW.md)
- [Data Processing Services](../docs/Services.md)
- [User Roles & Permissions](../architecture/RBAC_DOCUMENTATION.md)
- [BMS Workflow](../docs/Services.md#bmeworkflowservice)

---

*Last Updated: January 6, 2026*
*Version: 1.0*
