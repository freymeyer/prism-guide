# Chapter 3 — The `users` App

> The `users` app handles everything about **who** can use the system and **what** they can do. It defines the custom User model with 5 roles, the account approval workflow, two-factor authentication, organization structure, the Philippine address hierarchy, and protected area boundaries.

---

## Table of Contents

- [3.1 Custom User Model](#31-custom-user-model)
- [3.2 User Profiles](#32-user-profiles)
- [3.3 Organization Structure](#33-organization-structure)
- [3.4 Address & Geography](#34-address--geography)
- [3.5 Protected Areas & Landscapes](#35-protected-areas--landscapes)
- [3.6 Activity Logging & Notifications](#36-activity-logging--notifications)
- [3.7 Authentication Flow](#37-authentication-flow)
- [3.8 Two-Factor Authentication (2FA)](#38-two-factor-authentication-2fa)
- [3.9 Account Approval Workflow](#39-account-approval-workflow)
- [3.10 Views](#310-views)
- [3.11 URL Routing](#311-url-routing)
- [3.12 Management Commands](#312-management-commands)

---

## 3.1 Custom User Model

**File:** `users/models.py`

### Django Concept: Custom User Model

Django ships with a built-in `User` model, but most real projects need to customize it. PRISM-Matutum replaces it entirely:

```python
# In settings.py:
AUTH_USER_MODEL = 'users.User'

# In users/models.py:
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    role = models.CharField(max_length=20, choices=ROLE_CHOICES)
    # ... custom fields
```

By extending `AbstractUser`, we keep all the built-in features (password hashing, session management, `is_staff`, `is_superuser`) **plus** add our own fields.

> **Important:** `AUTH_USER_MODEL` must be set **before the first migration**. Changing it later is extremely painful.

### The 5 Roles

| Role | Value | Description |
|------|-------|-------------|
| **PA Staff** | `pa_staff` | Field personnel who collect data using KoboToolbox |
| **Validator** | `validator` | Reviewers who approve/reject submitted data |
| **DENR Personnel** | `denr` | Department of Environment and Natural Resources staff — view reports and analytics |
| **System Admin** | `system_admin` | Full system access — user management, configuration |
| **Collaborator** | `collaborator` | External researchers with read-only access |

### User Model Fields

#### Identity & Role

| Field | Type | Purpose |
|-------|------|---------|
| *inherited* | — | `username`, `password`, `email`, `first_name`, `last_name`, `is_staff`, `is_superuser`, `is_active`, `date_joined` (from AbstractUser) |
| `role` | CharField(20) | One of the 5 roles above |
| `phone_number` | CharField | Contact number |
| `employee_id` | CharField (unique) | Organization employee ID |
| `position` | CharField | Job title/position |
| `bio` | TextField | Short biography |
| `profile_picture` | ImageField | Profile photo |

#### Organization Links

| Field | Type | Purpose |
|-------|------|---------|
| `agency` | ForeignKey → Agency | Which government agency/organization |
| `department` | ForeignKey → Department | Which department within the agency |
| `date_joined_organization` | DateField | When they joined the organization |

#### Account Management

| Field | Type | Purpose |
|-------|------|---------|
| `account_status` | CharField | `pending`, `approved`, `rejected`, `suspended` |
| `approval_requested_at` | DateTimeField | When the user registered |
| `approved_by` | ForeignKey → User (self) | Which admin approved the account |
| `approved_at` | DateTimeField | When the account was approved |
| `rejection_reason` | TextField | Why the account was rejected |
| `is_field_verified` | BooleanField | Has the user been verified in person? |
| `last_field_activity` | DateTimeField | Last time they performed field work |

#### Two-Factor Authentication

| Field | Type | Purpose |
|-------|------|---------|
| `is_2fa_enabled` | BooleanField | Is 2FA active? |
| `totp_secret` | CharField | TOTP secret key for generating codes |
| `backup_codes` | JSONField | Emergency backup codes |

### Role-Checking Methods

The User model provides convenience methods that are used throughout the application for permission checks:

```python
# These return True/False
user.is_pa_staff()
user.is_validator()
user.is_denr()
user.is_system_admin()
user.is_collaborator()

# Permission methods — used by mixins and decorators
user.can_collect_data()     # PA Staff only
user.can_validate()         # Validators + staff + superusers
user.can_view_reports()     # Validators + DENR + admins + staff
user.can_build_reports()    # Validators + admins
user.can_manage_users()     # System admins + superusers
```

### Account Lifecycle Methods

```python
user.request_approval()               # Set status to 'pending'
user.approve_account(admin_user)       # Set status to 'approved', record who approved
user.reject_account(admin_user, reason) # Set status to 'rejected'
user.suspend_account(admin_user, reason) # Set status to 'suspended'
```

---

## 3.2 User Profiles

### Django Concept: One-to-One Relationships (Profile Pattern)

Instead of cramming every field into the User model, Django projects commonly use **profiles** — separate models linked to User via `OneToOneField`:

```python
class PAStaffProfile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='pa_staff_profile',
        limit_choices_to={'role': 'pa_staff'}  # Only PA Staff can have this profile
    )
    assigned_tablet_id = models.CharField(max_length=100)
    # ... role-specific fields
```

The `limit_choices_to` parameter ensures only users with the correct role can be linked.

### Profile (Generic)

Basic profile information for any user.

| Field | Type | Purpose |
|-------|------|---------|
| `user` | OneToOne → User | |
| `birthdate` | DateField | |
| `phone_number` | CharField | |
| `gender` | CharField | |
| `image` | ImageField | Profile photo |
| `address` | ForeignKey → Address | Where the user lives |

### PAStaffProfile

Extended profile for PA Staff — tracks their field equipment, training, and performance.

| Field | Type | Purpose |
|-------|------|---------|
| `assigned_tablet_id` | CharField | ID of their assigned data collection tablet |
| `assigned_gps_device` | CharField | ID of their GPS device |
| `training_completed` | BooleanField | Have they completed required training? |
| `training_completion_date` | DateField | When they finished training |
| `certifications` | JSONField | List of certifications held |
| `assigned_areas` | JSONField | Areas they're responsible for |
| `specialization` | CharField | `fauna`, `flora`, `both`, `landscape`, `general` |
| `total_submissions` | IntegerField | Lifetime submission count |
| `approved_submissions` | IntegerField | How many were approved |
| `rejected_submissions` | IntegerField | How many were rejected |
| `emergency_contact_name` / `phone` | CharField | Emergency contact info |
| `is_available_for_field_work` | BooleanField | Currently available? |

### ValidatorProfile

Extended profile for Validators — tracks their expertise and validation statistics.

| Field | Type | Purpose |
|-------|------|---------|
| `expertise_areas` | JSONField | List of areas of expertise (e.g., `["birds", "mammals"]`) |
| `total_validations` | IntegerField | Total validations performed |
| `approved_count` | IntegerField | Number approved |
| `rejected_count` | IntegerField | Number rejected |
| `pending_count` | IntegerField | Currently pending |
| `notification_preferences` | JSONField | What notifications they want to receive |

---

## 3.3 Organization Structure

The system models a hierarchical organizational structure:

```
Agency (e.g., DENR Region XI)
  └── Department (e.g., Protected Area Management Division)
        └── Sub-Department (e.g., Biodiversity Monitoring Unit)
```

### Agency

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField (unique) | Organization name |
| `agency_type` | CharField | `government`, `lgu`, `ngo`, `academic`, `private` |
| `acronym` | CharField | Short name (e.g., `"DENR"`) |
| `parent_agency` | ForeignKey → Agency (self) | Parent organization (hierarchical) |
| `contact_person` / `contact_email` / `contact_phone` | CharField | Primary contact |
| `is_active` | BooleanField | Active/inactive |

### Department

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField | Department name |
| `agency` | ForeignKey → Agency | Which organization this belongs to |
| `parent_department` | ForeignKey → Department (self) | Sub-department hierarchy |
| `code` | CharField | Department code |
| `head` | ForeignKey → User | Department head |
| `office_location` | CharField | Physical office location |
| `is_active` | BooleanField | Active/inactive |

---

## 3.4 Address & Geography

### Philippine Geographic Hierarchy

PRISM-Matutum models the full Philippine administrative geography:

```
Country (Philippines)
  └── Region (Region XII - SOCCSKSARGEN)
        └── Province (South Cotabato)
              └── Municipality (Tupi)
                    └── Barangay (Kablon)
```

Each level has its own model:

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Country` | `id_country`, `name` | Top level |
| `Region` | `id_region`, `country` (FK), `name` | Administrative region |
| `Province` | `id_province`, `region` (FK), `name` | Province |
| `Municipality` | `id_municipality`, `province` (FK), `name` | City/Municipality |
| `Barangay` | `id_barangay`, `municipality` (FK), `name` | Smallest administrative unit |

### Address

Composite model that links all geographic levels together:

| Field | Type | Purpose |
|-------|------|---------|
| `country` | ForeignKey → Country | |
| `region` | ForeignKey → Region | |
| `province` | ForeignKey → Province | |
| `municipality` | ForeignKey → Municipality | |
| `barangay` | ForeignKey → Barangay | |

This is used by `Profile` (user addresses), `ProtectedArea`, and `FGDSession`.

---

## 3.5 Protected Areas & Landscapes

### ProtectedArea

Defines the boundary of a Protected Area using PostGIS:

| Field | Type | Purpose |
|-------|------|---------|
| `protecetedAreaName` | CharField | Protected area name (e.g., "Mt. Matutum Protected Landscape") |
| `address` | ForeignKey → Address | Location reference |
| `boundary` | MultiPolygonField (SRID 4326) | The geographic boundary polygon |
| `area_hectares` | FloatField | Total area in hectares |

**Key method:** `contains_point(lat, lng)` — Uses PostGIS to check if a GPS coordinate falls inside this protected area. This is used to auto-assign observations to protected areas.

### Landscape

Images associated with protected areas (for display in reports and dashboards):

| Field | Type | Purpose |
|-------|------|---------|
| `protected_area` | ForeignKey → ProtectedArea | Which PA this image belongs to |
| `image_url` | URLField | Link to the landscape image |
| `remarks` | TextField | Description of the image |

---

## 3.6 Activity Logging & Notifications

### UserActivityLog

Audit trail for user actions:

| Field | Type | Purpose |
|-------|------|---------|
| `user` | ForeignKey → User | Who performed the action |
| `activity_type` | CharField | `login`, `logout`, `data_submission`, `validation`, `report_generation`, `user_management`, `settings_change` |
| `description` | TextField | Human-readable description |
| `ip_address` | GenericIPAddressField | Client IP |
| `user_agent` | CharField | Browser/device info |
| `metadata` | JSONField | Additional context data |

### UserNotification

In-app notification system:

| Field | Type | Purpose |
|-------|------|---------|
| `user` | ForeignKey → User | Recipient |
| `notification_type` | CharField | 11 types (see below) |
| `title` | CharField | Notification title |
| `message` | TextField | Notification body |
| `is_read` | BooleanField | Has the user seen it? |
| `read_at` | DateTimeField | When it was read |
| `related_submission_id` | IntegerField | Optional link to a submission |
| `related_observation_id` | IntegerField | Optional link to an observation |
| `related_event_id` | IntegerField | Optional link to an event |
| `expires_at` | DateTimeField | Auto-expire date |

**Notification Types:**
1. `validation_update` — Your submission was approved/rejected
2. `assignment` — You've been assigned to a survey
3. `system` — System-wide announcement
4. `report_ready` — Your report is ready for download
5. `survey_reminder` — Upcoming survey reminder
6. `on_hold` — Your submission is on hold (needs corrections)
7. `event_created` — New event posted
8. `event_updated` — Event details changed
9. `event_cancelled` — Event was cancelled
10. `event_attendee_added` — You've been added as an attendee
11. `event_reminder` — Upcoming event reminder

---

## 3.7 Authentication Flow

### Django Concept: Authentication Backends

Django uses **authentication backends** to verify credentials. PRISM-Matutum uses the standard `ModelBackend`:

```python
# settings.py
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
]
```

### The Login Flow

```
User visits /accounts/login/
        │
        ▼
CustomLoginView renders login form
        │
        ▼
User submits username + password
        │
        ▼
Django checks credentials via ModelBackend
        │
        ├── Invalid → Error message
        │
        ├── Valid + 2FA disabled → session created → redirect to role-based dashboard
        │
        └── Valid + 2FA enabled → redirect to /accounts/2fa/verify/
                                        │
                                        ▼
                                  User enters TOTP code
                                        │
                                        ├── Correct → session created → redirect
                                        └── Incorrect → retry
```

### Role-Based Redirect

After successful login, the root URL `home_redirect` in `prism/urls.py` routes users based on their role:

| Role | Redirected To |
|------|-------------|
| System Admin / Staff / Superuser | `/dashboard/admin-dashboard/` |
| PA Staff | `/dashboard/pa-staff-home/` |
| Validator | `/dashboard/queue/` (validation queue) |
| DENR | `/dashboard/reports/archive/` (report archive) |
| Fallback | `/dashboard/` |

---

## 3.8 Two-Factor Authentication (2FA)

### How It Works

PRISM-Matutum uses **TOTP** (Time-based One-Time Password) via the `pyotp` library:

1. **Setup:** User visits `/accounts/2fa/setup/` → system generates a secret key → displays QR code → user scans with an authenticator app (Google Authenticator, Authy, etc.)
2. **Login:** After password verification, user is asked for the 6-digit code from their authenticator app
3. **Backup codes:** During setup, the system generates one-time backup codes stored as JSON in the `backup_codes` field — for emergency access if the phone is lost

### Related Views

| View | URL | Purpose |
|------|-----|---------|
| `two_factor_setup_view` | `/accounts/2fa/setup/` | Generate secret, show QR code |
| `two_factor_verify_view` | `/accounts/2fa/verify/` | Verify TOTP code during login |
| `two_factor_backup_codes_view` | `/accounts/2fa/backup-codes/` | View/regenerate backup codes |
| `two_factor_disable_view` | `/accounts/2fa/disable/` | Turn off 2FA |
| `two_factor_status_view` | `/accounts/2fa/status/` | Check 2FA status (API) |

---

## 3.9 Account Approval Workflow

New users don't get immediate access. There's a manual approval process:

```
User registers at /accounts/register/
        │
        ▼
User object created with account_status = 'pending'
        │
        ▼
User sees "Registration Pending" page
        │
        ▼
System Admin sees new user in /dashboard/admin/users/ (or /accounts/management/pending/)
        │
        ├── Approve → account_status = 'approved' → user can now log in
        │
        ├── Reject → account_status = 'rejected' + reason → user notified
        │
        └── (Later) Suspend → account_status = 'suspended' → access revoked
```

The `DashboardAccessMiddleware` enforces this — only users with `account_status = 'approved'` can access `/dashboard/` URLs.

---

## 3.10 Views

**File:** `users/views.py`

| View | Type | URL | Purpose |
|------|------|-----|---------|
| `CustomLoginView` | CBV (LoginView) | `/accounts/login/` | Login with custom template |
| `user_registration_view` | FBV | `/accounts/register/` | Registration form |
| `registration_pending_view` | FBV | `/accounts/registration/pending/` | "Your account is pending approval" page |
| `two_factor_verify_view` | FBV | `/accounts/2fa/verify/` | Enter TOTP code |
| `two_factor_setup_view` | FBV | `/accounts/2fa/setup/` | QR code setup |
| `two_factor_backup_codes_view` | FBV | `/accounts/2fa/backup-codes/` | View/regenerate backup codes |
| `two_factor_disable_view` | FBV | `/accounts/2fa/disable/` | Disable 2FA |
| `two_factor_status_view` | FBV | `/accounts/2fa/status/` | Check 2FA status |
| `CustomPasswordResetView` | CBV | `/accounts/password-reset/` | Password reset request |
| `CustomPasswordResetConfirmView` | CBV | `/accounts/password-reset/<uidb64>/<token>/` | Set new password |
| `CustomPasswordChangeView` | CBV | `/accounts/password-change/` | Change password (logged in) |
| `session_status_view` | FBV | `/accounts/session/status/` | Check if session is valid (AJAX) |
| `pending_users_list_view` | FBV | `/accounts/management/pending/` | Admin: view pending users |
| `user_approval_view` | FBV | `/accounts/management/approve/<user_id>/` | Admin: approve a user |
| `user_management_dashboard_view` | FBV | `/accounts/management/dashboard/` | Admin: user management hub |

---

## 3.11 URL Routing

**File:** `users/urls.py` — `app_name = 'users'`

| Group | URLs |
|-------|------|
| **Auth** | `login/`, `logout/` |
| **Registration** | `register/`, `registration/pending/` |
| **User Management** | `management/dashboard/`, `management/pending/`, `management/approve/<user_id>/` |
| **2FA** | `2fa/verify/`, `2fa/setup/`, `2fa/backup-codes/`, `2fa/disable/`, `2fa/status/` |
| **Password Reset** | `password-reset/`, `password-reset/done/`, `password-reset/<uidb64>/<token>/`, `password-reset/complete/` |
| **Password Change** | `password-change/`, `password-change/done/` |
| **Session** | `session/status/` |

---

## 3.12 Management Commands

**Directory:** `users/management/commands/`

| Command | Purpose |
|---------|---------|
| `create_test_users` | Creates one test user for each role (pa_staff, validator, denr, system_admin, collaborator) with predefined credentials. **Useful for development and testing.** |
| `populate_usersfromjson` | Bulk import users from a JSON file — useful for initial data migration |

---

**Next:** [Chapter 4 — The `species` App →](04-species-app.md)

**Previous:** [← Chapter 2: The `dashboard` App](02-dashboard-app.md)
