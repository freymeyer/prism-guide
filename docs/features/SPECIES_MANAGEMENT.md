# Species Management Documentation

## 📋 Overview

The Species Management system in PRISM-Matutum provides comprehensive tools for managing biodiversity data, including master species checklists (fauna and flora), taxonomic classifications, conservation status, and linkages to field observations.

### Key Features

- **Dual Checklists**: Separate management for Fauna and Flora species
- **Full Taxonomy**: Kingdom → Phylum → Class → Order → Family → Genus hierarchy
- **Conservation Tracking**: IUCN threat categories and endemicity status
- **Common Names**: Multiple local/common names per species
- **Media Gallery**: Photo/audio attachments per species
- **Research Articles**: Linked scientific publications
- **Observation History**: Complete record of field sightings
- **Auto-Classification**: Automatic species type assignment based on taxonomy

---

## 🏗️ Architecture

### Core Components

```
species/models.py
├── Taxonomy Models         # Kingdom, Phylum, Class, Order, Family, Genus, Taxonomy
├── Status Models           # Endemicity, ThreatAssessment
├── Characteristics         # FaunalCharacteristics, FloralCharacteristics
├── Main Checklists         # FaunaChecklist, FloraChecklist
├── Common Names            # FaunaCommonName, FloraCommonName
├── Research & Articles     # RelatedResearch, Articles
└── Media                   # FieldMedia, FaunaFieldMedia, FloraFieldMedia

dashboard/views_species.py
├── SpeciesListView         # Browse species with filters
├── SpeciesDetailView       # Comprehensive species information
├── SpeciesCreateView       # Add new species (Admin only)
├── SpeciesUpdateView       # Edit species (Admin only)
├── SpeciesDeleteView       # Remove species (Admin only)
├── SpeciesObservationsView # View all observations for a species
└── SpeciesMediaView        # Manage species media

dashboard/views_articles.py
├── FaunaArticleCreateView  # Add articles to fauna species
├── FloraArticleCreateView  # Add articles to flora species
└── Article Update/Delete   # Article management
```

### Data Relationships

```
FaunaChecklist
    ├── id_taxonomy → Taxonomy
    ├── id_endemicity → Endemicity
    ├── id_threat_category → ThreatAssessment
    ├── id_faunal_characteristics → FaunalCharacteristics
    ├── common_names → FaunaCommonName → SpeciesCommonName
    ├── research → RelatedFaunaResearch → RelatedResearch
    ├── articles → FaunaRelatedArticle → Articles
    ├── media → FaunaFieldMedia → FieldMedia
    └── field_observations → WildlifeObservation (reverse FK)

FloraChecklist
    ├── id_taxonomy → Taxonomy
    ├── id_endemicity → Endemicity
    ├── id_threat_category → ThreatAssessment
    ├── id_floral_characteristics → FloralCharacteristics
    ├── common_names → FloraCommonName → SpeciesCommonName
    ├── research → RelatedFloraResearch → RelatedResearch
    ├── articles → FloraRelatedArticle → Articles
    ├── media → FloraFieldMedia → FieldMedia
    └── field_observations → WildlifeObservation (reverse FK)
```

---

## 📦 Models

### 1. FaunaChecklist Model

**Table**: `fauna_checklist`

**Purpose**: Master checklist of all fauna (animal) species

```python
class FaunaChecklist(models.Model):
    id_fauna_checklist = AutoField(primary_key=True)
    scientific_name = CharField(max_length=255, unique=True, indexed=True)
    id_taxonomy = ForeignKey(Taxonomy)
    id_species_type = ForeignKey(SpeciesType)  # Auto-assigned
    id_endemicity = ForeignKey(Endemicity)
    id_faunal_characteristics = ForeignKey(FaunalCharacteristics)
    id_threat_category = ForeignKey(ThreatAssessment)
    total_recorded_population = IntegerField(null=True)
    remarks = TextField(null=True)
    created_at = DateTimeField(auto_now_add=True)
    edited_at = DateTimeField(auto_now=True)
```

**Auto-Classification Logic**:

