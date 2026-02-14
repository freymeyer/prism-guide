# User & Organization Management System Documentation

## Overview

The User & Organization Management System provides comprehensive administration of users, agencies, departments, and geographic addresses. The system implements a sophisticated approval workflow for new user registrations, hierarchical organization structures, and a four-tier geographic address system for the Philippines.

**Key Features:**
- **User Approval Workflow** - Pending → Approved/Rejected/Suspended status management
- **Organization Hierarchy** - Agencies with optional parent agencies, Departments with sub-departments
- **Role-Based Access Control** - 5 user roles with distinct permissions
- **Geographic Addresses** - 4-tier hierarchy (Region → Province → Municipality → Barangay)
- **Unified Management Hub** - Single interface for Users, Agencies, Departments, Addresses
- **Activity Logging** - Audit trail for user management actions
- **Staff Verification** - Field verification status for PA Staff

---

## Architecture

### Three-Tier Management System

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER INTERFACE LAYER                          │
│  - User & Organization Hub (Unified Interface)                  │
│  - Address Management Hub (4-tier Geographic System)            │
│  - Admin Dashboard (Statistics & Quick Actions)                 │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ORCHESTRATION LAYER                            │
│  - StaffUserRequiredMixin (Access Control)                      │
│  - User Management Views (Approve/Reject/Update/Delete)         │
│  - Agency Management Views (CRUD operations)                    │
│  - Department Management Views (CRUD operations)                │
│  - Address Management Views (CRUD for 4 address levels)         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                  │
│  - User Model (Extended AbstractUser with approval workflow)    │
│  - Agency Model (Hierarchical organization structure)           │
│  - Department Model (Agency sub-units with heads)               │
│  - Address Models (Region, Province, Municipality, Barangay)    │
│  - UserActivityLog (Audit trail)                                │
│  - UserNotification (System notifications)                      │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Backend Framework:** Django 5.2 Class-Based Views
- **Authentication:** Django AbstractUser with custom fields
- **Database:** PostgreSQL 16 with foreign key constraints
- **Access Control:** Role-based permissions with mixins
- **File Storage:** ImageField for profile pictures
- **Validation:** Django forms with custom validators

---

## Models & Database Schema

### 1. Agency

Government agencies and organizations with hierarchical structure.

```python
class Agency(models.Model):
    """Government agencies and organizations (e.g., DENR, LGU, NGO)"""
    
    # Identification
    id_agency = AutoField(primary_key=True)
    name = CharField(max_length=200, unique=True)
    agency_type = CharField(max_length=20, choices=AGENCY_TYPES, default='government')
    acronym = CharField(max_length=20, blank=True)
    description = TextField(blank=True)
    
    # Hierarchy
    parent_agency = ForeignKey('self', null=True, blank=True, 
                              related_name='sub_agencies')
    
    # Contact Information
    contact_person = CharField(max_length=100, blank=True)
    contact_email = EmailField(blank=True)
    contact_phone = CharField(max_length=15, blank=True)
    address = TextField(blank=True)
    
    # Status
    is_active = BooleanField(default=True)
    
    # Metadata
    created_at = DateTimeField(auto_now_add=True)
    updated_at = DateTimeField(auto_now=True)
    
    # Methods
    def get_hierarchy_path(self) -> str
    # Returns: "Parent Agency > Sub Agency > Current Agency"
```

**AGENCY_TYPES:**
- `government` - Government Agency
- `lgu` - Local Government Unit
- `ngo` - Non-Government Organization
- `academic` - Academic Institution
- `private` - Private Organization

**Relationships:**
- Self-referential hierarchy via `parent_agency`
- One-to-Many with Department (via `departments`)
- One-to-Many with User (via `users`)

**Indexes:**
- `['agency_type', 'is_active']` - for filtering by type and status

**Key Methods:**
- `get_hierarchy_path()`: Returns full path like "DENR > DENR Region XII > MMPL Office"

---

### 2. Department

Departments within agencies with optional sub-department structure.

```python
class Department(models.Model):
    """Departments within organizations (e.g., Research Dept, Field Operations)"""
    
    # Identification
    id_department = AutoField(primary_key=True)
    name = CharField(max_length=200)
    code = CharField(max_length=20, blank=True)
    description = TextField(blank=True)
    
    # Organization Structure
    agency = ForeignKey(Agency, related_name='departments')
    parent_department = ForeignKey('self', null=True, blank=True,
                                  related_name='sub_departments')
    head = ForeignKey(User, null=True, blank=True,
                     related_name='headed_departments')
    
    # Contact Information
    office_location = CharField(max_length=200, blank=True)
    phone_extension = CharField(max_length=10, blank=True)
    email = EmailField(blank=True)
    
    # Status
    is_active = BooleanField(default=True)
    
    # Metadata
    created_at = DateTimeField(auto_now_add=True)
    updated_at = DateTimeField(auto_now=True)
    
    # Methods
    def get_hierarchy_path(self) -> str
    def get_all_staff(self) -> QuerySet[User]
```

**Constraints:**
- `unique_together`: ['agency', 'name'] - Department names unique within agency

**Indexes:**
- `['agency', 'is_active']` - for filtering departments by agency

**Key Methods:**
- `get_hierarchy_path()`: Returns department hierarchy path
- `get_all_staff()`: Returns all users in department and sub-departments (recursive)

---

### 3. User (Extended AbstractUser)

Extended Django user model with approval workflow and role-based access.

