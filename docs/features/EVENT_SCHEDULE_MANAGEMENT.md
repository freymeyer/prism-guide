# Event & Schedule Management Documentation

## 📅 Overview

The PRISM-Matutum Event & Schedule Management module provides three integrated systems for planning, organizing, and tracking field activities, community engagement, and administrative operations. It supports protected area management through comprehensive scheduling, event coordination, and community consultation tracking.

### Three-Module System
1. **Events System**: Meetings, workshops, announcements, inspections
2. **Scheduled Surveys**: Field monitoring surveys with staff assignments
3. **Focus Group Discussions (FGD)**: Community consultations with participant tracking and findings management

---

## 🏗️ Architecture

### Module Relationships
```
Protected Area Operations
    ↓
├─ Events (General Organization)
│   ├─ Meetings
│   ├─ Workshops/Training
│   ├─ Announcements
│   ├─ Field Activities
│   ├─ Inspections
│   └─ Community Engagement
│
├─ Scheduled Surveys (Field Monitoring)
│   ├─ Transect Route Assignments
│   ├─ Staff Assignments (PA Staff)
│   ├─ Time-based Status Tracking
│   └─ Submission Linkage
│
└─ FGD Sessions (Community Research)
    ├─ Session Management
    ├─ Participant Demographics
    ├─ Key Findings/Themes
    └─ Attachments (Audio, Consent, Photos)
```

### Technology Stack
- **Backend**: Django 5.2 Class-Based Views (CBV)
- **Models**: 8 models across 3 modules
- **Notifications**: Celery async tasks for user alerts
- **Calendar Views**: Month/Week/Day layout options
- **File Uploads**: Audio recordings, consent forms, photo evidence

---

## 📊 Models & Database Schema

## 1. Events System

### Event Model
**Location**: `dashboard/models.py`

```python
class Event(models.Model):
    EVENT_TYPES = [
        ('meeting', 'Meeting'),
        ('workshop', 'Workshop/Training'),
        ('announcement', 'Announcement'),
        ('field_activity', 'Field Activity'),
        ('inspection', 'Inspection'),
        ('community', 'Community Engagement'),
        ('other', 'Other'),
    ]
    
    title = models.CharField(max_length=255)
    event_type = models.CharField(max_length=30, choices=EVENT_TYPES)
    description = models.TextField(blank=True)
    start_datetime = models.DateTimeField()
    end_datetime = models.DateTimeField(null=True, blank=True)
    location = models.CharField(max_length=255, blank=True)
    
    protected_area = models.ForeignKey('users.ProtectedArea', ...)
    organizer = models.ForeignKey(User, related_name='organized_events')
    attendees = models.ManyToManyField(User, related_name='attended_events')
    
    is_public = models.BooleanField(default=True)
    created_by = models.ForeignKey(User, related_name='created_events')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

**Indexes**:
- `event_type + start_datetime` (DESC)
- `protected_area + start_datetime` (DESC)
- `is_public + start_datetime` (DESC)

**Properties**:
```python
@property
def is_past(self) -> bool:
    """Check if event has already occurred"""
    return self.start_datetime < timezone.now()

@property
def is_ongoing(self) -> bool:
    """Check if event is currently happening"""
    now = timezone.now()
    return self.start_datetime <= now <= (self.end_datetime or self.start_datetime)
