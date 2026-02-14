# PRISM-Matutum Role-Based Access Control (RBAC) Documentation

**Document Version:** 1.0  
**Last Updated:** January 6, 2026  
**Author:** Documentation for Capstone Project  

---

## Table of Contents

1. [Overview](#overview)
2. [User Roles Definition](#user-roles-definition)
3. [User Model & Fields](#user-model--fields)
4. [Account Approval Workflow](#account-approval-workflow)
5. [Permission Methods](#permission-methods)
6. [Role Capabilities Matrix](#role-capabilities-matrix)
7. [Permission Mixins](#permission-mixins)
8. [Organization Structure](#organization-structure)
9. [Security Implementation](#security-implementation)
10. [Code Examples](#code-examples)

---

## 1. Overview

PRISM-Matutum implements a comprehensive **Role-Based Access Control (RBAC)** system that governs what users can access and perform within the system. The RBAC system is built into the User model and enforced through Django mixins, decorators, and permission checks throughout the application.

### Key RBAC Components

1. **User Roles** - 5 distinct roles with specific responsibilities
2. **Account Status** - Approval workflow for new users
3. **Permission Methods** - Python methods that check user capabilities
4. **Access Mixins** - Class-based view mixins for role enforcement
5. **Function Decorators** - Function-based view decorators for role checks
6. **Organization Hierarchy** - Agency and Department structure for user organization

### RBAC Philosophy

- **Least Privilege**: Users receive only the permissions necessary for their role
- **Separation of Duties**: Different roles handle different stages of the workflow
- **Account Approval**: New users require administrator approval before accessing the system
- **Hierarchical Organization**: Users belong to agencies and departments for organizational structure

---

## 2. User Roles Definition

### 2.1 PA Staff (`pa_staff`)

**Primary Function**: Field data collectors

**Responsibilities**:
- Collect biodiversity data using KoboToolbox mobile app
- Submit wildlife observations, resource use incidents, disturbance records
- View their own submission history
- Track validation status of their submissions
- View approved observations

**Access Level**: Limited - Own data only

**Typical Users**: Park rangers, field technicians, monitoring staff

**Key Characteristics**:
- Must complete field verification training (`is_field_verified`)
- Can only see their own submissions
- Cannot validate or approve data
- Cannot manage users or system settings

---

### 2.2 Validator (`validator`)

**Primary Function**: Data quality assurance and validation

**Responsibilities**:
- Review pending submissions in validation queue
- Approve, reject, or hold submissions for clarification
- Add validation remarks and feedback
- Assign submissions to themselves or other validators
- Access validation history and audit trails
- Generate and view reports

**Access Level**: High - All submissions for validation

**Typical Users**: Senior biologists, data quality specialists, research coordinators

**Key Characteristics**:
- Core role in data quality workflow
- Can validate all submissions, not just assigned ones
- Can access advanced analytics and reporting tools
- Cannot manage users but can view user information

---

### 2.3 DENR Personnel (`denr`)

**Primary Function**: Government oversight and reporting

**Responsibilities**:
- View approved biodiversity data
- Generate comprehensive reports
- Export data for government reporting
- View analytics and trends
- Access report archives

**Access Level**: Read-only - All approved data

**Typical Users**: DENR officials, government auditors, policy makers

**Key Characteristics**:
- Read-only access to validated data
- Cannot validate submissions
- Can generate but not edit data
- Focus on reporting and oversight

---

### 2.4 System Administrator (`system_admin`)

**Primary Function**: System management and user administration

**Responsibilities**:
- Approve or reject user registrations
- Manage agencies and departments
- Configure system settings
- Manage choice lists and form mappings
- Sync with KoboToolbox
- Manage taxonomy and species data
- Access all system features

**Access Level**: Full - Complete system access

**Typical Users**: IT administrators, system managers, project leads

**Key Characteristics**:
- Only role that can manage users
- Can create/edit/delete agencies and departments
- Can access Django admin interface
- Bypasses most permission checks

---

### 2.5 Collaborator / External Researcher (`collaborator`)

**Primary Function**: External research access

**Responsibilities**:
- View approved biodiversity data
- Access species database
- View public reports
- Export data for research purposes

**Access Level**: Read-only - Approved data only

**Typical Users**: University researchers, NGO partners, external consultants

**Key Characteristics**:
- Read-only access to approved observations
- Cannot submit, validate, or manage data
- Cannot access user management or system settings
- Focus on data consumption for research

---

## 3. User Model & Fields

### Core User Model Structure

The `User` model extends Django's `AbstractUser` with additional fields for RBAC:

```python
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    # Role Selection
    role = models.CharField(
        max_length=20,
        choices=USER_ROLES,
        default='pa_staff'
    )
    
    # Account Status
    account_status = models.CharField(
        max_length=20,
        choices=ACCOUNT_STATUS_CHOICES,
        default='pending'
    )
    
    # Organization Links
    agency = models.ForeignKey(Agency, ...)
    department = models.ForeignKey(Department, ...)
    
    # PA Staff Fields
    employee_id = models.CharField(...)
    position = models.CharField(...)
    is_field_verified = models.BooleanField(default=False)
    
    # Approval Workflow
    approval_requested_at = models.DateTimeField(...)
    approved_by = models.ForeignKey('self', ...)
    approved_at = models.DateTimeField(...)
    rejection_reason = models.TextField(...)
```

### Key Model Fields

| Field | Type | Purpose |
|-------|------|---------|
| `role` | CharField | User's role (pa_staff, validator, denr, system_admin, collaborator) |
| `account_status` | CharField | Approval status (pending, approved, rejected, suspended) |
| `agency` | ForeignKey | Organization the user belongs to |
| `department` | ForeignKey | Department within the organization |
| `employee_id` | CharField | Unique employee identifier |
| `position` | CharField | Job position/title |
| `is_field_verified` | Boolean | Whether PA Staff completed field training |
| `date_joined_organization` | DateField | Date joined for years of service calculation |
| `approved_by` | ForeignKey | Admin who approved the account |
| `approved_at` | DateTimeField | When account was approved |
| `rejection_reason` | TextField | Reason if account was rejected |
| `is_2fa_enabled` | Boolean | Two-factor authentication status |

---

## 4. Account Approval Workflow

All new users must be approved by a System Administrator before accessing the system. Staff and superusers bypass this check.

### Account Status States

```
┌─────────────────────────────────────────────────────────────────┐
│                    ACCOUNT STATUS FLOW                          │
└─────────────────────────────────────────────────────────────────┘

    NEW USER
    REGISTRATION
         │
         ↓
    ┌─────────┐
    │ PENDING │  ← Default status for new registrations
    └────┬────┘
         │
         │ (Admin Review)
         │
         ├─────────────┬─────────────┬───────────────┐
         ↓             ↓             ↓               ↓
    ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐
    │APPROVED │  │ REJECTED │  │SUSPENDED │  │ (PENDING)  │
    └─────────┘  └──────────┘  └──────────┘  └────────────┘
         │             │             │               │
         │             │             │               │
         ↓             ↓             ↓               ↓
   Full Access   No Access    Temp. Locked   Awaiting Approval
    (Active)     (Inactive)    (Inactive)
```

### Account Status Values

| Status | Description | Active | Can Login |
|--------|-------------|--------|-----------|
| **pending** | Awaiting admin approval | No | No |
| **approved** | Account approved and active | Yes | Yes |
| **rejected** | Account rejected by admin | No | No |
| **suspended** | Temporarily suspended | No | No |

### Approval Workflow Methods

#### `request_approval()`
Marks account as pending approval (sets `approval_requested_at` timestamp).

```python
user.request_approval()
```

#### `approve_account(admin_user)`
Approves a user account. Only System Administrators can approve.

```python
user.approve_account(admin_user=request.user)
# - Sets account_status = 'approved'
# - Sets approved_by = admin_user
# - Sets approved_at = now
# - Sets is_active = True
# - Creates activity log entry
# - Sends notification to user
```

#### `reject_account(admin_user, reason)`
Rejects a user account with a reason.

```python
user.reject_account(
    admin_user=request.user,
    reason="Insufficient credentials provided"
)
# - Sets account_status = 'rejected'
# - Sets rejection_reason
# - Sets is_active = False
# - Creates activity log entry
# - Sends notification to user
```

#### `suspend_account(admin_user, reason)`
Suspends an active user account.

```python
user.suspend_account(
    admin_user=request.user,
    reason="Policy violation"
)
# - Sets account_status = 'suspended'
# - Sets is_active = False
# - Creates activity log entry
# - Sends notification to user
```

### Approval Process Flow

```python
# In dashboard/views_admin.py
class UserApproveView(StaffUserRequiredMixin, View):
    def post(self, request, pk):
        user = get_object_or_404(User, pk=pk)
        
        # Approve the user
        user.approve_account(admin_user=request.user)
        
        messages.success(
            request,
            f"User {user.get_full_name()} has been approved."
        )
        return redirect('dashboard:user_management')
```

---

## 5. Permission Methods

The User model includes several permission check methods that return boolean values based on role and status.

### Role Check Methods

These methods check if the user has a specific role:

```python
# Role checks - Return True/False
user.is_pa_staff()        # role == 'pa_staff'
user.is_validator()       # role == 'validator'
user.is_denr()            # role == 'denr'
user.is_system_admin()    # role == 'system_admin'
user.is_collaborator()    # role == 'collaborator'
```

### Capability Check Methods

These methods check if the user can perform specific actions:

#### `can_collect_data()`
Can the user submit field data?

```python
def can_collect_data(self):
    return (
        self.role == 'pa_staff' and
        self.is_field_verified and
        self.account_status == 'approved'
    )
```

**Requirements**:
- Role: PA Staff
- Field verified: True
- Account status: Approved

---

#### `can_validate()`
Can the user validate submissions?

```python
def can_validate(self):
    return (
        self.role == 'validator' and
        self.account_status == 'approved'
    )
```

**Requirements**:
- Role: Validator
- Account status: Approved

---

#### `can_view_reports()`
Can the user view generated reports (read-only)?

```python
def can_view_reports(self):
    return (
        self.role in ['pa_staff', 'collaborator', 'denr', 'validator'] and
        self.account_status == 'approved'
    )
```

**Requirements**:
- Role: PA Staff, Collaborator, DENR, or Validator
- Account status: Approved

---

#### `can_build_reports()`
Can the user create and generate new reports?

```python
def can_build_reports(self):
    return (
        self.role in ['denr', 'validator'] and
        self.account_status == 'approved'
    )
```

**Requirements**:
- Role: DENR or Validator (NOT PA Staff)
- Account status: Approved

---

#### `can_manage_users()`
Can the user manage other users?

```python
def can_manage_users(self):
    return (
        self.role == 'system_admin' and
        self.account_status == 'approved'
    )
```

**Requirements**:
- Role: System Administrator
- Account status: Approved

---

### Permission Method Usage

```python
# In a view
if request.user.can_validate():
    # Show validation queue
    submissions = DataSubmission.objects.filter(validation_status='---')
else:
    # Deny access
    return HttpResponseForbidden()

# In a template
{% if user.can_build_reports %}
    <a href="{% url 'dashboard:report_builder' %}">Create Report</a>
{% endif %}
```

---

## 6. Role Capabilities Matrix

This table shows what each role can and cannot do:

| Capability | PA Staff | Validator | DENR | System Admin | Collaborator |
|------------|----------|-----------|------|--------------|--------------|
| **Data Collection** |
| Submit field data | ✅ (if field verified) | ❌ | ❌ | ✅ | ❌ |
| View own submissions | ✅ | ❌ | ❌ | ✅ | ❌ |
| Edit own submissions | ✅ (before validation) | ❌ | ❌ | ✅ | ❌ |
| Delete own submissions | ✅ (before validation) | ❌ | ❌ | ✅ | ❌ |
| **Validation** |
| Access validation queue | ❌ | ✅ | ❌ | ✅ | ❌ |
| Approve submissions | ❌ | ✅ | ❌ | ✅ | ❌ |
| Reject submissions | ❌ | ✅ | ❌ | ✅ | ❌ |
| View validation history | ❌ | ✅ | ❌ | ✅ | ❌ |
| Assign submissions | ❌ | ✅ | ❌ | ✅ | ❌ |
| **Data Viewing** |
| View own observations | ✅ | ❌ | ❌ | ✅ | ❌ |
| View all approved data | ❌ | ✅ | ✅ | ✅ | ✅ |
| View pending data | ❌ | ✅ | ❌ | ✅ | ❌ |
| View rejected data | ❌ | ✅ | ❌ | ✅ | ❌ |
| **Reporting** |
| View reports | ✅ (read-only) | ✅ | ✅ | ✅ | ✅ (read-only) |
| Generate reports | ❌ | ✅ | ✅ | ✅ | ❌ |
| Export data | ❌ | ✅ | ✅ | ✅ | ✅ (approved only) |
| Download shapefiles | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Species Management** |
| View species list | ✅ | ✅ | ✅ | ✅ | ✅ |
| Add/Edit species | ❌ | ✅ | ❌ | ✅ | ❌ |
| Delete species | ❌ | ❌ | ❌ | ✅ | ❌ |
| Add species articles | ❌ | ✅ | ❌ | ✅ | ❌ |
| **GIS Features** |
| View interactive map | ✅ | ✅ | ✅ | ✅ | ✅ |
| Add monitoring locations | ❌ | ✅ | ❌ | ✅ | ❌ |
| Edit monitoring locations | ❌ | ✅ | ❌ | ✅ | ❌ |
| Add transect routes | ❌ | ✅ | ❌ | ✅ | ❌ |
| **User Management** |
| View users | ❌ | ❌ | ❌ | ✅ | ❌ |
| Approve users | ❌ | ❌ | ❌ | ✅ | ❌ |
| Edit user roles | ❌ | ❌ | ❌ | ✅ | ❌ |
| Delete users | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Organization Management** |
| View agencies | ❌ | ✅ (read-only) | ✅ (read-only) | ✅ | ❌ |
| Manage agencies | ❌ | ❌ | ❌ | ✅ | ❌ |
| Manage departments | ❌ | ❌ | ❌ | ✅ | ❌ |
| **System Configuration** |
| Manage choice lists | ❌ | ❌ | ❌ | ✅ | ❌ |
| Sync KoboToolbox | ❌ | ❌ | ❌ | ✅ | ❌ |
| Manage taxonomy | ❌ | ✅ | ❌ | ✅ | ❌ |
| Access Django admin | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Dashboards** |
| Create custom dashboards | ✅ | ✅ | ✅ | ✅ | ✅ |
| Add widgets | ✅ | ✅ | ✅ | ✅ | ✅ |
| Share dashboards | ✅ | ✅ | ✅ | ✅ | ✅ |

**Legend**:
- ✅ = Allowed
- ❌ = Not allowed
- ✅ (condition) = Allowed with conditions

---

## 7. Permission Mixins

Permission mixins are class-based view mixins that enforce role-based access control. They are defined in [dashboard/permissions.py](dashboard/permissions.py).

### Base Mixins

#### `DashboardAccessMixin`

**Purpose**: Base mixin for all dashboard views. Ensures user is authenticated and has an approved account.

**Behavior**:
- Requires authentication (redirects to login if not authenticated)
- Checks `account_status == 'approved'` (staff/superusers bypass this)
- Redirects unapproved users to login with error message

**Usage**:
```python
class MyView(DashboardAccessMixin, TemplateView):
    template_name = 'my_template.html'
```

**Inheritance Chain**: All other permission mixins inherit from this.

---

#### `MultiRoleAccessMixin`

**Purpose**: Allows access to multiple user roles. Staff and superusers bypass role checks.

**Configuration**:
```python
class MyView(MultiRoleAccessMixin, ListView):
    allowed_roles = ['validator', 'system_admin']
```

**Behavior**:
- Checks if `user.role` is in `allowed_roles`
- Staff and superusers bypass role check
- Returns 403 for AJAX requests, redirects for normal requests

**Error Handling**:
- AJAX: Returns JSON with error message (403 status)
- Normal: Shows error message and redirects to appropriate dashboard

---

### Role-Specific Mixins

#### `PAStaffAccessMixin`

**Purpose**: Restricts access to PA Staff only.

**Requirements**:
- Role: pa_staff
- Field verified: True (additional check)

**Usage**:
```python
class MySubmissionsView(PAStaffAccessMixin, ListView):
    model = DataSubmission
```

---

#### `ValidatorAccessMixin`

**Purpose**: Restricts access to Validators and Staff/Superusers.

**Requirements**:
- Role: validator OR staff/superuser

**Usage**:
```python
class ValidationQueueView(ValidatorAccessMixin, ListView):
    template_name = 'dashboard/validation_queue.html'
```

---

#### `DENRAccessMixin`

**Purpose**: Restricts access to DENR and Validators.

**Allowed Roles**: `['denr', 'validator']`

**Usage**:
```python
class DENRDashboardView(DENRAccessMixin, TemplateView):
    template_name = 'dashboard/denr_dashboard.html'
```

---

#### `CollaboratorAccessMixin`

**Purpose**: Restricts access to Collaborators, Validators, and DENR.

**Allowed Roles**: `['collaborator', 'validator', 'denr']`

**Usage**:
```python
class ReportArchiveView(CollaboratorAccessMixin, ListView):
    model = GeneratedReport
```

---

#### `AdminAccessMixin`

**Purpose**: Restricts access to System Administrators only.

**Allowed Roles**: `['system_admin']`

**Usage**:
```python
class UserManagementView(AdminAccessMixin, TemplateView):
    template_name = 'dashboard/admin/user_management.html'
```

---

#### `StaffUserRequiredMixin`

**Purpose**: Restricts access to staff users (Django staff flag or System Admin role).

**Requirements**:
- `is_staff == True` OR `role == 'system_admin'`

**Usage**:
```python
class AdminDashboardView(StaffUserRequiredMixin, TemplateView):
    template_name = 'dashboard/admin/admin_dashboard.html'
```

---

### Permission-Based Mixins

These mixins use the permission check methods from the User model:

#### `CanValidateMixin`

Uses `user.can_validate()` to check validation permissions.

```python
class SomeValidationView(CanValidateMixin, FormView):
    pass
```

---

#### `CanManageUsersMixin`

Uses `user.can_manage_users()` to check user management permissions.

```python
class UserApprovalView(CanManageUsersMixin, View):
    pass
```

---

#### `CanViewReportsMixin`

Uses `user.can_view_reports()` to check report viewing permissions.

```python
class ReportArchiveView(CanViewReportsMixin, ListView):
    pass
```

---

#### `CanBuildReportsMixin`

Uses `user.can_build_reports()` to check report generation permissions.

```python
class ReportBuilderView(CanBuildReportsMixin, FormView):
    pass
```

---

### Queryset Filtering Mixin

#### `RoleBasedQuerysetMixin`

**Purpose**: Automatically filters querysets based on user role.

**Default Behavior**:
- **PA Staff**: Only their own records
- **Validators**: Assigned or unassigned records
- **DENR/Admin**: All records

**Usage**:
```python
class MySubmissionsView(RoleBasedQuerysetMixin, ListView):
    model = DataSubmission
    
    def get_role_filtered_queryset(self, queryset):
        # Override to customize filtering
        if self.request.user.is_pa_staff():
            return queryset.filter(username=self.request.user.username)
        return queryset
```

---

## 8. Organization Structure

Users belong to **Agencies** and **Departments** for organizational hierarchy.

### Agency Model

Represents government agencies, NGOs, academic institutions, etc.

```python
class Agency(models.Model):
    AGENCY_TYPES = [
        ('government', 'Government Agency'),
        ('lgu', 'Local Government Unit'),
        ('ngo', 'Non-Government Organization'),
        ('academic', 'Academic Institution'),
        ('private', 'Private Organization'),
    ]
    
    name = models.CharField(max_length=200, unique=True)
    agency_type = models.CharField(choices=AGENCY_TYPES)
    acronym = models.CharField(max_length=20)
    parent_agency = models.ForeignKey('self', ...)  # Hierarchical
    
    # Contact info
    contact_person = models.CharField(...)
    contact_email = models.EmailField(...)
    contact_phone = models.CharField(...)
```

**Features**:
- Hierarchical structure (parent_agency)
- Multiple types (government, LGU, NGO, academic, private)
- Contact information
- Active/inactive status

---

### Department Model

Represents departments within agencies.

```python
class Department(models.Model):
    name = models.CharField(max_length=200)
    agency = models.ForeignKey(Agency, ...)
    parent_department = models.ForeignKey('self', ...)  # Hierarchical
    code = models.CharField(...)  # Department code
    head = models.ForeignKey(User, ...)  # Department head
```

**Features**:
- Belongs to an Agency
- Hierarchical structure (parent_department)
- Department head assignment
- Office location and contact info

---

### Organization Hierarchy Example

```
DENR (Agency)
├── Region XI Office (Agency - child of DENR)
│   ├── Biodiversity Management Section (Department)
│   │   ├── Field Operations Unit (Department - child)
│   │   └── Data Management Unit (Department - child)
│   └── Protected Areas Division (Department)
└── CENRO (Agency - child of DENR)
```

---

## 9. Security Implementation

### Authentication Flow

```
1. User visits protected URL
   ↓
2. DashboardAccessMixin checks authentication
   ↓ (if not authenticated)
3. Redirect to login page
   ↓ (if authenticated)
4. Check account_status
   ↓ (if not approved)
5. Redirect to login with error message
   ↓ (if approved)
6. Check role permissions (MultiRoleAccessMixin)
   ↓ (if role not allowed)
7. Show error and redirect to appropriate dashboard
   ↓ (if role allowed)
8. Render view
```

### Permission Checking Levels

PRISM-Matutum implements **defense in depth** with multiple layers of permission checks:

1. **URL-Level**: Mixins on views prevent access
2. **View-Level**: Explicit permission checks in view logic
3. **Template-Level**: Conditional rendering based on permissions
4. **Object-Level**: Check ownership before editing/deleting

### Staff and Superuser Privileges

Throughout the system, Django `staff` and `superuser` accounts bypass most role checks:

```python
# In MultiRoleAccessMixin
if request.user.is_staff or request.user.is_superuser:
    # Bypass role check
    pass
```

This allows system administrators to access all features regardless of their assigned role.

---

## 10. Code Examples

### Example 1: Protecting a Class-Based View

```python
from dashboard.permissions import ValidatorAccessMixin
from django.views.generic import ListView
from api.models import DataSubmission

class ValidationQueueView(ValidatorAccessMixin, ListView):
    """
    Validation queue - only accessible by Validators and Admins
    """
    model = DataSubmission
    template_name = 'dashboard/validation_queue.html'
    paginate_by = 20
    
    def get_queryset(self):
        # Only show pending submissions
        return DataSubmission.objects.filter(
            validation_status='---'
        ).order_by('-received_date')
```

---

### Example 2: Protecting a Function-Based View

```python
from dashboard.decorators import validator_required
from django.shortcuts import render

@validator_required
def validation_workspace(request, pk):
    """
    Validation workspace - validators only
    """
    submission = get_object_or_404(DataSubmission, pk=pk)
    
    context = {
        'submission': submission,
        'can_validate': request.user.can_validate()
    }
    return render(request, 'dashboard/validation_workspace.html', context)
```

---

### Example 3: Multiple Roles

```python
from dashboard.permissions import MultiRoleAccessMixin

class ReportArchiveView(MultiRoleAccessMixin, ListView):
    """
    Report archive - accessible by DENR, Validators, and Collaborators
    """
    allowed_roles = ['denr', 'validator', 'collaborator']
    model = GeneratedReport
    template_name = 'dashboard/reports/report_archive_list.html'
```

---

### Example 4: Template-Level Permission Checks

```django
{% extends 'dashboard/base.html' %}

{% block content %}
<h1>Dashboard</h1>

{% if user.is_pa_staff %}
    <a href="{% url 'dashboard:my_submissions' %}">My Submissions</a>
{% endif %}

{% if user.is_validator %}
    <a href="{% url 'dashboard:validation_queue' %}">Validation Queue</a>
{% endif %}

{% if user.can_manage_users %}
    <a href="{% url 'dashboard:user_management' %}">Manage Users</a>
{% endif %}

{% if user.can_build_reports %}
    <a href="{% url 'dashboard:report_builder' %}">Create Report</a>
{% elif user.can_view_reports %}
    <a href="{% url 'dashboard:report_archive_list' %}">View Reports</a>
{% endif %}
{% endblock %}
```

---

### Example 5: Object-Level Permissions

```python
from dashboard.permissions import user_can_edit_submission

def edit_submission(request, pk):
    submission = get_object_or_404(DataSubmission, pk=pk)
    
    # Check if user can edit this specific submission
    if not user_can_edit_submission(request.user, submission):
        messages.error(request, "You cannot edit this submission.")
        return redirect('dashboard:my_submissions')
    
    # ... edit logic here
```

---

### Example 6: Custom Queryset Filtering

```python
class MySubmissionsView(PAStaffAccessMixin, RoleBasedQuerysetMixin, ListView):
    model = DataSubmission
    template_name = 'dashboard/my_submissions.html'
    
    def get_role_filtered_queryset(self, queryset):
        # PA Staff only see their own submissions
        return queryset.filter(
            username=self.request.user.username
        ).order_by('-received_date')
```

---

## Summary

PRISM-Matutum's RBAC system provides:

1. **5 Distinct Roles** with clear responsibilities
2. **Account Approval Workflow** for new user registration
3. **Permission Methods** on User model for capability checks
4. **Mixins and Decorators** for enforcing access control
5. **Organization Structure** with agencies and departments
6. **Multi-Layer Security** with defense in depth approach
7. **Flexible Role Assignment** with staff/superuser bypass

The system ensures that users only access features and data appropriate for their role, while maintaining flexibility for administrative oversight.

---

**Next Documentation**:
- [PERMISSION_SYSTEM.md](PERMISSION_SYSTEM.md) - Detailed permission decorators and mixins
- [URL_ROUTING.md](URL_ROUTING.md) - Complete URL structure with role requirements

---

*Document created as part of PRISM-Matutum Capstone Project Documentation*  
*For questions or updates, refer to the project repository.*
