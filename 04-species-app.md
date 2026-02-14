# Chapter 4 — The `species` App

> The `species` app is the **taxonomic reference library** of PRISM-Matutum. It holds the complete biological classification hierarchy (Kingdom → Genus), fauna and flora checklists, morphological characteristics, common names, research articles, and field media. Other apps reference this data when identifying species in observations.

---

## Table of Contents

- [4.1 App Overview](#41-app-overview)
- [4.2 Taxonomy Hierarchy](#42-taxonomy-hierarchy)
- [4.3 Species Checklists](#43-species-checklists)
- [4.4 Characteristics & Morphology](#44-characteristics--morphology)
- [4.5 Common Names, Research & Articles](#45-common-names-research--articles)
- [4.6 Field Media & Image Pipeline](#46-field-media--image-pipeline)
- [4.7 Services](#47-services)
- [4.8 Celery Tasks](#48-celery-tasks)
- [4.9 Management Commands](#49-management-commands)
- [4.10 How Other Apps Use Species Data](#410-how-other-apps-use-species-data)

---

## 4.1 App Overview

Unlike the other three custom apps, the `species` app has **no URL configuration of its own**. All species-related web pages are served through the **dashboard app** (under `/dashboard/species/`). The `species` app focuses on:

1. **Data models** — the taxonomic hierarchy and species checklists
2. **Services** — business logic for taxonomy management and biodiversity analytics
3. **Background tasks** — image downloading from KoboToolbox
4. **Admin interface** — Django admin configuration for managing species data

```
species/
├── models.py          # All species models (~592 lines)
├── services.py        # TaxonomyService, SpeciesService, BiodiversityAnalyticsService
├── services/          # Additional services (ImageDownloadService)
├── tasks.py           # Celery tasks for image downloads
├── admin.py           # Django admin with image previews and download actions
├── management/commands/  # JSON import, bulk image download
├── forms.py           # Species forms (used by dashboard views)
├── signals.py         # Empty (no signals currently)
└── tests/             # Tests
```

---

## 4.2 Taxonomy Hierarchy

### Django Concept: Hierarchical Data with Foreign Keys

Biological taxonomy follows a strict hierarchy — Kingdom is the broadest, Genus is the most specific. PRISM-Matutum models each level as a separate table linked by foreign keys:

```
Kingdom (e.g., Animalia)
  └── Phylum (e.g., Chordata)
        └── TaxonomyClass (e.g., Aves — birds)
              └── TaxonomyOrder (e.g., Accipitriformes — raptors)
                    └── Family (e.g., Accipitridae — hawks/eagles)
                          └── Genus (e.g., Pithecophaga)
```

### Individual Level Models

Each level is a simple lookup table:

```python
class Kingdom(models.Model):
    id_kingdom = models.AutoField(primary_key=True)
    taxon_name = models.CharField(max_length=100, unique=True)

class Phylum(models.Model):
    id_phylum = models.AutoField(primary_key=True)
    taxon_name = models.CharField(max_length=100, unique=True)

# ... same pattern for TaxonomyClass, TaxonomyOrder, Family, Genus
```

> **Naming note:** `TaxonomyClass` and `TaxonomyOrder` are used instead of `Class` and `Order` because those are reserved words in Python/SQL.

### Taxonomy (Composite Model)

The `Taxonomy` model links all six levels together to form a complete classification:

| Field | Type | Purpose |
|-------|------|---------|
| `id_taxonomy` | AutoField (PK) | |
| `id_kingdom` | ForeignKey → Kingdom | |
| `id_phylum` | ForeignKey → Phylum | |
| `id_class` | ForeignKey → TaxonomyClass | |
| `id_order` | ForeignKey → TaxonomyOrder | |
| `id_family` | ForeignKey → Family | |
| `id_genus` | ForeignKey → Genus | |
| `updated_at` | DateTimeField | |

**Unique constraint:** The combination of all six levels must be unique — you can't have two identical classifications.

**Example:** The Philippine Eagle would have a `Taxonomy` record linking:
- Kingdom: Animalia
- Phylum: Chordata
- Class: Aves
- Order: Accipitriformes
- Family: Accipitridae
- Genus: Pithecophaga

---

## 4.3 Species Checklists

The checklists are the master lists of known species in the protected area. There are two separate checklists — fauna (animals) and flora (plants).

### Status Lookup Tables

| Model | Purpose | Example Values |
|-------|---------|---------------|
| `SpeciesType` | Species category | Bird, Mammal, Reptile, Amphibian, Arthropod, Plant, Other |
| `Endemicity` | How endemic is the species? | Endemic, Native, Introduced, Migratory |
| `ThreatAssessment` | Conservation threat level | Critically Endangered, Endangered, Vulnerable, Near Threatened, Least Concern |

### FaunaChecklist

The master list of animal species.

| Field | Type | Purpose |
|-------|------|---------|
| `id_fauna_checklist` | AutoField (PK) | |
| `scientific_name` | CharField (unique, indexed) | Scientific name (e.g., `"Pithecophaga jefferyi"`) |
| `id_taxonomy` | ForeignKey → Taxonomy | Full taxonomic classification |
| `id_species_type` | ForeignKey → SpeciesType | Bird, Mammal, etc. — **auto-set on save** |
| `id_endemicity` | ForeignKey → Endemicity | Endemic, Native, etc. |
| `id_faunal_characteristics` | ForeignKey → FaunalCharacteristics | Morphological traits |
| `id_threat_category` | ForeignKey → ThreatAssessment | Conservation status |
| `total_recorded_population` | IntegerField | Known population count |
| `remarks` | TextField | Additional notes |

**Auto species-type mapping:** On save, the model automatically sets `id_species_type` based on the taxonomy class:
- Class Aves → SpeciesType "Bird"
- Class Mammalia → SpeciesType "Mammal"
- Class Reptilia → SpeciesType "Reptile"
- Class Amphibia → SpeciesType "Amphibian"
- Class Insecta (or similar) → SpeciesType "Arthropod"

### FloraChecklist

The master list of plant species. Same structure as `FaunaChecklist` but with flora-specific characteristics:

| Field | Type | Purpose |
|-------|------|---------|
| `scientific_name` | CharField (unique, indexed) | |
| `id_taxonomy` | ForeignKey → Taxonomy | |
| `id_species_type` | ForeignKey → SpeciesType | **Auto-set to "Plant"** |
| `id_endemicity` | ForeignKey → Endemicity | |
| `id_floral_characteristics` | ForeignKey → FloralCharacteristics | Leaf, stem, flower, fruit traits |
| `id_threat_category` | ForeignKey → ThreatAssessment | |
| `total_recorded_population` | IntegerField | |

---

## 4.4 Characteristics & Morphology

### FaunalCharacteristics

Simple characteristics for animals:

| Field | Type | Purpose |
|-------|------|---------|
| `description` | TextField | Physical description |
| `vocalization_link` | URLField | Link to audio recording of the species |

### FloralCharacteristics

Detailed plant morphology — links to multiple lookup tables:

| Field | Type | Purpose |
|-------|------|---------|
| `id_leaf_type` | ForeignKey → LeafType | Simple, Compound, etc. |
| `id_leaf_shape` | ForeignKey → LeafShape | Ovate, Lanceolate, Cordate, etc. |
| `id_leaf_margin` | ForeignKey → LeafMargin | Serrate, Entire, Lobed, etc. |
| `id_leaf_apex` | ForeignKey → LeafApex | Acute, Obtuse, Acuminate, etc. |
| `id_leaf_base` | ForeignKey → LeafBase | Cuneate, Rounded, Cordate, etc. |
| `id_stem_type` | ForeignKey → StemType | Woody, Herbaceous, etc. |
| `id_flower_type` | ForeignKey → FlowerType | Complete, Incomplete, etc. |
| `id_fruit_type` | ForeignKey → FruitType | Berry, Drupe, Capsule, etc. |
| `leaf_color_description` / `flower_color_description` | CharField | Color descriptions |

### Morphology Lookup Tables

Each lookup has an `image` field for visual identification:

| Model | Examples |
|-------|---------|
| `LeafType` | Simple, Compound, Trifoliate |
| `LeafShape` | Ovate, Elliptic, Lanceolate, Linear, Cordate |
| `LeafMargin` | Entire, Serrate, Dentate, Lobed |
| `LeafApex` | Acute, Obtuse, Acuminate, Emarginate |
| `LeafBase` | Cuneate, Rounded, Truncate, Cordate |
| `StemType` | Woody, Herbaceous, Climbing |
| `FlowerType` | Complete, Incomplete, Unisexual |
| `FruitType` | Berry, Drupe, Capsule, Pod |

---

## 4.5 Common Names, Research & Articles

### Django Concept: Junction Tables (Many-to-Many without `ManyToManyField`)

Sometimes you need explicit junction tables instead of Django's automatic M2M. PRISM-Matutum uses explicit junction models to keep the structure clean:

```python
# Instead of:
# class FaunaChecklist(models.Model):
#     common_names = models.ManyToManyField(SpeciesCommonName)

# PRISM-Matutum uses:
class FaunaCommonName(models.Model):
    id_fauna_checklist = models.ForeignKey(FaunaChecklist, ...)
    id_species_common_name = models.ForeignKey(SpeciesCommonName, ...)
```

This allows more control and makes admin management easier.

### SpeciesCommonName

| Field | Type | Purpose |
|-------|------|---------|
| `name` | CharField | Common name (e.g., "Philippine Eagle") |
| `link` | URLField | Reference link |
| `id_articles` | ForeignKey → Articles | Related article |

### Junction Tables

| Model | Links | Purpose |
|-------|-------|---------|
| `FaunaCommonName` | FaunaChecklist ↔ SpeciesCommonName | Common names for fauna |
| `FloraCommonName` | FloraChecklist ↔ SpeciesCommonName | Common names for flora |
| `RelatedFaunaResearch` | FaunaChecklist ↔ RelatedResearch | Research papers about fauna |
| `RelatedFloraResearch` | FloraChecklist ↔ RelatedResearch | Research papers about flora |
| `FaunaRelatedArticle` | FaunaChecklist ↔ Articles | Articles about fauna |
| `FloraRelatedArticle` | FloraChecklist ↔ Articles | Articles about flora |

### RelatedResearch & Articles

| Model | Fields | Purpose |
|-------|--------|---------|
| `RelatedResearch` | `title`, `abstract`, `link`, `type` | Academic research papers |
| `Articles` | `title`, `abstract`, `type`, `url` | General articles and references |

---

## 4.6 Field Media & Image Pipeline

### FieldMedia

Stores media files (photos, videos, audio) collected in the field.

| Field | Type | Purpose |
|-------|------|---------|
| `description` | CharField | What this media shows |
| `media_type` | CharField | Type of media (photo, video, audio, document) |
| `captured_by` | CharField | Who took it |
| `latitude` / `longitude` | FloatField | Where it was captured |
| `link` | URLField | Original URL (usually a KoboToolbox URL) |
| `source_url` | URLField | Source URL for download |
| `image_file` | ImageField | Locally stored copy of the image |
| `thumbnail` | ImageField | Locally generated thumbnail |
| `download_status` | CharField | `pending`, `downloading`, `downloaded`, `failed`, `not_needed` |
| `content_hash` | CharField | SHA hash for deduplication |
| `downloaded_at` | DateTimeField | When the local copy was made |
| `file_size` | IntegerField | File size in bytes |
| `download_attempts` | IntegerField | How many times download was tried |
| `last_download_error` | TextField | Error message from last failed attempt |

### The Image Download Pipeline

Field photos are hosted on KoboToolbox's servers. To display them in the app, PRISM-Matutum can either:

1. **Proxy** — Use the `/api/kobo-image/` endpoint to fetch and relay the image on each request (avoids CORS issues)
2. **Download** — Queue a Celery task to download the image locally for faster access

```
KoboToolbox Server (eu.kobotoolbox.org)
        │
        │  Option A: Proxy (on-demand)
        ├──────► /api/kobo-image/?url=... ──► Browser
        │
        │  Option B: Local download (background)
        └──────► Celery task: download_field_media_image
                        │
                        ▼
                media/field_media/<file>  ──► Browser
```

**Key methods on FieldMedia:**
- `is_downloaded()` — Check if a local copy exists
- `get_image_url()` — Returns local path if downloaded, otherwise the original URL
- `get_thumbnail_url()` — Returns local thumbnail path
- `schedule_download()` — Queue a Celery task to download this media

### Junction Tables for Media

| Model | Links | Purpose |
|-------|-------|---------|
| `FaunaFieldMedia` | FaunaChecklist ↔ FieldMedia | Photos of fauna species |
| `FloraFieldMedia` | FloraChecklist ↔ FieldMedia | Photos of flora species |

---

## 4.7 Services

**File:** `species/services.py` (~665 lines)

### TaxonomyService

Manages the taxonomy hierarchy.

| Method | Purpose |
|--------|---------|
| `get_or_create_taxonomy_level(model_class, taxon_name)` | Get or create a level (Kingdom, Phylum, etc.) by name |
| `get_or_create_taxonomy(kingdom, phylum, class, order, family, genus)` | Get or create a complete `Taxonomy` record from all 6 level names |
| `get_taxonomy_hierarchy()` | Returns the full hierarchy tree for display |

### SpeciesService

CRUD operations for species checklists.

| Method | Purpose |
|--------|---------|
| `check_duplicate_species(scientific_name)` | Check if a species already exists in either checklist |
| `create_fauna_entry(data)` | Create a new FaunaChecklist entry with all related data (taxonomy, characteristics, common names) |
| `create_flora_entry(data)` | Create a new FloraChecklist entry |

### FloraLookupService

| Method | Purpose |
|--------|---------|
| `get_all_lookups()` | Returns all flora morphology lookup values (leaf types, shapes, margins, etc.) for form dropdowns |

### StatusLookupService

| Method | Purpose |
|--------|---------|
| `get_all_statuses()` | Returns all endemicity and threat assessment options |

### BiodiversityAnalyticsService

Provides biodiversity metrics for analytics dashboards and reports.

| Method | Purpose |
|--------|---------|
| `calculate_species_abundance(species_type, date_from, date_to, protected_area)` | Count observations per species in a time range and area |
| `calculate_species_richness(date_from, date_to, protected_area, transect_route)` | Count unique species observed — a key biodiversity indicator |
| `calculate_frequency_monitoring(species_id, species_type, date_from, date_to)` | How often a species is observed over time |
| `calculate_population_trends(species_id, species_type, date_from, date_to)` | Population trend over time (increasing, decreasing, stable) |

---

## 4.8 Celery Tasks

**File:** `species/tasks.py` (~112 lines)

All species tasks route to the `media_downloads` queue (configured in `prism/celery.py`).

| Task | Retries | Purpose |
|------|---------|---------|
| `download_field_media_image(field_media_id)` | 3 | Download a single image from KoboToolbox to local storage. Updates `download_status`, `image_file`, `file_size`, `content_hash` |
| `download_multiple_images(field_media_ids)` | — | Orchestrator: queues individual `download_field_media_image` tasks for each ID |
| `retry_failed_downloads()` | — | Finds all `FieldMedia` with `download_status='failed'` and re-queues them |
| `cleanup_orphaned_images()` | — | Removes locally stored images that no longer have a `FieldMedia` record |

---

## 4.9 Management Commands

**Directory:** `species/management/commands/`

| Command | Purpose |
|---------|---------|
| `add_species_byjson` | Bulk import species from a JSON file. The command reads a structured JSON with taxonomy, endemicity, threat status, and common names, then creates all the necessary records. A JSON template is provided at `species/management/commands/species_import_template.json` |
| `download_field_images` | Batch download all pending field media images. Finds all `FieldMedia` with `download_status='pending'` and processes them |

---

## 4.10 How Other Apps Use Species Data

The species app acts as a **reference data provider** for the rest of the system:

### api app
- **WildlifeObservation** has `fauna_species` (FK → FaunaChecklist) and `flora_species` (FK → FloraChecklist) — auto-linked when observations are processed
- **Choice** model can link to `FaunaChecklist` and `FloraChecklist` — for KoboToolbox form dropdowns
- **ObservationMedia** links to `FieldMedia` — shared media between species records and observations

### dashboard app
- **Species views** (`views_species.py`) provide the UI for viewing and managing species
- **Report services** (`report_services.py`) query species data for biodiversity metrics (species richness, endemic counts, threatened species)
- **Analytics widgets** use `BiodiversityAnalyticsService` for charts and statistics
- **Validation workspace** shows species info when reviewing wildlife observations

### Users within Profiles
- **ValidatorProfile** tracks `expertise_areas` which can include species types
- **PAStaffProfile** tracks `specialization` (fauna, flora, both, etc.)

---

**Next:** [Chapter 5 — Data Flow Pipeline →](05-data-flow-pipeline.md)

**Previous:** [← Chapter 3: The `users` App](03-users-app.md)
