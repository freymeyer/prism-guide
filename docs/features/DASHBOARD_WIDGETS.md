# Dashboard & Widget System Documentation

## 📋 Overview

The PRISM-Matutum Dashboard & Widget System provides customizable, role-aware dashboards that allow users to create personalized views of biodiversity data through a flexible widget-based interface.

### Key Features

- **Customizable Dashboards**: Users can create multiple dashboards
- **Drag-and-Drop Widget Layout**: Grid-based positioning system (12-column layout)
- **Role-Specific Defaults**: Auto-generated dashboards tailored to user roles
- **Widget Library**: 14+ widget types for data visualization
- **Dashboard Templates**: Pre-configured dashboard layouts for common use cases
- **Real-Time Data**: Auto-refresh capabilities for dynamic updates
- **Dashboard Sharing**: Share custom dashboards with other users

---

## 🏗️ Architecture

### Core Components

```
api/models/dashboards.py
├── Dashboard              # Container for widgets
├── DashboardWidget        # Individual widget instances
├── DashboardTemplate      # Reusable dashboard layouts
└── WidgetDataCache        # Performance optimization

dashboard/
├── views_dashboard.py     # Dashboard CRUD operations
├── views_widgets.py       # Widget data providers
└── templates/dashboard/
    ├── custom_dashboard.html
    ├── template_gallery.html
    └── widgets/           # Individual widget templates
```

### Data Flow

```
User Request → CustomDashboardView
    ↓
Fetch Dashboard → Get Widgets → For Each Widget:
    ↓
views_widgets.get_widget_data()
    ↓
Widget-specific data provider (e.g., get_counter_data)
    ↓
Query database → Process data → Return JSON
    ↓
Render widget template → Display in grid
```

---

## 📦 Models

### 1. Dashboard Model

**Location**: `api/models/dashboards.py`

```python
class Dashboard(models.Model):
    user = ForeignKey(User)              # Dashboard owner
    name = CharField(max_length=200)     # Dashboard name
    description = TextField()             # Optional description
    is_default = BooleanField()          # User's default dashboard
    is_shared = BooleanField()           # Shared with others
    shared_with = ManyToManyField(User)  # Specific users with access
    layout_config = JSONField()          # Grid settings
    created_at = DateTimeField()
    updated_at = DateTimeField()
```

**Key Methods**:
- `get_widgets()` - Returns ordered list of widgets
- `duplicate(new_name)` - Creates copy with all widgets
- `save()` - Ensures only one default dashboard per user

### 2. DashboardWidget Model

**Location**: `api/models/dashboards.py`

```python
class DashboardWidget(models.Model):
    dashboard = ForeignKey(Dashboard)
    widget_type = CharField(choices=WIDGET_TYPE_CHOICES)
    title = CharField(max_length=200)
    configuration = JSONField()       # Widget-specific settings
    
    # Grid Positioning (12-column grid)
    row_position = IntegerField()     # 0-indexed row
    column_position = IntegerField()  # 0-11 (12-column grid)
    width = IntegerField()            # 1-12 columns
    height = IntegerField()           # Grid rows
    
    is_visible = BooleanField()       # Show/hide toggle
    refresh_interval = IntegerField() # Auto-refresh (seconds)
    
    created_at = DateTimeField()
    updated_at = DateTimeField()
```

**Widget Types**:
- `counter` - Data Counter / KPI
- `pie_chart` - Pie Chart
- `bar_chart` - Bar Chart
- `line_chart` - Line Chart
- `data_table` - Data Table
- `map` - Spatial Map
- `image` - Image Display
- `text` - Text / Notes
- `recent_activity` - Recent Activity Feed
- `assigned_submissions` - Assigned Submissions
- `validation_stats` - Validation Statistics
- `species_summary` - Species Summary
- `disturbance_alerts` - Disturbance Alerts
- `schedule_calendar` - Schedule Calendar

**Key Methods**:
- `get_data()` - Fetches widget data
- `move_to(row, column)` - Repositions widget
- `resize(width, height)` - Resizes widget

### 3. DashboardTemplate Model

**Location**: `api/models/dashboards.py`

```python
class DashboardTemplate(models.Model):
    name = CharField(max_length=200)
    description = TextField()
    category = CharField()              # e.g., 'Biodiversity', 'Validation'
    is_system_template = BooleanField() # Protected system template
    created_by = ForeignKey(User)
    configuration = JSONField()         # Complete dashboard config
    thumbnail = ImageField()            # Preview image
    usage_count = IntegerField()        # Times instantiated
```