```python
def save(self, *args, **kwargs):
    # Auto-assign species type based on taxonomic class
    tax_class = self.id_taxonomy.id_class.taxon_name.lower()
    
    mapping = {
        'aves': 'Bird',
        'mammalia': 'Mammal',
        'reptilia': 'Reptile',
        'amphibia': 'Amphibian',
        'insecta': 'Arthropod',
        'arachnida': 'Arthropod',
        'crustacea': 'Arthropod',
    }
    
    matched_type = mapping.get(tax_class, 'Other')
    self.id_species_type, _ = SpeciesType.objects.get_or_create(type=matched_type)
    
    super().save(*args, **kwargs)
```

**Key Methods**:
- `get_observations()` - Returns all `WildlifeObservation` records linked to this species

**Relationships**:
- `common_names` - Many-to-Many through `FaunaCommonName`
- `research` - Many-to-Many through `RelatedFaunaResearch`
- `articles` - Many-to-Many through `FaunaRelatedArticle`
- `media` - Many-to-Many through `FaunaFieldMedia`
- `field_observations` - Reverse FK from `WildlifeObservation`

### 2. FloraChecklist Model

**Table**: `flora_checklist`

**Purpose**: Master checklist of all flora (plant) species

```python
class FloraChecklist(models.Model):
    id_flora_checklist = AutoField(primary_key=True)
    scientific_name = CharField(max_length=255, unique=True, indexed=True)
    id_taxonomy = ForeignKey(Taxonomy)
    id_species_type = ForeignKey(SpeciesType)  # Auto-assigned as 'Plant'
    id_endemicity = ForeignKey(Endemicity)
    id_floral_characteristics = ForeignKey(FloralCharacteristics)
    id_threat_category = ForeignKey(ThreatAssessment)
    total_recorded_population = IntegerField(null=True)
    remarks = TextField(null=True)
    created_at = DateTimeField(auto_now_add=True)
    edited_at = DateTimeField(auto_now=True)
```

**Auto-Classification Logic**:

```python
def save(self, *args, **kwargs):
    kingdom = self.id_taxonomy.id_kingdom.taxon_name.lower()
    if kingdom == 'plantae':
        self.id_species_type, _ = SpeciesType.objects.get_or_create(type='Plant')
    else:
        self.id_species_type, _ = SpeciesType.objects.get_or_create(type='Other')
    super().save(*args, **kwargs)
```

### 3. Taxonomy Model

**Table**: `taxonomy`

**Purpose**: Complete taxonomic hierarchy for species classification

```python
class Taxonomy(models.Model):
    id_taxonomy = AutoField(primary_key=True)
    id_kingdom = ForeignKey(Kingdom)
    id_phylum = ForeignKey(Phylum)
    id_class = ForeignKey(TaxonomyClass)
    id_order = ForeignKey(TaxonomyOrder)
    id_family = ForeignKey(Family)
    id_genus = ForeignKey(Genus)
```

**Hierarchy**:
```
Kingdom (e.g., Animalia, Plantae)
    ↓
Phylum (e.g., Chordata, Magnoliophyta)
    ↓
Class (e.g., Aves, Mammalia)
    ↓
Order (e.g., Accipitriformes, Primates)
    ↓
Family (e.g., Accipitridae, Hominidae)
    ↓
Genus (e.g., Pithecophaga, Homo)
    ↓
Species (e.g., Pithecophaga jefferyi)
```

### 4. FaunalCharacteristics Model

**Table**: `faunal_characteristics`

**Purpose**: Physical and behavioral characteristics of fauna

```python
class FaunalCharacteristics(models.Model):
    id_faunal_characteristics = AutoField(primary_key=True)
    description = TextField()  # Physical description, behavior notes
    vocalization_link = URLField(null=True, blank=True)  # Link to sound recording
```

### 5. FloralCharacteristics Model

**Table**: `floral_characteristics`

**Purpose**: Morphological characteristics of flora

