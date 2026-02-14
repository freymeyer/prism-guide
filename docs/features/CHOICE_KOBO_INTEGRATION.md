# Choice List & KoboToolbox Integration Documentation

**PRISM-Matutum Dashboard Module**  
**Section 3.8 - Feature Documentation**

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Choice Lists](#choice-lists)
4. [Individual Choices](#individual-choices)
5. [Form Choice Mappings](#form-choice-mappings)
6. [KoboToolbox Integration](#kobotoolbox-integration)
7. [Bidirectional Sync](#bidirectional-sync)
8. [User Workflows](#user-workflows)
9. [API & Services](#api--services)
10. [Testing Guide](#testing-guide)

---

## Overview

### Purpose

The Choice List & KoboToolbox Integration system provides a **bidirectional synchronization mechanism** between Django and KoboToolbox, allowing administrators to:

- Manage standardized choice lists (dropdown options) in the Django admin
- Automatically sync choices to KoboToolbox forms
- Import choices from KoboToolbox forms back to Django
- Link specific form questions to database-managed choice lists
- Maintain data consistency across field data collection and database records

### Key Benefits

✅ **Single Source of Truth**: Manage choices in one place (Django database)  
✅ **Automatic Propagation**: Changes sync to all linked Kobo forms automatically  
✅ **Dynamic Updates**: Add new species, locations, or options without editing forms  
✅ **Version Control**: Track updates with `FormUpdateLog`  
✅ **Flexible Mapping**: Link multiple forms to the same choice list

---

## System Architecture

### Data Models

```
┌─────────────────┐
│   ChoiceList    │  (Container for related choices)
│  - name         │
│  - description  │
└────────┬────────┘
         │ 1
         │
         │ N
┌────────▼────────┐
│     Choice      │  (Individual choice item)
│  - name         │  (internal value, e.g., 'seen')
│  - label        │  (display text, e.g., 'Seen (Visual observation)')
│  - order        │  (sort order)
│  - active       │  (enable/disable)
│  - fauna_species│  (optional link to species)
│  - flora_species│  (optional link to species)
│  - monitoring_  │  (optional link to location)
│    location     │
└─────────────────┘
         │
         │ referenced by
         │
┌────────▼─────────────┐
│ FormChoiceMapping    │  (Links Kobo form to ChoiceList)
│  - asset_uid         │  (Kobo form ID)
│  - question_name     │  (Kobo question using this list)
│  - choice_list       │  (FK to ChoiceList)
│  - auto_update       │  (enable auto-sync)
└──────────────────────┘
```

### File Structure

```
api/
├── models/
│   └── choices.py              # ChoiceList, Choice, FormChoiceMapping
├── form_services.py            # Kobo API interaction
├── excel_processor.py          # XLSForm parsing & modification
└── management/commands/
    └── initialize_choice_lists.py  # Populate default choices

dashboard/
├── views_admin.py              # Choice management views
├── forms.py                    # Choice forms
└── templates/dashboard/admin/
    ├── choice_form_hub.html    # Unified management interface
    ├── choice_list_*.html      # Choice list CRUD
    ├── choice_*.html           # Individual choice CRUD
    ├── form_mapping_*.html     # Mapping CRUD
    └── kobo_form_browse.html   # Browse Kobo forms
```

---

## Choice Lists

### What are Choice Lists?

**ChoiceLists** are containers for groups of related dropdown options used across the system. Each list represents a category of choices (e.g., "modes of observation", "weather conditions").

### Model Definition

**File**: `api/models/choices.py`

```python
class ChoiceList(models.Model):
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

### Standard Choice Lists

The system includes predefined choice lists initialized via management command:

| Choice List Name | Description | Use Case |
|------------------|-------------|----------|
| `mode_of_observation` | How wildlife was observed | Wildlife Observation forms |
| `type_of_resource_use` | Types of resource extraction | Resource Use Incident forms |
| `type_of_disturbance` | Environmental disturbances | Disturbance Event forms |
| `severity_level` | Impact severity (Low/Medium/High/Critical) | Incident severity rating |
| `weather_condition` | Weather during monitoring | Field monitoring conditions |
| `monitoring_location` | Field monitoring sites | **Auto-populated from MonitoringLocation model** |

### CRUD Operations

#### 1. **List All Choice Lists**

**View**: `ChoiceListListView`  
**URL**: `/dashboard/admin/choices/`  
**Template**: `choice_list_list.html`

**Features**:
- Shows all choice lists with count of active choices
- Displays number of form mappings per list
- Search by name or description
- Pagination (25 per page)

**Code Example**:
```python
class ChoiceListListView(StaffUserRequiredMixin, ListView):
    model = ChoiceList
    template_name = 'dashboard/admin/choice_list_list.html'
    paginate_by = 25
    
    def get_queryset(self):
        return ChoiceList.objects.annotate(
            choice_count=Count('choices', filter=Q(choices__active=True)),
            mapping_count=Count('formchoicemapping')
        ).order_by('name')
```

#### 2. **Create Choice List**

**View**: `ChoiceListCreateView`  
**URL**: `/dashboard/admin/choices/create/`  
**Form Fields**:
- Name (unique, e.g., `animal_behavior`)
- Description (optional, e.g., "Animal behaviors observed in the field")

**Success Message**: `"Choice list 'X' created successfully."`  
**Redirect**: Detail view of created list

#### 3. **View Choice List Details**

**View**: `ChoiceListDetailView`  
**URL**: `/dashboard/admin/choices/<pk>/`  
**Shows**:
- List metadata (name, description, dates)
- All choices within the list (paginated)
- Associated form mappings
- Statistics: active choices, inactive choices, auto-update mappings

**Actions Available**:
- Add new choice
- Bulk add choices
- Sync to Kobo forms (if mappings exist with `auto_update=True`)
- Edit or delete the list

#### 4. **Update Choice List**

**View**: `ChoiceListUpdateView`  
**URL**: `/dashboard/admin/choices/<pk>/edit/`  
**Fields**: Name, Description  
**Success Message**: `"Choice list 'X' updated successfully."`

#### 5. **Delete Choice List**

**View**: `ChoiceListDeleteView`  
**URL**: `/dashboard/admin/choices/<pk>/delete/`  
**Template**: `choice_list_confirm_delete.html`  
**Warning**: Shows count of choices and mappings that will be affected

**Cascade Behavior**:
- All `Choice` records in this list are deleted (CASCADE)
- `FormChoiceMapping` records are NOT deleted (must be removed manually first)

---

## Individual Choices

### What are Choices?

**Choices** are the individual options within a ChoiceList. Each choice has:
- **name**: Internal identifier (e.g., `seen`, `heard`)
- **label**: Human-readable display text (e.g., "Seen (Visual observation)")
- **order**: Integer for sorting
- **active**: Boolean to enable/disable

### Model Definition

```python
class Choice(models.Model):
    choice_list = models.ForeignKey(ChoiceList, on_delete=models.CASCADE, related_name='choices')
    name = models.CharField(max_length=100)
    label = models.CharField(max_length=200)
    order = models.IntegerField(default=0)
    active = models.BooleanField(default=True)
    
    # Optional FK links to related models
    fauna_species = models.ForeignKey('species.FaunaChecklist', null=True, blank=True, ...)
    flora_species = models.ForeignKey('species.FloraChecklist', null=True, blank=True, ...)
    monitoring_location = models.ForeignKey('MonitoringLocation', null=True, blank=True, ...)
    
    class Meta:
        ordering = ['order', 'name']
        unique_together = ['choice_list', 'name']
```

### Optional Links to Models

Choices can reference other database models to provide **dynamic, database-driven options**:

#### Example: Monitoring Locations

```python
# In initialize_choice_lists.py
locations = MonitoringLocation.objects.filter(is_active=True).order_by('name')
for location in locations:
    Choice.objects.create(
        choice_list=location_list,
        name=location.name.lower().replace(' ', '_'),
        label=f'{location.name} ({location.get_location_type_display()})',
        monitoring_location=location  # Link to actual location record
    )
```

**Benefit**: When a new monitoring location is added to the system, it can automatically appear in KoboToolbox forms.

### CRUD Operations

#### 1. **Add Choice to List**

**View**: `ChoiceCreateView`  
**URL**: `/dashboard/admin/choices/<choice_list_pk>/add/`  
**Form Fields**:
- Name (unique within list)
- Label (display text)
- Order (integer)
- Active (checkbox, default True)
- Optional: fauna_species, flora_species, monitoring_location

**Code Example**:
```python
def form_valid(self, form):
    form.instance.choice_list = self.get_choice_list()
    messages.success(self.request, f'Choice "{form.instance.label}" added successfully.')
    return super().form_valid(form)
```

#### 2. **Edit Choice**

**View**: `ChoiceUpdateView`  
**URL**: `/dashboard/admin/choices/edit/<pk>/`  
**Editable**: All fields except `choice_list` (cannot move to another list)

#### 3. **Delete Choice**

**View**: `ChoiceDeleteView`  
**URL**: `/dashboard/admin/choices/delete/<pk>/`  
**Warning**: Confirm before deletion

#### 4. **Toggle Active Status**

**View**: `ChoiceToggleActiveView` (AJAX-compatible)  
**URL**: `/dashboard/admin/choices/<pk>/toggle/`  
**Method**: POST  
**Action**: Flips `active` from True ↔ False

**Use Case**: Temporarily disable a choice without deleting it (e.g., deprecated species name)

#### 5. **Bulk Add Choices**

**View**: `ChoiceBulkAddView`  
**URL**: `/dashboard/admin/choices/<choice_list_pk>/bulk-add/`  
**Template**: `choice_bulk_add.html`

**Input Format** (textarea):
```
name1|Label 1
name2|Label 2
name3|Label 3
```

**Logic**:
- Parse each line as `name|label`
- If choice with `name` exists: update label
- If choice is new: create with auto-incremented order
- Returns counts: `{created: X, updated: Y, skipped: Z}`

**Code Excerpt**:
```python
for i, choice_data in enumerate(choices_data):
    existing = Choice.objects.filter(
        choice_list=choice_list,
        name=choice_data['name']
    ).first()
    
    if existing:
        if existing.label != choice_data['label']:
            existing.label = choice_data['label']
            existing.save()
            updated_count += 1
        else:
            skipped_count += 1
    else:
        Choice.objects.create(
            choice_list=choice_list,
            name=choice_data['name'],
            label=choice_data['label'],
            order=max_order + i + 1,
            active=True
        )
        created_count += 1
```

---

## Form Choice Mappings

### What are Form Choice Mappings?

**FormChoiceMapping** links a specific question in a KoboToolbox form to a Django ChoiceList, enabling automatic synchronization.

### Model Definition

```python
class FormChoiceMapping(models.Model):
    asset_uid = models.CharField(max_length=50)  # Kobo form ID (e.g., 'abc123xyz')
    question_name = models.CharField(max_length=200)  # Question/choice list name in Kobo
    choice_list = models.ForeignKey(ChoiceList, on_delete=models.CASCADE)
    auto_update = models.BooleanField(default=False)  # Enable automatic sync
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

### Mapping Configuration

#### Fields Explained

| Field | Description | Example |
|-------|-------------|---------|
| `asset_uid` | KoboToolbox form unique identifier | `aXb2Cd3Ef4Gh5Ij6` |
| `question_name` | Name of the choice list in the Kobo form's XLSForm | `mode_of_observation` |
| `choice_list` | FK to Django ChoiceList | → `ChoiceList(name='mode_of_observation')` |
| `auto_update` | If True, changes to Django choices automatically push to Kobo | `True` / `False` |

#### Example Mapping

```python
FormChoiceMapping.objects.create(
    asset_uid='aXb2Cd3Ef4Gh5Ij6',
    question_name='mode_of_observation',
    choice_list=ChoiceList.objects.get(name='mode_of_observation'),
    auto_update=True
)
```

**Result**: Whenever choices in `mode_of_observation` ChoiceList are updated in Django, they will automatically sync to the Kobo form with asset UID `aXb2Cd3Ef4Gh5Ij6`.

### CRUD Operations

#### 1. **List All Mappings**

**View**: `FormChoiceMappingListView`  
**URL**: `/dashboard/admin/form-mappings/`  
**Template**: `form_mapping_list.html`

**Filters**:
- By Choice List (dropdown)
- By Auto-Update status (True/False/All)
- Search by asset_uid or question_name

**Display**:
| Asset UID | Question Name | Choice List | Auto-Update | Actions |
|-----------|---------------|-------------|-------------|---------|
| aXb2Cd... | mode_of_observation | Mode of Observation | ✅ Yes | Edit, Delete, Sync |

#### 2. **Create Mapping**

**View**: `FormChoiceMappingCreateView`  
**URL**: `/dashboard/admin/form-mappings/create/?asset_uid=<uid>&choice_list=<id>`  
**Form Fields**:
- Asset UID (Kobo form ID)
- Question Name (choice list name in Kobo form)
- Choice List (FK to ChoiceList)
- Auto Update (checkbox)

**Pre-fill Support**: If accessed from Kobo Form Browser with `?asset_uid=X`, the field is pre-filled.

**Success**: `"Form mapping created successfully."`

#### 3. **Edit Mapping**

**View**: `FormChoiceMappingUpdateView`  
**URL**: `/dashboard/admin/form-mappings/<pk>/edit/`

#### 4. **Delete Mapping**

**View**: `FormChoiceMappingDeleteView`  
**URL**: `/dashboard/admin/form-mappings/<pk>/delete/`

#### 5. **Manual Sync from Kobo**

**View**: `FormChoiceMappingSyncView` (POST only)  
**URL**: `/dashboard/admin/form-mappings/<pk>/sync/`  
**Action**: Fetch current choices from the Kobo form and update Django database

**Use Case**: Import existing choices from a Kobo form into Django

**Process**:
1. Download XLSForm from Kobo
2. Parse choices sheet
3. Create/update `Choice` records in Django
4. Success message: `"Successfully synced choices from form X"`

---

## KoboToolbox Integration

### Unified Management Interface

**View**: `ChoiceManagementView`  
**URL**: `/dashboard/admin/choice-management/`  
**Template**: `choice_form_hub.html`

This is the **main hub** for all choice-related operations, featuring a tabbed interface:

#### Tabs

1. **Choice Lists** (`?section=choices`)
   - View all choice lists
   - Quick stats: total lists, active/inactive choices
   - Create, edit, delete choice lists
   - Access individual list details

2. **Form Mappings** (`?section=mappings`)
   - View all form-choice mappings
   - Filter by choice list or auto-update status
   - Create, edit, delete mappings
   - Manual sync buttons

3. **Kobo Forms** (`?section=kobo`)
   - Browse available forms from KoboToolbox API
   - Health check status (API connection)
   - Quick-create mappings for unmapped forms
   - View form metadata (name, deployment status, submission count)

### Kobo Form Browser

**View**: `KoboFormBrowseView`  
**URL**: `/dashboard/admin/kobo-forms/`  
**Template**: `kobo_form_browse.html`

**Features**:
- **Health Check**: Verify API connection and display connected username
- **Form List**: Fetch all survey forms from KoboToolbox
- **Metadata Display**:
  - Form name
  - Asset UID (unique identifier)
  - Deployment status (deployed/draft)
  - Submission count
  - Last modified date
- **Mapping Status**: Shows which forms already have mappings (badge indicator)

**API Interaction**:
```python
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    
    # Check API health
    health_result, health_error = health_check()
    context['kobo_connected'] = health_result is not None
    
    # Fetch forms
    assets_response = list_assets()
    if 'error' not in assets_response:
        forms = []
        for asset in assets_response.get('results', []):
            if asset.get('asset_type') == 'survey':
                forms.append({
                    'uid': asset.get('uid'),
                    'name': asset.get('name'),
                    'deployment_status': asset.get('deployment_status'),
                    # ...
                })
        context['forms'] = forms
```

### API Health Check

**Function**: `health_check()`  
**File**: `api/form_services.py`  
**Endpoint**: `https://eu.kobotoolbox.org/api/v2/`

**Returns**:
- Success: `({"status": "connected", "user": {...}}, None)`
- Failure: `(None, "Connection error: ...")`

**Usage in Templates**:
```django
{% if kobo_connected %}
    <span class="badge bg-success">Connected as {{ kobo_user.username }}</span>
{% else %}
    <span class="badge bg-danger">Disconnected</span>
    <p>Error: {{ kobo_error }}</p>
{% endif %}
```

---

## Bidirectional Sync

### Overview

The system supports **two-way synchronization**:

1. **Django → Kobo**: Push updated choices from Django to KoboToolbox forms
2. **Kobo → Django**: Import existing choices from Kobo forms into Django

### Django → Kobo Sync (Push)

#### Process Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Admin updates choices in Django (add/edit/delete)               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                        Trigger Sync
                                │
                                ▼
                ┌───────────────────────────┐
                │ 1. Download XLSForm       │
                │    from KoboToolbox       │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 2. Parse form (survey +   │
                │    choices sheets)        │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 3. Fetch updated choices  │
                │    from Django database   │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 4. Replace choices in     │
                │    XLSForm (choices sheet)│
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 5. Upload updated XLSForm │
                │    to KoboToolbox         │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 6. Redeploy form          │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 7. Log update result      │
                │    (FormUpdateLog)        │
                └───────────────────────────┘
```

#### Trigger Sync

**Option 1: From Choice List Detail Page**

**View**: `ChoiceListSyncView` (POST)  
**URL**: `/dashboard/admin/choices/<pk>/sync/`  
**Button**: "Sync to Kobo Forms" (on choice list detail page)

**Logic**:
```python
def post(self, request, pk):
    choice_list = get_object_or_404(ChoiceList, pk=pk)
    
    # Find all mappings with auto_update=True
    mappings = FormChoiceMapping.objects.filter(
        choice_list=choice_list,
        auto_update=True
    )
    
    for mapping in mappings:
        success, message, details = process_form_with_choice_updates(mapping.asset_uid)
        # Handle result...
```

**Requirement**: At least one `FormChoiceMapping` with `auto_update=True` must exist.

**Option 2: Automatic Sync (Future Enhancement)**

Could be triggered via:
- Post-save signal on `Choice` model
- Scheduled Celery task
- Webhook from Django admin action

#### Core Function: `process_form_with_choice_updates()`

**File**: `api/form_services.py`  
**Signature**: `process_form_with_choice_updates(asset_uid: str) -> Tuple[bool, str, Dict]`

**Steps**:
1. Download XLSForm file from Kobo
2. Parse survey and choices sheets using `XLSFormProcessor`
3. Fetch updated choices from Django database via `FormChoiceMapping`
4. Replace choices in the XLSForm
5. Save modified XLSForm
6. Upload to Kobo (import/replace endpoint)
7. Redeploy form to make changes live
8. Log result in `FormUpdateLog`

**Returns**:
- `(True, "Form updated and redeployed successfully", {details})`
- `(False, "Error message", {details})`

**Code Excerpt**:
```python
def process_form_with_choice_updates(asset_uid: str) -> Tuple[bool, str, Dict]:
    processor = XLSFormProcessor()
    log_details = {}
    
    # Step 1: Download
    file_content, filename, error = download_asset_xls(asset_uid)
    if error:
        return False, f"Download failed: {error}", {}
    
    # Step 2: Parse
    sheets = processor.read_xlsform(file_content)
    choice_lists = processor.extract_choice_lists(sheets['choices'])
    
    # Step 3: Get updated choices from database
    mappings = FormChoiceMapping.objects.filter(asset_uid=asset_uid, auto_update=True)
    choice_updates = processor.get_updated_choices_from_database(mappings)
    
    # Step 4: Update form
    updated_sheets = processor.update_choices_in_form(sheets, choice_updates)
    
    # Step 5: Save
    updated_file_content = processor.save_xlsform(updated_sheets)
    
    # Step 6: Upload
    result, upload_error = upload_new_form(asset_uid, updated_file_content, filename)
    if upload_error:
        return False, f"Upload failed: {upload_error}", log_details
    
    # Step 7: Redeploy
    deploy_result, deploy_error = redeploy_asset(asset_uid)
    if deploy_error:
        return True, f"Form updated but deployment failed: {deploy_error}", log_details
    
    # Step 8: Log
    FormUpdateLog.objects.create(asset_uid=asset_uid, status='success', ...)
    
    return True, "Form choices updated and redeployed successfully", log_details
```

#### XLSForm Processing

**File**: `api/excel_processor.py`  
**Class**: `XLSFormProcessor`

**Key Methods**:
- `read_xlsform(file_content)`: Parse Excel file into survey/choices/settings sheets
- `extract_choice_lists(choices_sheet)`: Extract existing choices
- `get_updated_choices_from_database(mappings)`: Fetch Django choices
- `update_choices_in_form(sheets, choice_updates)`: Replace choice rows
- `save_xlsform(sheets)`: Write updated Excel file

### Kobo → Django Sync (Import)

#### Process Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Admin wants to import choices from an existing Kobo form        │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                    Trigger Import Sync
                                │
                                ▼
                ┌───────────────────────────┐
                │ 1. Download XLSForm       │
                │    from KoboToolbox       │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 2. Parse choices sheet    │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 3. Create/Update ChoiceList│
                │    in Django               │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 4. Create/Update Choice    │
                │    records                 │
                └───────────┬───────────────┘
                            │
                            ▼
                ┌───────────────────────────┐
                │ 5. Create FormChoiceMapping│
                └───────────────────────────┘
```

#### Trigger Import

**View**: `FormChoiceMappingSyncView` (POST)  
**URL**: `/dashboard/admin/form-mappings/<pk>/sync/`  
**Button**: "Sync from Kobo" (on form mapping row)

**Use Case**: You have an existing Kobo form with well-defined choices. Instead of manually re-entering them in Django, import them automatically.

#### Core Function: `sync_form_choices_to_database()`

**File**: `api/form_services.py`  
**Signature**: `sync_form_choices_to_database(asset_uid: str) -> Tuple[bool, str, Dict]`

**Steps**:
1. Download XLSForm from Kobo
2. Parse choices sheet
3. For each choice list in the form:
   - Create or get `ChoiceList` in Django
   - Create/update `Choice` records
4. Create `FormChoiceMapping` for each question using a choice list

**Returns**:
- `(True, "Synced X choice lists from form", {details})`
- `(False, "Error message", {})`

---

## User Workflows

### Workflow 1: Add New Species to Kobo Forms

**Scenario**: A new endemic species is discovered and needs to be added to field data collection forms.

**Steps**:

1. **Navigate** to Species Management (`/dashboard/species/`)
2. **Add** new species to `FaunaChecklist` or `FloraChecklist`
3. **Navigate** to Choice Management Hub (`/dashboard/admin/choice-management/`)
4. **Select** Choice Lists tab
5. **Find** the species choice list (e.g., `fauna_species_list`)
6. **Click** "Add Choice" or "Bulk Add"
7. **Enter** species data:
   - Name: `phasianus_colchicus_matutum`
   - Label: `Phasianus colchicus matutum (Mt. Matutum Pheasant)`
   - Link to fauna_species record
8. **Save** choice
9. **Click** "Sync to Kobo Forms" button on choice list detail page
10. **Verify** sync success message
11. **Check** KoboToolbox form to confirm species appears in dropdown

**Result**: Field staff can now select the new species in their mobile forms.

---

### Workflow 2: Create Mapping for New Kobo Form

**Scenario**: A new monitoring form is deployed in KoboToolbox. Admin needs to link it to Django choice lists.

**Steps**:

1. **Navigate** to Choice Management Hub → **Kobo Forms** tab
2. **Verify** API connection (green "Connected" badge)
3. **Browse** available forms
4. **Identify** the new form (e.g., "Monthly Wildlife Census 2026")
5. **Copy** the Asset UID (e.g., `aXb2Cd3Ef4Gh5Ij6`)
6. **Download** the XLSForm from Kobo manually (or use sync feature)
7. **Identify** which questions use choice lists (check XLSForm survey sheet, type = `select_one`)
8. **Switch** to **Form Mappings** tab
9. **Click** "Create New Mapping"
10. **Fill** form:
    - Asset UID: `aXb2Cd3Ef4Gh5Ij6`
    - Question Name: `mode_of_observation` (from XLSForm)
    - Choice List: Select "Mode of Observation" from dropdown
    - Auto Update: ✅ Check if you want automatic sync
11. **Save** mapping
12. **Repeat** for other questions in the form
13. **Test** sync: Click "Sync to Kobo" on the choice list

---

### Workflow 3: Import Choices from Existing Kobo Form

**Scenario**: An old Kobo form has a well-defined set of choices that you want to use in Django.

**Steps**:

1. **Navigate** to Choice Management Hub → **Kobo Forms** tab
2. **Find** the existing form
3. **Copy** Asset UID
4. **Switch** to **Form Mappings** tab
5. **Click** "Create New Mapping"
6. **Enter** Asset UID and Question Name
7. **Leave** Choice List blank initially
8. **Save** mapping
9. **Click** "Sync from Kobo" button (import icon)
10. **System** downloads form and creates:
    - New ChoiceList named `{asset_uid}_{question_name}`
    - All choices from the form
11. **Review** imported choices in Choice Lists tab
12. **Optionally** rename/reorganize the imported list

---

### Workflow 4: Temporarily Disable a Choice

**Scenario**: A monitoring location is temporarily closed for maintenance.

**Steps**:

1. **Navigate** to Choice List Detail for `monitoring_location`
2. **Find** the choice for the closed location (e.g., "Station Alpha")
3. **Click** toggle/deactivate button (or edit and uncheck "Active")
4. **Sync** to Kobo forms
5. **Result**: "Station Alpha" no longer appears in field forms
6. **When ready**: Toggle active status back to True and sync again

**Benefit**: No data is lost; choice is preserved but hidden from dropdowns.

---

## API & Services

### API Endpoints (Django)

| Endpoint | Method | Description | Access |
|----------|--------|-------------|--------|
| `/dashboard/admin/choice-management/` | GET | Main hub (tabbed interface) | Staff only |
| `/dashboard/admin/choices/` | GET | List all choice lists | Staff only |
| `/dashboard/admin/choices/create/` | GET, POST | Create choice list | Staff only |
| `/dashboard/admin/choices/<pk>/` | GET | Choice list detail | Staff only |
| `/dashboard/admin/choices/<pk>/edit/` | GET, POST | Edit choice list | Staff only |
| `/dashboard/admin/choices/<pk>/delete/` | GET, POST | Delete choice list | Staff only |
| `/dashboard/admin/choices/<pk>/sync/` | POST | Sync choices to Kobo | Staff only |
| `/dashboard/admin/choices/<choice_list_pk>/add/` | GET, POST | Add choice to list | Staff only |
| `/dashboard/admin/choices/edit/<pk>/` | GET, POST | Edit choice | Staff only |
| `/dashboard/admin/choices/delete/<pk>/` | GET, POST | Delete choice | Staff only |
| `/dashboard/admin/choices/<pk>/toggle/` | POST | Toggle active status | Staff only |
| `/dashboard/admin/choices/<choice_list_pk>/bulk-add/` | GET, POST | Bulk add choices | Staff only |
| `/dashboard/admin/form-mappings/` | GET | List all mappings | Staff only |
| `/dashboard/admin/form-mappings/create/` | GET, POST | Create mapping | Staff only |
| `/dashboard/admin/form-mappings/<pk>/edit/` | GET, POST | Edit mapping | Staff only |
| `/dashboard/admin/form-mappings/<pk>/delete/` | GET, POST | Delete mapping | Staff only |
| `/dashboard/admin/form-mappings/<pk>/sync/` | POST | Import from Kobo | Staff only |
| `/dashboard/admin/kobo-forms/` | GET | Browse Kobo forms | Staff only |

### KoboToolbox API Endpoints (External)

**Base URL**: `https://eu.kobotoolbox.org/api/v2`  
**Authentication**: Token-based (set in `settings.KOBOTOOLBOX_TOKEN`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Health check & user info |
| `/assets/?format=json` | GET | List all assets (forms) |
| `/assets/{asset_uid}/?format=json` | GET | Get asset details |
| `/assets/{asset_uid}/?format=xls` | GET | Download XLSForm file |
| `/imports/` | POST | Upload/replace form (with `destination` param) |
| `/assets/{asset_uid}/deployment/` | PATCH | Redeploy form with `version_id` |

### Service Functions

**File**: `api/form_services.py`

#### 1. `list_assets()`
Fetch all forms from KoboToolbox.
```python
def list_assets():
    """Fetch all assets available in kobotoolbox"""
    url = f"{BASE_URL}/assets/?format=json"
    response = requests.get(url, headers=HEADERS, timeout=30)
    return response.json()
```

#### 2. `get_asset_details(asset_uid)`
Get detailed information about a specific form.

#### 3. `download_asset_xls(asset_uid)`
Download the XLSForm Excel file.
```python
def download_asset_xls(asset_uid):
    url = f"{BASE_URL}/assets/{asset_uid}/?format=xls"
    response = requests.get(url, headers=HEADERS, timeout=60)
    return response.content, filename, None
```

#### 4. `upload_new_form(asset_uid, file_content, filename)`
Upload a modified XLSForm back to Kobo.
```python
def upload_new_form(asset_uid: str, file_content, filename: str = None):
    url = f"{BASE_URL}/imports/"
    files = {"file": (filename, f, "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")}
    data = {"destination": f"{BASE_URL}/assets/{asset_uid}/"}
    response = requests.post(url, headers=HEADERS, files=files, data=data, timeout=120)
    return response.json(), None
```

#### 5. `redeploy_asset(asset_uid)`
Redeploy form to publish changes.
```python
def redeploy_asset(asset_uid: str) -> Tuple[Optional[Dict], Optional[str]]:
    asset_details = get_asset_details(asset_uid)
    version_id = asset_details.get("version_id")
    
    url = f"{BASE_URL}/assets/{asset_uid}/deployment/"
    data = {"version_id": version_id}
    response = requests.patch(url, headers=HEADERS, json=data, timeout=30)
    
    if response.status_code == 204:
        return {"status": "success"}, None
    return response.json(), None
```

#### 6. `health_check()`
Verify API connection.

#### 7. `process_form_with_choice_updates(asset_uid)`
**Main sync function** (Django → Kobo). See [Bidirectional Sync](#bidirectional-sync).

#### 8. `sync_form_choices_to_database(asset_uid)`
Import choices from Kobo to Django (Kobo → Django).

---

## Testing Guide

### Test Scenarios

#### Scenario 1: Create and Sync New Choice List

**Steps**:
1. Create a new ChoiceList: `test_list`
2. Add 3 choices: `option1`, `option2`, `option3`
3. Create a FormChoiceMapping linking to a test Kobo form
4. Enable `auto_update=True`
5. Trigger sync
6. Verify choices appear in Kobo form

**Expected Results**:
- ✅ ChoiceList created
- ✅ 3 Choice records created
- ✅ Mapping created
- ✅ Sync completes without errors
- ✅ Kobo form shows all 3 options
- ✅ `FormUpdateLog` entry with `status='success'`

#### Scenario 2: Update Existing Choice and Re-sync

**Steps**:
1. Edit a choice label (e.g., `"Seen"` → `"Visually Observed"`)
2. Sync to Kobo
3. Download form from Kobo and verify change

**Expected Results**:
- ✅ Label updated in Django
- ✅ Sync completes successfully
- ✅ Kobo form reflects new label
- ✅ Form redeployed automatically

#### Scenario 3: Import Choices from Kobo Form

**Steps**:
1. Manually create a test form in Kobo with a custom choice list
2. In Django, create a FormChoiceMapping for this form
3. Click "Sync from Kobo"
4. Verify ChoiceList and Choices created in Django

**Expected Results**:
- ✅ New ChoiceList created in Django
- ✅ All choices imported with correct names and labels
- ✅ Mapping updated to reference new ChoiceList
- ✅ Success message displayed

#### Scenario 4: Deactivate Choice and Verify Hidden in Kobo

**Steps**:
1. Toggle a choice's `active` status to `False`
2. Sync to Kobo
3. Verify choice no longer appears in form dropdown

**Expected Results**:
- ✅ Choice `active=False` in Django
- ✅ Sync removes choice from Kobo form's choices sheet
- ✅ Form redeployed
- ✅ Choice hidden in mobile app dropdown

#### Scenario 5: Bulk Add Choices

**Steps**:
1. Navigate to a ChoiceList detail page
2. Click "Bulk Add Choices"
3. Paste:
   ```
   test1|Test Choice 1
   test2|Test Choice 2
   test3|Test Choice 3
   ```
4. Submit

**Expected Results**:
- ✅ 3 choices created
- ✅ Success message: `"Bulk add complete: 3 created"`
- ✅ Choices visible in list with correct order

#### Scenario 6: Handle API Connection Failure

**Steps**:
1. Temporarily disable internet or set invalid API token
2. Attempt to browse Kobo forms
3. Verify error message displayed

**Expected Results**:
- ✅ Error message: `"Connection error: ..."`
- ✅ Red "Disconnected" badge shown
- ✅ No crash or exception
- ✅ User can navigate away gracefully

---

### Automated Test Cases

**File**: `dashboard/tests/test_choice_system.py` (create if not exists)

```python
from django.test import TestCase, Client
from django.contrib.auth import get_user_model
from api.models import ChoiceList, Choice, FormChoiceMapping

class ChoiceSystemTests(TestCase):
    def setUp(self):
        self.client = Client()
        self.admin_user = get_user_model().objects.create_user(
            username='admin_test',
            password='testpass123',
            is_staff=True
        )
        self.client.login(username='admin_test', password='testpass123')
    
    def test_create_choice_list(self):
        """Test creating a new choice list"""
        response = self.client.post('/dashboard/admin/choices/create/', {
            'name': 'test_list',
            'description': 'Test description'
        })
        self.assertEqual(response.status_code, 302)  # Redirect on success
        self.assertTrue(ChoiceList.objects.filter(name='test_list').exists())
    
    def test_add_choice_to_list(self):
        """Test adding a choice to a list"""
        choice_list = ChoiceList.objects.create(name='test_list')
        response = self.client.post(f'/dashboard/admin/choices/{choice_list.pk}/add/', {
            'name': 'test_option',
            'label': 'Test Option',
            'order': 1,
            'active': True
        })
        self.assertEqual(response.status_code, 302)
        self.assertTrue(Choice.objects.filter(name='test_option').exists())
    
    def test_toggle_choice_active(self):
        """Test toggling choice active status"""
        choice_list = ChoiceList.objects.create(name='test_list')
        choice = Choice.objects.create(
            choice_list=choice_list,
            name='test',
            label='Test',
            active=True
        )
        response = self.client.post(f'/dashboard/admin/choices/{choice.pk}/toggle/')
        choice.refresh_from_db()
        self.assertFalse(choice.active)
    
    def test_bulk_add_choices(self):
        """Test bulk adding multiple choices"""
        choice_list = ChoiceList.objects.create(name='test_list')
        response = self.client.post(
            f'/dashboard/admin/choices/{choice_list.pk}/bulk-add/',
            {'choices_text': 'opt1|Option 1\nopt2|Option 2\nopt3|Option 3'}
        )
        self.assertEqual(Choice.objects.filter(choice_list=choice_list).count(), 3)
    
    def test_form_mapping_creation(self):
        """Test creating a form choice mapping"""
        choice_list = ChoiceList.objects.create(name='test_list')
        response = self.client.post('/dashboard/admin/form-mappings/create/', {
            'asset_uid': 'test_uid_123',
            'question_name': 'test_question',
            'choice_list': choice_list.pk,
            'auto_update': True
        })
        self.assertEqual(response.status_code, 302)
        self.assertTrue(FormChoiceMapping.objects.filter(asset_uid='test_uid_123').exists())
```

---

### Manual Testing Checklist

Use this checklist for comprehensive testing:

#### Choice Lists
- [ ] Can create new choice list
- [ ] Can edit choice list name/description
- [ ] Can delete choice list (with confirmation)
- [ ] Can view choice list details
- [ ] Can search/filter choice lists
- [ ] Pagination works correctly

#### Individual Choices
- [ ] Can add single choice to list
- [ ] Can edit choice name/label/order
- [ ] Can delete choice (with confirmation)
- [ ] Can toggle active/inactive status
- [ ] Can bulk add multiple choices via textarea
- [ ] Bulk add handles duplicates correctly (update vs create)
- [ ] Choices display in correct order

#### Form Mappings
- [ ] Can create new mapping
- [ ] Can edit existing mapping
- [ ] Can delete mapping (with confirmation)
- [ ] Can filter mappings by choice list
- [ ] Can filter mappings by auto_update status
- [ ] Can search mappings by asset_uid or question_name

#### Kobo Integration
- [ ] Health check shows connection status
- [ ] Form browser displays all Kobo forms
- [ ] Form metadata is accurate (name, UID, submission count)
- [ ] Mapped forms show indicator/badge
- [ ] Can click "Create Mapping" from form browser (pre-fills asset_uid)

#### Sync (Django → Kobo)
- [ ] Sync button appears only when auto_update mappings exist
- [ ] Sync completes without errors
- [ ] Success message displayed
- [ ] Kobo form reflects changes
- [ ] Form is automatically redeployed
- [ ] FormUpdateLog entry created with status 'success'

#### Sync (Kobo → Django)
- [ ] Import sync creates ChoiceList
- [ ] Import sync creates all Choice records
- [ ] Choice names and labels match Kobo form
- [ ] Success message displayed

#### Error Handling
- [ ] API connection failure shows error message
- [ ] Invalid asset_uid shows error
- [ ] Missing mapping shows warning
- [ ] Duplicate choice names prevented
- [ ] Form validation works correctly

#### Permissions
- [ ] Non-staff users cannot access any choice management pages
- [ ] Staff users can access all features
- [ ] Superusers have full access

---

## Summary

The **Choice List & KoboToolbox Integration** system provides a robust, bidirectional synchronization mechanism that:

✅ **Centralizes** choice management in Django  
✅ **Automates** form updates to KoboToolbox  
✅ **Maintains** data consistency across field and database  
✅ **Supports** dynamic options (species, locations)  
✅ **Logs** all sync operations for audit trails  
✅ **Enables** bulk operations for efficiency  
✅ **Provides** flexible mapping configurations  

### Key Takeaways

1. **ChoiceLists** group related options (e.g., observation modes, weather conditions)
2. **Choices** are individual items with name (internal), label (display), and order
3. **FormChoiceMappings** link Django ChoiceLists to specific Kobo form questions
4. **Auto-update** enables automatic sync when `auto_update=True`
5. **Bidirectional sync** supports both Django→Kobo and Kobo→Django workflows
6. **Unified Hub** (`choice_form_hub.html`) provides single interface for all operations
7. **XLSFormProcessor** handles form parsing and modification
8. **FormUpdateLog** tracks all sync operations for debugging and audit

---

## Next Steps

- Review existing choice lists and add missing options
- Create mappings for all active Kobo forms
- Enable `auto_update` for production forms
- Test sync workflow end-to-end
- Document any project-specific choice lists
- Train field staff on new dropdown options

---

*Documentation created for PRISM-Matutum Capstone Project*  
*Last Updated: January 6, 2026*  
*Author: System Documentation Team*