**Key Methods**:
- `instantiate_for_user(user, dashboard_name)` - Creates dashboard from template

### 4. WidgetDataCache Model

**Location**: `api/models/dashboards.py`

Caches widget data to improve performance for expensive queries.

```python
class WidgetDataCache(models.Model):
    widget = OneToOneField(DashboardWidget)
    cached_data = JSONField()
    expires_at = DateTimeField()
```

**Key Methods**:
- `is_expired()` - Checks if cache is stale
- `refresh(data, ttl_seconds)` - Updates cache

---

## 🎨 Views

### CustomDashboardView

**Purpose**: Main dashboard display view

**Location**: `dashboard/views_dashboard.py`

**Features**:
- Displays dashboard with all widgets
- Auto-creates default dashboard if none exists
- Role-aware default widget configuration
- Loads data for all visible widgets

**Access Control**: `DashboardAccessMixin` (any logged-in user)

**Template**: `dashboard/custom_dashboard.html`

**Context Variables**:
- `dashboard` - Dashboard object
- `widgets_with_data` - List of `{'widget': widget, 'data': data}` dicts
- `user_dashboards` - User's other dashboards

**Role-Specific Default Dashboards**:

#### PA Staff Default Widgets:
```python
[
    {'widget_type': 'counter', 'title': 'My Total Submissions'},
    {'widget_type': 'counter', 'title': 'Pending Review'},
    {'widget_type': 'counter', 'title': 'Approved'},
    {'widget_type': 'counter', 'title': 'This Week'},
    {'widget_type': 'recent_activity', 'title': 'My Recent Submissions'}
]
```

#### Validator Default Widgets:
```python
[
    {'widget_type': 'counter', 'title': 'Pending Validations'},
    {'widget_type': 'counter', 'title': 'Assigned to Me'},
    {'widget_type': 'counter', 'title': 'Approval Rate'},
    {'widget_type': 'counter', 'title': 'Total Species'},
    {'widget_type': 'assigned_submissions', 'title': 'My Assigned Submissions'},
    {'widget_type': 'recent_activity', 'title': 'Recent Activity'}
]
```

#### DENR/Other Roles Default Widgets:
```python
[
    {'widget_type': 'counter', 'title': 'Total Species'},
    {'widget_type': 'counter', 'title': 'Total Observations'}
]
```

### DashboardListView

**Purpose**: List all user's dashboards

**URL**: `/dashboards/`

**Features**:
- Shows user's created dashboards
- Displays default dashboard first
- Pagination (20 per page)

### DashboardCreateView

**Purpose**: Create new dashboard

**URL**: `/dashboards/new/`

**Form Fields**:
- `name` - Dashboard name
- `description` - Optional description
- `is_default` - Set as default
- `is_shared` - Allow sharing

### DashboardUpdateView

**Purpose**: Edit existing dashboard

**URL**: `/dashboards/<pk>/edit/`

**Security**: Users can only edit their own dashboards

### DashboardDeleteView

**Purpose**: Delete dashboard

**URL**: `/dashboards/<pk>/delete/`

**Security**: Cannot delete if only dashboard (validation required)

### WidgetCreateView

**Purpose**: Add widget to dashboard

**URL**: `/dashboards/<dashboard_pk>/widgets/add/`

**Features**:
- Pre-select widget type via `?type=counter` URL parameter
- Auto-positions at bottom of dashboard
- Redirects to configuration modal after creation

**Form Fields**:
- `widget_type` - Type of widget
- `title` - Widget title
- `width` - Grid width (1-12)
- `height` - Grid height
- `refresh_interval` - Auto-refresh (seconds)

### WidgetUpdateView

**Purpose**: Edit widget settings

**URL**: `/widgets/<pk>/edit/`

**Editable Fields**:
- `title`
- `width`
- `height`
- `is_visible`
- `refresh_interval`

### WidgetDeleteView

**Purpose**: Remove widget from dashboard

**URL**: `/widgets/<pk>/delete/`

### TemplateGalleryView

**Purpose**: Browse dashboard templates

**URL**: `/dashboards/templates/`

**Features**:
- Lists system templates and user-created templates
- Sorted by usage count
- One-click instantiation
- Pagination (12 per page)

---

## 📊 Widget Data Providers

**Location**: `dashboard/views_widgets.py`

### Main Entry Point

