# GIS & Spatial Features Documentation

## 🗺️ Overview

The PRISM-Matutum GIS & Spatial Features module provides comprehensive geospatial functionality for managing protected areas, transect routes, monitoring locations, and spatial observations. Built on **PostGIS** (PostgreSQL spatial extension) and **Leaflet.js**, it enables field data visualization, spatial analysis, and geographic export capabilities.

### Core Capabilities
- **Protected Area Management**: Define PA boundaries with polygon geometries
- **Transect Route System**: Create linear transect routes connecting monitoring stations
- **Monitoring Locations**: Manage point locations for field stations and facilities
- **Interactive Mapping**: Multi-layer Leaflet.js visualization with clustering
- **Spatial Assignment**: Auto-link observations to transects/PAs via proximity
- **GeoJSON API**: RFC 7946-compliant endpoints for map data
- **Shapefile Export**: ESRI Shapefile generation with ZIP packaging

---

## 🏗️ Architecture

### Spatial Data Stack
```
Frontend: Leaflet.js 1.9.4 + Clustering + Heatmaps
    ↓
GeoJSON API (api_geo.py) - RFC 7946 compliant
    ↓
Django GIS Views (views_gis.py) - 12 CBVs
    ↓
PostGIS Database (PostgreSQL 16 + PostGIS extension)
    ↓
Spatial Models: ProtectedArea, TransectRoute, MonitoringLocation, TransectSegment
```

### Geometry Types
| Model | Geometry Field | SRID | Type |
|-------|---------------|------|------|
| `ProtectedArea.boundary` | `MultiPolygonField` | 4326 | Area boundaries |
| `TransectRoute.route_geometry` | `LineStringField` | 4326 | Route paths |
| `MonitoringLocation.geometry` | `PointField` | 4326 | Station coordinates |
| `TransectSegment.segment_geometry` | `LineStringField` | 4326 | Inter-station segments |

**SRID 4326**: WGS84 latitude/longitude coordinate system (standard for GPS data)

---

## 📊 Models & Relationships

### ProtectedArea Model
**Location**: `users/models.py`

```python
class ProtectedArea(models.Model):
    protecetedAreaName = models.CharField(max_length=255)
    boundary = gis_models.MultiPolygonField(srid=4326, null=True, blank=True)
    area_hectares = models.DecimalField(max_digits=10, decimal_places=2)
    address = models.ForeignKey('Address', ...)
    
    # Reverse relations:
    # - landscape_set (Landscape objects)
    # - transect_routes (TransectRoute objects)
```

**Purpose**: Represents a legally protected conservation area with spatial boundaries.

### TransectRoute Model
**Location**: `api/models/monitoring.py`

```python
class TransectRoute(models.Model):
    name = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    protected_area = models.ForeignKey('users.ProtectedArea', related_name='transect_routes')
    stations = models.ManyToManyField('MonitoringLocation', related_name='transect_routes')
    route_geometry = gis_models.LineStringField(srid=4326, null=True, blank=True)
    is_active = models.BooleanField(default=True)
    
    def update_geometry_from_stations(self):
        """Generate LineString connecting stations in sequence"""
        coords = [(loc.geometry.x, loc.geometry.y) 
                  for loc in self.stations.all().order_by('name') if loc.geometry]
        if len(coords) >= 2:
            self.route_geometry = LineString(coords, srid=4326)
            self.save()
```

**Purpose**: Linear transect paths for systematic biodiversity monitoring.

### MonitoringLocation Model
**Location**: `api/models/monitoring.py`

```python
class MonitoringLocation(models.Model):
    LOCATION_TYPES = [
        ('facility', 'Facility/Station'),
        ('transect_station', 'Transect Station'),
        ('other', 'Other'),
    ]
    
    name = models.CharField(max_length=255)
    geometry = gis_models.PointField(srid=4326, null=True, blank=True)
    location_type = models.CharField(max_length=50, choices=LOCATION_TYPES)
    description = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)
    
    # Reverse relations:
    # - transect_routes (ManyToMany)
    # - landscape_monitorings (LandscapeMonitoring objects)
```