```python
class FloralCharacteristics(models.Model):
    id_floral_characteristics = AutoField(primary_key=True)
    id_leaf_type = ForeignKey(LeafType, null=True)
    id_leaf_shape = ForeignKey(LeafShape, null=True)
    id_leaf_margin = ForeignKey(LeafMargin, null=True)
    id_leaf_apex = ForeignKey(LeafApex, null=True)
    id_leaf_base = ForeignKey(LeafBase, null=True)
    leaf_color_description = TextField(null=True)
    id_stem_type = ForeignKey(StemType, null=True)
    id_flower_type = ForeignKey(FlowerType, null=True)
    flower_color_description = TextField(null=True)
    id_fruit_type = ForeignKey(FruitType, null=True)
    fruit_color_description = TextField(null=True)
```

### 6. Endemicity Model

**Table**: `endemicity`

**Purpose**: Geographic distribution status

```python
class Endemicity(models.Model):
    id_endemicity = AutoField(primary_key=True)
    endemicity_status = CharField(max_length=255)
```

**Example Values**:
- Endemic to Philippines
- Endemic to Mindanao
- Endemic to Mount Apo
- Native (not endemic)
- Introduced/Exotic

### 7. ThreatAssessment Model

**Table**: `threat_assessment`

**Purpose**: IUCN conservation status

```python
class ThreatAssessment(models.Model):
    id_threat_assessment = AutoField(primary_key=True)
    threat_category_status = CharField(max_length=255)
```

**IUCN Categories**:
- **EX** - Extinct
- **EW** - Extinct in the Wild
- **CR** - Critically Endangered
- **EN** - Endangered
- **VU** - Vulnerable
- **NT** - Near Threatened
- **LC** - Least Concern
- **DD** - Data Deficient
- **NE** - Not Evaluated

### 8. Common Names Models

**Tables**: `fauna_common_name`, `flora_common_name`

**Purpose**: Link species to local/common names

```python
class FaunaCommonName(models.Model):
    id_fauna_common_name = AutoField(primary_key=True)
    id_fauna_checklist = ForeignKey(FaunaChecklist, related_name='common_names')
    id_species_common_name = ForeignKey(SpeciesCommonName)
    
    class Meta:
        unique_together = ['id_fauna_checklist', 'id_species_common_name']

class SpeciesCommonName(models.Model):
    id_species_common_name = AutoField(primary_key=True)
    name = CharField(max_length=255)  # e.g., "Philippine Eagle", "Haring Ibon"
```

### 9. Articles & Research Models

**Purpose**: Link scientific publications to species

```python
class Articles(models.Model):
    id_articles = AutoField(primary_key=True)
    title = CharField(max_length=500)
    abstract = TextField()
    url = URLField()
    type = CharField(max_length=100)  # Journal, Report, Thesis, etc.

class FaunaRelatedArticle(models.Model):
    id_fauna_checklist = ForeignKey(FaunaChecklist, related_name='articles')
    id_articles = ForeignKey(Articles)
```

### 10. Media Models

**Purpose**: Store species photos, audio, video

```python
class FieldMedia(models.Model):
    id_field_media = AutoField(primary_key=True)
    link = URLField()  # URL to media file
    description = TextField(null=True)
    media_type = CharField(max_length=50)  # 'photo', 'audio', 'video'

class FaunaFieldMedia(models.Model):
    id_fauna_checklist = ForeignKey(FaunaChecklist, related_name='media')
    id_field_media = ForeignKey(FieldMedia)
```

---

## 🎯 Views & Features

### Species List View

**URL**: `/species/?tab=fauna` or `/species/?tab=flora`

**Access**: Validators, DENR, Collaborators, Admin

**Purpose**: Browse and filter species checklists

#### Features

**1. Dual Tab Interface**:
- Fauna tab: Lists all animal species
- Flora tab: Lists all plant species
- Switch between tabs preserves filters

**2. Advanced Filtering**:

**Fauna Filters**:
- Species Type (Bird, Mammal, Reptile, Amphibian, Arthropod)
- Conservation Status (CR, EN, VU, NT, LC, etc.)
- Endemicity Status
- Family
- Search by scientific/common name

**Flora Filters**:
- Conservation Status
- Endemicity Status
- Family
- Search by scientific name

