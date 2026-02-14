# System Administrator Workflow Documentation

## 📋 Overview

This document provides comprehensive documentation for System Administrator users in the PRISM-Matutum system. System Admins have full access to all system functions and are responsible for user management, system configuration, data integrity, and overall system administration.

**Role Code:** `system_admin` (or `is_superuser=True`)  
**Primary Function:** System management, user administration, and configuration

---

## 🎯 Role Definition & Capabilities

### What System Admins Can Do:
- ✅ **Full Access** - Access all system features and functions
- ✅ Approve/reject user registrations
- ✅ Manage user roles and permissions
- ✅ Create and manage agencies and departments
- ✅ Manage geographic data (regions, provinces, municipalities, barangays)
- ✅ Configure choice lists and KoboToolbox integration
- ✅ Access Django admin panel
- ✅ View all submissions and observations
- ✅ Perform validation tasks
- ✅ Generate system-wide reports
- ✅ Monitor system health and performance
- ✅ Configure system settings

### System Admin Responsibilities:
- 🔧 System configuration and maintenance
- 👥 User onboarding and role assignment
- 🏛️ Organizational structure management
- 🗺️ Geographic data accuracy
- 📋 Choice list synchronization with KoboToolbox
- 📊 System performance monitoring
- 🔒 Security and access control
- 📈 Data quality oversight

---

## 🏠 1. Admin Dashboard