**Purpose**: Point locations for field stations, monitoring facilities, and transect waypoints.

### TransectSegment Model
**Location**: `api/models/transect_segments.py`

```python
class TransectSegment(models.Model):
    transect_route = models.ForeignKey('TransectRoute', related_name='segments')
    start_station = models.ForeignKey('MonitoringLocation', related_name='segments_start')
    end_station = models.ForeignKey('MonitoringLocation', related_name='segments_end')
    segment_geometry = gis_models.LineStringField(srid=4326, null=True, blank=True)
    segment_name = models.CharField(max_length=255)
    sequence_order = models.IntegerField()
    length_meters = models.FloatField(null=True, blank=True)
    is_active = models.BooleanField(default=True)
```

**Purpose**: Auto-generated line segments between consecutive transect stations. Used for spatial assignment of observations.

**Auto-Assignment Logic**: When observations with GPS coordinates are saved, the system:
1. Finds the nearest `TransectSegment` within 50m buffer
2. Assigns `observation.transect_segment` and `observation.protected_area`
3. Uses PostGIS spatial indexing for performance

---

## 🎯 Views & URLs

### Protected Area Views

#### `ProtectedAreaListView`
**URL**: `/validator-dashboard/protected-areas/`  
**Template**: `dashboard/protected_area_list.html`  
**Permission**: `ValidatorAccessMixin`

**Features**:
- Paginated list of all PAs (20/page)
- Summary cards: Total PAs, Landscapes, Transect Routes
- Address display with barangay/municipality/province
- Per-PA statistics: landscape count, transect count

**Context Data**:
```python
{
    'protected_areas': QuerySet[ProtectedArea],
    'total_pas': int,
    'total_landscapes': int,
    'total_transect_routes': int,
    'pa_stats': [{'pa': ProtectedArea, 'landscape_count': int, 'transect_count': int, 'address_display': str}]
}
```

#### `ProtectedAreaDetailView`
**URL**: `/validator-dashboard/protected-areas/<int:pk>/`  
**Template**: `dashboard/protected_area_detail.html`  
**Permission**: `ValidatorAccessMixin`

**Features**:
- Full PA information with address hierarchy
- Associated landscapes list
- Transect routes with stats (stations, segments, observations)
- Monitoring locations within PA
- Recent observations (last 10)
- Interactive Leaflet map

**Statistics Provided**:
```python
{
    'total_landscapes': int,
    'total_transect_routes': int,
    'active_transect_routes': int,
    'total_monitoring_locations': int,
    'total_transect_segments': int,
    'total_wildlife_observations': int,
    'unique_species_observed': int,
    'recent_observations_30_days': int,
    'resource_use_incidents': int,
    'disturbance_records': int,
    'landscape_monitoring_records': int
}
```

#### `ProtectedAreaUpdateView`
**URL**: `/validator-dashboard/protected-areas/<int:pk>/edit/`  
**Template**: `dashboard/protected_area_form.html`  
**Permission**: `ValidatorAccessMixin`

**Editable Fields**: `protecetedAreaName` (note: typo preserved from legacy database)

---

### Transect Route Views

#### `TransectRouteListView`
**URL**: `/validator-dashboard/transect-routes/`  
**Template**: `dashboard/transect_route_list.html`  
**Permission**: `StaffAndValidatorAccessMixin`

**Features**:
- Filterable by: `protected_area`, `is_active`, `search` (name)
- Route statistics: stations, segments, observations
- Interactive map showing all route geometries
- Export route data to Shapefile/GeoJSON

**Map Data Structure** (JSON):
```javascript
{
  id: int,
  name: string,
  protected_area: string,
  coordinates: [[lat, lon], ...],
  is_active: boolean,
  station_count: int,
  segment_count: int,
  observation_count: int
}
```

