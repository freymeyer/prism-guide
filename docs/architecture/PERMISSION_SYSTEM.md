# PRISM-Matutum Permission System & Decorators Documentation

**Document Version:** 1.0  
**Last Updated:** January 6, 2026  
**Author:** Documentation for Capstone Project  

---

## Table of Contents

1. [Overview](#overview)
2. [Function-Based View Decorators](#function-based-view-decorators)
3. [Class-Based View Mixins](#class-based-view-mixins)
4. [AJAX-Specific Decorators](#ajax-specific-decorators)
5. [Rate Limiting System](#rate-limiting-system)
6. [Permission Utility Functions](#permission-utility-functions)
7. [Implementation Patterns](#implementation-patterns)
8. [Best Practices](#best-practices)
9. [Code Examples](#code-examples)

---

## 1. Overview

PRISM-Matutum implements a multi-layered permission system using both **decorators** (for function-based views) and **mixins** (for class-based views). This document focuses on the decorator system and utility functions defined in [dashboard/decorators.py](dashboard/decorators.py) and [dashboard/permissions.py](dashboard/permissions.py).

### Permission Enforcement Methods

| Method | Use Case | Location |
|--------|----------|----------|
| **Decorators** | Function-based views | `dashboard/decorators.py` |
| **Mixins** | Class-based views | `dashboard/permissions.py` |
| **Permission Methods** | Business logic checks | `users/models.py` (User model) |
| **Template Tags** | Template-level rendering | `{% if user.can_validate %}` |

### Permission Check Flow

```
┌─────────────────────────────────────────────────────────────┐
│                  PERMISSION CHECK FLOW                      │
└─────────────────────────────────────────────────────────────┘

Request
   │
   ↓
[1] Authentication Check
   ├─ Not authenticated → Redirect to login
   └─ Authenticated → Continue
   │
   ↓
[2] Account Status Check
   ├─ Not approved → Error message + Redirect
   └─ Approved → Continue
   │
   ↓
[3] Role Check
   ├─ Role not in allowed_roles → Access denied
   └─ Role allowed → Continue
   │
   ↓
[4] Capability Check (optional)
   ├─ user.can_validate() == False → Access denied
   └─ user.can_validate() == True → Continue
   │
   ↓
[5] Rate Limit Check (optional)
   ├─ Limit exceeded → 429 Too Many Requests
   └─ Within limit → Continue
   │
   ↓
Execute View
```

---

## 2. Function-Based View Decorators

Function-based view decorators are defined in [dashboard/decorators.py](dashboard/decorators.py) and [dashboard/permissions.py](dashboard/permissions.py).

### 2.1 Core Decorators

#### `@dashboard_access_required`

**Purpose**: Ensures user is authenticated and has an approved account.

**Source**: `dashboard/permissions.py`

**Behavior**:
- Checks if user is authenticated
- Checks if `account_status == 'approved'`
- Staff and superusers bypass account status check
- Redirects unapproved users to login

**Usage**:
```python
from dashboard.permissions import dashboard_access_required

@dashboard_access_required
def my_view(request):
    return render(request, 'my_template.html')
```

**Error Response**:
- Redirects to `admin:login`
- Shows message: "Your account is {status}. Please contact an administrator."

---

#### `@multi_role_required(*roles)`

**Purpose**: Restricts access to specific roles. Factory function that creates a decorator.

**Source**: `dashboard/permissions.py`

**Parameters**:
- `*roles` - Variable number of role strings (e.g., 'validator', 'denr')

**Behavior**:
- Checks if `user.role` is in the provided roles list
- Staff and superusers bypass role check
- Returns JSON for AJAX, redirects for normal requests

**Usage**:
```python
from dashboard.permissions import multi_role_required

@multi_role_required('validator', 'denr')
def report_view(request):
    return render(request, 'report.html')
```

**Error Responses**:
- **AJAX**: `JsonResponse({'error': '...', 'status': 'forbidden'}, status=403)`
- **Normal**: Redirects to `admin:index` with error message

---

### 2.2 Role-Specific Decorators

These are convenience decorators that call `multi_role_required()` with predefined roles.

#### `@validator_required`

Restricts access to **Validators only**.

**Source**: `dashboard/permissions.py`

```python
from dashboard.permissions import validator_required

@validator_required
def validation_workspace(request, pk):
    # Only validators can access
    pass
```

**Equivalent to**: `@multi_role_required('validator')`

---

#### `@pa_staff_required`

Restricts access to **PA Staff only**.

```python
from dashboard.permissions import pa_staff_required

@pa_staff_required
def my_submissions(request):
    # Only PA Staff can access
    pass
```

**Equivalent to**: `@multi_role_required('pa_staff')`

---

#### `@denr_required`

Restricts access to **DENR and Validators**.

```python
from dashboard.permissions import denr_required

@denr_required
def denr_dashboard(request):
    # DENR and Validators can access
    pass
```

**Equivalent to**: `@multi_role_required('denr', 'validator')`

---

#### `@admin_required`

Restricts access to **System Administrators only**.

```python
from dashboard.permissions import admin_required

@admin_required
def user_management(request):
    # Only System Admins can access
    pass
```

**Equivalent to**: `@multi_role_required('system_admin')`

---

#### `@collaborator_required`

Restricts access to **Collaborators, Validators, and DENR**.

```python
from dashboard.permissions import collaborator_required

@collaborator_required
def research_data(request):
    # Collaborators, Validators, and DENR can access
    pass
```

**Equivalent to**: `@multi_role_required('collaborator', 'validator', 'denr')`

---

### 2.3 Capability-Based Decorators

These decorators use the User model's capability check methods.

#### `@can_view_reports`

Restricts access to users who **can view reports**.

**Allowed Roles**: PA Staff, Collaborator, DENR, Validator

```python
from dashboard.permissions import can_view_reports

@can_view_reports
def report_archive(request):
    # Users who can view reports can access
    pass
```

**Equivalent to**: `@multi_role_required('pa_staff', 'collaborator', 'denr', 'validator')`

---

#### `@can_build_reports`

Restricts access to users who **can generate reports**.

**Allowed Roles**: DENR, Validator

```python
from dashboard.permissions import can_build_reports

@can_build_reports
def report_builder(request):
    # Only DENR and Validators can generate reports
    pass
```

**Equivalent to**: `@multi_role_required('denr', 'validator')`

---

#### `@can_validate_required`

Restricts access to users who **can validate submissions**.

**Source**: `dashboard/decorators.py`

**Uses**: `user.can_validate()` method

```python
from dashboard.decorators import can_validate_required

@can_validate_required
def approve_submission(request, pk):
    # Only users with can_validate() == True can access
    pass
```

**Requirements**:
- Role: validator
- Account status: approved

---

### 2.4 Backward Compatible Decorators (Deprecated)

These decorators are maintained for backward compatibility but are deprecated.

#### `@validator_required` (Old)

**Source**: `dashboard/decorators.py`

**Status**: **DEPRECATED** - Use the new `@validator_required` from `permissions.py`

```python
# Old way (deprecated)
from dashboard.decorators import validator_required

# New way (recommended)
from dashboard.permissions import validator_required
```

---

#### `@validator_or_admin_required` (Old)

**Source**: `dashboard/decorators.py`

**Status**: **DEPRECATED** - Redirects to new validator decorator

```python
# This is deprecated
from dashboard.decorators import validator_or_admin_required
```

---

## 3. Class-Based View Mixins

Mixins are used with Django's class-based views to enforce permissions at the view level.

### 3.1 Base Mixins

#### `DashboardAccessMixin`

**Purpose**: Base mixin for all dashboard views.

**Source**: `dashboard/permissions.py`

**Behavior**:
- Requires authentication
- Checks account approval status
- Staff/superusers bypass approval check

**Usage**:
```python
from dashboard.permissions import DashboardAccessMixin
from django.views.generic import TemplateView

class MyView(DashboardAccessMixin, TemplateView):
    template_name = 'my_template.html'
```

---

#### `MultiRoleAccessMixin`

**Purpose**: Allows access to multiple roles.

**Configuration**: Set `allowed_roles` class attribute.

**Usage**:
```python
from dashboard.permissions import MultiRoleAccessMixin
from django.views.generic import ListView

class ReportListView(MultiRoleAccessMixin, ListView):
    allowed_roles = ['validator', 'denr']
    model = GeneratedReport
```

---

### 3.2 Role-Specific Mixins

#### `PAStaffAccessMixin`

Restricts to PA Staff only. Adds additional check for `is_field_verified`.

```python
from dashboard.permissions import PAStaffAccessMixin

class MySubmissionsView(PAStaffAccessMixin, ListView):
    model = DataSubmission
```

---

#### `ValidatorAccessMixin`

Restricts to Validators and Staff/Superusers.

```python
from dashboard.permissions import ValidatorAccessMixin

class ValidationQueueView(ValidatorAccessMixin, ListView):
    model = DataSubmission
```

---

#### `DENRAccessMixin`

Allows DENR and Validators.

```python
from dashboard.permissions import DENRAccessMixin

class DENRDashboardView(DENRAccessMixin, TemplateView):
    template_name = 'dashboard/denr_dashboard.html'
```

---

#### `CollaboratorAccessMixin`

Allows Collaborators, Validators, and DENR.

```python
from dashboard.permissions import CollaboratorAccessMixin

class ResearchDataView(CollaboratorAccessMixin, ListView):
    model = WildlifeObservation
```

---

#### `AdminAccessMixin`

Restricts to System Administrators only.

```python
from dashboard.permissions import AdminAccessMixin

class UserManagementView(AdminAccessMixin, TemplateView):
    template_name = 'dashboard/admin/user_management.html'
```

---

#### `StaffUserRequiredMixin`

Requires Django staff flag or System Admin role.

```python
from dashboard.permissions import StaffUserRequiredMixin

class AdminDashboardView(StaffUserRequiredMixin, TemplateView):
    template_name = 'dashboard/admin/admin_dashboard.html'
```

---

### 3.3 Capability-Based Mixins

#### `CanValidateMixin`

Uses `user.can_validate()` method.

```python
from dashboard.permissions import CanValidateMixin

class ApprovalView(CanValidateMixin, FormView):
    pass
```

---

#### `CanManageUsersMixin`

Uses `user.can_manage_users()` method.

```python
from dashboard.permissions import CanManageUsersMixin

class UserApprovalView(CanManageUsersMixin, View):
    pass
```

---

#### `CanViewReportsMixin`

Uses `user.can_view_reports()` method.

```python
from dashboard.permissions import CanViewReportsMixin

class ReportArchiveView(CanViewReportsMixin, ListView):
    model = GeneratedReport
```

---

#### `CanBuildReportsMixin`

Uses `user.can_build_reports()` method.

```python
from dashboard.permissions import CanBuildReportsMixin

class ReportBuilderView(CanBuildReportsMixin, FormView):
    pass
```

---

### 3.4 Queryset Filtering Mixin

#### `RoleBasedQuerysetMixin`

**Purpose**: Automatically filters querysets based on user role.

**Default Behavior**:
- **PA Staff**: Only their own records
- **Validators**: Assigned or unassigned records
- **DENR/Admin**: All records

**Usage**:
```python
from dashboard.permissions import RoleBasedQuerysetMixin

class MyView(RoleBasedQuerysetMixin, ListView):
    model = DataSubmission
    
    def get_role_filtered_queryset(self, queryset):
        # Override to customize filtering
        if self.request.user.is_pa_staff():
            return queryset.filter(username=self.request.user.username)
        return queryset
```

---

## 4. AJAX-Specific Decorators

These decorators are designed for AJAX endpoints and return JSON errors instead of redirecting.

### 4.1 `@ajax_validator_required`

**Purpose**: Validates that the user is a validator for AJAX endpoints.

**Source**: `dashboard/decorators.py`

**Returns**: JSON error response (403 status)

**Usage**:
```python
from dashboard.decorators import ajax_validator_required

@ajax_validator_required
def api_validate(request, submission_id):
    # AJAX endpoint for validation
    return JsonResponse({'status': 'ok'})
```

**Error Response**:
```json
{
    "error": "Access denied. Validator privileges required.",
    "status": "forbidden"
}
```

---

### 4.2 `@ajax_dashboard_required`

**Purpose**: Requires approved account for AJAX endpoints.

**Source**: `dashboard/decorators.py`

**Usage**:
```python
from dashboard.decorators import ajax_dashboard_required

@ajax_dashboard_required
def api_dashboard_stats(request):
    return JsonResponse({...})
```

**Error Response**:
```json
{
    "error": "Your account is pending.",
    "status": "forbidden"
}
```

---

## 5. Rate Limiting System

PRISM-Matutum implements a sophisticated rate limiting system using Redis/cache backend to prevent abuse.

### 5.1 `@rate_limit` Decorator

**Purpose**: Generic rate limiting decorator with minute and hour-based limits.

**Source**: `dashboard/decorators.py`

**Parameters**:
- `requests_per_minute` (int): Max requests per minute (default: 60)
- `requests_per_hour` (int, optional): Max requests per hour (default: None)
- `key_func` (callable, optional): Custom cache key generator
- `scope` (str): Scope identifier (default: 'default')

**Usage**:
```python
from dashboard.decorators import rate_limit

@rate_limit(requests_per_minute=30)
def api_validate(request, submission_id):
    # Limited to 30 requests per minute
    pass

@rate_limit(requests_per_minute=10, requests_per_hour=100, scope='report_generation')
def generate_report(request):
    # Limited to 10/min and 100/hour
    pass
```

**Response Headers**:
- `X-RateLimit-Limit`: Maximum requests allowed
- `X-RateLimit-Remaining`: Requests remaining in current window
- `X-RateLimit-Reset`: Unix timestamp when limit resets
- `Retry-After`: Seconds to wait before retrying (on 429 error)

**Error Response (429 Too Many Requests)**:

**AJAX/API**:
```json
{
    "error": "Rate limit exceeded",
    "message": "Too many requests. Please try again in 45 seconds.",
    "retry_after": 45,
    "limit": 30,
    "window": "minute"
}
```

**HTML**:
```html
<h1>429 Too Many Requests</h1>
<p>Rate limit exceeded. Please try again in 45 seconds.</p>
```

---

### 5.2 `@rate_limit_by_ip` Decorator

**Purpose**: Rate limiting based on IP address (for unauthenticated endpoints).

**Parameters**:
- `requests_per_minute` (int): Max requests per minute per IP

**Usage**:
```python
from dashboard.decorators import rate_limit_by_ip

@rate_limit_by_ip(requests_per_minute=20)
def public_api_endpoint(request):
    # Limited to 20 requests per minute per IP
    pass
```

**IP Detection**:
- Uses `HTTP_X_FORWARDED_FOR` header if present (proxy support)
- Falls back to `REMOTE_ADDR`
- Handles comma-separated IP lists (takes first IP)

---

### 5.3 `@api_rate_limit` Decorator

**Purpose**: Default rate limit for API endpoints.

**Defaults**:
- 30 requests per minute
- 500 requests per hour

**Usage**:
```python
from dashboard.decorators import api_rate_limit

@api_rate_limit
def api_view(request):
    # Automatically rate-limited
    pass
```

**Equivalent to**:
```python
@rate_limit(requests_per_minute=30, requests_per_hour=500, scope='api')
```

---

### 5.4 Rate Limit Cache Keys

Rate limits use Redis/cache with the following key structure:

```
rate_limit:{scope}:{user_id}:{path}:minute:{minute_timestamp}
rate_limit:{scope}:{user_id}:{path}:hour:{hour_timestamp}
```

**Example Keys**:
```
rate_limit:api:123:/dashboard/api/validate/:minute:1234567
rate_limit:api:123:/dashboard/api/validate/:hour:12345
rate_limit:ip:192.168.1.1:/dashboard/public/:minute:1234567
```

---

## 6. Permission Utility Functions

Utility functions provide fine-grained permission checks for specific objects.

**Source**: `dashboard/permissions.py`

### 6.1 `user_can_edit_submission(user, submission)`

**Purpose**: Check if user can edit a specific submission.

**Rules**:
- Validators: Can edit any submission
- PA Staff: Can edit only their own unvalidated submissions

**Usage**:
```python
from dashboard.permissions import user_can_edit_submission

if user_can_edit_submission(request.user, submission):
    # Allow editing
else:
    # Deny editing
```

---

### 6.2 `user_can_delete_submission(user, submission)`

**Purpose**: Check if user can delete a specific submission.

**Rules**:
- PA Staff: Can delete only their own unvalidated submissions
- Validators: Cannot delete (only reject)

**Usage**:
```python
from dashboard.permissions import user_can_delete_submission

if user_can_delete_submission(request.user, submission):
    submission.delete()
```

---

### 6.3 `user_can_validate_submission(user, submission)`

**Purpose**: Check if user can validate a submission.

**Rules**:
- Only validators with approved accounts

**Usage**:
```python
from dashboard.permissions import user_can_validate_submission

if user_can_validate_submission(request.user, submission):
    # Show validation buttons
```

---

### 6.4 `get_dashboard_home_url(user)`

**Purpose**: Get appropriate dashboard URL based on user role.

**Returns**: URL name (string)

**Usage**:
```python
from dashboard.permissions import get_dashboard_home_url
from django.urls import reverse

home_url = reverse(get_dashboard_home_url(request.user))
return redirect(home_url)
```

**Return Values**:
- System Admin → `'admin:index'`
- PA Staff → `'dashboard:my_submissions'`
- Validator → `'dashboard:validation_queue'`
- Collaborator → `'dashboard:report_archive_list'`
- DENR → `'dashboard:report_archive_list'`

---

## 7. Implementation Patterns

### 7.1 Combining Decorators

Decorators can be stacked to combine authentication, role, and rate limiting checks:

```python
from dashboard.permissions import validator_required
from dashboard.decorators import rate_limit

@validator_required
@rate_limit(requests_per_minute=30)
def api_validate(request, submission_id):
    # 1. Check if validator
    # 2. Check rate limit
    # 3. Execute view
    pass
```

**Order matters**: Place permission decorators first, then rate limiting.

---

### 7.2 Custom Rate Limit Keys

Create custom cache keys for specific rate limiting scenarios:

```python
from dashboard.decorators import rate_limit

def custom_key_func(request):
    # Rate limit per user per submission type
    submission_type = request.GET.get('type', 'unknown')
    return f"rate_limit:custom:{request.user.id}:{submission_type}"

@rate_limit(requests_per_minute=20, key_func=custom_key_func)
def filtered_submissions(request):
    pass
```

---

### 7.3 Class-Based View with Multiple Mixins

```python
from dashboard.permissions import ValidatorAccessMixin, RoleBasedQuerysetMixin
from django.views.generic import ListView

class ValidationQueueView(ValidatorAccessMixin, RoleBasedQuerysetMixin, ListView):
    """
    Combines permission checking and queryset filtering
    """
    model = DataSubmission
    
    def get_role_filtered_queryset(self, queryset):
        return queryset.filter(validation_status='---')
```

**Mixin Order**: Permission mixins should come first in the inheritance chain.

---

### 7.4 Template-Level Permission Checks

```django
{% if user.is_validator %}
    <a href="{% url 'dashboard:validation_queue' %}">Validation Queue</a>
{% endif %}

{% if user.can_manage_users %}
    <a href="{% url 'dashboard:user_management' %}">Manage Users</a>
{% endif %}

{% if perms.dashboard.can_edit_submission %}
    <button>Edit Submission</button>
{% endif %}
```

---

## 8. Best Practices

### 8.1 Choose the Right Permission Method

| Scenario | Recommended Method |
|----------|-------------------|
| Function-based view | Decorator (`@validator_required`) |
| Class-based view | Mixin (`ValidatorAccessMixin`) |
| AJAX endpoint | AJAX decorator (`@ajax_validator_required`) |
| API endpoint | Rate limit + role decorator |
| Template rendering | Template tag (`{% if user.can_validate %}`) |
| Business logic | User method (`if user.can_validate():`) |

---

### 8.2 Security Principles

1. **Defense in Depth**: Apply multiple layers of permission checks
2. **Fail Closed**: Default to denying access
3. **Check Early**: Validate permissions before expensive operations
4. **Audit Trail**: Log access attempts and denials
5. **Rate Limiting**: Always apply rate limits to API endpoints

---

### 8.3 Performance Considerations

1. **Cache Permission Checks**: Use `@cached_property` for expensive checks
2. **Database Optimization**: Use `select_related()` and `prefetch_related()`
3. **Rate Limit Cache**: Ensure Redis is properly configured
4. **Minimize Middleware**: Only use necessary middleware

---

### 8.4 Error Handling

Always provide clear error messages:

```python
@validator_required
def my_view(request):
    if not submission.can_be_validated():
        messages.error(request, "This submission cannot be validated at this time.")
        return redirect('dashboard:validation_queue')
    # ... continue
```

---

## 9. Code Examples

### Example 1: Function-Based View with Multiple Decorators

```python
from dashboard.permissions import validator_required
from dashboard.decorators import rate_limit
from django.http import JsonResponse

@validator_required
@rate_limit(requests_per_minute=30, requests_per_hour=200)
def api_validate_submission(request, pk):
    """
    API endpoint to validate a submission
    - Requires validator role
    - Rate limited to 30/min, 200/hour
    """
    submission = get_object_or_404(DataSubmission, pk=pk)
    
    # Additional permission check
    if not user_can_validate_submission(request.user, submission):
        return JsonResponse({
            'error': 'You cannot validate this submission'
        }, status=403)
    
    # Perform validation
    decision = request.POST.get('decision')
    submission.validation_status = decision
    submission.validated_by = request.user
    submission.save()
    
    return JsonResponse({
        'status': 'success',
        'message': 'Submission validated'
    })
```

---

### Example 2: Class-Based View with Custom Filtering

```python
from dashboard.permissions import PAStaffAccessMixin, RoleBasedQuerysetMixin
from django.views.generic import ListView
from api.models import DataSubmission

class MySubmissionsView(PAStaffAccessMixin, RoleBasedQuerysetMixin, ListView):
    """
    PA Staff submissions list
    - Only accessible by PA Staff with field verification
    - Automatically filters to user's own submissions
    """
    model = DataSubmission
    template_name = 'dashboard/my_submissions.html'
    paginate_by = 20
    context_object_name = 'submissions'
    
    def get_role_filtered_queryset(self, queryset):
        # Only show current user's submissions
        return queryset.filter(
            username=self.request.user.username
        ).order_by('-received_date')
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        
        # Add statistics
        all_submissions = self.get_role_filtered_queryset(
            DataSubmission.objects.all()
        )
        context['total_submissions'] = all_submissions.count()
        context['pending_submissions'] = all_submissions.filter(
            validation_status='---'
        ).count()
        context['approved_submissions'] = all_submissions.filter(
            validation_status__icontains='ok'
        ).count()
        
        return context
```

---

### Example 3: AJAX Endpoint with Rate Limiting

```python
from dashboard.decorators import ajax_validator_required, rate_limit
from django.http import JsonResponse
from django.views.decorators.http import require_POST

@require_POST
@ajax_validator_required
@rate_limit(requests_per_minute=20, scope='bulk_assign')
def api_bulk_assign(request):
    """
    Bulk assign submissions to validators
    - AJAX only
    - Validator required
    - Rate limited to prevent abuse
    """
    submission_ids = request.POST.getlist('submission_ids[]')
    validator_id = request.POST.get('validator_id')
    
    if not submission_ids:
        return JsonResponse({
            'error': 'No submissions selected'
        }, status=400)
    
    validator = get_object_or_404(User, id=validator_id, role='validator')
    
    # Bulk assign
    updated = DataSubmission.objects.filter(
        id__in=submission_ids,
        validation_status='---'
    ).update(validated_by=validator)
    
    return JsonResponse({
        'status': 'success',
        'message': f'{updated} submissions assigned to {validator.get_full_name()}',
        'count': updated
    })
```

---

### Example 4: Custom Permission Check in View

```python
from dashboard.permissions import ValidatorAccessMixin, user_can_validate_submission
from django.views.generic import DetailView
from django.contrib import messages
from django.shortcuts import redirect

class ValidationWorkspaceView(ValidatorAccessMixin, DetailView):
    """
    Detailed view for validating a single submission
    """
    model = DataSubmission
    template_name = 'dashboard/validation_workspace.html'
    context_object_name = 'submission'
    
    def dispatch(self, request, *args, **kwargs):
        # Get the submission
        self.object = self.get_object()
        
        # Additional check: Can this user validate this specific submission?
        if not user_can_validate_submission(request.user, self.object):
            messages.error(
                request,
                "You cannot validate this submission. It may already be validated or assigned to another validator."
            )
            return redirect('dashboard:validation_queue')
        
        return super().dispatch(request, *args, **kwargs)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        
        # Add validation history
        context['validation_history'] = self.object.validation_history.all()
        
        # Add related observations
        context['observations'] = self.object.get_related_observations()
        
        return context
```

---

### Example 5: Rate Limiting Public API

```python
from dashboard.decorators import rate_limit_by_ip
from django.http import JsonResponse
from django.views.decorators.http import require_GET

@require_GET
@rate_limit_by_ip(requests_per_minute=10)
def public_species_api(request):
    """
    Public API endpoint for species data
    - No authentication required
    - Rate limited by IP address
    """
    species_type = request.GET.get('type', 'fauna')
    
    if species_type == 'fauna':
        from species.models import FaunaChecklist
        species = FaunaChecklist.objects.filter(
            is_active=True
        ).values('common_name', 'scientific_name')
    else:
        from species.models import FloraChecklist
        species = FloraChecklist.objects.filter(
            is_active=True
        ).values('common_name', 'scientific_name')
    
    return JsonResponse({
        'species': list(species),
        'count': len(species)
    })
```

---

## Summary

PRISM-Matutum's permission system provides:

1. **Flexible Decorators** for function-based views
2. **Powerful Mixins** for class-based views
3. **AJAX-Specific Handlers** with JSON error responses
4. **Rate Limiting** to prevent abuse
5. **Utility Functions** for object-level permissions
6. **Multiple Layers** of security (defense in depth)

### Key Takeaways

- Always use the appropriate permission method for your view type
- Apply rate limiting to all API endpoints
- Check permissions early in the request lifecycle
- Provide clear error messages to users
- Use utility functions for object-level permission checks
- Stack decorators/mixins for multiple layers of security

---

**Related Documentation**:
- [RBAC_DOCUMENTATION.md](RBAC_DOCUMENTATION.md) - User roles and capabilities
- [URL_ROUTING.md](URL_ROUTING.md) - URL structure with permissions

---

*Document created as part of PRISM-Matutum Capstone Project Documentation*  
*For questions or updates, refer to the project repository.*