**3. Statistics Sidebar**:

```python
statistics = {
    'total_species': 487,
    'by_type': {
        'Bird': 156,
        'Mammal': 89,
        'Reptile': 67,
        'Amphibian': 45,
        'Arthropod': 130
    },
    'by_conservation_status': {
        'Critically Endangered': 23,
        'Endangered': 56,
        'Vulnerable': 78,
        'Near Threatened': 45,
        'Least Concern': 285
    },
    'by_endemicity': {
        'Endemic to Philippines': 234,
        'Endemic to Mindanao': 89,
        'Native': 145,
        'Introduced': 19
    },
    'species_with_observations': 342,
    'total_observations': 12456
}
```

**4. Data Visualizations**:
- Pie Chart: Species by Type
- Pie Chart: Conservation Status Distribution
- Pie Chart: Endemicity Distribution
- Bar Chart: Top 10 Families
- Bar Chart: Top 8 Orders

**5. Species List Display**:

| Column | Data |
|--------|------|
| Scientific Name | *Italicized*, clickable |
| Common Names | Comma-separated |
| Family | Taxonomic family |
| Conservation Status | Color-coded badge |
| Endemicity | Badge |
| Observations | Count with link |
| Actions | View Details, Edit (Admin only) |

**6. Pagination**: 50 species per page

### Species Detail View

**URL**: `/species/<species_type>/<id>/`

**Access**: Validators, DENR, Collaborators, Admin

**Purpose**: Comprehensive species profile

#### Sections

**1. Species Header**:
- Scientific name (*italicized*)
- Common names
- Conservation status badge (color-coded)
- Endemicity badge
- Photo gallery (if available)

**2. Taxonomy Card**:
```
Kingdom: Animalia
Phylum: Chordata
Class: Aves
Order: Accipitriformes
Family: Accipitridae
Genus: Pithecophaga
Species: jefferyi
```

**3. Characteristics**:

**For Fauna**:
- Physical description
- Behavior notes
- Vocalization (audio link if available)

**For Flora**:
- Leaf: Type, Shape, Margin, Apex, Base, Color
- Stem: Type
- Flower: Type, Color description
- Fruit: Type, Color description

**4. Population & Status**:
- Total Recorded Population
- Conservation Status details
- Endemicity information
- IUCN threat category

**5. Observation Summary**:
- Total observations: 47
- First observation: 2020-03-15
- Most recent observation: 2026-01-05
- Top collectors (top 5)

**6. Observation Timeline Chart**:
- Line chart showing monthly observation counts for last 12 months

**7. Observation Map**:
- Interactive Leaflet map
- Markers for each observation location (limited to 50 for performance)
- Popup shows: Date, Collector, Count, Behavior notes

**8. Recent Observations Table**:
- Date/Time
- Location
- Collector
- Individual Count
- Behavior/Notes
- Transect (if applicable)
- View Details link

**9. Research & Articles**:
- Linked scientific publications
- Abstract preview
- External links
- Add Article button (Validators only)

**10. Media Gallery**:
- Photo thumbnails with lightbox
- Audio files (if fauna)
- Upload Media button (Validators only)

### Species Create View

**URL**: `/species/<fauna|flora>/create/`

**Access**: System Admin only

**Purpose**: Add new species to checklist

#### Form Sections

**1. Basic Information**:
- Scientific Name (unique, required)
- Remarks (optional notes)

**2. Taxonomy Selection**:
- Kingdom dropdown
- Phylum dropdown (filtered by Kingdom)
- Class dropdown (filtered by Phylum)
- Order dropdown (filtered by Class)
- Family dropdown (filtered by Order)
- Genus dropdown (filtered by Family)

**3. Status Selection**:
- Conservation Status (IUCN category)
- Endemicity Status

**4. Characteristics**:

**Fauna Form**:
- Physical Description (textarea)
- Vocalization Link (URL, optional)

**Flora Form**:
- Leaf Type, Shape, Margin, Apex, Base (dropdowns)
- Leaf Color Description (textarea)
- Stem Type (dropdown)
- Flower Type (dropdown)
- Flower Color Description (textarea)
- Fruit Type (dropdown)
- Fruit Color Description (textarea)