```python
def get_widget_data(widget):
    """
    Fetches data for any widget type
    
    Args:
        widget: DashboardWidget instance
    
    Returns:
        dict: Widget data ready for rendering
    """
```

### Counter Widget Data Provider

```python
def get_counter_data(widget, config):
    """
    Returns single numeric value with optional trend
    
    Config Options:
        - metric: Which metric to display
        - comparison_period: 'week', 'month', 'year'
    
    Supported Metrics:
        - pending_validations
        - assigned_to_me
        - total_species
        - observations_today
        - disturbances_this_week
        - approval_rate
        - my_total_submissions
        - my_pending_submissions
        - my_approved_submissions
        - my_weekly_submissions
    
    Returns:
        {
            'value': int,
            'label': str,
            'trend': 'up' | 'down' | None,
            'trend_percentage': float
        }
    """
```

**Example Usage**:
```python
widget = DashboardWidget.objects.get(id=1)
data = get_counter_data(widget, {'metric': 'pending_validations'})
# {'value': 42, 'label': 'Pending Validations', 'trend': 'up', 'trend_percentage': 15.3}
```

### Pie Chart Data Provider

```python
def get_pie_chart_data(widget, config):
    """
    Returns data for pie chart visualization
    
    Config Options:
        - data_source: What data to chart
        - date_range: Days to include (default 30)
        - protected_area_filter: Filter by PA ID
        - species_type_filter: Filter by species type
        - user_filter: 'me' | 'all'
        - endemicity_filter: Filter species
        - threat_category_filter: Filter by IUCN status
    
    Supported Data Sources:
        - validation_status
        - species_by_type
        - disturbance_types
        - resource_use_types
        - observation_methods
    
    Returns:
        {
            'labels': [str],
            'values': [int],
            'colors': [hex_color]
        }
    """
```

**Example Configuration**:
```json
{
    "data_source": "validation_status",
    "date_range": 30,
    "user_filter": "me",
    "protected_area_filter": 5
}
```

### Bar Chart Data Provider

```python
def get_bar_chart_data(widget, config):
    """
    Returns data for bar chart visualization
    
    Config Options:
        - data_source: What data to chart
        - date_range: Days to include
        - group_by: 'day', 'week', 'month'
        - protected_area_filter: Filter by PA
        - species_type_filter: Filter by species type
    
    Supported Data Sources:
        - submissions_over_time
        - observations_by_species
        - validations_by_user
        - species_observations_count
        - disturbances_by_location
    
    Returns:
        {
            'labels': [str],
            'values': [int],
            'datasets': [{
                'label': str,
                'data': [int],
                'backgroundColor': hex_color
            }]
        }
    """
```

### Line Chart Data Provider

```python
def get_line_chart_data(widget, config):
    """
    Returns time-series data for line chart
    
    Config Options:
        - data_source: What metric to track
        - date_range: Days to include
        - group_by: 'day', 'week', 'month'
        - species_filter: Filter by species ID
        - multiple_series: Show multiple lines
    
    Supported Data Sources:
        - submissions_trend
        - species_sightings_trend
        - validation_completion_trend
        - disturbance_frequency
    
    Returns:
        {
            'labels': [date_str],
            'datasets': [{
                'label': str,
                'data': [float],
                'borderColor': hex_color,
                'fill': bool
            }]
        }
    """
```

### Data Table Data Provider

```python
def get_data_table_data(widget, config):
    """
    Returns tabular data
    
    Config Options:
        - data_source: What data to display
        - limit: Number of rows
        - sort_by: Column to sort by
        - sort_order: 'asc' | 'desc'
        - filters: Dict of filter criteria
    
    Supported Data Sources:
        - recent_submissions
        - recent_validations
        - top_species
        - active_pa_staff
        - flagged_observations
    
    Returns:
        {
            'columns': [{'name': str, 'field': str}],
            'rows': [dict],
            'total': int
        }
    """
```

### Recent Activity Data Provider

```python
def get_recent_activity_data(widget, config):
    """
    Returns recent activity feed
    
    Config Options:
        - limit: Number of items (default 10)
        - user_filter: 'me' | 'all'
        - activity_types: List of types to include
    
    Activity Types:
        - submission_created
        - validation_completed
        - observation_flagged
        - species_added
        - report_generated
    
    Returns:
        {
            'activities': [{
                'type': str,
                'user': str,
                'action': str,
                'target': str,
                'timestamp': datetime,
                'url': str
            }],
            'total': int
        }
    """
```