```

---

## 2. Scheduled Surveys System

### ScheduledSurvey Model
**Location**: `api/models/schedules.py`

```python
class ScheduledSurvey(models.Model):
    STATUS_CHOICES = [
        ('scheduled', 'Scheduled'),
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
        ('overdue', 'Overdue'),
        ('cancelled', 'Cancelled'),
    ]
    
    transect_route = models.ForeignKey('TransectRoute', on_delete=models.CASCADE)
    assigned_staff = models.ManyToManyField(User, related_name='scheduled_surveys')
    
    start_time = models.DateTimeField()
    end_time = models.DateTimeField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='scheduled')
    
    related_submissions = models.ManyToManyField('DataSubmission', blank=True)
    notes = models.TextField(blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

**Key Methods**:
```python
def check_and_update_status(self):
    """Auto-update status based on current time"""
    now = timezone.now()
    
    if self.status == 'scheduled' and now >= self.start_time:
        self.status = 'in_progress'
        self.save()
    elif self.status == 'in_progress' and now > self.end_time:
        # Check if submissions were completed
        if self.related_submissions.exists():
            self.status = 'completed'
        else:
            self.status = 'overdue'
        self.save()
```

**Signals**: Post-save signal creates notifications for assigned staff.

---

## 3. Focus Group Discussion (FGD) System

### FGDSession Model
**Location**: `api/models/fgd.py`

```python
class FGDSession(models.Model):
    STATUS_CHOICES = [
        ('scheduled', 'Scheduled'),
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
        ('cancelled', 'Cancelled'),
    ]
    
    title = models.CharField(max_length=255)
    date_conducted = models.DateField()
    start_time = models.TimeField(null=True, blank=True)
    end_time = models.TimeField(null=True, blank=True)
    location = models.CharField(max_length=255)
    
    protected_area = models.ForeignKey('users.ProtectedArea', ...)
    barangay = models.ForeignKey('users.Barangay', null=True, blank=True)
    
    facilitator = models.ForeignKey(User, related_name='facilitated_fgd_sessions')
    co_facilitators = models.ManyToManyField(User, related_name='co_facilitated_fgd_sessions')
    participant_count = models.PositiveIntegerField(default=0)
    
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='scheduled')
    objectives = models.TextField(blank=True)
    methodology = models.TextField(blank=True)
    notes = models.TextField(blank=True)
    summary = models.TextField(blank=True)
    
    audio_recording = models.FileField(upload_to='fgd/audio/%Y/%m/', blank=True)
    consent_form = models.FileField(upload_to='fgd/consent/%Y/%m/', blank=True)
```

**Properties**:
```python
@property
def duration_minutes(self) -> Optional[int]:
    """Calculate session duration in minutes"""
    if self.start_time and self.end_time:
        start = datetime.combine(self.date_conducted, self.start_time)
        end = datetime.combine(self.date_conducted, self.end_time)
        if end < start:  # Handle midnight crossing
            end += timedelta(days=1)
        return int((end - start).total_seconds() / 60)
    return None

@property
def finding_count(self) -> int:
    return self.findings.count()

@property
def high_priority_findings(self):
    return self.findings.filter(priority='high')
```

### FGDParticipant Model
**Location**: `api/models/fgd.py`

```python
class FGDParticipant(models.Model):
    GENDER_CHOICES = [
        ('male', 'Male'), ('female', 'Female'), ('other', 'Other'),
        ('prefer_not_to_say', 'Prefer not to say'),
    ]
    
    AGE_GROUP_CHOICES = [
        ('under_18', 'Under 18'), ('18-25', '18-25'), ('26-35', '26-35'),
        ('36-45', '36-45'), ('46-55', '46-55'), ('56-65', '56-65'),
        ('over_65', 'Over 65'),
    ]
    
    STAKEHOLDER_TYPE_CHOICES = [
        ('farmer', 'Farmer'), ('fisher', 'Fisher'),
        ('forest_user', 'Forest User'), ('indigenous', 'Indigenous Community Member'),
        ('local_official', 'Local Government Official'),
        ('teacher', 'Teacher/Educator'), ('student', 'Student'),
        ('business', 'Business Owner'), ('ngo', 'NGO Representative'),
        ('other', 'Other'),
    ]
    
    session = models.ForeignKey(FGDSession, related_name='participants')
    name = models.CharField(max_length=255, blank=True)  # Optional for privacy
    gender = models.CharField(max_length=20, choices=GENDER_CHOICES, blank=True)
    age_group = models.CharField(max_length=20, choices=AGE_GROUP_CHOICES, blank=True)
    occupation = models.CharField(max_length=100, blank=True)
    stakeholder_type = models.CharField(max_length=30, choices=STAKEHOLDER_TYPE_CHOICES)
    barangay = models.ForeignKey('users.Barangay', null=True, blank=True)
    notes = models.TextField(blank=True)
```

**Purpose**: Captures participant demographics for representation analysis while respecting privacy (name is optional).

### FGDFinding Model
**Location**: `api/models/fgd.py`

```python
class FGDFinding(models.Model):
    CATEGORY_CHOICES = [
        ('resource_use', 'Resource Use Pattern'),
        ('threat', 'Identified Threat'),
        ('recommendation', 'Community Recommendation'),
        ('concern', 'Community Concern'),
        ('traditional_knowledge', 'Traditional Knowledge'),
        ('conservation_practice', 'Conservation Practice'),
        ('livelihood', 'Livelihood Related'),
        ('governance', 'Governance/Policy'),
        ('other', 'Other'),
    ]
    
    PRIORITY_CHOICES = [('high', 'High'), ('medium', 'Medium'), ('low', 'Low')]
    
    session = models.ForeignKey(FGDSession, related_name='findings')
    category = models.CharField(max_length=50, choices=CATEGORY_CHOICES)
    title = models.CharField(max_length=255)
    description = models.TextField()
    
    priority = models.CharField(max_length=20, choices=PRIORITY_CHOICES, default='medium')
    is_actionable = models.BooleanField(default=False)
    recommended_action = models.TextField(blank=True)
    
    # Species linkages
    related_fauna_species = models.ManyToManyField('species.FaunaChecklist', blank=True)
    related_flora_species = models.ManyToManyField('species.FloraChecklist', blank=True)
    
    # Spatial context
    location_description = models.CharField(max_length=255, blank=True)
    latitude = models.DecimalField(max_digits=10, decimal_places=7, null=True, blank=True)
    longitude = models.DecimalField(max_digits=10, decimal_places=7, null=True, blank=True)
    
    # Evidence
    supporting_quotes = models.TextField(blank=True)
    photo_evidence = models.ImageField(upload_to='fgd/findings/%Y/%m/', blank=True)
    
    # Follow-up tracking
    follow_up_required = models.BooleanField(default=False)
    follow_up_completed = models.BooleanField(default=False)
    follow_up_notes = models.TextField(blank=True)
    
    recorded_by = models.ForeignKey(User, ...)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

**Purpose**: Structured insights from FGD discussions, categorized for analysis and reporting.

### FGDAttachment Model
**Location**: `api/models/fgd.py`

```python
class FGDAttachment(models.Model):
    ATTACHMENT_TYPE_CHOICES = [
        ('photo', 'Photo'),
        ('document', 'Document'),
        ('map', 'Participatory Map'),
        ('diagram', 'Diagram/Chart'),
        ('other', 'Other'),
    ]
    
    session = models.ForeignKey(FGDSession, related_name='attachments')
    attachment_type = models.CharField(max_length=20, choices=ATTACHMENT_TYPE_CHOICES)
    title = models.CharField(max_length=255)
    file = models.FileField(upload_to='fgd/attachments/%Y/%m/')
    description = models.TextField(blank=True)
    uploaded_by = models.ForeignKey(User, ...)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## 🎯 Views & URL Patterns

## 1. Events Module

### `EventListView`
**URL**: `/validator-dashboard/events/`  
**Template**: `dashboard/event_list.html`  
**Permission**: `EventAccessMixin` (all authenticated approved users)

**Features**:
- Filter by: `event_type`, `protected_area`, `time` (upcoming/past/all)
- Visibility control: Public events + user's involved events
- Paginated (12/page)
- Summary stats: upcoming count, past count

**Query Logic**:
```python
queryset = Event.objects.filter(
    Q(is_public=True) | Q(organizer=user) | Q(attendees=user) | Q(created_by=user)
).distinct()
```

**Context Data**:
```python
{
    'events': QuerySet[Event],
    'event_types': EVENT_TYPES,
    'protected_areas': QuerySet[ProtectedArea],
    'can_manage_events': bool,  # Role-based: denr, validator, system_admin
    'upcoming_count': int,
    'past_count': int,
    'current_filters': dict
}
```

### `EventDetailView`
**URL**: `/validator-dashboard/events/<int:pk>/`  
**Template**: `dashboard/event_detail.html`  
**Permission**: `EventAccessMixin`

**Features**:
- Full event information
- Attendee list
- Organizer details
- Edit/Delete actions (if authorized)

**Context**:
```python
{
    'event': Event,
    'can_manage_events': bool,
    'is_attendee': bool,
    'is_organizer': bool
}
```

### `EventCreateView`
**URL**: `/validator-dashboard/events/create/`  
**Template**: `dashboard/event_form.html`  
**Permission**: `EventManagementMixin` (denr, validator, system_admin)

**Notification Workflow** (Async via Celery):
1. **Public Events**: Notify all users in protected area
2. **Attendee Invitations**: Send invitation notifications to selected attendees
3. **Task**: `send_event_notifications.delay(notification_type='created', event_id, user_ids)`

**Form Handling**:
```python
def form_valid(self, form):
    form.instance.created_by = self.request.user
    if not form.instance.organizer:
        form.instance.organizer = self.request.user
    
    response = super().form_valid(form)
    
    # Queue async notifications
    send_event_notifications.delay('created', event.id, ...)
    
    return response
```

### `EventUpdateView`
**URL**: `/validator-dashboard/events/<int:pk>/edit/`  
**Permission**: `EventManagementMixin`

**Change Detection**:
- Tracks original attendees and event details
- Compares changed fields: `title`, `start_datetime`, `end_datetime`, `location`, `description`, `event_type`
- Sends different notification types:
  - **Updated**: Existing attendees (if important fields changed)
  - **Invitation**: Newly added attendees

**Notification Logic**:
```python
# Detect new attendees
new_attendee_ids = current_attendees - original_attendees

# Notify existing attendees of changes
if important_fields_changed:
    send_event_notifications.delay('updated', event_id, existing_attendee_ids, changed_fields)

# Invite new attendees
if new_attendee_ids:
    send_event_notifications.delay('invitation', event_id, new_attendee_ids)
```

### `EventDeleteView`
**URL**: `/validator-dashboard/events/<int:pk>/delete/`  
**Template**: `dashboard/event_confirm_delete.html`

**Cancellation Notifications**:
- Captures event data before deletion
- Sends cancellation notifications to all attendees + organizer
- Task: `send_event_cancellation_notifications.delay(event_data)`

---

## 2. Scheduled Surveys Module

### `ScheduledSurveyCalendarView`
**URL**: `/validator-dashboard/schedules/`  
**Template**: `dashboard/schedule_calendar.html`  
**Permission**: `StaffAndValidatorAccessMixin` (pa_staff, validator, denr, system_admin)

**View Modes**:
- **Month View**: Calendar grid (42 cells, 6 weeks)
- **Week View**: Monday-Sunday layout
- **Day View**: Single day timeline

**Query Parameters**:
- `view` (default: 'month'): View type
- `date` (YYYY-MM-DD): Target date

**Auto-Status Updates**: Calls `survey.check_and_update_status()` for all displayed surveys.

**Context Data**:
```python
{
    'view_type': str,  # 'month', 'week', 'day'
    'current_date': date,
    'first_day': date,
    'last_day': date,
    'scheduled_surveys': QuerySet[ScheduledSurvey],
    'surveys_by_date': dict,  # {date: [survey_data, ...]}
    'calendar_grid': list,  # For month view: [[day_data]*7]*6
    'total_surveys': int,
    'status_counts': dict,  # {'scheduled': 5, 'in_progress': 2, ...}
    'prev_date': date,
    'next_date': date
}
```

**Calendar Grid Structure** (Month View):
```python
calendar_grid = [
    [  # Week 1
        {
            'date': date(2024, 1, 1),
            'is_current_month': True,
            'is_today': False,
            'surveys': [survey_data, ...]
        },
        # ... 6 more days
    ],
    # ... 5 more weeks
]
```

### `ScheduledSurveyCreateView`
**URL**: `/validator-dashboard/schedules/create/`  
**Template**: `dashboard/schedule_form.html`  
**Permission**: `ValidatorAccessMixin`

**Features**:
- Select transect route
- Assign multiple PA staff members
- Set start/end times
- Add notes

**Post-Save Actions**:
1. Validate assigned staff have `pa_staff` role
2. Create `UserNotification` for each assigned staff
3. Log activity in `UserActivityLog`
4. Success message: "X staff members have been notified"

**Notification Example**:
```python
UserNotification.objects.create(
    user=staff_member,
    notification_type='assignment',
    title='New Survey Assignment',
    message=f'You have been assigned to survey {route.name} on {start_time:%Y-%m-%d at %H:%M}. Notes: {notes or "No additional notes."}'
)
```

### `ScheduledSurveyDetailView`
**URL**: `/validator-dashboard/schedules/<int:pk>/`  
**Template**: `dashboard/schedule_detail.html`  
**Permission**: `StaffAndValidatorAccessMixin`

**Features**:
- Survey metadata
- Assigned staff list with submission counts
- Related submissions list
- Progress percentage calculation
- Status update form (POST)
- Transect route details (stations, segments)

**Progress Calculation**:
```python
total_expected = len(assigned_staff)  # Simplified
actual_submissions = related_submissions.count()
progress_percentage = min(100, (actual_submissions / total_expected) * 100)
```

**Status Update Handling** (POST):
```python
def post(self, request, pk):
    survey = self.get_object()
    new_status = request.POST.get('status')
    
    if new_status in valid_choices:
        old_status = survey.status
        survey.status = new_status
        survey.save()
        
        # Log activity
        UserActivityLog.objects.create(...)
        
        # Notify assigned staff
        for staff in survey.assigned_staff.all():
            UserNotification.objects.create(
                user=staff,
                notification_type='system',
                title='Survey Status Updated',
                message=f'Survey status changed to {new_status}'
            )
```

### `ScheduledSurveyUpdateView`
**URL**: `/validator-dashboard/schedules/<int:pk>/edit/`  
**Permission**: `ValidatorAccessMixin`

**Change Tracking**:
- Detects added/removed/continuing staff
- Compares date/time/route changes
- Sends differentiated notifications:
  - **Added staff**: Assignment notification
  - **Removed staff**: Removal notification
  - **Continuing staff**: Update notification (if schedule changed)

**Activity Logging**:
```python
changes = []
if old.start_time != new.start_time:
    changes.append(f"start time: {old.start_time} → {new.start_time}")
if old.status != new.status:
    changes.append(f"status: {old.status} → {new.status}")
if len(added_staff) > 0:
    changes.append(f"added {len(added_staff)} staff")

UserActivityLog.objects.create(
    activity_type='schedule_update',
    description=f'Updated survey: {", ".join(changes)}',
    metadata={'changes': changes, 'staff_added': len(added_staff), ...}
)
```

### `ScheduledSurveyDeleteView`
**URL**: `/validator-dashboard/schedules/<int:pk>/delete/` (POST only)  
**Permission**: `ValidatorAccessMixin`

**Deletion Rules**:
- **Cannot delete** if `related_submissions.exists()`
- **Alternative**: Change status to `cancelled`

**Workflow**:
1. Check for related submissions
2. Notify assigned staff of cancellation
3. Log deletion activity
4. Delete survey
5. Success message with notification count

---

## 3. Focus Group Discussion (FGD) Module

### `FGDSessionListView`
**URL**: `/validator-dashboard/fgd/`  
**Template**: `dashboard/fgd/fgd_session_list.html`  
**Permission**: `FGDAccessMixin` (all authenticated approved users)

**Role-Based Filtering**:
- **pa_staff**: Only sessions from their protected area
- **collaborator**: Only completed sessions
- **Others**: All sessions

**Filters**:
- `protected_area`, `status`, `category` (from findings), `search` (title/location)
- `date_from`, `date_to` (date range)

**Context Data**:
```python
{
    'sessions': QuerySet[FGDSession],
    'protected_areas': QuerySet[ProtectedArea],
    'status_choices': STATUS_CHOICES,
    'category_choices': CATEGORY_CHOICES,
    'current_filters': dict,
    'stats': {
        'total_sessions': int,
        'completed_sessions': int,
        'total_participants': int,
        'total_findings': int,
        'high_priority_findings': int
    },
    'can_manage': bool
}
```

### `FGDSessionDetailView`
**URL**: `/validator-dashboard/fgd/<int:pk>/`  
**Template**: `dashboard/fgd/fgd_session_detail.html`  
**Permission**: `FGDAccessMixin`

**Features**:
- Session metadata (date, time, location, facilitators)
- Participant demographics summary
- Findings grouped by category
- Priority summary (high/medium/low counts)
- Attachments list
- CRUD actions (if authorized)

**Demographics Analysis**:
```python
demographics = {
    'total': participants.count(),
    'by_gender': {'Male': 12, 'Female': 15, ...},
    'by_age_group': {'26-35': 10, '36-45': 8, ...},
    'by_stakeholder': {'Farmer': 18, 'Fisher': 5, ...}
}
```

**Findings Organization**:
```python
findings_by_category = {
    'Resource Use Pattern': [Finding1, Finding2, ...],
    'Identified Threat': [Finding3, ...],
    'Community Recommendation': [Finding4, Finding5, ...]
}
```

### `FGDSessionCreateView`
**URL**: `/validator-dashboard/fgd/create/`  
**Template**: `dashboard/fgd/fgd_session_form.html`  
**Permission**: `FGDManagementMixin` (denr, validator, system_admin)

**Form Fields**:
```python
fields = [
    'title', 'date_conducted', 'start_time', 'end_time',
    'location', 'protected_area', 'barangay',
    'facilitator', 'co_facilitators', 'participant_count',
    'status', 'objectives', 'methodology', 'notes', 'summary',
    'audio_recording', 'consent_form'
]
```

**Post-Save**: Sets `created_by` to current user, redirects to detail view.

### `FGDSessionUpdateView`
**URL**: `/validator-dashboard/fgd/<int:pk>/edit/`  
**Permission**: `FGDManagementMixin`

### `FGDSessionDeleteView`
**URL**: `/validator-dashboard/fgd/<int:pk>/delete/`  
**Template**: `dashboard/fgd/fgd_session_confirm_delete.html`  
**Permission**: `FGDManagementMixin`

**Cascade Behavior**: Deleting a session also deletes:
- All `FGDParticipant` records
- All `FGDFinding` records
- All `FGDAttachment` records

---

### `FGDParticipantCreateView`
**URL**: `/validator-dashboard/fgd/<int:session_pk>/participants/add/`  
**Template**: `dashboard/fgd/fgd_participant_form.html`  
**Permission**: `FGDManagementMixin`

**Form Fields**:
```python
fields = ['name', 'gender', 'age_group', 'occupation', 'stakeholder_type', 'barangay', 'notes']
```

**Privacy Note**: Name is optional for participant privacy protection.

---

### `FGDFindingCreateView`
**URL**: `/validator-dashboard/fgd/<int:session_pk>/findings/add/`  
**Template**: `dashboard/fgd/fgd_finding_form.html`  
**Permission**: `FGDManagementMixin`

**Form Fields**:
```python
fields = [
    'category', 'title', 'description', 'priority',
    'is_actionable', 'recommended_action',
    'related_fauna_species', 'related_flora_species',
    'location_description', 'latitude', 'longitude',
    'supporting_quotes', 'photo_evidence',
    'follow_up_required', 'follow_up_notes'
]
```

**Post-Save**: Sets `recorded_by` to current user, links to parent session.

### `FGDFindingUpdateView`
**URL**: `/validator-dashboard/fgd/findings/<int:pk>/edit/`  
**Permission**: `FGDManagementMixin`

### `FGDFindingDeleteView`
**URL**: `/validator-dashboard/fgd/findings/<int:pk>/delete/`  
**Template**: `dashboard/fgd/fgd_finding_confirm_delete.html`  
**Permission**: `FGDManagementMixin`

---

### FGD API Endpoint

#### `api_fgd_stats`
**URL**: `/validator-dashboard/api/fgd-stats/` (GET)  
**Authentication**: Required

**Response**:
```json
{
  "total_sessions": 25,
  "completed_sessions": 18,
  "scheduled_sessions": 7,
  "total_participants": 342,
  "total_findings": 156,
  "findings_by_category": {
    "Resource Use Pattern": 45,
    "Identified Threat": 32,
    "Community Recommendation": 50,
    "Traditional Knowledge": 15,
    ...
  },
  "findings_by_priority": {
    "high": 38,
    "medium": 74,
    "low": 44
  },
  "actionable_findings": 62,
  "pending_followups": 18
}
```

**Usage**: Powers FGD dashboard widgets and analytics.

---

## 🔐 Security & Permissions

### Access Control Matrix

| View | Permission Mixin | Roles Allowed |
|------|-----------------|---------------|
| **Events** |
| `EventListView` | `EventAccessMixin` | All authenticated approved users |
| `EventDetailView` | `EventAccessMixin` | All authenticated approved users |
| `EventCreateView` | `EventManagementMixin` | denr, validator, system_admin |
| `EventUpdateView` | `EventManagementMixin` | denr, validator, system_admin |
| `EventDeleteView` | `EventManagementMixin` | denr, validator, system_admin |
| **Scheduled Surveys** |
| `ScheduledSurveyCalendarView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |
| `ScheduledSurveyCreateView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `ScheduledSurveyDetailView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |
| `ScheduledSurveyUpdateView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `ScheduledSurveyDeleteView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| **FGD Sessions** |
| `FGDSessionListView` | `FGDAccessMixin` | All authenticated approved users |
| `FGDSessionDetailView` | `FGDAccessMixin` | All authenticated approved users |
| `FGDSessionCreateView` | `FGDManagementMixin` | denr, validator, system_admin |
| `FGDSessionUpdateView` | `FGDManagementMixin` | denr, validator, system_admin |
| `FGDSessionDeleteView` | `FGDManagementMixin` | denr, validator, system_admin |
| `FGDParticipantCreateView` | `FGDManagementMixin` | denr, validator, system_admin |
| `FGDFindingCreateView` | `FGDManagementMixin` | denr, validator, system_admin |
| `FGDFindingUpdateView` | `FGDManagementMixin` | denr, validator, system_admin |
| `FGDFindingDeleteView` | `FGDManagementMixin` | denr, validator, system_admin |

### Event Visibility Rules
- **Public Events** (`is_public=True`): Visible to all users
- **Private Events** (`is_public=False`): Visible only to:
  - Organizer
  - Attendees
  - Creator

### FGD Access Rules
- **PA Staff**: Can only view FGD sessions from their assigned protected area
- **Collaborators**: Can only view completed FGD sessions (read-only)
- **DENR/Validators/Admins**: Full access to all sessions

---

## 🧪 Testing Scenarios

### 1. Event Creation with Notifications
```python
# Create public event
event = Event.objects.create(
    title="Biodiversity Workshop",
    event_type='workshop',
    start_datetime=timezone.now() + timedelta(days=7),
    protected_area=pa,
    organizer=validator_user,
    is_public=True,
    created_by=validator_user
)
event.attendees.add(pa_staff_user, denr_user)

# Verify async notification queued
from dashboard.tasks import send_event_notifications
# Mock: assert send_event_notifications.delay.called_with('created', event.id, ...)

# Verify notifications created
assert UserNotification.objects.filter(
    notification_type='invitation',
    user__in=[pa_staff_user, denr_user]
).exists()
```

### 2. Event Update with Change Detection
```python
# Update event time
event.start_datetime = timezone.now() + timedelta(days=10)
event.save()

# In EventUpdateView, changed_fields should include 'start_datetime'
# Existing attendees should receive 'updated' notification
```

### 3. Scheduled Survey Auto-Status Update
```python
# Create scheduled survey
survey = ScheduledSurvey.objects.create(
    transect_route=route,
    start_time=timezone.now() - timedelta(hours=1),  # Started 1 hour ago
    end_time=timezone.now() + timedelta(hours=2),  # Ends in 2 hours
    status='scheduled'
)

# Trigger auto-update
survey.check_and_update_status()
survey.refresh_from_db()

assert survey.status == 'in_progress'
```

### 4. Scheduled Survey Overdue Detection
```python
survey = ScheduledSurvey.objects.create(
    start_time=timezone.now() - timedelta(days=2),
    end_time=timezone.now() - timedelta(days=1),  # Ended yesterday
    status='in_progress'
)

survey.check_and_update_status()
survey.refresh_from_db()

# If no submissions, should be overdue
if not survey.related_submissions.exists():
    assert survey.status == 'overdue'
```

### 5. Calendar Grid Generation
```python
view = ScheduledSurveyCalendarView()
first_day = date(2024, 3, 1)  # Friday
last_day = date(2024, 3, 31)  # Sunday
surveys_by_date = {}

calendar_grid = view.generate_calendar_grid(first_day, last_day, surveys_by_date)

assert len(calendar_grid) == 5  # 5 weeks
assert len(calendar_grid[0]) == 7  # 7 days per week

# First cell should be Monday before March 1
assert calendar_grid[0][0]['date'] == date(2024, 2, 26)
assert not calendar_grid[0][0]['is_current_month']

# March 1 (Friday) should be in first week, index 4
assert calendar_grid[0][4]['date'] == date(2024, 3, 1)
assert calendar_grid[0][4]['is_current_month']
```

### 6. FGD Participant Demographics Aggregation
```python
session = FGDSession.objects.create(
    title="Community Consultation",
    date_conducted=date.today(),
    protected_area=pa,
    facilitator=validator_user
)

# Add participants
FGDParticipant.objects.create(
    session=session, gender='male', age_group='26-35', stakeholder_type='farmer'
)
FGDParticipant.objects.create(
    session=session, gender='female', age_group='36-45', stakeholder_type='farmer'
)
FGDParticipant.objects.create(
    session=session, gender='female', age_group='26-35', stakeholder_type='fisher'
)

# Aggregate demographics (as in FGDSessionDetailView)
demographics = {
    'by_gender': {},
    'by_age_group': {},
    'by_stakeholder': {}
}

for p in session.participants.all():
    if p.gender:
        key = p.get_gender_display()
        demographics['by_gender'][key] = demographics['by_gender'].get(key, 0) + 1

assert demographics['by_gender'] == {'Male': 1, 'Female': 2}
assert demographics['by_stakeholder'] == {'Farmer': 2, 'Fisher': 1}
```

### 7. FGD Finding Priority Filtering
```python
session = FGDSession.objects.create(...)

# Add findings with different priorities
FGDFinding.objects.create(
    session=session,
    category='threat',
    title='Illegal logging increasing',
    priority='high'
)
FGDFinding.objects.create(
    session=session,
    category='recommendation',
    title='Increase patrols',
    priority='medium'
)

high_priority = session.findings.filter(priority='high')
assert high_priority.count() == 1
assert high_priority.first().title == 'Illegal logging increasing'

# Test priority summary (as in view)
priority_summary = {
    'high': session.findings.filter(priority='high').count(),
    'medium': session.findings.filter(priority='medium').count(),
    'low': session.findings.filter(priority='low').count()
}
assert priority_summary == {'high': 1, 'medium': 1, 'low': 0}
```

### 8. FGD Finding with Species Linkage
```python
from species.models import FaunaChecklist

fauna = FaunaChecklist.objects.create(
    common_name="Philippine Eagle",
    scientific_name="Pithecophaga jefferyi"
)

finding = FGDFinding.objects.create(
    session=session,
    category='traditional_knowledge',
    title='Eagle nesting site identified',
    description='Community members report eagle nest in Old Growth Forest area',
    priority='high',
    latitude=6.5,
    longitude=125.0
)
finding.related_fauna_species.add(fauna)

# Verify linkage
assert finding.related_fauna_species.count() == 1
assert finding.related_fauna_species.first().common_name == "Philippine Eagle"

# Verify reverse relationship
assert fauna.fgd_findings.count() == 1
```

### 9. Event Deletion with Cascade Notifications
```python
event = Event.objects.create(
    title="Monthly Meeting",
    event_type='meeting',
    start_datetime=timezone.now() + timedelta(days=5),
    organizer=validator_user
)
event.attendees.add(pa_staff1, pa_staff2, denr_user)

attendee_ids = list(event.attendees.values_list('id', flat=True))

# Delete event
event.delete()

# Verify cancellation notifications queued
# Mock: assert send_event_cancellation_notifications.delay.called
# with event_data containing attendee_ids
```

### 10. Scheduled Survey Staff Assignment Validation
```python
# Create validator user (wrong role)
validator = User.objects.create(username='val1', role='validator')

# Attempt to assign non-PA staff to survey
form = ScheduledSurveyForm(data={
    'transect_route': route.id,
    'assigned_staff': [validator.id],  # Wrong role
    'start_time': timezone.now(),
    'end_time': timezone.now() + timedelta(hours=4)
})

# In ScheduledSurveyCreateView.post(), check:
for staff_member in form.cleaned_data['assigned_staff']:
    if staff_member.role != 'pa_staff':
        # Warning message should be displayed
        assert messages.warning.called_with(
            request,
            f"User {staff_member.username} does not have PA Staff role but was assigned to survey."
        )
```

---

## 📈 Integration with Other Modules

### Dashboard Widgets
**Events**: `get_upcoming_events_data(widget, config)`  
**Surveys**: `get_schedule_calendar_data(widget, config)`  
**FGD**: `api_fgd_stats` endpoint

### Notification System
- All three modules integrate with `UserNotification` model
- Celery async tasks for events: `send_event_notifications`, `send_event_cancellation_notifications`
- Inline notifications for surveys (schedule changes)

### Spatial Integration
- **Scheduled Surveys**: Linked to `TransectRoute` (GIS module)
- **FGD Findings**: Optional GPS coordinates for spatially-specific insights
- **Events**: Linked to `ProtectedArea`

### Species Management
- **FGD Findings**: Many-to-many relationships with `FaunaChecklist` and `FloraChecklist`
- Enables tracking community-reported species observations and traditional knowledge

### Validation Workflow
- **Scheduled Surveys**: Linked to `DataSubmission` via `related_submissions` M2M
- Status automatically updates to `completed` when submissions are validated

---

## 📚 Related Documentation

- **Database Models**: [docs/DatabaseModel.md](../DatabaseModel.md) - Full schema
- **GIS Features**: [docs/features/GIS_SPATIAL_FEATURES.md](GIS_SPATIAL_FEATURES.md) - TransectRoute integration
- **Species Management**: [docs/features/SPECIES_MANAGEMENT.md](SPECIES_MANAGEMENT.md) - Species linkages in FGD findings
- **Validation System**: [docs/features/VALIDATION_SYSTEM.md](VALIDATION_SYSTEM.md) - Submission linkage

---

## 🎓 Key Takeaways

1. **Three-Module System**: Events (organizational), Surveys (field monitoring), FGD (community research) serve distinct but complementary purposes.

2. **Auto-Status Updates**: `ScheduledSurvey.check_and_update_status()` automatically transitions surveys through lifecycle based on timestamps.

3. **Async Notifications**: Event CRUD operations use Celery tasks for scalable notification delivery to multiple users.

4. **Demographics Analysis**: FGD participant tracking enables representation analysis while protecting privacy (optional names).

5. **Finding Categorization**: 9 finding categories support structured community insight capture for analysis and reporting.

6. **Calendar Views**: Month/Week/Day layouts provide flexible scheduling visualization with status-based color coding.

7. **Role-Based Access**: PA Staff see only their area's content; collaborators have read-only completed sessions; validators/DENR/admins have full access.

8. **Change Tracking**: Event and survey updates detect changes and send differentiated notifications (added/removed/updated).

---

**Documentation Version**: 1.0  
**Last Updated**: 2024  
**Module**: Dashboard Event & Schedule Management  
**Related Files**:
- `dashboard/views_events.py` (366 lines)
- `dashboard/views_schedule.py` (590 lines)
- `dashboard/views_fgd.py` (413 lines)
- `dashboard/models.py` (Event model)
- `api/models/schedules.py` (ScheduledSurvey)
- `api/models/fgd.py` (FGDSession, FGDParticipant, FGDFinding, FGDAttachment)