```python
class User(AbstractUser):
    """Extended user model with role-based access control"""
    
    # Role & Permissions
    role = CharField(max_length=20, choices=USER_ROLES, default='pa_staff')
    
    # Contact Information
    phone_number = CharField(max_length=15, blank=True,
                            validators=[phone_validator])
    
    # Organization & Department
    agency = ForeignKey(Agency, null=True, blank=True,
                       related_name='users')
    department = ForeignKey(Department, null=True, blank=True,
                           related_name='users')
    
    # PA Staff Specific Fields
    employee_id = CharField(max_length=50, blank=True, unique=True, null=True)
    position = CharField(max_length=100, blank=True)
    date_joined_organization = DateField(null=True, blank=True)
    
    # Account Approval Workflow
    account_status = CharField(max_length=20, choices=ACCOUNT_STATUS_CHOICES,
                              default='pending')
    approval_requested_at = DateTimeField(null=True, blank=True)
    approved_by = ForeignKey('self', null=True, blank=True,
                            related_name='approved_users')
    approved_at = DateTimeField(null=True, blank=True)
    rejection_reason = TextField(blank=True)
    
    # Account Status
    is_active = BooleanField(default=True)
    is_field_verified = BooleanField(default=False)
    
    # Profile
    profile_picture = ImageField(upload_to='user_profiles/', null=True, blank=True)
    bio = TextField(blank=True)
    
    # Metadata
    created_at = DateTimeField(auto_now_add=True)
    updated_at = DateTimeField(auto_now=True)
    last_field_activity = DateTimeField(null=True, blank=True)
    
    # Two-Factor Authentication
    is_2fa_enabled = BooleanField(default=False)
    totp_secret = CharField(max_length=32, blank=True, null=True)
    backup_codes = JSONField(default=list, blank=True)
    
    # Role Check Methods
    def is_pa_staff(self) -> bool
    def is_validator(self) -> bool
    def is_denr(self) -> bool
    def is_system_admin(self) -> bool
    def is_collaborator(self) -> bool
    
    # Permission Methods
    def can_collect_data(self) -> bool
    def can_validate(self) -> bool
    def can_view_reports(self) -> bool
    def can_build_reports(self) -> bool
    def can_manage_users(self) -> bool
    
    # Years of Service
    def get_years_of_service(self) -> Tuple[int, int, int]
    def get_years_of_service_display(self) -> str
    
    # Approval Workflow Methods
    def request_approval(self) -> None
    def approve_account(self, admin_user) -> None
    def reject_account(self, admin_user, reason) -> None
    def suspend_account(self, admin_user, reason) -> None
```

**USER_ROLES:**
- `pa_staff` - PA Staff (Field data collectors)
- `validator` - Validator (Data validation specialists)
- `denr` - DENR Personnel (Department staff)
- `system_admin` - System Administrator (Full access)
- `collaborator` - Collaborator / External Researcher (View-only)

**ACCOUNT_STATUS_CHOICES:**
- `pending` - Pending Approval (default on registration)
- `approved` - Approved (can access system)
- `rejected` - Rejected (cannot access system)
- `suspended` - Suspended (temporarily blocked)

**Indexes:**
- `['role', 'is_active']` - for role-based queries
- `['employee_id']` - for staff ID lookups

**Approval Workflow Logic:**

**`approve_account(admin_user)`** - Approves user account
1. Validates admin has `can_manage_users()` permission
2. Sets `account_status='approved'`, `is_active=True`
3. Records `approved_by`, `approved_at` timestamp
4. Clears `rejection_reason`
5. Creates UserActivityLog entry
6. Sends UserNotification to user

**`reject_account(admin_user, reason)`** - Rejects user account
1. Validates admin has `can_manage_users()` permission
2. Sets `account_status='rejected'`, `is_active=False`
3. Records `rejection_reason`
4. Creates UserActivityLog entry
5. Sends UserNotification with reason

**`suspend_account(admin_user, reason)`** - Suspends user account
1. Validates admin has `can_manage_users()` permission
2. Sets `account_status='suspended'`, `is_active=False`
3. Records `rejection_reason` (reused for suspension reason)
4. Creates UserActivityLog entry
5. Sends UserNotification with reason

---

### 4. UserActivityLog

Audit trail of user management actions and system events.

```python
class UserActivityLog(models.Model):
    """Log user activities for audit trail"""
    
    # References
    user = ForeignKey(User, related_name='activity_logs')
    
    # Activity Details
    activity_type = CharField(max_length=50)
    # Types: 'user_management', 'data_submission', 'validation', 'login', etc.
    description = TextField()
    metadata = JSONField(default=dict, blank=True)
    
    # Context
    ip_address = GenericIPAddressField(null=True, blank=True)
    user_agent = TextField(blank=True)
    
    # Timestamp
    created_at = DateTimeField(auto_now_add=True)
```

**Activity Types:**
- `user_management` - Account approval/rejection/suspension
- `data_submission` - Field data submission
- `validation` - Observation validation actions
- `login` - User login events
- `password_change` - Password changes
- `profile_update` - Profile modifications

---

### 5. UserNotification

System notifications for users (in-app messages).

```python
class UserNotification(models.Model):
    """System notifications for users"""
    
    # References
    user = ForeignKey(User, related_name='notifications')
    
    # Notification Content
    notification_type = CharField(max_length=50)
    # Types: 'system', 'validation', 'submission', 'report', 'approval'
    title = CharField(max_length=200)
    message = TextField()
    
    # Status
    is_read = BooleanField(default=False)
    read_at = DateTimeField(null=True, blank=True)
    
    # Optional Link
    link_url = CharField(max_length=500, blank=True)
    
    # Timestamp
    created_at = DateTimeField(auto_now_add=True)
```

---

### 6. Address Models (4-Tier Hierarchy)

Philippine geographic address system with cascading foreign keys.

#### Country

```python
class Country(models.Model):
    id_country = AutoField(primary_key=True)
    name = CharField(max_length=100)
```

#### Region

```python
class Region(models.Model):
    id_region = AutoField(primary_key=True)
    country = ForeignKey(Country)
    name = CharField(max_length=100)
```

**Example:** Region XII (SOCCSKSARGEN)

#### Province

```python
class Province(models.Model):
    id_province = AutoField(primary_key=True)
    region = ForeignKey(Region)
    name = CharField(max_length=100)
```

**Example:** South Cotabato

#### Municipality

```python
class Municipality(models.Model):
    id_municipality = AutoField(primary_key=True)
    province = ForeignKey(Province)
    name = CharField(max_length=100)
```

**Example:** Tupi

#### Barangay