#### `TransectRouteDetailView`
**URL**: `/validator-dashboard/transect-routes/<int:pk>/`  
**Template**: `dashboard/transect_route_detail.html`  
**Permission**: `StaffAndValidatorAccessMixin`

**Features**:
- Route metadata and geometry visualization
- Linked monitoring stations list
- Auto-generated segments list
- Observation points mapped by type (wildlife/resource/disturbance/landscape)
- Multi-layer Leaflet map with:
  - Route line (blue polyline)
  - Station markers (clustered)
  - Observation points (colored by type)
  - Heatmap layer toggle

**Filter Controls**:
```python
date_from, date_to, collected_by, transect, species, 
resource_type, disturbance_type, monitoring_location
```

#### `TransectRouteCreateView`
**URL**: `/validator-dashboard/transect-routes/create/`  
**Template**: `dashboard/transect_route_form.html`  
**Permission**: `ValidatorAccessMixin`

**Form Fields**:
```python
fields = ['name', 'description', 'protected_area', 'stations', 'is_active']
```

**Post-Save Actions**:
1. Calls `update_geometry_from_stations()` to generate route line
2. Redirects to detail view
3. Displays success message

---

### Monitoring Location Views

#### `MonitoringLocationListView`
**URL**: `/validator-dashboard/monitoring-locations/`  
**Template**: `dashboard/monitoring_location_list.html`  
**Permission**: `StaffAndValidatorAccessMixin`

**Features**:
- Paginated list (50/page)
- Filter by: `location_type`, `is_active`, `search` (name)
- Summary statistics
- Associated transect routes display

**Location Types**:
- `facility` - Facility/Station
- `transect_station` - Transect Station
- `other` - Other

#### `MonitoringLocationDetailView`
**URL**: `/validator-dashboard/monitoring-locations/<int:pk>/`  
**Template**: `dashboard/monitoring_location_detail.html`  
**Permission**: `StaffAndValidatorAccessMixin`

**Features**:
- Location metadata and coordinates
- Associated transect routes with statistics
- Landscape monitoring records
- Recent observations within proximity
- Point marker on Leaflet map

#### `MonitoringLocationCreateView`
**URL**: `/validator-dashboard/monitoring-locations/create/`  
**Template**: `dashboard/monitoring_location_form.html`  
**Permission**: `ValidatorAccessMixin`

**Form Fields**:
```python
fields = ['name', 'location_type', 'geometry', 'description', 'is_active']
```

**Geometry Input**: Uses `LeafletWidget` for interactive point placement.

---

### Transect Segment Views

#### `TransectSegmentListView`
**URL**: `/validator-dashboard/transect-segments/`  
**Template**: `dashboard/transect_segment_list.html`  
**Permission**: `ValidatorAccessMixin`

**Features**:
- Auto-generated segments display
- Filter by: `transect_route`, `protected_area`, `is_active`
- Segment statistics: length (meters), observation count
- Sequence order display

**Segment Structure**:
```
Segment Name Format: "{Route Name} - Segment {Sequence}"
Example: "Mt. Matutum Trail 1 - Segment 1"
```

#### `TransectSegmentDetailView`
**URL**: `/validator-dashboard/transect-segments/<int:pk>/`  
**Template**: `dashboard/transect_segment_detail.html`  
**Permission**: `ValidatorAccessMixin`

**Features**:
- Segment geometry visualization
- Start/End station details
- Length in meters
- Associated observations list
- Segment line on Leaflet map

---

### Interactive Map View

#### `InteractiveMapView`
**URL**: `/validator-dashboard/map/`  
**Template**: `dashboard/interactive_map.html`  
**Permission**: `StaffAndValidatorAccessMixin`

**Features**: The flagship GIS visualization interface.