### Assigned Submissions Widget

**Purpose**: Shows submissions assigned to current validator

**Config Options**:
- `limit` - Number of submissions (default 10)
- `status_filter` - Filter by validation status

**Returns**:
```python
{
    'submissions': [DataSubmission],
    'total_assigned': int,
    'pending_count': int
}
```

### Validation Stats Widget

**Purpose**: Validator performance metrics

**Returns**:
```python
{
    'total_validated': int,
    'approved': int,
    'rejected': int,
    'approval_rate': float,
    'avg_validation_time': timedelta,
    'validations_this_week': int
}
```

### Species Summary Widget

**Purpose**: Species diversity overview

**Config Options**:
- `species_type_filter` - Filter by type
- `endemicity_filter` - Filter by endemicity
- `threat_category_filter` - Filter by IUCN status

**Returns**:
```python
{
    'total_species': int,
    'fauna_count': int,
    'flora_count': int,
    'endemic_count': int,
    'threatened_count': int,
    'by_type': [{
        'type': str,
        'count': int
    }]
}
```

### Disturbance Alerts Widget

**Purpose**: Recent disturbance notifications

**Config Options**:
- `severity_filter` - Filter by severity
- `limit` - Number of alerts

**Returns**:
```python
{
    'alerts': [{
        'disturbance': DisturbanceRecord,
        'severity': str,
        'location': str,
        'reported': datetime
    }],
    'total_active': int
}
```

### Schedule Calendar Widget

**Purpose**: Upcoming survey schedules

**Config Options**:
- `days_ahead` - Days to show (default 30)

**Returns**:
```python
{
    'events': [{
        'title': str,
        'date': date,
        'type': str,
        'location': str
    }],
    'total_upcoming': int
}
```

---

## 🔧 API Endpoints

### Widget Management APIs

#### Move Widget
```
POST /api/widgets/<widget_id>/move/
Body: {
    "row_position": int,
    "column_position": int
}
Response: {"success": true}
```

#### Resize Widget
```
POST /api/widgets/<widget_id>/resize/
Body: {
    "width": int,
    "height": int
}
Response: {"success": true}
```

#### Toggle Visibility
```
POST /api/widgets/<widget_id>/toggle/
Response: {
    "success": true,
    "is_visible": bool
}
```

#### Refresh Widget Data
```
GET /api/widgets/<widget_id>/refresh/
Response: {
    "success": true,
    "data": {...}
}
```

### Dashboard Management APIs

#### Duplicate Dashboard
```
POST /dashboards/<pk>/duplicate/
Body: {"new_name": str}
Response: {
    "success": true,
    "new_dashboard_id": int
}
```

#### Share Dashboard
```
POST /dashboards/<pk>/share/
Body: {
    "user_ids": [int]
}
Response: {"success": true}
```

#### Export Dashboard Configuration
```
GET /dashboards/<pk>/export/
Response: {
    "dashboard": {...},
    "widgets": [...]
}
```

### Template Management

#### Instantiate Template
```
POST /templates/<template_id>/instantiate/
Body: {"dashboard_name": str}
Response: {
    "success": true,
    "dashboard_id": int,
    "redirect_url": str
}
```

---

## 📝 Usage Examples

### Example 1: Creating a Custom Dashboard

```python
# In your view or management command
from api.models import Dashboard, DashboardWidget

# Create dashboard
dashboard = Dashboard.objects.create(
    user=request.user,
    name="Biodiversity Monitoring Dashboard",
    description="Track wildlife observations and species diversity",
    is_default=True
)

# Add KPI widgets
DashboardWidget.objects.create(
    dashboard=dashboard,
    widget_type='counter',
    title='Total Observations',
    configuration={'metric': 'total_observations'},
    row_position=0,
    column_position=0,
    width=3,
    height=2
)

# Add pie chart
DashboardWidget.objects.create(
    dashboard=dashboard,
    widget_type='pie_chart',
    title='Observations by Species Type',
    configuration={
        'data_source': 'species_by_type',
        'date_range': 90
    },
    row_position=0,
    column_position=3,
    width=6,
    height=4
)

# Add data table
DashboardWidget.objects.create(
    dashboard=dashboard,
    widget_type='data_table',
    title='Recent Submissions',
    configuration={
        'data_source': 'recent_submissions',
        'limit': 10,
        'sort_by': 'submission_time',
        'sort_order': 'desc'
    },
    row_position=1,
    column_position=0,
    width=12,
    height=4
)
```