```python
class Barangay(models.Model):
    id_barangay = AutoField(primary_key=True)
    municipality = ForeignKey(Municipality)
    name = CharField(max_length=100)
```

**Example:** Poblacion

#### Address (Combined)

```python
class Address(models.Model):
    id_address = AutoField(primary_key=True)
    country = ForeignKey(Country)
    region = ForeignKey(Region)
    province = ForeignKey(Province)
    municipality = ForeignKey(Municipality)
    barangay = ForeignKey(Barangay)
    
    def __str__(self):
        return f"{self.barangay.name}, {self.municipality.name}, {self.province.name}"
```

**Full Address Example:** "Poblacion, Tupi, South Cotabato, Region XII, Philippines"

---

## Views & URL Patterns

### Administrative Hub Views

#### 1. `AdminDashboardView` - Main Admin Dashboard

```python
class AdminDashboardView(StaffUserRequiredMixin, TemplateView)
```

**URL:** `/dashboard/admin/`  
**Template:** `dashboard/admin/admin_dashboard.html`  
**Permissions:** Staff users only (is_staff or is_superuser)  
**HTTP Methods:** GET

**Features:**
- System health overview with key statistics
- User statistics (total, active, pending approval, by role)
- Data submission statistics (total, pending, validated, rejected)
- Observation statistics (wildlife, resource incidents)
- Species statistics (fauna, flora counts)
- Geographic data statistics (regions, provinces, municipalities, barangays)
- Organization statistics (agencies, departments)
- Report statistics (total, pending, completed)
- Recent pending approvals list (top 5 users)
- Recent pending submissions list (top 10)

**Context Data:**
- `user_stats`: Dictionary with user counts by status and role
- `submission_stats`: Dictionary with submission counts by status
- `observation_stats`: Wildlife and resource use incident counts
- `species_stats`: Fauna and flora checklist counts
- `geo_stats`: Address hierarchy counts
- `org_stats`: Agency and department counts
- `pending_users`: Queryset of pending user approvals
- `recent_submissions`: Queryset of pending data submissions
- `report_stats`: Report generation statistics

---

#### 2. `UserOrganizationHubView` - Unified Management Hub

```python
class UserOrganizationHubView(StaffUserRequiredMixin, TemplateView)
```

**URL:** `/dashboard/admin/user-organization-hub/`  
**Template:** `dashboard/admin/user_organization_hub.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET

**Features:**
- Tabbed interface with 4 sections: Users, Agencies, Departments, Addresses
- **Users Section:**
  - 4 tabs: Pending, Active, Staff, All
  - Search by username, email, name
  - Pagination (15 per page)
  - Quick approve/reject buttons
- **Agencies Section:**
  - Filter by agency type, active status
  - Search by name, acronym, description
  - Annotated with department count, user count
  - Create/Edit/Delete actions
- **Departments Section:**
  - Filter by agency
  - Search by name, code, description
  - Annotated with user count
  - Shows department head
  - Create/Edit/Delete actions
- **Addresses Section:**
  - Nested tabs: Regions, Provinces, Municipalities, Barangays
  - Hierarchical display
  - Annotated with child counts
  - CRUD operations

**Query Parameters:**
- `section`: 'users' | 'agencies' | 'departments' | 'addresses' (default: 'users')
- `tab`: For users: 'pending' | 'active' | 'staff' | 'all' (default: 'pending')
- `address_tab`: For addresses: 'regions' | 'provinces' | 'municipalities' | 'barangays'
- `page`: Page number for pagination
- `search`: Search query
- `type`: Agency type filter (for agencies section)
- `active`: Active status filter (for agencies section)
- `agency`: Agency filter (for departments section)

**Context Data:**
- `active_section`: Current section ('users', 'agencies', 'departments', 'addresses')
- `active_tab`: Current tab within section
- `user_stats`: User count statistics
- `pending_users_page`, `active_users_page`, `staff_users_page`, `all_users_page`: Paginated user querysets
- `agencies_page`: Paginated agency queryset with annotations
- `agency_stats`: Agency count statistics
- `departments_page`: Paginated department queryset
- `department_stats`: Department count statistics
- `regions_page`, `provinces_page`, `municipalities_page`, `barangays_page`: Paginated address querysets
- `address_stats`: Address hierarchy count statistics

---

### User Management Views

#### 3. `UserManagementView` - Legacy User Management

```python
class UserManagementView(StaffUserRequiredMixin, TemplateView)
```

**URL:** `/dashboard/admin/users/`  
**Template:** `dashboard/admin/user_management.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET

**Note:** This view is being deprecated in favor of UserOrganizationHubView but remains for backward compatibility.

---

#### 4. `UserApproveView` - Approve User Account

```python
class UserApproveView(StaffUserRequiredMixin, View)
```

**URL:** `/dashboard/admin/users/<pk>/approve/`  
**Permissions:** Staff users only  
**HTTP Methods:** POST

**Features:**
- Calls `user.approve_account(request.user)`
- Creates activity log entry
- Sends notification to user
- Redirects back to User & Organization Hub

**POST Processing:**
1. Get user by primary key
2. Validate admin has permission
3. Call `approve_account()` method
4. Show success message
5. Redirect to hub with `?section=users&tab=pending`

---

#### 5. `UserRejectView` - Reject User Account

```python
class UserRejectView(StaffUserRequiredMixin, View)
```

**URL:** `/dashboard/admin/users/<pk>/reject/`  
**Permissions:** Staff users only  
**HTTP Methods:** POST

**Features:**
- Calls `user.reject_account(request.user, reason="Rejected by admin")`
- Creates activity log entry
- Sends notification to user with reason
- Redirects back to User & Organization Hub

**POST Processing:**
1. Get user by primary key
2. Validate admin has permission
3. Call `reject_account()` with default reason
4. Show success message
5. Redirect to hub with `?section=users&tab=pending`

**Note:** Uses default rejection reason. For custom reasons, users should use the user update form.

---

#### 6. `UserUpdateView` - Edit User Details

