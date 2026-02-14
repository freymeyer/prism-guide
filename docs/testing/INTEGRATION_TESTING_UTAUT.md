# PRISM-Matutum Integration Testing Guide - UTAUT Framework

## 📋 Document Overview

This guide provides comprehensive integration testing procedures for PRISM-Matutum, incorporating UTAUT (Unified Theory of Acceptance and Use of Technology) evaluation at each integration point. Integration tests verify that system components work together correctly and deliver a cohesive user experience.

---

## 🎯 Integration Testing Philosophy

**Integration testing with UTAUT** goes beyond verifying technical integration—it ensures that connected components create a seamless experience that users will accept and adopt.

### Key Principles

1. **End-to-End User Flows**: Test complete user journeys, not isolated components
2. **Cross-Component Validation**: Verify data flows correctly between system layers
3. **User Experience Continuity**: Ensure smooth transitions between features
4. **UTAUT at Every Touchpoint**: Evaluate acceptance factors at each integration point

---

## 🔄 Integration Testing Scope

### System Integration Points

```
┌─────────────────────────────────────────────────────────┐
│                    External Systems                      │
├─────────────────────────────────────────────────────────┤
│  KoboToolbox API  →  Celery Tasks  →  Django Backend   │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Data Layer (PostgreSQL + PostGIS)      │
├─────────────────────────────────────────────────────────┤
│  DataSubmission → Validation → Processed Records        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Business Logic Layer                   │
├─────────────────────────────────────────────────────────┤
│  Services → Workflows → Validation Rules                │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Presentation Layer                     │
├─────────────────────────────────────────────────────────┤
│  Views → Templates → JavaScript → User Interface        │
└─────────────────────────────────────────────────────────┘
```

---

## 🧪 Integration Test Scenarios

## Integration Test 1: KoboToolbox to DataSubmission Flow

### Test Objective
Verify that field data submitted through KoboToolbox is correctly synchronized, stored, and made available for validation.

### Components Involved
- KoboToolbox API
- Celery Task: `sync_kobotoolbox_data`
- `DataSubmission` Model
- `SubmissionService`

### Test Steps

#### 1. Preconditions
```bash
# Setup test environment
docker-compose exec web python manage.py shell
```

```python
from users.models import User
from api.models import KoboForm

# Create test PA Staff user
pa_staff = User.objects.create_user(
    username='test_pa_staff',
    email='pa_staff@test.com',
    role='pa_staff',
    account_status='approved'
)

# Verify Kobo form exists
form = KoboForm.objects.first()
print(f"Testing with form: {form.name}")
```

#### 2. Trigger Data Sync
```bash
# Manually trigger KoboToolbox sync
docker-compose exec web python manage.py sync_kobotoolbox_data
```

#### 3. Verify Data Import

**Functional Verification:**
```python
from api.models import DataSubmission

# Check that submissions were created
recent_submissions = DataSubmission.objects.filter(
    created_at__gte=timezone.now() - timedelta(hours=1)
)

assert recent_submissions.exists(), "No submissions created from Kobo sync"
print(f"✅ {recent_submissions.count()} submissions imported")

# Verify submission structure
submission = recent_submissions.first()
assert submission.raw_data is not None, "Raw data not stored"
assert submission.submission_id, "Submission ID missing"
assert submission.kobo_form_id, "Kobo form reference missing"
print("✅ Submission structure is valid")
```

**UTAUT Evaluation:**

**Performance Expectancy:**
```python
# Time sync performance
import time
start_time = time.time()

# Run sync (in test, use smaller dataset)
call_command('sync_kobotoolbox_data', limit=10)

sync_time = time.time() - start_time
print(f"Sync time for 10 submissions: {sync_time:.2f}s")

# PE Target: <30s for 100 submissions
assert sync_time < 5.0, "Sync is too slow for small dataset"
print("✅ PE: Sync performance is acceptable")
```

**Effort Expectancy:**
```python
# Verify data doesn't require manual cleanup
submission = recent_submissions.first()

# Check data quality
assert submission.raw_data.get('_id'), "Missing Kobo ID"
assert submission.submitted_by, "Submitter not linked"
print("✅ EE: Data is ready for validation without manual intervention")
```