**5. Population Data**:
- Total Recorded Population (integer, optional)

#### Validation

- Scientific name must be unique across both Fauna and Flora checklists
- All required taxonomy levels must be selected
- Conservation status and endemicity are required

#### Process

```python
with transaction.atomic():
    # 1. Save characteristics
    characteristics = characteristics_form.save()
    
    # 2. Save species with characteristics
    species = species_form.save(commit=False)
    if species_type == 'fauna':
        species.id_faunal_characteristics = characteristics
    else:
        species.id_floral_characteristics = characteristics
    species.save()
    
    # 3. Auto-assign species_type based on taxonomy
    # (handled by model's save() method)
    
    # 4. Create activity log
    UserActivityLog.objects.create(
        user=request.user,
        activity_type='species_management',
        description=f'Created new {species_type} species: {species.scientific_name}'
    )
```

### Species Update View

**URL**: `/species/<species_type>/<id>/edit/`

**Access**: System Admin only

**Purpose**: Edit existing species

**Features**:
- Pre-populated form with existing data
- Update taxonomy, status, characteristics
- Cannot change scientific name if observations exist (safety check)
- Characteristics edited in-place (not new record)

### Species Delete View

**URL**: `/species/<species_type>/<id>/delete/`

**Access**: System Admin only

**Purpose**: Remove species from checklist

**Safety Checks**:
```python
# Check for linked observations
observations_count = species.get_observations().count()

if observations_count > 0:
    messages.error(
        request,
        f"Cannot delete species: {observations_count} observations are linked. "
        "Please delete or reassign observations first."
    )
    return redirect('species_detail', species_type=species_type, pk=pk)

# Proceed with deletion
species.delete()
```

### Species Observations View

**URL**: `/species/<species_type>/<id>/observations/`

**Access**: Validators, DENR, Collaborators, Admin

**Purpose**: View all field observations for a specific species

**Features**:
- Paginated list of all observations (20 per page)
- Filter by:
  - Date range
  - Protected Area
  - Collector (PA Staff)
  - Transect Route
- Sort by:
  - Date (newest/oldest)
  - Location
  - Individual count
- Export to CSV
- Each observation links to full `WildlifeObservation` detail

### Species Media Management

**URL**: `/species/<species_type>/<id>/media/`

**Access**: Validators and Admin only

**Purpose**: Manage photos, audio, and video for species

**Features**:

**1. Upload Media**:
```python
POST /species/<species_type>/<id>/media/
Content-Type: multipart/form-data

{
    'media_file': <file>,
    'media_type': 'photo|audio|video',
    'description': 'Optional description'
}
```

**2. Delete Media**:
- Remove media from species
- Media file remains in FieldMedia table (may be linked to other records)

**3. Gallery View**:
- Grid of thumbnails (photos)
- Audio player (for fauna vocalizations)
- Video embed

### Article Management

**URLs**:
- `/species/fauna/<id>/article/add/` - Add article to fauna species
- `/species/flora/<id>/article/add/` - Add article to flora species
- `/articles/<id>/edit/` - Edit article
- `/articles/<id>/delete/` - Delete article

**Access**: Validators and Admin

**Form Fields**:
- Title (max 500 chars)
- Abstract (textarea)
- URL (link to full article)
- Type (Journal Article, Technical Report, Thesis, Conference Proceeding, Book Chapter)

**Validation**:
- URL must be valid
- Title required

---

## 🔄 Integration with Observations

### Automatic Linkage

When a submission is validated and processed:

```python
# In BMSDataProcessor.process_wildlife_observation()

# Get or create taxonomy
taxonomy = get_or_create_taxonomy(form_data)

# Match species
fauna_species = FaunaChecklist.objects.filter(
    scientific_name__iexact=species_name
).first()

flora_species = FloraChecklist.objects.filter(
    scientific_name__iexact=species_name
).first()

# Create observation with species link
observation = WildlifeObservation.objects.create(
    species=species_name,
    fauna_species=fauna_species,  # Linked FK
    flora_species=flora_species,  # Linked FK
    species_count=count,
    # ... other fields
)
```