```python
class UserUpdateView(StaffUserRequiredMixin, UpdateView)
```

**URL:** `/dashboard/admin/users/<pk>/edit/`  
**Template:** `dashboard/admin/user_form.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET, POST

**Features:**
- Edit all user fields except password
- Update role, agency, department assignments
- Change account status
- Update profile information
- Set field verification status

**Form Fields:**
- `username`, `email`, `first_name`, `last_name`
- `phone_number`
- `role` (dropdown)
- `agency`, `department` (dropdowns)
- `employee_id`, `position`
- `date_joined_organization`
- `account_status` (dropdown)
- `is_active`, `is_field_verified` (checkboxes)
- `bio`

---

#### 7. `UserDeleteView` - Delete User Account

```python
class UserDeleteView(StaffUserRequiredMixin, DeleteView)
```

**URL:** `/dashboard/admin/users/<pk>/delete/`  
**Permissions:** Staff users only  
**HTTP Methods:** POST

**Features:**
- Soft delete by setting `is_active=False`
- Or hard delete (removes from database)
- Confirms deletion with message

**Warning:** Deleting users with associated data (submissions, validations) may cause cascading deletions. Consider suspension instead.

---

### Agency Management Views

#### 8. `AgencyManagementView` - Agency List

```python
class AgencyManagementView(StaffUserRequiredMixin, TemplateView)
```

**URL:** `/dashboard/admin/agencies/`  
**Template:** `dashboard/admin/agency_management.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET

**Features:**
- List all agencies with pagination
- Filter by agency type, active status
- Search by name, acronym
- Shows department count, user count
- Hierarchical display (parent → children)
- Create/Edit/Delete action buttons

---

#### 9. `AgencyCreateView` - Create Agency

```python
class AgencyCreateView(StaffUserRequiredMixin, CreateView)
```

**URL:** `/dashboard/admin/agencies/create/`  
**Template:** `dashboard/admin/agency_form.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET, POST

**Form Fields:**
- `name` (required, unique)
- `agency_type` (dropdown: Government, LGU, NGO, Academic, Private)
- `acronym`
- `description`
- `parent_agency` (dropdown of existing agencies)
- `contact_person`, `contact_email`, `contact_phone`
- `address` (textarea)
- `is_active` (checkbox, default: True)

**POST Processing:**
1. Validate form data
2. Create Agency record
3. Show success message
4. Redirect to User & Organization Hub with `?section=agencies`

---

#### 10. `AgencyUpdateView` - Edit Agency

```python
class AgencyUpdateView(StaffUserRequiredMixin, UpdateView)
```

**URL:** `/dashboard/admin/agencies/<pk>/edit/`  
**Template:** `dashboard/admin/agency_form.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET, POST

**Features:**
- Edit all agency fields
- Change parent agency (reorganize hierarchy)
- Update contact information
- Toggle active status

---

#### 11. `AgencyDeleteView` - Delete Agency

```python
class AgencyDeleteView(StaffUserRequiredMixin, DeleteView)
```

**URL:** `/dashboard/admin/agencies/<pk>/delete/`  
**Permissions:** Staff users only  
**HTTP Methods:** POST

**Features:**
- Delete agency if no associated users or departments
- Shows error if deletion would violate constraints
- Redirects to hub after successful deletion

**Constraint Checks:**
- Cannot delete if `users.count() > 0`
- Cannot delete if `departments.count() > 0`
- Must reassign users/departments before deletion

---

### Department Management Views

#### 12. `DepartmentManagementView` - Department List

```python
class DepartmentManagementView(StaffUserRequiredMixin, TemplateView)
```

**URL:** `/dashboard/admin/departments/`  
**Template:** `dashboard/admin/department_management.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET

**Features:**
- List all departments grouped by agency
- Shows parent department if hierarchical
- Displays department head
- Shows user count
- Filter by agency
- Search by name, code

---

#### 13. `DepartmentCreateView` - Create Department

```python
class DepartmentCreateView(StaffUserRequiredMixin, CreateView)
```

**URL:** `/dashboard/admin/departments/create/`  
**Template:** `dashboard/admin/department_form.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET, POST

**Form Fields:**
- `name` (required)
- `code` (department abbreviation)
- `agency` (dropdown, required)
- `parent_department` (dropdown of departments in same agency)
- `description`
- `head` (dropdown of users in agency)
- `office_location`, `phone_extension`, `email`
- `is_active` (checkbox)

**POST Processing:**
1. Validate form data
2. Ensure `parent_department` is from same agency
3. Create Department record
4. Redirect to hub with `?section=departments`

---

#### 14. `DepartmentUpdateView` - Edit Department

```python
class DepartmentUpdateView(StaffUserRequiredMixin, UpdateView)
```