**Facilitating Conditions:**
```python
# Check error handling
from api.tasks import sync_kobotoolbox_data

# Simulate error scenario
try:
    result = sync_kobotoolbox_data.apply(kwargs={'invalid_param': True})
except Exception as e:
    print(f"✅ FC: Errors are caught and logged: {e}")

# Verify sync continues after errors
assert DataSubmission.objects.exists(), "System should remain functional after errors"
```

#### 4. Verify PA Staff Can See Submission

**Functional Test:**
```python
from django.test import Client

client = Client()
client.login(username='test_pa_staff', password='testpass123')

response = client.get('/dashboard/my-submissions/')
assert response.status_code == 200
print("✅ PA Staff can access submissions")
```

**UTAUT Verification:**
```python
content = response.content.decode()

# PE: Submission is immediately visible
assert str(submission.submission_id) in content or submission.kobo_form.name in content
print("✅ PE: Submission appears immediately after sync")

# EE: Submission data is formatted for readability
assert 'date' in content.lower() and 'status' in content.lower()
print("✅ EE: Submission display is clear")

# SI: Shows submission came from legitimate source (Kobo)
assert 'kobo' in content.lower() or submission.kobo_form.name in content
print("✅ SI: Data source is transparent")
```

### Integration Success Criteria

**Functional:**
- ✅ All Kobo submissions are imported
- ✅ No data loss or corruption
- ✅ Submissions link to correct PA Staff user
- ✅ Data appears in PA Staff dashboard

**UTAUT:**
- ✅ PE: Sync completes in acceptable time (<30s per 100 records)
- ✅ EE: No manual data cleanup required
- ✅ SI: Data source attribution is clear
- ✅ FC: Error handling prevents system failure

---

## Integration Test 2: Submission → Validation → Approval Flow

### Test Objective
Verify the complete validation workflow from submission through approval, including all notifications and state changes.

### Components Involved
- `DataSubmission` Model
- `WorkflowService`
- `ValidationService`
- Validator Dashboard Views
- PA Staff Dashboard Views
- Notification System

### Test Steps

#### 1. Create Test Submission

```python
from api.models import DataSubmission
from api.submission_services import DataSubmissionService

# Create submission programmatically
submission_data = {
    'species_observed': 'Philippine Eagle',
    'observation_date': '2026-01-06',
    'location': 'Transect A',
    'observer_name': 'Test PA Staff'
}

submission = DataSubmissionService.create_submission(
    kobo_form_id='test_form',
    raw_data=submission_data,
    submitted_by=pa_staff_user
)

print(f"✅ Test submission created: {submission.id}")
assert submission.validation_status == '---', "Initial status should be pending"
```

#### 2. Validator Sees Submission in Queue

**Functional Test:**
```python
# Login as validator
validator = User.objects.create_user(
    username='test_validator',
    role='validator',
    account_status='approved'
)

client = Client()
client.login(username='test_validator', password='testpass123')

# Access validation queue
response = client.get('/dashboard/validation/queue/')
assert response.status_code == 200

content = response.content.decode()
assert str(submission.id) in content or str(submission.submission_id) in content
print("✅ Submission appears in validation queue")
```

**UTAUT Evaluation:**
```python
# PE: Queue helps validator prioritize work
assert 'pending' in content.lower()
assert 'priority' in content.lower() or 'date' in content.lower()
print("✅ PE: Queue provides prioritization tools")

# EE: Submission is easy to identify and select
assert submission.submitted_by.get_full_name() in content or pa_staff_user.username in content
print("✅ EE: Submissions are clearly labeled")
```

#### 3. Validator Opens Validation Workspace

**Functional Test:**
```python
# Open validation workspace
response = client.get(f'/dashboard/validation/workspace/{submission.id}/')
assert response.status_code == 200

content = response.content.decode()
print("✅ Validation workspace loads successfully")
```