### Species Verification

During validation, the system verifies species against checklists:

```python
from dashboard.utils import SpeciesVerifier

verification = SpeciesVerifier.verify_species(submission.form_data)

# Result:
{
    'species_name': 'Pithecophaga jefferyi',
    'match_found': True,
    'species_type': 'fauna',
    'species_id': 123,
    'confidence': 1.0,  # Exact match
    'common_names': ['Philippine Eagle', 'Haring Ibon'],
    'conservation_status': 'Critically Endangered',
    'endemicity': 'Endemic to Philippines'
}
```

---

## 📊 Analytics & Reports

### Species Statistics

Available through dashboard widgets and API endpoints:

```python
from dashboard.views_widgets import get_species_summary_data

data = get_species_summary_data(widget, config)

# Returns:
{
    'total_species': 487,
    'fauna_count': 356,
    'flora_count': 131,
    'endemic_count': 234,
    'threatened_count': 157,  # CR + EN + VU
    'by_type': [
        {'type': 'Bird', 'count': 156},
        {'type': 'Mammal', 'count': 89},
        # ...
    ],
    'recent_additions': [
        {' species': 'Homo sapiens neanderthalensis', 'added': '2026-01-01'},
        # ...
    ]
}
```

### Observation Trends

```python
# Monthly observation counts per species
monthly_data = species.get_observations().annotate(
    month=TruncMonth('submission_time')
).values('month').annotate(
    count=Count('id')
).order_by('month')
```

### Top Observed Species

```python
top_species = FaunaChecklist.objects.annotate(
    obs_count=Count('field_observations')
).order_by('-obs_count')[:20]
```

---

## 🧪 Testing Scenarios

### Scenario 1: Creating a New Fauna Species

**Steps**:
1. Login as System Admin
2. Navigate to Species List → Fauna tab
3. Click "Add New Species"
4. Fill form:
   - Scientific Name: "Tarsius syrichta"
   - Taxonomy: Animalia → Chordata → Mammalia → Primates → Tarsiidae → Tarsius
   - Conservation Status: Near Threatened
   - Endemicity: Endemic to Philippines
   - Description: "Small nocturnal primate with large eyes"
5. Submit

**Expected Results**:
- ✅ Species created successfully
- ✅ Species Type auto-assigned as "Mammal" (from class Mammalia)
- ✅ Appears in species list
- ✅ Can be searched by scientific name
- ✅ Activity log created

### Scenario 2: Adding Common Names

**Steps**:
1. Navigate to species detail for "Tarsius syrichta"
2. Click "Add Common Name"
3. Enter "Philippine Tarsier"
4. Click "Add Common Name" again
5. Enter "Mawmag"
6. Save

**Expected Results**:
- ✅ Both names displayed on species detail page
- ✅ Species searchable by either common name
- ✅ Unique constraint prevents duplicate common name entries

### Scenario 3: Linking Observations

**Steps**:
1. Create field submission with species "Tarsius syrichta"
2. Validate and approve submission
3. Navigate to species detail page

**Expected Results**:
- ✅ Observation count increased by 1
- ✅ Observation appears in "Recent Observations"
- ✅ Observation location plotted on map
- ✅ Species page shows "First Observation" date

### Scenario 4: Uploading Species Media

**Steps**:
1. Navigate to species detail
2. Click "Upload Media"
3. Select photo file
4. Add description: "Adult specimen in natural habitat"
5. Submit

**Expected Results**:
- ✅ Photo appears in species gallery
- ✅ Thumbnail clickable for full-size view
- ✅ Description displayed
- ✅ Can be used in species profile cards

### Scenario 5: Adding Research Article

**Steps**:
1. Navigate to species detail
2. Click "Add Article"
3. Fill form:
   - Title: "Conservation Status of Philippine Tarsier"
   - Abstract: "Comprehensive study of population trends..."
   - URL: "https://journal.com/article-123"
   - Type: Journal Article
4. Submit