**URL:** `/dashboard/admin/departments/<pk>/edit/`  
**Template:** `dashboard/admin/department_form.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET, POST

**Features:**
- Edit department details
- Change department head
- Reorganize hierarchy (change parent department)
- Update contact information

---

#### 15. `DepartmentDeleteView` - Delete Department

```python
class DepartmentDeleteView(StaffUserRequiredMixin, DeleteView)
```

**URL:** `/dashboard/admin/departments/<pk>/delete/`  
**Permissions:** Staff users only  
**HTTP Methods:** POST

**Features:**
- Delete department if no associated users
- Shows error if users assigned
- Redirects to hub after deletion

**Constraint Checks:**
- Cannot delete if `users.count() > 0`
- Must reassign users before deletion

---

### Address Management Views

#### 16. `AddressManagementView` - Address Hub

```python
class AddressManagementView(StaffUserRequiredMixin, TemplateView)
```

**URL:** `/dashboard/admin/addresses/`  
**Template:** `dashboard/admin/address_management.html`  
**Permissions:** Staff users only  
**HTTP Methods:** GET

**Features:**
- Tabbed interface for 4 levels: Regions, Provinces, Municipalities, Barangays
- Hierarchical display showing parent relationships
- Pagination (50 items per page)
- Statistics for each level
- Create/Edit/Delete actions per level

**Query Parameters:**
- `tab`: 'regions' | 'provinces' | 'municipalities' | 'barangays' (default: 'regions')
- `page`: Page number

**Context Data:**
- `active_tab`: Current address level
- `regions_page`, `provinces_page`, `municipalities_page`, `barangays_page`: Paginated querysets
- `stats`: Count statistics for all 4 levels

---

#### Region Views (17-20)

**`RegionListView`** - `/dashboard/admin/addresses/regions/`  
**`RegionCreateView`** - `/dashboard/admin/addresses/region/add/`  
**`RegionUpdateView`** - `/dashboard/admin/addresses/region/<pk>/edit/`  
**`RegionDeleteView`** - `/dashboard/admin/addresses/region/<pk>/delete/`

**Form Fields (Create/Update):**
- `name` (required)
- `country` (dropdown, foreign key)

**Delete Logic:**
- Cannot delete if provinces exist
- Must delete child provinces first (cascading prevention)

---

#### Province Views (21-24)

**`ProvinceListView`** - `/dashboard/admin/addresses/provinces/`  
**`ProvinceCreateView`** - `/dashboard/admin/addresses/province/add/`  
**`ProvinceUpdateView`** - `/dashboard/admin/addresses/province/<pk>/edit/`  
**`ProvinceDeleteView`** - `/dashboard/admin/addresses/province/<pk>/delete/`

**Form Fields (Create/Update):**
- `name` (required)
- `region` (dropdown, foreign key)

**Delete Logic:**
- Cannot delete if municipalities exist
- Must delete child municipalities first

---

#### Municipality Views (25-28)

**`MunicipalityListView`** - `/dashboard/admin/addresses/municipalities/`  
**`MunicipalityCreateView`** - `/dashboard/admin/addresses/municipality/add/`  
**`MunicipalityUpdateView`** - `/dashboard/admin/addresses/municipality/<pk>/edit/`  
**`MunicipalityDeleteView`** - `/dashboard/admin/addresses/municipality/<pk>/delete/`

**Form Fields (Create/Update):**
- `name` (required)
- `province` (dropdown, foreign key)

**Delete Logic:**
- Cannot delete if barangays exist
- Must delete child barangays first

---

#### Barangay Views (29-32)

**`BarangayListView`** - `/dashboard/admin/addresses/barangays/`  
**`BarangayCreateView`** - `/dashboard/admin/addresses/barangay/add/`  
**`BarangayUpdateView`** - `/dashboard/admin/addresses/barangay/<pk>/edit/`  
**`BarangayDeleteView`** - `/dashboard/admin/addresses/barangay/<pk>/delete/`

**Form Fields (Create/Update):**
- `name` (required)
- `municipality` (dropdown, foreign key)

**Delete Logic:**
- Cannot delete if addresses reference this barangay
- Must reassign addresses before deletion

---

## Security & Permissions

### Access Control Mixin

#### StaffUserRequiredMixin

Restricts all administrative views to staff users only.

```python
class StaffUserRequiredMixin(UserPassesTestMixin):
    """Restrict access to staff users (is_staff or is_superuser)"""
    login_url = '/accounts/login/'
    
    def test_func(self):
        return self.request.user.is_authenticated and (
            self.request.user.is_staff or self.request.user.is_superuser
        )
    
    def handle_no_permission(self):
        messages.error(
            self.request,
            "Access denied. This section is restricted to DENR staff only."
        )
        return super().handle_no_permission()
```

**Logic:**
- Checks `user.is_authenticated AND (user.is_staff OR user.is_superuser)`
- Redirects to login if not authenticated
- Shows error message and redirects if not authorized

---

### User Permission Methods

User model includes permission check methods used throughout the system:

```python
# Role Checks
user.is_pa_staff() -> bool
user.is_validator() -> bool
user.is_denr() -> bool
user.is_system_admin() -> bool
user.is_collaborator() -> bool

# Functional Permissions
user.can_collect_data() -> bool       # PA Staff with field verification
user.can_validate() -> bool           # Validators with approved status
user.can_view_reports() -> bool       # All roles except unapproved
user.can_build_reports() -> bool      # Validators, DENR only
user.can_manage_users() -> bool       # System Admins only
```

**Permission Logic:**

**`can_collect_data()`**
```python
return (
    self.role == 'pa_staff' and 
    self.is_field_verified and 
    self.account_status == 'approved'
)
```

**`can_validate()`**
```python
return (
    self.role == 'validator' and 
    self.account_status == 'approved'
)
```

**`can_manage_users()`**
```python
return (
    self.role == 'system_admin' and 
    self.account_status == 'approved'
)
```

---

### Access Control Matrix

| View/Feature | PA Staff | Collaborator | Validator | DENR | System Admin |
|--------------|----------|--------------|-----------|------|--------------|
| AdminDashboardView | ❌ | ❌ | ❌ | ✅ | ✅ |
| UserOrganizationHubView | ❌ | ❌ | ❌ | ✅ | ✅ |
| UserApprove/Reject | ❌ | ❌ | ❌ | ❌ | ✅* |
| UserUpdate | ❌ | ❌ | ❌ | ❌ | ✅* |
| UserDelete | ❌ | ❌ | ❌ | ❌ | ✅* |
| AgencyCRUD | ❌ | ❌ | ❌ | ✅ | ✅ |
| DepartmentCRUD | ❌ | ❌ | ❌ | ✅ | ✅ |
| AddressManagement | ❌ | ❌ | ❌ | ✅ | ✅ |

**Legend:**
- ✅ Full access
- ✅* Full access (requires `can_manage_users()` for user management)
- ❌ No access

**Note:** User approval/rejection/suspension specifically requires `can_manage_users()` which is only available to `role='system_admin'` with `account_status='approved'`.

---

## User Registration & Approval Workflow

### Complete Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: USER REGISTRATION                                       │
│ URL: /users/register/                                           │
└───────────────┬─────────────────────────────────────────────────┘
                │
                ▼
     User fills UserRegistrationForm
       - Username, email, password
       - First name, last name, phone
       - Agency, department (dropdowns)
       - Employee ID, position
       - Role selection
       - Date joined organization
                │
                ▼
         Account Created:
       - account_status = 'pending'
       - is_active = False
       - approval_requested_at = now()
                │
                ▼
       Notifications Sent:
         - All System Admins notified
         - User sees "Pending Approval" page
                │
                │
┌───────────────┴─────────────────────────────────────────────────┐
│ STEP 2: ADMIN REVIEW                                            │
│ URL: /dashboard/admin/user-organization-hub/?section=users&tab=pending │
└───────────────┬─────────────────────────────────────────────────┘
                │
                ▼
   Admin views pending users list
     - Filters by account_status='pending'
     - Reviews user information:
       * Organization affiliation verified
       * Role appropriate for position
       * Employee ID checked
                │
       ┌────────┴────────┐
       │                 │
       ▼                 ▼
   APPROVE           REJECT
       │                 │
       ▼                 ▼
┌─────────────────┐ ┌─────────────────┐
│ user.approve_   │ │ user.reject_    │
│ account(admin)  │ │ account(admin,  │
│                 │ │ reason)         │
│ Sets:           │ │ Sets:           │
│ - status =      │ │ - status =      │
│   'approved'    │ │   'rejected'    │
│ - is_active =   │ │ - is_active =   │
│   True          │ │   False         │
│ - approved_by = │ │ - rejection_    │
│   admin         │ │   reason = text │
│ - approved_at = │ │                 │
│   now()         │ │ Creates:        │
│                 │ │ - Activity log  │
│ Creates:        │ │ - Notification  │
│ - Activity log  │ │   with reason   │
│ - Notification  │ │                 │
└─────────────────┘ └─────────────────┘
```