**UTAUT Evaluation:**
```python
# PE: All necessary information is present
assert 'species' in content.lower()
assert 'date' in content.lower()
assert 'location' in content.lower()
print("✅ PE: Workspace shows all validation-relevant data")

# EE: Action buttons are clear
assert 'approve' in content.lower()
assert 'reject' in content.lower()
print("✅ EE: Validation actions are obvious")

# Check load time
start = time.time()
response = client.get(f'/dashboard/validation/workspace/{submission.id}/')
load_time = time.time() - start
assert load_time < 1.5, f"Workspace load time: {load_time:.2f}s"
print(f"✅ PE: Workspace loads in {load_time:.2f}s")
```

#### 4. Validator Approves Submission

**Functional Test:**
```python
# Submit approval
response = client.post('/dashboard/api/validate/', {
    'submission_id': submission.id,
    'action': 'approve',
    'remarks': 'Data quality is excellent. All fields complete.'
})

assert response.status_code == 200
data = response.json()
assert data['status'] == 'success'
print("✅ Approval submitted successfully")

# Verify status change
submission.refresh_from_db()
assert submission.validation_status == 'approved'
assert submission.validated_by == validator
assert submission.validated_at is not None
print("✅ Submission status updated to approved")
```

**UTAUT Evaluation:**
```python
# PE: Approval is instant
start = time.time()
client.post('/dashboard/api/validate/', {
    'submission_id': submission.id,
    'action': 'approve',
    'remarks': 'Test'
})
approval_time = time.time() - start
assert approval_time < 1.0, f"Approval time: {approval_time:.2f}s"
print(f"✅ PE: Approval completes in {approval_time:.2f}s")

# SI: Validator attribution is recorded
assert submission.validated_by == validator
print("✅ SI: Validator accountability is maintained")
```

#### 5. PA Staff Sees Approval Status

**Functional Test:**
```python
# Switch to PA Staff client
client.logout()
client.login(username='test_pa_staff', password='testpass123')

# View submission detail
response = client.get(f'/dashboard/submission/{submission.id}/')
assert response.status_code == 200

content = response.content.decode()
assert 'approved' in content.lower()
print("✅ PA Staff can see approval status")
```

**UTAUT Evaluation:**
```python
# PE: Status update is immediate (no delay)
# Already verified submission was updated instantly

# EE: Approval is clearly communicated
assert 'approved' in content.lower() or '✓' in content or '✔' in content
print("✅ EE: Approval status is visually clear")

# SI: Validator feedback is visible
assert submission.validation_remarks in content
print("✅ SI: Validator feedback is transparent")

# FC: Notification sent (if implemented)
from dashboard.models import Notification
notification_exists = Notification.objects.filter(
    user=pa_staff_user,
    submission=submission
).exists()
if notification_exists:
    print("✅ FC: PA Staff received approval notification")
```

#### 6. Verify Data Processing Pipeline

**Functional Test:**
```python
from api.models import WildlifeObservation

# Check if approval triggered data processing
# (Depending on your workflow, approved data may be processed into WildlifeObservation)
wildlife_obs = WildlifeObservation.objects.filter(
    submission=submission
).first()

if wildlife_obs:
    print("✅ Approval triggered data processing")
    assert wildlife_obs.species_name == submission_data['species_observed']
    print("✅ Processed data matches submission")
```

### Integration Success Criteria

**Functional:**
- ✅ Submission moves through workflow states correctly
- ✅ Validator actions update database immediately
- ✅ PA Staff sees updated status
- ✅ Audit trail is complete
- ✅ Notifications are sent (if implemented)

**UTAUT:**
- ✅ PE: Workflow is efficient (<2 min per validation)
- ✅ EE: Interface is intuitive at each step
- ✅ SI: Roles and accountability are clear
- ✅ FC: System handles errors gracefully

---

## Integration Test 3: Species Management ↔ Observations Linking

### Test Objective
Verify that species management and wildlife observations are correctly integrated, allowing users to link observations to species and view all observations for a species.

### Components Involved
- `Species` Model (Fauna/Flora)
- `WildlifeObservation` Model
- Species Detail View
- Observation List View

### Test Steps

#### 1. Create Species