**Layer System**:
| Layer | Type | Data Source | Visualization |
|-------|------|------------|---------------|
| Observations | Point (Clustered) | Wildlife observations | Marker clusters (color by type) |
| Resource Use | Point | Resource use incidents | Orange markers |
| Disturbance | Point | Disturbance records | Red markers |
| Landscape | Point | Landscape monitoring | Green markers |
| Transect Routes | LineString | TransectRoute.route_geometry | Blue polylines |
| PA Boundaries | MultiPolygon | ProtectedArea.boundary | Purple polygons |
| Monitoring Locations | Point | MonitoringLocation.geometry | Station icons |
| Heatmap | Density | Observation points | Gradient overlay |

**Basemap Options**:
- **CartoDB Light** (default): Clean, low-contrast base
- **OpenStreetMap**: Standard OSM tiles
- **Satellite**: ESRI World Imagery
- **Dark**: CartoDB Dark Matter

**Filter Controls**:
```javascript
{
  observation_type: ['wildlife', 'resource_use', 'disturbance', 'landscape'],
  date_from: 'YYYY-MM-DD',
  date_to: 'YYYY-MM-DD',
  validation_status: ['pending', 'approved', 'rejected'],
  protected_area: PA_ID,
  species: 'search_term',
  collected_by: USER_ID
}
```

**Map Initialization**:
```javascript
// Default center: Philippines (12.8797, 121.7740)
// Auto-fit bounds to show all data layers
// Clustering: maxClusterRadius = 50px
// Heatmap: radius = 25, blur = 15
```

---

## 🔌 GeoJSON API Endpoints

**Base Path**: `/validator-dashboard/api/geo/`  
**Format**: RFC 7946 (GeoJSON Specification)  
**Authentication**: `@login_required` + `@validator_required`  
**Rate Limit**: Applied via `@api_rate_limit` decorator

### `api_protected_areas_geojson`
**Endpoint**: `GET /validator-dashboard/api/geo/protected-areas/`

**Query Parameters**:
- `pa_id` (int): Filter by specific PA
- `active_only` (bool, default: `true`)

**Response**:
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "MultiPolygon",
        "coordinates": [[[[lon, lat], ...]]]
      },
      "properties": {
        "id": 1,
        "name": "Mount Matutum Protected Landscape",
        "area_hectares": 14000.00,
        "address": {
          "province": "South Cotabato",
          "municipality": "Tupi",
          "barangay": "Kablon"
        }
      }
    }
  ]
}
```

### `api_transect_routes_geojson`
**Endpoint**: `GET /validator-dashboard/api/geo/transect-routes/`

**Query Parameters**:
- `pa_id` (int): Filter by protected area
- `active_only` (bool, default: `true`)
- `include_stations` (bool, default: `false`): Include station details

**Response**:
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "LineString",
        "coordinates": [[lon, lat], [lon, lat], ...]
      },
      "properties": {
        "id": 1,
        "name": "Transect Route 1",
        "description": "Primary monitoring transect",
        "protected_area": {"id": 1, "name": "Mt. Matutum"},
        "is_active": true,
        "station_count": 5,
        "stations": [  // Only if include_stations=true
          {"id": 1, "name": "Station A", "type": "transect_station"}
        ]
      }
    }
  ]
}
```

### `api_monitoring_locations_geojson`
**Endpoint**: `GET /validator-dashboard/api/geo/monitoring-locations/`

**Query Parameters**:
- `location_type` (`facility` | `transect_station` | `other`)
- `active_only` (bool, default: `true`)
- `pa_id` (int): Filter by PA (via transect routes)