### Step-by-Step Process

**Step 1: User Registration**

1. User navigates to `/users/register/`
2. Fills out `UserRegistrationForm`:
   - Username (unique)
   - Email (valid email format)
   - Password (with confirmation)
   - First name, Last name
   - Phone number (optional, validated format)
   - Agency (dropdown of active agencies)
   - Department (dropdown of departments in selected agency)
   - Employee ID (optional, unique if provided)
   - Position
   - Role (dropdown: PA Staff, Validator, DENR, Collaborator)
   - Date joined organization (optional)
3. Form submission creates User with:
   - `account_status = 'pending'`
   - `is_active = False` (cannot login yet)
   - `approval_requested_at = timezone.now()`
4. System sends notifications to all System Admins
5. User redirected to "Registration Pending" page with instructions

**Step 2: Admin Review**

1. System Admin receives notification
2. Navigates to User & Organization Hub
3. Clicks "Users" section → "Pending" tab
4. Sees list of pending users with:
   - Full name, email, username
   - Requested role
   - Agency and department
   - Employee ID
   - Registration date
5. Reviews user credentials:
   - Verifies employee ID with HR records
   - Confirms agency affiliation
   - Checks role appropriateness
6. **Decision Point:**

**Option A: Approve Account**
1. Click "Approve" button (green checkmark)
2. POST to `/dashboard/admin/users/<pk>/approve/`
3. Backend calls `user.approve_account(request.user)`:
   ```python
   user.account_status = 'approved'
   user.approved_by = admin_user
   user.approved_at = timezone.now()
   user.is_active = True
   user.rejection_reason = ''
   user.save()
   
   # Create activity log
   UserActivityLog.objects.create(
       user=user,
       activity_type='user_management',
       description=f"Account approved by {admin_user.get_full_name()}",
       metadata={'approved_by': admin_user.id}
   )
   
   # Send notification
   UserNotification.objects.create(
       user=user,
       notification_type='system',
       title='Account Approved',
       message=f'Your account has been approved. You can now access the system.'
   )
   ```
4. User can now login with credentials
5. User has access based on assigned role

**Option B: Reject Account**
1. Click "Reject" button (red X)
2. POST to `/dashboard/admin/users/<pk>/reject/`
3. Backend calls `user.reject_account(request.user, reason)`:
   ```python
   user.account_status = 'rejected'
   user.rejection_reason = reason
   user.is_active = False
   user.save()
   
   # Create activity log
   UserActivityLog.objects.create(
       user=user,
       activity_type='user_management',
       description=f"Account rejected by {admin_user.get_full_name()}",
       metadata={'rejected_by': admin_user.id, 'reason': reason}
   )
   
   # Send notification
   UserNotification.objects.create(
       user=user,
       notification_type='system',
       title='Account Rejected',
       message=f'Your account has been rejected. Reason: {reason}'
   )
   ```
4. User cannot login
5. User notified with rejection reason

**Step 3: Post-Approval Setup**

For approved users:
1. User logs in for first time
2. Completes profile setup (optional):
   - Upload profile picture
   - Fill bio
   - Update contact information
3. PA Staff users may need:
   - Field verification training
   - Setting `is_field_verified=True` by admin
   - Assignment to protected area

---

## Testing Scenarios

### 1. User Approval Workflow

**Setup:**
1. Logout (or use incognito browser)
2. Navigate to `/users/register/`
3. Fill registration form:
   - Username: `pa_staff_001`
   - Email: `pa.staff@example.com`
   - Password: (strong password)
   - Name: Juan Dela Cruz
   - Agency: DENR Region XII
   - Department: Mt. Matutum Office
   - Role: PA Staff
   - Employee ID: 2024-001
4. Submit form

**Expected Results:**
- User created with `account_status='pending'`
- User redirected to "Pending Approval" page
- Cannot login yet (`is_active=False`)
- System Admins receive notification

**Admin Actions:**
1. Login as System Admin
2. Navigate to Admin Dashboard
3. See "1 Pending Account" badge
4. Open User & Organization Hub → Users → Pending tab
5. See new user in list
6. Click "Approve" button

**Expected Results After Approval:**
- User `account_status` changes to 'approved'
- User `is_active` becomes True
- User receives "Account Approved" notification
- UserActivityLog entry created
- User can now login