```python
from species.models import FaunaSpecies, TaxonomyClass

# Create taxonomy hierarchy
taxonomy_class = TaxonomyClass.objects.get_or_create(
    name='Aves',
    common_name='Birds'
)[0]

# Create species
species = FaunaSpecies.objects.create(
    scientific_name='Pithecophaga jefferyi',
    common_name='Philippine Eagle',
    taxonomy_class=taxonomy_class,
    conservation_status='CR'  # Critically Endangered
)

print(f"✅ Species created: {species.scientific_name}")
```

#### 2. Create Wildlife Observation Linked to Species

```python
from api.models import WildlifeObservation
from django.contrib.gis.geos import Point

observation = WildlifeObservation.objects.create(
    submission=submission,
    species=species,  # Link to species
    species_name=species.common_name,
    observed_at=timezone.now(),
    location=Point(125.0, 7.5, srid=4326),
    observer=pa_staff_user,
    observation_notes='Adult eagle spotted in canopy'
)

print(f"✅ Observation linked to species: {observation.id}")
```

#### 3. View Species Detail with Observations

**Functional Test:**
```python
client = Client()
client.login(username='test_validator', password='testpass123')

# View species detail page
response = client.get(f'/dashboard/species/{species.id}/')
assert response.status_code == 200

content = response.content.decode()
print("✅ Species detail page loads")
```

**UTAUT Evaluation:**
```python
# PE: Species page shows linked observations
assert str(observation.id) in content or 'observation' in content.lower()
print("✅ PE: Species page provides observation history")

# EE: Navigation between species and observations is intuitive
assert 'view observations' in content.lower() or 'observations' in content
print("✅ EE: Observation access is clearly available")

# SI: Species information is authoritative
assert species.scientific_name in content
assert 'Critically Endangered' in content or 'CR' in content
print("✅ SI: Species data appears authoritative")
```

#### 4. View Observations for Species

**Functional Test:**
```python
# Navigate to species observations
response = client.get(f'/dashboard/species/{species.id}/observations/')
assert response.status_code == 200

content = response.content.decode()
assert observation.observation_notes in content
print("✅ Observation list for species works")
```

**UTAUT Evaluation:**
```python
# PE: Filtering observations by species provides insights
observations_count = WildlifeObservation.objects.filter(species=species).count()
assert str(observations_count) in content
print(f"✅ PE: Shows {observations_count} observations for species")

# EE: Easy to navigate from species to observations and back
assert 'species' in content.lower() and species.common_name in content
print("✅ EE: Breadcrumb/navigation maintains context")
```

### Integration Success Criteria

**Functional:**
- ✅ Species and observations are correctly linked
- ✅ Species detail shows observation count
- ✅ Observations page filters by species
- ✅ Navigation between views works

**UTAUT:**
- ✅ PE: Integration provides valuable insights (observation history)
- ✅ EE: Navigation is intuitive
- ✅ SI: Data relationships are clear

---

## Integration Test 4: GIS Map ↔ Spatial Data

### Test Objective
Verify that the interactive map correctly displays spatial data from various sources (observations, transect routes, protected areas).

### Components Involved
- Interactive Map View
- GeoJSON API Endpoints
- PostGIS Database
- Leaflet.js Frontend

### Test Steps

#### 1. Create Spatial Data

```python
from django.contrib.gis.geos import Point, LineString, Polygon
from dashboard.models import MonitoringLocation, TransectRoute, ProtectedArea

# Create monitoring location
location = MonitoringLocation.objects.create(
    name='Observation Point Alpha',
    location_type='observation_point',
    coordinates=Point(125.1, 7.6, srid=4326),
    description='Primary eagle observation point'
)

# Create transect route
route = TransectRoute.objects.create(
    name='Trail A',
    route_line=LineString(
        [(125.1, 7.6), (125.15, 7.65), (125.2, 7.7)],
        srid=4326
    ),
    length_km=5.2
)

print("✅ Spatial data created")
```

#### 2. Test GeoJSON API Endpoints