**Response**:
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [lon, lat]
      },
      "properties": {
        "id": 1,
        "name": "Station Alpha",
        "type": "transect_station",
        "type_display": "Transect Station",
        "description": "Main monitoring station",
        "is_active": true,
        "transect_routes": [
          {"id": 1, "name": "Route 1"}
        ]
      }
    }
  ]
}
```

### `api_transect_segments_geojson`
**Endpoint**: `GET /validator-dashboard/api/geo/transect-segments/`

**Query Parameters**:
- `route_id` (int): Filter by transect route
- `pa_id` (int): Filter by protected area
- `active_only` (bool, default: `true`)

**Response**:
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "LineString",
        "coordinates": [[lon, lat], [lon, lat]]
      },
      "properties": {
        "id": 1,
        "name": "Route 1 - Segment 1",
        "route_id": 1,
        "route_name": "Transect Route 1",
        "sequence_order": 1,
        "length_meters": 250.5,
        "start_station": {"id": 1, "name": "Station A"},
        "end_station": {"id": 2, "name": "Station B"},
        "is_active": true
      }
    }
  ]
}
```

### `api_observations_geojson`
**Endpoint**: `GET /validator-dashboard/api/geo/observations/`

**Query Parameters**:
- `observation_type` (`wildlife` | `resource_use` | `disturbance` | `landscape`)
- `validation_status` (`pending` | `approved` | `rejected`)
- `date_from` (YYYY-MM-DD)
- `date_to` (YYYY-MM-DD)
- `pa_id` (int)
- `route_id` (int)

**Response**: FeatureCollection of Point geometries with observation details.

### `api_map_statistics`
**Endpoint**: `GET /validator-dashboard/api/geo/map-statistics/`

**Response**:
```json
{
  "total_observations": 1250,
  "wildlife_count": 800,
  "resource_use_count": 200,
  "disturbance_count": 150,
  "landscape_count": 100,
  "total_transect_routes": 15,
  "active_routes": 12,
  "total_monitoring_locations": 45,
  "total_protected_areas": 3,
  "recent_observations_7_days": 85
}
```

### `api_export_map_data`
**Endpoint**: `POST /validator-dashboard/api/geo/export/`

**Request Body**:
```json
{
  "layers": ["observations", "routes", "locations", "boundaries"],
  "format": "shapefile",  // or "geojson"
  "filters": {
    "date_from": "2024-01-01",
    "pa_id": 1
  }
}
```

**Response**: Triggers async Celery task, returns task ID for progress tracking.

---

## 🛠️ Utility Functions

### Shapefile Export
**File**: `dashboard/utils_geo.py`

#### `export_shapefile(queryset, filename_prefix="observations")`

Exports Django QuerySet to ESRI Shapefile format (ZIP archive).

**Parameters**:
- `queryset`: Django QuerySet with spatial data (must have `location` field)
- `filename_prefix`: Base name for output files

**Returns**: Path to ZIP file containing `.shp`, `.shx`, `.dbf`, `.prj`, `.cpg`

**Schema**:
```python
{
    'geometry': 'Point',
    'properties': {
        'id': 'int',
        'species': 'str:254',
        'date': 'date',
        'observer': 'str:100',
        'status': 'str:50',
        'habitat': 'str:100',
        'behavior': 'str:100',
        'count': 'int',
        'latitude': 'float',
        'longitude': 'float',
    }
}
```

**CRS**: WGS84 (EPSG:4326)

**Example**:
```python
from dashboard.utils_geo import export_shapefile
from api.models import WildlifeObservation

approved_observations = WildlifeObservation.objects.filter(
    validation_status='approved',
    latitude__isnull=False
)

zip_path = export_shapefile(approved_observations, filename_prefix="wildlife_approved")
# Returns: /tmp/tmpXXXX/wildlife_approved_shapefile.zip
```

**Dependencies**: `fiona`, `shapely`

#### Helper Functions
```python
_safe_str(value, max_length=254) -> str
    # Safely convert to string with length limit

_format_date(value) -> Optional[str]
    # Format date/datetime to ISO 8601 (YYYY-MM-DD)
```

---

## 📍 Frontend Integration

### Leaflet.js Configuration

**CDN**: `https://unpkg.com/leaflet@1.9.4/dist/leaflet.js`