**Expected Results**:
- ✅ Article appears in "Research & Articles" section
- ✅ Abstract preview shown
- ✅ External link functional
- ✅ Can be edited/deleted by validators

### Scenario 6: Species Filtering

**Steps**:
1. Navigate to Species List → Fauna
2. Apply filters:
   - Species Type: Bird
   - Conservation Status: Endangered
   - Endemicity: Endemic to Mindanao
3. Search: "Pitta"

**Expected Results**:
- ✅ Results filtered correctly
- ✅ Only matching species displayed
- ✅ Statistics sidebar updates to reflect filtered data
- ✅ Charts regenerate with filtered data
- ✅ Pagination works with filtered results

### Scenario 7: Deleting Species with Observations

**Steps**:
1. Attempt to delete species with linked observations
2. Confirm deletion

**Expected Results**:
- ✅ Deletion prevented
- ✅ Error message: "Cannot delete species: X observations are linked"
- ✅ Species remains in database
- ✅ Redirected to species detail page

---

## 🔧 Configuration & Utilities

### Species Services

**Location**: `species/services.py`

```python
from species.services import SpeciesService, TaxonomyService

# Check for duplicate species
check = SpeciesService.check_duplicate_species('Homo sapiens')
# {'exists': True, 'in_fauna': True, 'in_flora': False}

# Create complete taxonomy
taxonomy = TaxonomyService.get_or_create_taxonomy({
    'kingdom': 'Animalia',
    'phylum': 'Chordata',
    'class': 'Aves',
    'order': 'Accipitriformes',
    'family': 'Accipitridae',
    'genus': 'Pithecophaga'
})
```

### Species Verification Utility

**Location**: `dashboard/utils.py`

```python
from dashboard.utils import SpeciesVerifier

# Verify species from field data
result = SpeciesVerifier.verify_species(form_data)

# Verify single species name
verification = SpeciesVerifier._verify_single_species(
    'species_1',
    'Pithecophaga jefferyi'
)
```

### Auto-Classification

Species are automatically classified based on their taxonomy:

**Fauna** (based on Class):
- Aves → Bird
- Mammalia → Mammal
- Reptilia → Reptile
- Amphibia → Amphibian
- Insecta/Arachnida/Crustacea → Arthropod
- Others → Other

**Flora** (based on Kingdom):
- Plantae → Plant
- Others → Other

---

## 🔒 Security & Permissions

### Access Control Matrix

| Action | PA Staff | Validator | DENR | Collaborator | Admin |
|--------|----------|-----------|------|--------------|-------|
| View Species List | ❌ | ✅ | ✅ | ✅ | ✅ |
| View Species Detail | ❌ | ✅ | ✅ | ✅ | ✅ |
| View Observations | ❌ | ✅ | ✅ | ✅ | ✅ |
| Create Species | ❌ | ❌ | ❌ | ❌ | ✅ |
| Edit Species | ❌ | ❌ | ❌ | ❌ | ✅ |
| Delete Species | ❌ | ❌ | ❌ | ❌ | ✅ |
| Add Articles | ❌ | ✅ | ❌ | ❌ | ✅ |
| Upload Media | ❌ | ✅ | ❌ | ❌ | ✅ |

### Permission Mixins

```python
# View species (read-only)
class SpeciesListView(CollaboratorAccessMixin, TemplateView):
    # Accessible to: Validator, DENR, Collaborator, Admin
    pass

# Manage species (CRUD)
class SpeciesCreateView(AdminAccessMixin, View):
    # Accessible to: Admin only
    pass

# Manage articles/media
class FaunaArticleCreateView(ValidatorAccessMixin, CreateView):
    # Accessible to: Validator, Admin
    pass
```

---

## 📚 Related Documentation

- [Wildlife Observation Records](../docs/DatabaseModel.md#wildlife-observation)
- [Taxonomy Management](../docs/generated/species_DEEP_DIVE.md)
- [Data Processing](../docs/Services.md#bmsdataprocessor)
- [Validation Workflow](VALIDATION_SYSTEM.md)

---

*Last Updated: January 6, 2026*
*Version: 1.0*