**Functional Test:**
```python
client = Client()
client.login(username='test_validator', password='testpass123')

# Test observations GeoJSON endpoint
response = client.get('/dashboard/api/observations-geojson/')
assert response.status_code == 200
data = response.json()

assert data['type'] == 'FeatureCollection'
assert 'features' in data
print(f"✅ Observations GeoJSON returns {len(data['features'])} features")

# Test routes GeoJSON endpoint
response = client.get('/dashboard/api/transect-routes-geojson/')
assert response.status_code == 200
data = response.json()
print(f"✅ Routes GeoJSON returns {len(data['features'])} features")
```

**UTAUT Evaluation:**
```python
# PE: API responds quickly
start = time.time()
response = client.get('/dashboard/api/observations-geojson/')
api_time = time.time() - start
assert api_time < 1.0, f"GeoJSON API time: {api_time:.2f}s"
print(f"✅ PE: GeoJSON API responds in {api_time:.2f}s")

# Verify data structure is correct for Leaflet
feature = data['features'][0] if data['features'] else None
if feature:
    assert 'geometry' in feature
    assert 'properties' in feature
    print("✅ EE: GeoJSON structure is standard-compliant")
```

#### 3. Test Interactive Map Page

**Functional Test:**
```python
# Load interactive map
response = client.get('/dashboard/gis/map/')
assert response.status_code == 200

content = response.content.decode()
print("✅ Interactive map page loads")
```

**UTAUT Evaluation:**
```python
# EE: Map libraries are loaded
assert 'leaflet' in content.lower()
print("✅ EE: Leaflet library is loaded")

# PE: Map provides data visualization tools
assert 'layer' in content.lower() or 'filter' in content.lower()
print("✅ PE: Map includes data exploration tools")

# Check JavaScript initializes map correctly
assert 'L.map' in content or 'map(' in content
print("✅ Technical: Map initialization code present")
```

#### 4. Verify Map Interactions (Manual/Automated)

**Manual Testing Required:**
- [ ] Map loads and displays tiles
- [ ] Observation markers appear
- [ ] Clicking marker shows popup with observation details
- [ ] Transect routes display as lines
- [ ] Layer toggle works
- [ ] Map zoom/pan is smooth

**Automated Test (Selenium - if available):**
```python
# Example with Selenium
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait

driver = webdriver.Chrome()
driver.get('http://localhost:8000/dashboard/gis/map/')

# Wait for map to load
wait = WebDriverWait(driver, 10)
map_element = wait.until(
    lambda d: d.find_element(By.ID, 'map')
)

# Check map has loaded
assert map_element.is_displayed()
print("✅ Map is visible")

# Click on a marker (if any)
markers = driver.find_elements(By.CLASS_NAME, 'leaflet-marker-icon')
if markers:
    markers[0].click()
    # Check popup appears
    popup = driver.find_element(By.CLASS_NAME, 'leaflet-popup')
    assert popup.is_displayed()
    print("✅ Marker popup works")

driver.quit()
```

### Integration Success Criteria

**Functional:**
- ✅ GeoJSON APIs return valid data
- ✅ Map displays all spatial layers
- ✅ User interactions work (click, zoom, pan)
- ✅ Popups show correct data

**UTAUT:**
- ✅ PE: Map provides valuable spatial insights
- ✅ EE: Map controls are intuitive
- ✅ FC: Map loads even with slow connections (progressive loading)

---

## Integration Test 5: Report Generation End-to-End

### Test Objective
Verify that the report generation system correctly aggregates data from multiple sources and produces downloadable reports.

### Components Involved
- Report Builder View
- `ReportService`
- Celery Task: `generate_report`
- Report Template Engine
- PDF/Excel Export

### Test Steps

#### 1. Access Report Builder

**Functional Test:**
```python
client = Client()
# Login as user with report generation permission (system_admin or denr)
admin_user = User.objects.create_user(
    username='test_admin',
    role='system_admin',
    account_status='approved'
)
client.login(username='test_admin', password='testpass123')

response = client.get('/dashboard/reports/builder/')
assert response.status_code == 200
print("✅ Report builder is accessible")
```

**UTAUT Evaluation:**
```python
content = response.content.decode()

# EE: Builder interface is intuitive
assert 'report type' in content.lower()
assert 'date range' in content.lower()
print("✅ EE: Report options are clear")
```