**Plugins**:
- **MarkerCluster**: `leaflet.markercluster@1.4.1` - Cluster large point datasets
- **Heatmap**: `leaflet.heat@0.2.0` - Density visualization
- **GeoJSON**: Built-in L.geoJSON() support

**Standard Map Setup**:
```javascript
const map = L.map('map').setView([12.8797, 121.7740], 10);

L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {
    attribution: '© OpenStreetMap © CARTO',
    maxZoom: 19
}).addTo(map);
```

**Loading GeoJSON Data**:
```javascript
fetch('/validator-dashboard/api/geo/transect-routes/?pa_id=1')
    .then(response => response.json())
    .then(geojson => {
        L.geoJSON(geojson, {
            style: feature => ({
                color: feature.properties.is_active ? 'blue' : 'gray',
                weight: 3
            }),
            onEachFeature: (feature, layer) => {
                layer.bindPopup(`<strong>${feature.properties.name}</strong>`);
            }
        }).addTo(map);
    });
```

**Marker Clustering**:
```javascript
const markers = L.markerClusterGroup({
    maxClusterRadius: 50,
    spiderfyOnMaxZoom: true,
    showCoverageOnHover: true
});

observations.forEach(obs => {
    const marker = L.marker([obs.lat, obs.lon]);
    marker.bindPopup(obs.species);
    markers.addLayer(marker);
});

map.addLayer(markers);
```

**Heatmap Layer**:
```javascript
const heatData = observations.map(obs => [obs.lat, obs.lon, obs.intensity]);

L.heatLayer(heatData, {
    radius: 25,
    blur: 15,
    maxZoom: 17,
    gradient: {0.0: 'blue', 0.5: 'lime', 0.7: 'yellow', 1.0: 'red'}
}).addTo(map);
```

---

## 🔐 Security & Permissions

### Access Control Matrix
| View | Permission Mixin | Roles Allowed |
|------|-----------------|---------------|
| `ProtectedAreaListView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `ProtectedAreaDetailView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `ProtectedAreaUpdateView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `MonitoringLocationListView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |
| `MonitoringLocationDetailView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |
| `MonitoringLocationCreateView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `TransectRouteListView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |
| `TransectRouteDetailView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |
| `TransectRouteCreateView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `TransectSegmentListView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `TransectSegmentDetailView` | `ValidatorAccessMixin` | validator, denr, system_admin |
| `InteractiveMapView` | `StaffAndValidatorAccessMixin` | pa_staff, validator, denr, system_admin |

**API Endpoints**: All GeoJSON endpoints require `@login_required` + `@validator_required` decorators.

**Data Visibility**:
- **pa_staff**: Can view PAs/routes assigned to their organization
- **validator/denr**: Can view all PAs/routes
- **system_admin**: Full access + can edit geometries

---

## 🧪 Testing Scenarios

### 1. Protected Area Boundary Management
```python
# Create PA with boundary
pa = ProtectedArea.objects.create(
    protecetedAreaName="Test PA",
    boundary=MultiPolygon([Polygon([(125.0, 6.0), (125.1, 6.0), (125.1, 6.1), (125.0, 6.1), (125.0, 6.0)])]),
    area_hectares=1000.00
)

# Verify GeoJSON API
response = client.get('/validator-dashboard/api/geo/protected-areas/', {'pa_id': pa.id})
assert response.json()['features'][0]['properties']['name'] == "Test PA"
assert response.json()['features'][0]['geometry']['type'] == "MultiPolygon"
```

### 2. Transect Route Geometry Generation
```python
# Create route with stations
route = TransectRoute.objects.create(name="Route 1", protected_area=pa)
station_a = MonitoringLocation.objects.create(name="A", geometry=Point(125.0, 6.0))
station_b = MonitoringLocation.objects.create(name="B", geometry=Point(125.05, 6.05))
route.stations.add(station_a, station_b)

# Generate geometry
route.update_geometry_from_stations()
route.refresh_from_db()

assert route.route_geometry is not None
assert route.route_geometry.geom_type == "LineString"
assert len(route.route_geometry.coords) == 2
```