### Example 2: Creating a Dashboard Template

```python
from api.models import DashboardTemplate

template = DashboardTemplate.objects.create(
    name="Validator Workstation",
    description="Essential tools for validation workflow",
    category="Validation",
    is_system_template=True,
    created_by=admin_user,
    configuration={
        'layout_config': {'grid_size': 12},
        'widgets': [
            {
                'widget_type': 'counter',
                'title': 'Pending Validations',
                'configuration': {'metric': 'pending_validations'},
                'row_position': 0,
                'column_position': 0,
                'width': 4,
                'height': 2
            },
            {
                'widget_type': 'assigned_submissions',
                'title': 'My Queue',
                'configuration': {'limit': 15},
                'row_position': 1,
                'column_position': 0,
                'width': 8,
                'height': 5
            },
            {
                'widget_type': 'validation_stats',
                'title': 'My Performance',
                'configuration': {},
                'row_position': 1,
                'column_position': 8,
                'width': 4,
                'height': 5
            }
        ]
    }
)

# User instantiates template
dashboard = template.instantiate_for_user(
    user=validator_user,
    dashboard_name="My Validation Workspace"
)
```

### Example 3: Custom Widget Configuration

```python
# Create a configurable bar chart widget
widget = DashboardWidget.objects.create(
    dashboard=my_dashboard,
    widget_type='bar_chart',
    title='Species Observations by Month',
    configuration={
        'data_source': 'submissions_over_time',
        'date_range': 365,
        'group_by': 'month',
        'protected_area_filter': 3,
        'species_type_filter': 'Bird',
        'chart_options': {
            'show_legend': True,
            'stacked': False,
            'show_data_labels': True
        }
    },
    row_position=2,
    column_position=0,
    width=12,
    height=5,
    refresh_interval=300  # Auto-refresh every 5 minutes
)
```

### Example 4: Programmatically Fetching Widget Data

```python
from dashboard.views_widgets import get_widget_data

widget = DashboardWidget.objects.get(id=123)
data = get_widget_data(widget)

# For counter widget
print(f"{data['label']}: {data['value']}")
if data['trend']:
    print(f"Trend: {data['trend']} {data['trend_percentage']}%")

# For chart widget
print(f"Labels: {data['labels']}")
print(f"Values: {data['values']}")
```

---

## 🎯 User Workflows

### Workflow 1: PA Staff Creating Personal Dashboard

1. **Login** → Navigate to "My Dashboards"
2. **Click** "Create New Dashboard"
3. **Enter** name: "My Field Work Tracker"
4. **Set** as default dashboard
5. **Add Widgets**:
   - Counter: "My Submissions This Month"
   - Line Chart: "My Observation Trend"
   - Recent Activity: "My Recent Submissions"
6. **Arrange** widgets using drag-and-drop
7. **Save** and set auto-refresh intervals

### Workflow 2: Validator Using Template

1. **Navigate** to Template Gallery
2. **Browse** system templates
3. **Select** "Validator Workstation" template
4. **Click** "Use Template"
5. **Customize** dashboard name
6. **Instantiate** → Auto-generates full dashboard
7. **Adjust** widget positions as needed

### Workflow 3: Admin Creating System Template

1. **Create** a well-designed dashboard
2. **Test** with sample data
3. **Navigate** to Admin → Templates
4. **Export** current dashboard as template
5. **Mark** as "System Template"
6. **Add** thumbnail and description
7. **Publish** to template gallery

---

## 🔒 Security & Permissions

### Dashboard Access Control

- **Ownership**: Users can only edit their own dashboards
- **Sharing**: Dashboard owner controls who can view
- **Templates**: System templates locked to admins

### Permission Checks

```python
# In views_dashboard.py
class DashboardUpdateView(LoginRequiredMixin, UpdateView):
    def get_queryset(self):
        # Users can only edit their own dashboards
        return Dashboard.objects.filter(user=self.request.user)

# In CustomDashboardView
def get_queryset(self):
    # Users can access own dashboards or shared ones
    return Dashboard.objects.filter(
        Q(user=self.request.user) |
        Q(shared_with=self.request.user)
    ).distinct()
```

### Data Filtering

Widget data is automatically filtered based on user role:
- **PA Staff**: Only see their own submissions
- **Validators**: See assigned submissions + queue
- **DENR**: See all validated data
- **System Admin**: Full access