#### 2. Generate Report

**Functional Test:**
```python
from dashboard.models import GeneratedReport

# Submit report generation request
response = client.post('/dashboard/reports/generate/', {
    'report_type': 'semestral_bms',
    'date_from': '2026-01-01',
    'date_to': '2026-06-30',
    'geographic_scope': 'all'
})

assert response.status_code == 302  # Redirect after submission
print("✅ Report generation request submitted")

# Check report was created
report = GeneratedReport.objects.filter(
    generated_by=admin_user,
    report_type='semestral_bms'
).latest('created_at')

assert report.status == 'PENDING' or report.status == 'PROCESSING'
print(f"✅ Report created with status: {report.status}")
```

**UTAUT Evaluation:**
```python
# PE: Report generation is triggered immediately
assert report.created_at >= timezone.now() - timedelta(seconds=10)
print("✅ PE: Report generation starts immediately")

# EE: User gets confirmation
# (Check redirect or message)
```

#### 3. Simulate Report Processing (Celery Task)

**Functional Test:**
```python
# In test environment, run Celery task synchronously
from dashboard.tasks import generate_report_task

# Run task
result = generate_report_task.apply(args=[report.id])

# Check task completed
report.refresh_from_db()
assert report.status == 'COMPLETED', f"Report status: {report.status}"
print("✅ Report generation completed")

# Verify report file exists
assert report.file_path
assert os.path.exists(report.file_path.path)
print(f"✅ Report file generated: {report.file_path.name}")
```

**UTAUT Evaluation:**
```python
# PE: Report generation time is reasonable
generation_time = (report.completed_at - report.created_at).total_seconds()
assert generation_time < 60, f"Generation time: {generation_time}s"
print(f"✅ PE: Report generated in {generation_time:.1f}s")

# Check report content
import openpyxl
wb = openpyxl.load_workbook(report.file_path.path)
assert len(wb.sheetnames) > 0
print(f"✅ Report contains {len(wb.sheetnames)} sheets")
```

#### 4. Download Report

**Functional Test:**
```python
# Access report archive
response = client.get('/dashboard/reports/archive/')
assert response.status_code == 200

content = response.content.decode()
assert report.report_name in content or 'semestral' in content.lower()
print("✅ Report appears in archive")

# Download report
response = client.get(f'/dashboard/reports/download/{report.id}/')
assert response.status_code == 200
assert 'application' in response['Content-Type']
print("✅ Report download works")
```

**UTAUT Evaluation:**
```python
# PE: Download is immediate
start = time.time()
response = client.get(f'/dashboard/reports/download/{report.id}/')
download_time = time.time() - start
assert download_time < 3.0, f"Download time: {download_time:.2f}s"
print(f"✅ PE: Download completes in {download_time:.2f}s")

# EE: File format is user-friendly
assert report.file_path.name.endswith('.xlsx') or report.file_path.name.endswith('.pdf')
print("✅ EE: Report format is familiar (Excel/PDF)")
```

### Integration Success Criteria

**Functional:**
- ✅ Report builder accepts configuration
- ✅ Celery task generates report
- ✅ Report file is created
- ✅ Download works
- ✅ Report contains expected data

**UTAUT:**
- ✅ PE: Report generation is timely (<2 min for standard reports)
- ✅ EE: Report builder is intuitive
- ✅ FC: Reports are reliably generated and stored

---

## 🐳 Integration Testing with Docker

### Test Environment Setup

```bash
# 1. Start test environment
docker-compose up -d

# 2. Run migrations
docker-compose exec web python manage.py migrate

# 3. Create test data
docker-compose exec web python manage.py shell < create_test_data.py

# 4. Run integration tests
docker-compose exec web python manage.py test dashboard.tests.test_integration
```

### Test Database Isolation

```python
# dashboard/tests/test_integration.py

from django.test import TransactionTestCase

class IntegrationTest(TransactionTestCase):
    """Use TransactionTestCase for tests that need Celery or multiple DB transactions"""
    
    def setUp(self):
        # Setup runs before each test
        self.create_test_users()
        self.create_test_data()
        
    def tearDown(self):
        # Cleanup after each test
        # TransactionTestCase automatically rolls back DB
        pass
```