### 3. Transect Segment Auto-Generation
```python
# After route creation, segments should be auto-generated
segments = TransectSegment.objects.filter(transect_route=route)
assert segments.count() == 1  # One segment between 2 stations
segment = segments.first()
assert segment.start_station == station_a
assert segment.end_station == station_b
assert segment.segment_name == "Route 1 - Segment 1"
```

### 4. Observation Spatial Assignment
```python
# Create observation near segment
observation = WildlifeObservation.objects.create(
    latitude=6.025,  # Midpoint between A and B
    longitude=125.025,
    species="Tarsius syrichta"
)

# Auto-assignment should occur on save
assert observation.transect_segment == segment
assert observation.protected_area == pa
```

### 5. GeoJSON API Filtering
```python
# Test active_only filter
inactive_route = TransectRoute.objects.create(name="Route 2", is_active=False)
response = client.get('/validator-dashboard/api/geo/transect-routes/', {'active_only': 'true'})
feature_names = [f['properties']['name'] for f in response.json()['features']]
assert "Route 1" in feature_names
assert "Route 2" not in feature_names
```

### 6. Interactive Map Layer Loading
```python
# Test map view context
response = client.get('/validator-dashboard/map/')
assert 'transect_routes' in response.context
assert 'monitoring_locations' in response.context
assert 'pa_boundaries' in response.context

# Verify JSON serialization
import json
routes_json = json.loads(response.context['transect_routes'])
assert isinstance(routes_json, list)
```

### 7. Shapefile Export
```python
from dashboard.utils_geo import export_shapefile

observations = WildlifeObservation.objects.filter(latitude__isnull=False)[:10]
zip_path = export_shapefile(observations, filename_prefix="test_export")

assert os.path.exists(zip_path)
assert zipfile.is_zipfile(zip_path)

with zipfile.ZipFile(zip_path, 'r') as zf:
    assert 'test_export.shp' in zf.namelist()
    assert 'test_export.dbf' in zf.namelist()
    assert 'test_export.prj' in zf.namelist()
```

### 8. Monitoring Location Types
```python
# Test location type filtering
facility = MonitoringLocation.objects.create(
    name="HQ", location_type='facility', geometry=Point(125.0, 6.0)
)
station = MonitoringLocation.objects.create(
    name="S1", location_type='transect_station', geometry=Point(125.1, 6.1)
)

response = client.get('/validator-dashboard/api/geo/monitoring-locations/', 
                       {'location_type': 'transect_station'})
locations = response.json()['features']
assert len(locations) == 1
assert locations[0]['properties']['type'] == 'transect_station'
```

### 9. Spatial Query Performance
```python
# Test PostGIS spatial indexing
import time
from django.contrib.gis.db.models.functions import Distance

start = time.time()
nearby_segments = TransectSegment.objects.filter(
    segment_geometry__distance_lte=(Point(125.0, 6.0), 50)  # 50m buffer
).annotate(distance=Distance('segment_geometry', Point(125.0, 6.0)))
list(nearby_segments)  # Force query execution
duration = time.time() - start

assert duration < 0.5  # Should complete in <500ms with spatial index
```

### 10. Map Bounds Auto-Fit
```python
# Test frontend auto-fit logic
response = client.get('/validator-dashboard/transect-routes/')
routes_json = response.context['routes_for_map']

# Verify coordinates are in [lat, lon] format for Leaflet
routes = json.loads(routes_json)
if routes:
    assert isinstance(routes[0]['coordinates'], list)
    assert len(routes[0]['coordinates'][0]) == 2  # [lat, lon]
```

---

## 🚀 Performance Optimization