**URL:** `/dashboard/admin/`  
**View:** `AdminDashboardView` ([views_admin.py](dashboard/views_admin.py#L54-L137))  
**Template:** [admin_dashboard.html](dashboard/templates/dashboard/admin/admin_dashboard.html)

### 1.1 Dashboard Overview

The Admin Dashboard provides a comprehensive view of system health and activity.

#### Quick Stats Cards

| Stat Card | Data Displayed | Purpose |
|-----------|----------------|---------|
| **Total Users** | Total count + Active/Pending breakdown | Monitor user base growth and pending approvals |
| **Total Submissions** | Total + Validated/Pending breakdown | Track data collection activity |
| **Total Observations** | Combined wildlife + resource use counts | Monitor biodiversity data accumulation |
| **Species Catalogued** | Total species + Flora/Fauna breakdown | Track taxonomic database completeness |

**Code Reference:**
```python
context['user_stats'] = {
    'total_users': User.objects.count(),
    'active_users': User.objects.filter(is_active=True).count(),
    'pending_approval': User.objects.filter(account_status='pending').count(),
    'pa_staff': User.objects.filter(role='pa_staff', is_active=True).count(),
    'validators': User.objects.filter(role='validator', is_active=True).count(),
    'denr_staff': User.objects.filter(role='denr', is_active=True).count(),
}
```

#### Additional Statistics

**Geographic Data Stats:**
- Regions, Provinces, Municipalities, Barangays counts
- Coverage metrics

**Organization Stats:**
- Total agencies (active/inactive)
- Total departments
- Organizational hierarchy health

**Report Stats:**
- Total reports generated
- Pending reports
- Completed reports

### 1.2 Recent Activity Panels

#### Pending User Approvals (Top 5)
- Shows users awaiting approval
- Quick actions: Approve/Reject
- User details: Name, Email, Agency, Role requested

#### Recent Submissions (Top 10)
- Latest data submissions
- Validation status
- Quick links to validation workspace

#### System Health Indicators
- Database connection status
- KoboToolbox API status
- Celery task queue status
- Redis cache status

### 1.3 Quick Access Links

- **Django Admin** - Full Django admin panel
- **User Management** - Approve users, manage roles
- **Organization Hub** - Agencies, departments, addresses
- **Choice Management** - KoboToolbox choice lists
- **Validation Queue** - Review submissions
- **Reports** - Generate and view reports

---

## 👥 2. User & Organization Management Hub

**URL:** `/dashboard/admin/user-organization-hub/`  
**View:** `UserOrganizationHubView` ([views_admin.py](dashboard/views_admin.py#L143-L306))  
**Template:** [user_organization_hub.html](dashboard/templates/dashboard/admin/user_organization_hub.html)

### 2.1 Hub Overview

Unified interface for managing:
- **Users** - Registration approvals, role assignment
- **Agencies** - Government, LGU, NGO, academic organizations
- **Departments** - Organizational subdivisions
- **Addresses** - Geographic hierarchy (Region/Province/Municipality/Barangay)

#### Top-Level Statistics

Four stat cards:
- **Users** - Total count + pending badge
- **Agencies** - Total count
- **Departments** - Total count
- **Locations** - Total barangays (lowest level)

### 2.2 USERS Section

#### User Management Tabs

**A. Pending Users Tab**

Shows users with `account_status='pending'`

**Displayed Information:**
- Username
- Full Name
- Email
- Agency (if provided)
- Requested Role
- Registration Date

**Actions:**
```html
<form method="post" action="{% url 'dashboard:user_approve' user.id %}">
    <button type="submit" class="btn btn-success">Approve</button>
</form>

<form method="post" action="{% url 'dashboard:user_reject' user.id %}">
    <textarea name="rejection_reason" required></textarea>
    <button type="submit" class="btn btn-danger">Reject</button>
</form>
```

**Approval Process:**
1. Review user details
2. Verify agency affiliation
3. Confirm role request is appropriate
4. Click "Approve"
5. System sets `account_status='approved'`
6. Email notification sent to user
7. User can now login

**Rejection Process:**
1. Click "Reject"
2. **Must provide rejection reason**
3. System sets `account_status='rejected'`
4. Email sent with rejection reason
5. User can re-register with corrections

---

**B. Active Users Tab**

Shows users with `is_active=True` and `account_status='approved'`

**Displayed Information:**
- Username, Full Name, Email
- Current Role (badge with color)
- Agency/Department
- Last Login
- Status indicators (field verified, email verified)

**Actions:**
- **Edit User** - Modify role, agency, permissions
- **Deactivate** - Set `is_active=False`
- **Reset Password** - Generate password reset link
- **View Activity** - See user's activity log

---

**C. Staff Users Tab**

Shows users with roles: `pa_staff`, `validator`, `denr`, `system_admin`

**Purpose:** Quick access to operational staff
**Sorting:** By role, then username

---

**D. All Users Tab**

Complete user list with all statuses

**Features:**
- Search by username, email, name
- Filter by role, agency, status
- Bulk actions (export CSV, send notifications)

### 2.3 AGENCIES Section

**Agency Types:**
```python
AGENCY_TYPES = [
    ('government', 'Government Agency'),
    ('lgu', 'Local Government Unit'),
    ('ngo', 'Non-Government Organization'),
    ('academic', 'Academic Institution'),
    ('private', 'Private Organization'),
]
```

**Displayed Information:**
- Agency Name
- Acronym
- Type (colored badge)
- Parent Agency (if hierarchical)
- Department Count
- User Count
- Active status

**Actions:**
- **Add Agency** - Create new organization
- **Edit Agency** - Modify details, set parent
- **Deactivate** - Mark as inactive (soft delete)
- **View Departments** - See sub-departments
- **View Users** - See assigned users

**Agency Form Fields:**
- Name (required)
- Acronym (optional)
- Description
- Agency Type (dropdown)
- Parent Agency (optional, for hierarchies)
- Contact Information (address, phone, email, website)
- Active status

**Hierarchical Example:**
```
DENR (Government)
├── DENR Region XI (Government)
│   ├── DENR PENRO South Cotabato (Government)
│   └── DENR CENRO Koronadal (Government)
└── Protected Areas Management Bureau (Government)
```

### 2.4 DEPARTMENTS Section

**Purpose:** Sub-organizational units within agencies

**Displayed Information:**
- Department Name
- Code (short identifier)
- Parent Agency
- Department Head (user)
- Parent Department (if nested)
- User Count

**Actions:**
- **Add Department** - Create within agency
- **Edit Department** - Modify details, assign head
- **Delete** - Remove department (if no users assigned)

**Department Form Fields:**
- Name (required)
- Code (required, alphanumeric)
- Agency (required, dropdown)
- Parent Department (optional, for sub-departments)
- Department Head (optional, user dropdown)
- Description
- Active status

**Example Structure:**
```
DENR Region XI
├── Administrative Division
├── Technical Services Division
│   ├── Biodiversity Section
│   ├── Protected Areas Section
│   └── Wildlife Section
└── Legal Division
```

### 2.5 ADDRESSES Section

**Purpose:** Manage Philippine geographic hierarchy

**Four-Level Hierarchy:**
1. **Region** (17 regions)
2. **Province** (81 provinces)
3. **Municipality/City** (~1,634 municipalities/cities)
4. **Barangay** (~42,000 barangays)

#### A. Regions Tab

**Displayed:**
- Region Name
- PSGCCode (Philippine Standard Geographic Code)
- Country (Philippines)
- Province Count

**Actions:**
- Add Region
- Edit Region
- View Provinces

#### B. Provinces Tab

**Displayed:**
- Province Name
- Region (parent)
- PSGCCode
- Municipality Count

**Actions:**
- Add Province (select region)
- Edit Province
- View Municipalities

#### C. Municipalities Tab

**Displayed:**
- Municipality Name
- Province → Region
- PSGCCode
- Barangay Count

**Actions:**
- Add Municipality (select province)
- Edit Municipality
- View Barangays

#### D. Barangays Tab

**Displayed:**
- Barangay Name
- Municipality → Province → Region
- PSGCCode

**Actions:**
- Add Barangay (select municipality)
- Edit Barangay
- Delete Barangay

**Search & Filter:**
- Search by name (searches across hierarchy)
- Filter by parent (e.g., all municipalities in Province X)

**Pagination:** 15 items per page per section

---

## 📋 3. Choice List & KoboToolbox Integration

**URL:** `/dashboard/admin/choice-form-hub/`  
**View:** `ChoiceFormHubView`  
**Template:** [choice_form_hub.html](dashboard/templates/dashboard/admin/choice_form_hub.html)

### 3.1 What are Choice Lists?

Choice Lists are predefined option sets used in KoboToolbox forms (dropdown menus, radio buttons, checkboxes).

**Example:**
```
Choice List: "Wildlife Species"
├── Macaca fascicularis (Long-tailed Macaque)
├── Macaca nemestrina (Pig-tailed Macaque)
├── Paradoxurus hermaphroditus (Common Palm Civet)
└── Viverra tangalunga (Malay Civet)
```

### 3.2 Choice List Management

#### Viewing Choice Lists

**Displayed Information:**
- List Name
- Description
- Number of Choices
- Last Updated
- Used in Forms (count)

**Actions:**
- **Create New List**
- **Edit List** (name, description)
- **View Choices** - See all options in list
- **Delete List** (if not in use)

#### Managing Choices Within List

**URL:** `/dashboard/admin/choice-list/<int:pk>/`

**Displayed:**
- Choice Name
- Choice Code (value sent to database)
- Order (display order in form)
- Active status

**Actions:**
- **Add Choice** - Single entry
- **Bulk Add Choices** - Import multiple from CSV/Excel
- **Edit Choice** - Modify name, code, order
- **Deactivate Choice** - Hide from forms without deleting
- **Reorder Choices** - Drag-and-drop or set order number

**Bulk Add Format:**
```csv
name,code,order
Long-tailed Macaque,macaca_fascicularis,1
Pig-tailed Macaque,macaca_nemestrina,2
Common Palm Civet,paradoxurus_hermaphroditus,3
```

### 3.3 Form Choice Mappings

**Purpose:** Link KoboToolbox form questions to Django choice lists

**Mapping Table:**
| Kobo Form | Question Path | Choice List | Sync Direction |
|-----------|---------------|-------------|----------------|
| Wildlife Survey v3 | `record_wildlife/species` | Wildlife Species | Bi-directional |
| Resource Use | `record_resourceuse/resourceuse_type` | Resource Use Types | Django → Kobo |

**Fields:**
- **Kobo Form** (asset_uid)
- **Question Path** (XPath in form)
- **Choice List** (Django ChoiceList)
- **Sync Direction:**
  - **Django → Kobo** - Django is source of truth
  - **Kobo → Django** - Kobo is source of truth
  - **Bi-directional** - Sync both ways

**Actions:**
- **Create Mapping** - Link question to choice list
- **Edit Mapping** - Change sync direction
- **Sync Now** - Manually trigger synchronization
- **Delete Mapping** - Unlink (does not delete data)

### 3.4 KoboToolbox Form Browser

**URL:** `/dashboard/admin/kobo-forms/`

**Purpose:** Browse and inspect forms from KoboToolbox account

**Displayed:**
- Form Name
- Asset UID (unique identifier)
- Version
- Deployment Status
- Submission Count
- Last Modified

**Actions:**
- **View Form Structure** - Inspect questions and structure
- **Create Choice Mapping** - Link to Django choice list
- **Sync Submissions** - Manually trigger data import
- **Preview Form** - Open in KoboToolbox

**API Integration:**
```python
from api.form_services import list_assets, health_check

# Check KoboToolbox connection
status = health_check()
# Returns: {'status': 'healthy', 'assets_count': 15}

# List all forms
forms = list_assets()
# Returns: [{
#     'uid': 'aXyzGKq2yx...',
#     'name': 'Wildlife Survey v3',
#     'version_id': 'v123',
#     'deployment_status': 'deployed',
#     'submission_count': 450
# }, ...]
```

### 3.5 Choice Synchronization

**Manual Sync:**
```bash
docker-compose exec web python manage.py sync_kobotoolbox_choices
```

**Automated Sync:**
- Celery task runs every 6 hours
- Syncs all mappings with "Django → Kobo" or "Bi-directional"

**Sync Process:**
1. Fetch choice list from Django
2. Transform to KoboToolbox format
3. Update form via KoboToolbox API
4. Log sync result
5. Send notification if errors

**Sync Log:**
```
[2026-01-06 14:30:00] Choice sync started for "Wildlife Species"
[2026-01-06 14:30:05] Successfully synced 45 choices to form "Wildlife Survey v3"
[2026-01-06 14:30:05] Sync completed successfully
```

---

## 🗺️ 4. Geographic Data Management

### 4.1 Importance of Geographic Accuracy

Accurate geographic data is critical for:
- **Spatial Analysis** - Mapping observations
- **Protected Area Coverage** - Determining if observations are within PA boundaries
- **Reporting** - Generating regional/provincial reports
- **Data Quality** - Validating GPS coordinates

### 4.2 Maintaining Address Hierarchy

**Best Practices:**
1. **Use Official PSGCCodes** - Philippine Standard Geographic Codes
2. **Verify Names** - Match official PSGC names
3. **Complete Hierarchy** - Ensure all parent relationships are set
4. **No Orphaned Records** - Every address must have a parent (except regions)

**Common Issues:**
- Typos in names → Fix immediately
- Missing parent relationships → Set correct parent
- Duplicate entries → Merge or delete duplicates
- Outdated names (due to LGU renaming) → Update to current official name

### 4.3 Bulk Import

**CSV Format for Barangays:**
```csv
name,psgcCode,municipality_id
Barangay Poblacion,063001001,1234
Barangay Upper Suyan,063001002,1234
Barangay Roxas,063001003,1234
```

**Import Process:**
1. Prepare CSV with correct format
2. Upload via admin interface
3. System validates:
   - Required fields present
   - Parent exists
   - No duplicates
4. Import executes
5. Review import log for errors

---

## 🔐 5. User Approval Workflow

### 5.1 Registration Process (User Side)

1. User visits registration page
2. Fills out form:
   - Username, Email, Password
   - Full Name
   - Agency (optional)
   - Role request (PA Staff, Validator, DENR, Collaborator)
   - Justification for role
3. Submits form
4. System creates user with `account_status='pending'`
5. Email sent to admins: "New user registration pending approval"

### 5.2 Approval Process (Admin Side)

#### Step 1: Review User Details

**Check:**
- ✅ Real name (not fake/test)
- ✅ Valid agency email (if government role)
- ✅ Appropriate role request
- ✅ Clear justification

**Red Flags:**
- ❌ Suspicious email (temp email services)
- ❌ Role request doesn't match agency
- ❌ No justification provided
- ❌ Duplicate account attempt

#### Step 2: Verification (if needed)

**For sensitive roles (Validator, DENR):**
- Contact agency to verify employment
- Check if user has prior training
- Verify need for requested role

#### Step 3: Approve or Reject

**Approve:**
```python
def user_approve(request, user_id):
    user = User.objects.get(pk=user_id)
    user.account_status = 'approved'
    user.is_active = True
    user.save()
    
    # Send approval email
    send_mail(
        subject='Account Approved - PRISM-Matutum',
        message=f'Your account has been approved. You can now login.',
        from_email='noreply@prism-matutum.org',
        recipient_list=[user.email]
    )
    
    # Log activity
    UserActivityLog.objects.create(
        user=request.user,
        activity_type='user_approval',
        description=f'Approved user: {user.username}'
    )
    
    messages.success(request, f'User {user.username} approved.')
    return redirect('dashboard:admin_user_management')
```

**Reject:**
```python
def user_reject(request, user_id):
    user = User.objects.get(pk=user_id)
    rejection_reason = request.POST.get('rejection_reason')
    
    user.account_status = 'rejected'
    user.save()
    
    # Send rejection email with reason
    send_mail(
        subject='Account Registration - PRISM-Matutum',
        message=f'Your registration was not approved.\n\nReason: {rejection_reason}\n\nYou may re-register with corrections.',
        from_email='noreply@prism-matutum.org',
        recipient_list=[user.email]
    )
    
    messages.info(request, f'User {user.username} rejected.')
    return redirect('dashboard:admin_user_management')
```

### 5.3 Post-Approval Setup

**For PA Staff:**
- Assign to appropriate agency/department
- Set field verification status (if applicable)
- Provide training materials link

**For Validators:**
- Create ValidatorProfile
- Set validation permissions
- Assign to validation queue
- Provide validator training

**For DENR:**
- Assign agency (DENR office)
- Set report access levels
- Configure dashboard widgets

---

## 🧪 6. Testing Checklist for System Admins

### 6.1 Dashboard Access

- [ ] **Test:** Admin dashboard loads correctly
- [ ] **Test:** All statistic cards show accurate counts
- [ ] **Test:** Recent submissions display
- [ ] **Test:** Pending users list shows correct users
- [ ] **Test:** Quick links navigate correctly
- [ ] **Test:** Django admin link works

### 6.2 User Management

- [ ] **Test:** Can view pending users
- [ ] **Test:** Can approve user (email sent, status updated)
- [ ] **Test:** Can reject user with reason (email sent)
- [ ] **Test:** Can edit user role
- [ ] **Test:** Can deactivate user
- [ ] **Test:** Can assign user to agency
- [ ] **Test:** Search/filter users works
- [ ] **Test:** Export users to CSV works

### 6.3 Agency & Department Management

- [ ] **Test:** Can create new agency
- [ ] **Test:** Can set agency type and parent
- [ ] **Test:** Can edit agency details
- [ ] **Test:** Can deactivate agency
- [ ] **Test:** Can create department under agency
- [ ] **Test:** Can assign department head
- [ ] **Test:** Can create nested departments
- [ ] **Test:** Department user count is accurate
- [ ] **Test:** Cannot delete agency with users

### 6.4 Address Management

- [ ] **Test:** Can create new region
- [ ] **Test:** Can create province under region
- [ ] **Test:** Can create municipality under province
- [ ] **Test:** Can create barangay under municipality
- [ ] **Test:** PSGCCode validation works
- [ ] **Test:** Parent-child relationships are correct
- [ ] **Test:** Search across hierarchy works
- [ ] **Test:** Counts (provinces per region, etc.) are accurate
- [ ] **Test:** Bulk import barangays from CSV works

### 6.5 Choice List Management

- [ ] **Test:** Can create new choice list
- [ ] **Test:** Can add choices to list
- [ ] **Test:** Can bulk add choices from CSV
- [ ] **Test:** Can reorder choices
- [ ] **Test:** Can deactivate choice
- [ ] **Test:** Can edit choice name/code
- [ ] **Test:** Cannot delete list in use
- [ ] **Test:** Choice count displays correctly

### 6.6 KoboToolbox Integration

- [ ] **Test:** Can browse Kobo forms
- [ ] **Test:** Form details display correctly
- [ ] **Test:** Can create form-choice mapping
- [ ] **Test:** Can trigger manual sync
- [ ] **Test:** Sync updates Kobo form correctly
- [ ] **Test:** Sync errors are logged
- [ ] **Test:** Health check shows connection status

### 6.7 Performance

- [ ] **Test:** Dashboard loads in < 2 seconds
- [ ] **Test:** User hub with 1000+ users loads efficiently
- [ ] **Test:** Agency list paginates correctly
- [ ] **Test:** Address search is fast
- [ ] **Test:** Choice sync completes without timeout

---

## 🛠️ 7. Troubleshooting

### Common Issues

#### Issue: User approval not sending email

**Cause:** Email configuration not set or SMTP server issue

**Solution:**
```bash
# Check email settings in settings.py
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'your-email@gmail.com'
EMAIL_HOST_PASSWORD = 'your-app-password'

# Test email
docker-compose exec web python manage.py shell
>>> from django.core.mail import send_mail
>>> send_mail('Test', 'Test message', 'from@example.com', ['to@example.com'])
```

#### Issue: KoboToolbox sync failing

**Causes:**
- Invalid API token
- Network connectivity issue
- Form deleted in Kobo
- Rate limiting

**Solution:**
```bash
# Check Kobo connection
docker-compose exec web python manage.py shell
>>> from api.form_services import health_check
>>> health_check()

# Check logs
docker-compose logs -f web | grep "kobo"

# Re-sync manually
docker-compose exec web python manage.py sync_kobotoolbox_choices --form-uid=aXyzGKq2yx
```

#### Issue: Cannot delete agency (foreign key constraint)

**Cause:** Users or departments assigned to agency

**Solution:**
1. Reassign users to different agency
2. Delete or reassign departments
3. Then delete agency

OR

4. Deactivate agency instead of deleting (recommended)

#### Issue: Address hierarchy broken (orphaned records)

**Cause:** Parent record deleted or missing

**Solution:**
```python
# Find orphaned provinces (no region)
from users.models import Province
Province.objects.filter(region__isnull=True)

# Fix: Set correct region
orphaned = Province.objects.get(pk=123)
orphaned.region = Region.objects.get(name='Region XI')
orphaned.save()
```

---

## 📚 8. Best Practices

### 8.1 User Approval Guidelines

**Fast-Track Approval:**
- Valid government email (@denr.gov.ph, @lgu.gov.ph)
- Verification from existing admin
- Clear justification

**Additional Verification:**
- Personal email + government role
- New organization
- Validator or admin role request

**Reject Immediately:**
- Fake/test accounts
- No justification
- Inappropriate role request
- Duplicate account

### 8.2 Data Management

**Regular Maintenance:**
- Weekly: Review pending users
- Monthly: Audit user roles
- Quarterly: Review agency list (deactivate defunct organizations)
- Annually: Update address data (LGU changes)

**Data Integrity:**
- Always set parent relationships
- Use official PSGCCodes
- Verify spelling before saving
- Test choice syncs in staging first

### 8.3 Security

**Access Control:**
- Limit system admin role to trusted personnel
- Review admin activity logs monthly
- Rotate passwords regularly
- Use strong passwords (12+ characters)

**Monitoring:**
- Check failed login attempts
- Review unusual user activity
- Monitor submission validation rates
- Track API usage

---

## 🔗 9. Related Documentation

- **RBAC:** [RBAC_DOCUMENTATION.md](../architecture/RBAC_DOCUMENTATION.md)
- **Permission System:** [PERMISSION_SYSTEM.md](../architecture/PERMISSION_SYSTEM.md)
- **Choice System:** [CHOICE_SYSTEM_README.md](../../docs/CHOICE_SYSTEM_README.md)
- **KoboToolbox Integration:** [API.md](../../docs/API.md)
- **Testing:** [SYSTEM_ADMIN_TEST_CHECKLIST.md](../../dashboard/docs/SYSTEM_ADMIN_TEST_CHECKLIST.md)

---

## 📞 Support

**For System Admins:**
- Refer to Django admin documentation
- Check system logs: `docker-compose logs -f web`
- Celery logs: `docker-compose logs -f celery`
- Database backups: Review backup procedures

**For Developers:**
- Reference: [views_admin.py](dashboard/views_admin.py)
- Tests: [dashboard/tests/test_admin_functionality.py](dashboard/tests/)
- Models: [users/models.py](users/models.py), [api/models/choices.py](api/models/choices.py)

---

*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Capstone Documentation Project*