---

## ⚙️ Configuration Options

### Grid System

- **Columns**: 12-column responsive grid
- **Row Height**: Configurable (default 50px)
- **Breakpoints**:
  - Desktop: Full grid
  - Tablet: Stacked layout
  - Mobile: Single column

### Widget Configuration Schema

```json
{
    "data_source": "string",           // Required for data widgets
    "date_range": 30,                  // Days to include
    "protected_area_filter": 5,        // Filter by PA ID
    "species_type_filter": "Bird",     // Filter by species type
    "user_filter": "me",               // 'me' or 'all'
    "limit": 10,                       // Rows/items to show
    "sort_by": "field_name",           // Sort column
    "sort_order": "desc",              // 'asc' or 'desc'
    "group_by": "month",               // 'day', 'week', 'month'
    "chart_options": {                 // Chart-specific options
        "show_legend": true,
        "stacked": false,
        "show_data_labels": true,
        "colors": ["#28a745", "#dc3545"]
    }
}
```

### Auto-Refresh Configuration

```python
# In widget model
refresh_interval = 300  # 300 seconds = 5 minutes

# Minimum refresh interval: 10 seconds
# Recommended for real-time: 30-60 seconds
# Recommended for stats: 300-600 seconds (5-10 minutes)
```

---

## 🧪 Testing

### Test Scenarios

#### Dashboard Creation
- [ ] User can create new dashboard
- [ ] Default dashboard auto-created on first login
- [ ] Only one dashboard can be default
- [ ] Dashboard names must be unique per user

#### Widget Management
- [ ] User can add widgets to dashboard
- [ ] Widget type cannot be changed after creation
- [ ] Widgets can be moved and resized
- [ ] Widget deletion removes from dashboard only

#### Data Loading
- [ ] Counter widgets show correct metrics
- [ ] Chart widgets render with correct data
- [ ] Data respects user role permissions
- [ ] Empty states display when no data

#### Role-Based Defaults
- [ ] PA Staff gets field work dashboard
- [ ] Validators get validation dashboard
- [ ] DENR gets overview dashboard
- [ ] Admin gets system management dashboard

### Example Test

```python
from django.test import TestCase
from api.models import Dashboard, DashboardWidget
from users.models import User

class DashboardTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testpa',
            role='pa_staff'
        )
        
    def test_default_dashboard_creation(self):
        """Test PA Staff gets appropriate default dashboard"""
        self.client.force_login(self.user)
        response = self.client.get('/dashboards/')
        
        # Should auto-create default dashboard
        dashboard = Dashboard.objects.get(user=self.user, is_default=True)
        self.assertIsNotNone(dashboard)
        
        # Should have PA Staff widgets
        widgets = dashboard.widgets.all()
        self.assertGreater(len(widgets), 0)
        
        widget_types = [w.widget_type for w in widgets]
        self.assertIn('counter', widget_types)
        self.assertIn('recent_activity', widget_types)
```

---

## 🔧 Troubleshooting

### Common Issues

#### Issue: Dashboard not loading
**Solution**: Check that user has at least one dashboard. Auto-creation should trigger on first access.

#### Issue: Widget shows no data
**Solution**: Verify widget configuration is correct and user has permission to view the data source.

#### Issue: Auto-refresh not working
**Solution**: Check `refresh_interval` is set and >= 10 seconds. Verify JavaScript console for errors.

#### Issue: Dashboard layout broken
**Solution**: Reset `layout_config` to default or recreate dashboard. Check browser console for grid errors.

### Debug Commands

```bash
# Check user's dashboards
docker-compose exec web python manage.py shell
>>> from api.models import Dashboard
>>> Dashboard.objects.filter(user__username='testuser')

# Test widget data fetching
>>> from dashboard.views_widgets import get_widget_data
>>> from api.models import DashboardWidget
>>> widget = DashboardWidget.objects.first()
>>> data = get_widget_data(widget)
>>> print(data)

# Clear widget cache
>>> from api.models import WidgetDataCache
>>> WidgetDataCache.objects.all().delete()
```

---

## 📚 Related Documentation

- [User Roles & Permissions](../architecture/RBAC_DOCUMENTATION.md)
- [Validation System](VALIDATION_SYSTEM.md)
- [GIS Features](GIS_FEATURES.md)
- [Report Generation](REPORT_GENERATION.md)

---

*Last Updated: January 6, 2026*
*Version: 1.0*