### PostGIS Indexing
```sql
-- Spatial indexes (automatically created by Django GIS)
CREATE INDEX idx_protectedarea_boundary ON users_protectedarea USING GIST(boundary);
CREATE INDEX idx_transectroute_geometry ON api_transectroute USING GIST(route_geometry);
CREATE INDEX idx_monitoringlocation_geometry ON api_monitoringlocation USING GIST(geometry);
CREATE INDEX idx_transectsegment_geometry ON api_transectsegment USING GIST(segment_geometry);
```

### Query Optimization Patterns

**Bad** (N+1 queries):
```python
routes = TransectRoute.objects.all()
for route in routes:
    print(route.protected_area.name)  # Queries PA each time
    print(route.stations.count())      # Queries stations each time
```

**Good** (Eager loading):
```python
routes = TransectRoute.objects.select_related('protected_area') \
                               .prefetch_related('stations', 'segments')
for route in routes:
    print(route.protected_area.name)  # No additional query
    print(route.stations.count())      # No additional query
```

### GeoJSON Response Caching
```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
@login_required
@validator_required
def api_protected_areas_geojson(request):
    # ... implementation
```

### Marker Clustering for Large Datasets
```javascript
// Frontend optimization: Use clustering for >100 markers
if (observations.length > 100) {
    const markers = L.markerClusterGroup({maxClusterRadius: 50});
    // Add markers to cluster group
} else {
    const markers = L.featureGroup();
    // Add markers directly
}
```

---

## 🔗 Integration with Other Modules

### Wildlife Observation Module
- Observations automatically linked to `TransectSegment` via GPS proximity
- Map visualization uses `WildlifeObservation.latitude/longitude`
- Species filters apply to map layers

### Validation System
- Validation status filters available on map view
- Approved observations shown in distinct colors
- Validation workflow affects map data visibility

### KoboToolbox Sync
- GPS coordinates from Kobo forms populate `latitude/longitude` fields
- Spatial assignment occurs during submission processing
- Transect/PA fields auto-populated after validation

### Report Generation
- Shapefile exports include spatial metadata
- GeoJSON exports available for external GIS software
- Map screenshots included in PDF reports

---

## 📚 Related Documentation

- **Database Models**: [docs/DatabaseModel.md](../DatabaseModel.md) - Full schema definitions
- **API Specification**: [docs/API.md](../API.md) - REST API endpoints
- **Choice System**: [docs/CHOICE_SYSTEM_README.md](../CHOICE_SYSTEM_README.md) - Dynamic choice lists
- **Transect Segments**: [api/models/TRANSECT_SEGMENTS_README.md](../../api/models/TRANSECT_SEGMENTS_README.md) - Segment auto-generation logic

---

## 🎓 Key Takeaways

1. **PostGIS Foundation**: All spatial operations leverage PostgreSQL's PostGIS extension for performant geospatial queries.

2. **GeoJSON Standard**: API responses follow RFC 7946 for interoperability with external GIS tools (QGIS, ArcGIS).

3. **Auto-Assignment**: Observations are spatially linked to transects/PAs automatically using 50m buffer proximity.

4. **Multi-Layer Mapping**: Interactive map supports 8+ layers with dynamic filtering and clustering.

5. **Shapefile Export**: ESRI Shapefile generation enables offline analysis and data sharing.

6. **SRID 4326**: All geometries use WGS84 coordinate system (standard GPS format).

7. **Leaflet.js Integration**: Lightweight, mobile-friendly mapping library with extensive plugin ecosystem.

---

**Documentation Version**: 1.0  
**Last Updated**: 2024  
**Module**: Dashboard GIS & Spatial Features  
**Related Files**:
- `dashboard/views_gis.py` (1495 lines)
- `dashboard/api_geo.py` (584 lines)
- `dashboard/utils_geo.py` (185 lines)
- `api/models/monitoring.py`
- `api/models/transect_segments.py`