---

## 🔍 UTAUT Integration Evaluation Matrix

### Evaluation at Each Integration Point

| Integration Point | PE Metric | EE Metric | SI Metric | FC Metric |
|-------------------|-----------|-----------|-----------|-----------|
| **Kobo → Django** | Sync time <30s per 100 | Auto-sync, no manual steps | Data source is transparent | Error handling prevents data loss |
| **Submit → Validate** | Validation <2 min | Intuitive UI at each step | Validator accountability | Clear workflow states |
| **Species ↔ Observations** | Insights from linking | Easy navigation | Authoritative data | Relationships are maintained |
| **GIS Map ↔ Data** | Spatial analysis tools | Intuitive map controls | Professional cartography | Robust loading |
| **Report Generation** | Fast generation <2 min | Simple builder interface | Official reporting | Reliable export |

---

## 📊 Integration Test Metrics

### Quantitative Metrics

```python
# dashboard/tests/test_integration_metrics.py

class IntegrationPerformanceTests(TestCase):
    
    def test_end_to_end_workflow_time(self):
        """Measure complete submission-to-approval workflow"""
        start_time = time.time()
        
        # 1. Create submission (simulate Kobo sync)
        submission = create_test_submission()
        
        # 2. Validator accesses queue
        self.client.login(username='validator', password='test')
        self.client.get('/dashboard/validation/queue/')
        
        # 3. Validator opens workspace
        self.client.get(f'/dashboard/validation/workspace/{submission.id}/')
        
        # 4. Validator approves
        self.client.post('/dashboard/api/validate/', {
            'submission_id': submission.id,
            'action': 'approve',
            'remarks': 'Approved'
        })
        
        # 5. PA Staff sees approval
        self.client.logout()
        self.client.login(username='pa_staff', password='test')
        self.client.get(f'/dashboard/submission/{submission.id}/')
        
        total_time = time.time() - start_time
        
        # Target: Complete workflow in <5 seconds (excluding human decision time)
        self.assertLess(total_time, 5.0, 
            f"End-to-end workflow time: {total_time:.2f}s exceeds target")
```

### Qualitative Assessment

**Integration Smoothness Checklist:**

- [ ] **Data Consistency**: Data remains consistent across all integration points
- [ ] **State Synchronization**: UI reflects database state immediately
- [ ] **Error Propagation**: Errors are caught and handled at each layer
- [ ] **Transaction Integrity**: Database transactions are atomic (all-or-nothing)
- [ ] **User Experience Continuity**: Smooth transitions between features
- [ ] **Performance**: No noticeable lag at integration boundaries

---

## 🛠️ Common Integration Issues & Solutions

### Issue 1: Celery Task Not Executing

**Symptoms:**
- Reports stuck in "PENDING" status
- Kobo sync doesn't import data

**UTAUT Impact:**
- **PE**: System doesn't deliver promised functionality
- **FC**: Facilitating conditions not met (system unreliable)

**Solution:**
```bash
# Check Celery worker is running
docker-compose logs celery

# Restart Celery
docker-compose restart celery

# Run task manually for debugging
docker-compose exec web python manage.py shell
>>> from dashboard.tasks import generate_report_task
>>> generate_report_task.delay(report_id=1)
```

**Test Fix:**
```python
def test_celery_integration():
    """Verify Celery is processing tasks"""
    from dashboard.tasks import test_task
    
    result = test_task.apply_async()
    result.get(timeout=10)  # Wait for task to complete
    
    assert result.successful()
    print("✅ Celery integration works")
```

---

### Issue 2: GeoJSON Not Displaying on Map

**Symptoms:**
- Map loads but no data appears
- Console errors about invalid GeoJSON

**UTAUT Impact:**
- **PE**: Map doesn't provide spatial insights
- **EE**: Users confused by empty map