**Validation:**
- User login successful
- User redirected to role-appropriate dashboard (PA Staff Home)
- User can access field data submission forms
- User profile shows "Approved" status

---

### 2. Agency Hierarchy Management

**Setup:**
1. Login as DENR staff or System Admin
2. Navigate to User & Organization Hub → Agencies section
3. Create parent agency:
   - Name: Department of Environment and Natural Resources
   - Agency Type: Government Agency
   - Acronym: DENR
4. Create child agency:
   - Name: DENR Region XII
   - Agency Type: Government Agency
   - Acronym: DENR-12
   - Parent Agency: DENR (select from dropdown)

**Expected Results:**
- Both agencies created successfully
- Child agency displays with hierarchy indicator
- `get_hierarchy_path()` returns "DENR > DENR Region XII"
- Child agency shows parent in agency list

**Additional Test:**
1. Try to delete parent agency (DENR)
2. Should show error: "Cannot delete agency with sub-agencies"
3. Must delete child agencies first

**Validation:**
- Hierarchy properly established
- Deletion constraints enforced
- Agency list shows nested structure

---

### 3. Department User Assignment

**Setup:**
1. Create agency: DENR Region XII
2. Create departments:
   - Field Operations (no parent)
   - Mt. Matutum Team (parent: Field Operations)
3. Create user:
   - Role: PA Staff
   - Agency: DENR Region XII
   - Department: Mt. Matutum Team
   - Employee ID: 2024-002
4. Approve user

**Expected Results:**
- User successfully assigned to department
- User appears in department user count
- `department.get_all_staff()` includes user
- Parent department (Field Operations) also shows user in recursive staff count

**Test Department Head Assignment:**
1. Navigate to Department Edit for "Mt. Matutum Team"
2. Set Department Head to approved user
3. Save

**Expected Results:**
- User now appears as department head
- User listed in `headed_departments` relationship
- Department shows head name in list view

**Validation:**
- User-department relationship correct
- Hierarchical staff counting works
- Department head assignment functional

---

### 4. Address Hierarchy Integrity

**Setup:**
1. Navigate to Address Management Hub
2. Create address hierarchy:
   - Region: Region XII (SOCCSKSARGEN)
   - Province: South Cotabato
   - Municipality: Tupi
   - Barangay: Poblacion
3. Try to delete municipality before deleting barangays

**Expected Results:**
- All 4 levels created successfully
- Deletion of municipality blocked
- Error message: "Cannot delete municipality with existing barangays"
- Must delete barangay first, then municipality

**Test Cascading Display:**
1. Navigate to Municipalities tab
2. See "Tupi" with:
   - Province: South Cotabato
   - Region: Region XII
   - Barangay Count: 1
3. Navigate to Barangays tab
4. See "Poblacion" with full hierarchy:
   - Municipality: Tupi
   - Province: South Cotabato
   - Region: Region XII

**Validation:**
- Foreign key constraints enforced
- Cascading deletions prevented
- Hierarchical display accurate
- Child counts correct

---

### 5. User Rejection with Reason

**Setup:**
1. Create new user registration (pending status)
2. Login as System Admin
3. Navigate to Pending Users
4. Click "Reject" button for user

**Current Behavior:**
- Uses default reason: "Rejected by admin"
- User receives generic rejection notification

**Enhanced Test (if rejection reason form exists):**
1. Click "Reject" button
2. Fill rejection reason: "Invalid employee ID - does not match HR records"
3. Submit

**Expected Results:**
- User `account_status` = 'rejected'
- User `is_active` = False
- `rejection_reason` field populated with admin's reason
- UserActivityLog created with reason in metadata
- User receives notification with specific reason

**Validation:**
- User cannot login
- Notification includes specific reason
- Activity log shows admin who rejected and reason

---

### 6. Field Verification Status

**Setup:**
1. Create PA Staff user and approve
2. User logs in and tries to submit data
3. Without field verification, submission should be blocked

**Test Workflow:**
1. System Admin edits user
2. Checks "Field Verified" checkbox
3. Saves user
4. PA Staff user tries data submission again

**Expected Results:**
- `user.can_collect_data()` returns False before verification
- Data submission forms show "Field verification required" message
- After admin sets `is_field_verified=True`:
  - `user.can_collect_data()` returns True
  - User can access data submission forms
  - User can submit observations

**Validation:**
- Permission check enforced
- Field verification requirement prevents data collection
- Admin can toggle verification status

---

### 7. Years of Service Calculation

**Setup:**
1. Create PA Staff user
2. Set `date_joined_organization` to 3 years, 2 months, 15 days ago
3. View user profile

**Expected Results:**
- `get_years_of_service()` returns tuple: `(3, 2, 15)`
- `get_years_of_service_display()` returns: "3 years, 2 months"
- Profile displays service duration

**Edge Cases:**
- User with no `date_joined_organization`: Returns None/"Not set"
- User joined today: "0 days"
- User joined 11 months ago: "11 months"
- User joined 1 year, 0 months ago: "1 year"

**Validation:**
- Calculation accurate using relativedelta
- Display format human-readable
- Handles None values gracefully

---

### 8. Agency Type Filtering

**Setup:**
1. Create agencies of different types:
   - DENR (Government Agency)
   - Tupi Municipal Government (LGU)
   - SEARCA (Academic Institution)
   - Conservation International (NGO)
2. Navigate to Agencies section
3. Apply filter: Agency Type = "Government Agency"

**Expected Results:**
- Only DENR appears in list
- Filter UI shows selected type
- Other agencies hidden
- Clear filter option available

**Test Multiple Filters:**
1. Apply: Type = "Government Agency" AND Active = "Yes"
2. Only active government agencies shown
3. Combine with search: "DENR"

**Validation:**
- Filtering accurate
- Multiple filters combine correctly
- Search works with filters
- Clear filters resets view

---

### 9. Department Sub-Department Structure

**Setup:**
1. Create agency: DENR Region XII
2. Create parent department: Biodiversity Management Bureau
3. Create child department: Wildlife Section
4. Create grandchild department: Bird Monitoring Team
5. Assign users to each level