**Solution:**
```python
# Check GeoJSON format
response = self.client.get('/dashboard/api/observations-geojson/')
data = response.json()

# Validate GeoJSON structure
assert data['type'] == 'FeatureCollection'
for feature in data['features']:
    assert 'geometry' in feature
    assert 'type' in feature['geometry']
    assert 'coordinates' in feature['geometry']
    
    # Verify coordinates are [longitude, latitude] (not [lat, lon])
    coords = feature['geometry']['coordinates']
    assert -180 <= coords[0] <= 180  # Longitude
    assert -90 <= coords[1] <= 90    # Latitude
```

**Common Fix:**
```python
# In api_geo.py, ensure correct coordinate order:
{
    'type': 'Feature',
    'geometry': {
        'type': 'Point',
        'coordinates': [observation.location.x, observation.location.y]  # [lon, lat]
    },
    'properties': {...}
}
```

---

### Issue 3: Permission Denied Across Features

**Symptoms:**
- User can access one feature but not related feature
- 403 errors when navigating between views

**UTAUT Impact:**
- **EE**: Confusing user experience
- **SI**: System feels inconsistent
- **FC**: Workflow is blocked

**Solution:**
```python
# Test permission consistency
def test_permission_consistency(self):
    """Verify permissions are consistent across related views"""
    self.client.login(username='pa_staff', password='test')
    
    # PA Staff can view submissions
    response = self.client.get('/dashboard/my-submissions/')
    self.assertEqual(response.status_code, 200)
    
    # PA Staff can view submission detail
    submission = create_test_submission(submitted_by=self.pa_staff_user)
    response = self.client.get(f'/dashboard/submission/{submission.id}/')
    self.assertEqual(response.status_code, 200)
    
    # PA Staff can view their observations
    response = self.client.get('/dashboard/my-observations/')
    self.assertEqual(response.status_code, 200)
    
    # PA Staff CANNOT access validation queue
    response = self.client.get('/dashboard/validation/queue/')
    self.assertEqual(response.status_code, 403)
```

---

## ✅ Integration Testing Checklist

### Pre-Test Setup
- [ ] Docker containers are running
- [ ] Database is migrated
- [ ] Celery worker is active
- [ ] Redis is running
- [ ] Test data is created

### Critical Integration Tests
- [ ] IT-01: KoboToolbox → DataSubmission sync
- [ ] IT-02: Submission → Validation → Approval flow
- [ ] IT-03: Species ↔ Observations linking
- [ ] IT-04: GIS Map ↔ Spatial data display
- [ ] IT-05: Report generation end-to-end

### UTAUT Evaluation
- [ ] Performance Expectancy verified at each integration point
- [ ] Effort Expectancy evaluated for workflows
- [ ] Social Influence factors checked (roles, accountability)
- [ ] Facilitating Conditions confirmed (error handling, reliability)

### Performance Benchmarks
- [ ] Page load times <2s
- [ ] API response times <1s
- [ ] Background tasks complete within reasonable time
- [ ] No memory leaks or resource exhaustion

---

## 📈 Continuous Integration Testing

### Automated Testing in CI/CD

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      
      - name: Run integration tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/test_db
          REDIS_URL: redis://localhost:6379/0
        run: |
          python manage.py test dashboard.tests.test_integration --verbosity=2
      
      - name: Generate coverage report
        run: |
          coverage run --source='.' manage.py test dashboard.tests.test_integration
          coverage report
          coverage html
      
      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: htmlcov/
```

---

## 🎯 Success Criteria Summary

### Functional Success
- ✅ All integration tests pass
- ✅ No data loss at integration points
- ✅ Workflows complete end-to-end
- ✅ Error handling prevents system failures

### UTAUT Success
- ✅ **PE**: Integrations deliver promised value (time savings, insights)
- ✅ **EE**: Transitions between features are smooth and intuitive
- ✅ **SI**: Roles and accountability maintained across features
- ✅ **FC**: System is reliable and recovers from errors

### Performance Success
- ✅ End-to-end workflows complete in <5 seconds
- ✅ API integrations respond in <1 second
- ✅ Background tasks don't block user interactions
- ✅ System handles concurrent users without degradation

---

*Document Version: 1.0*  
*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Testing Documentation*