**Expected Results:**
- Three-level hierarchy established
- `get_hierarchy_path()` for grandchild: "Biodiversity Management Bureau > Wildlife Section > Bird Monitoring Team"
- `get_all_staff()` on parent includes all descendant users

**Test Recursive Staff Counting:**
1. Assign 2 users to parent
2. Assign 3 users to child
3. Assign 5 users to grandchild
4. Call `parent.get_all_staff()`

**Expected Results:**
- Returns 10 users total (2 + 3 + 5)
- Recursive traversal works correctly
- No duplicates in result set

**Validation:**
- Multi-level hierarchy supported
- Recursive methods work correctly
- Staff aggregation accurate

---

### 10. User Search Across Multiple Fields

**Setup:**
1. Create 5 users with varied information:
   - User 1: John Doe, `john.doe@denr.gov.ph`
   - User 2: Jane Smith, `jane.smith@example.com`
   - User 3: Juan Dela Cruz, `jdelacruz@gmail.com`
   - User 4: Maria Santos, `msantos@denr.gov.ph`
   - User 5: Pedro Reyes, `preyes@example.com`
2. Navigate to User & Organization Hub
3. Test searches:
   - Search: "john" → Should find John Doe
   - Search: "denr.gov.ph" → Should find users 1 and 4
   - Search: "cruz" → Should find Juan Dela Cruz
   - Search: "jane.smith@example.com" → Should find Jane Smith

**Expected Results:**
- Search across `username`, `email`, `first_name`, `last_name`
- Results update in real-time (if AJAX) or on form submit
- Pagination maintained with search
- "No results" message if no matches

**Validation:**
- Multi-field search works
- Partial matches found
- Case-insensitive search
- Search persists across pagination

---

## Integration Points

### 1. KoboToolbox Integration

User assignments affect KoboToolbox data submission and ownership.

**Integration:**
- PA Staff users with `can_collect_data()=True` can submit via Kobo forms
- `submitted_by` field links DataSubmission to User
- Agency and department context preserved in submissions

---

### 2. Validation System Integration

User roles determine validation workflow participation.

**Integration:**
- Only users with `can_validate()=True` appear in validator assignment dropdowns
- Validators must have `role='validator'` and `account_status='approved'`
- Validation history tracks `validated_by` user

---

### 3. Protected Area Integration

PA Staff users are assigned to specific protected areas.

**Integration:**
- User profile includes protected area assignments (via PAStaffProfile)
- Users can only submit data for their assigned protected areas
- Geographic filtering in dashboards based on user's area

---

### 4. Report Generation Integration

User roles determine report access and generation permissions.

**Integration:**
- `can_view_reports()`: All approved users (role-based filtering)
- `can_build_reports()`: Only Validators and DENR staff
- Report `requested_by` field links to User
- Report access control via `GeneratedReport.can_access(user)`

---

### 5. Notification System Integration

User management actions trigger system notifications.

**Integration:**
- Account approval/rejection sends UserNotification
- New pending user registration notifies System Admins
- UserActivityLog tracks all user management actions
- Email notifications (if configured) for critical events

---

## Key Takeaways

1. **Approval Workflow** - All new users start as `pending` and require System Admin approval before accessing the system, ensuring accountability and security.

2. **Hierarchical Organizations** - Agencies and Departments support multi-level hierarchies with parent-child relationships, reflecting real-world organizational structures.

3. **4-Tier Address System** - Philippine address hierarchy (Region → Province → Municipality → Barangay) with cascading foreign keys and deletion protection.

4. **Role-Based Permissions** - 5 distinct user roles with granular permission methods (`can_collect_data()`, `can_validate()`, etc.) enforced throughout the system.

5. **Activity Auditing** - All user management actions logged in UserActivityLog with metadata for compliance and troubleshooting.

6. **Field Verification** - PA Staff users require additional `is_field_verified=True` flag before data collection, ensuring proper training.

7. **Unified Management Hub** - Single interface combining Users, Agencies, Departments, and Addresses reduces navigation complexity.

8. **Constraint Enforcement** - Database-level foreign key constraints prevent orphaned records (e.g., cannot delete agency with users).

9. **Years of Service Tracking** - Automatic calculation of tenure based on `date_joined_organization` for PA Staff recognition.

10. **Flexible User Model** - Extended AbstractUser with custom fields avoids separate profile tables while maintaining Django's built-in auth features.

---

## Future Enhancements

### Planned Features

1. **Bulk User Import**
   - CSV upload for batch user creation
   - Email validation and duplicate detection
   - Auto-assignment to agencies/departments via lookup

2. **Advanced Approval Workflow**
   - Multi-step approval (Department Head → Admin)
   - Conditional auto-approval based on criteria
   - Email notifications at each step

3. **User Groups/Teams**
   - Create cross-department teams for projects
   - Team-based data access permissions
   - Team activity dashboards

4. **Enhanced Rejection Handling**
   - Custom rejection reason form (currently uses default)
   - Re-application workflow for rejected users
   - Rejection appeal process

5. **Two-Factor Authentication**
   - TOTP implementation using `totp_secret` field
   - Backup codes system
   - Optional/mandatory 2FA by role

6. **User Import/Export**
   - Export user list to CSV/Excel
   - Import users from HR systems
   - Bulk role/agency assignment

7. **Department Workflow Automation**
   - Auto-assign users to department on approval
   - Department-specific onboarding checklists
   - Department head approval for submissions

8. **Geographic Boundary Management**
   - Add polygon boundaries to regions/provinces
   - Visualize address hierarchy on map
   - Auto-assign users based on location

---

**Last Updated:** January 6, 2026  
**Document Version:** 1.0  
**Related Documentation:**
- [Dashboard & Widget System](DASHBOARD_WIDGET_SYSTEM.md)
- [Validation System](VALIDATION_QUEUE_WORKSPACE.md)
- [Report Generation System](REPORT_GENERATION_SYSTEM.md)
- [Role-Based Access Control](../architecture/RBAC_DOCUMENTATION.md)
