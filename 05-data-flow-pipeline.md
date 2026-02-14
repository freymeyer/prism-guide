# Chapter 5 — Data Flow Pipeline

> This chapter traces the complete journey of a single piece of field data — from a PA Staff member recording a wildlife sighting on their phone, all the way to that data appearing on a report, map, or analytics chart. Understanding this pipeline is key to understanding how the entire system fits together.

---

## Table of Contents

- [5.1 The Big Picture](#51-the-big-picture)
- [5.2 Stage 1: Data Collection (KoboToolbox)](#52-stage-1-data-collection-kobotoolbox)
- [5.3 Stage 2: Data Sync (Celery → DataSubmission)](#53-stage-2-data-sync-celery--datasubmission)
- [5.4 Stage 3: PA Staff Review](#54-stage-3-pa-staff-review)
- [5.5 Stage 4: Validator Review](#55-stage-4-validator-review)
- [5.6 Stage 5: Auto-Processing (Signals)](#56-stage-5-auto-processing-signals)
- [5.7 Stage 6: Structured Records → Dashboard](#57-stage-6-structured-records--dashboard)
- [5.8 Choice Sync Pipeline](#58-choice-sync-pipeline)
- [5.9 Image Download Pipeline](#59-image-download-pipeline)
- [5.10 Error Handling & Edge Cases](#510-error-handling--edge-cases)

---

## 5.1 The Big Picture

```
STAGE 1                STAGE 2              STAGE 3           STAGE 4
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  PA Staff   │    │    Celery    │    │   PA Staff   │    │  Validator   │
│  fills out  │───►│ syncs every  │───►│   reviews    │───►│  approves/   │
│  KoboToolbox│    │  10 minutes  │    │   own data   │    │  rejects     │
│  form on    │    │              │    │              │    │              │
│  mobile     │    │ Creates      │    │ Status:      │    │ Status:      │
│             │    │ DataSubmission│   │ pending_pa   │    │ approved     │
│             │    │ (raw JSON)   │    │ → ---        │    │ (or reject)  │
└─────────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘
                                                                  │
                                                                  │ Signal fires!
                                                                  ▼
STAGE 7                STAGE 6                              STAGE 5
┌─────────────┐    ┌──────────────┐                    ┌──────────────┐
│  Dashboard   │    │  Structured  │                    │ BMSDataProc  │
│  Maps        │◄───│   Records    │◄───────────────────│ + CommitSvc  │
│  Reports     │    │              │                    │              │
│  Analytics   │    │ WildlifeObs  │                    │ Parses JSON  │
│  Exports     │    │ ResourceUse  │                    │ Creates DB   │
│              │    │ Disturbance  │                    │ records      │
│              │    │ Landscape    │                    │ Links species│
└─────────────┘    └──────────────┘                    └──────────────┘
```

---

## 5.2 Stage 1: Data Collection (KoboToolbox)

### What is KoboToolbox?

[KoboToolbox](https://www.kobotoolbox.org/) is a free, open-source platform for mobile data collection. PA Staff install the KoboCollect app on their tablets/phones and fill out survey forms while in the field.

### The BMS Field Survey Form

The main form used by PRISM-Matutum is a **BMS (Biodiversity Monitoring System) Field Survey**. It captures:

- **Date, time, GPS coordinates** — When and where the observation happened
- **Observer identity** — Who is making the observation
- **Record type** — What kind of observation (wildlife sighting, resource use incident, disturbance, landscape monitoring)
- **Type-specific fields** — Species name, count, observation method (for wildlife); or disturbance type, action taken (for disturbances); etc.
- **Photos** — Field photos attached to the submission
- **Remarks** — Free-text notes

### How KoboToolbox Stores Data

KoboToolbox stores each submission as a JSON object. A wildlife observation might look like:

```json
{
  "_id": 12345,
  "_uuid": "abc-def-123-456",
  "_submission_time": "2026-01-15T09:30:00",
  "username": "juan.cruz",
  "group_location/geopoint": "6.357 125.073 1200 10",
  "group_observation/record_type": "wildlife_observation",
  "group_wildlife/species_name": "philippine_eagle",
  "group_wildlife/species_count": "1",
  "group_wildlife/observation_type": "seen",
  "group_evidence/photo": "attachment-url-here",
  "group_evidence/remarks": "Spotted near Station 3, adult male"
}
```

Note the **group prefixes** (`group_location/`, `group_wildlife/`, `group_evidence/`). These come from KoboToolbox's form structure and need to be flattened during processing.

### Form Configuration

Forms are defined as **XLSForm** (Excel) files that define:
- Questions and their types (text, integer, select_one, geopoint, image, etc.)
- Choice lists (dropdown options)
- Skip logic (show/hide questions based on answers)
- Groups and repeats

The form is deployed to KoboToolbox, where it becomes available on mobile devices.

---

## 5.3 Stage 2: Data Sync (Celery → DataSubmission)

### The Sync Process

Every 10 minutes, a Celery Beat task (`sync_all_kobotoolbox_data`) triggers the sync:

```
Celery Beat (every 10 min)
    │
    ▼
sync_all_kobotoolbox_data()              # api/tasks.py
    │
    ▼
DataSubmissionService.sync_submissions_to_database(asset_uid)  # api/submission_services.py
    │
    ├── 1. Call KoboToolbox API: GET /api/v2/assets/{uid}/data/
    │
    ├── 2. For each submission in response:
    │       ├── Check if submission_uuid already exists in DB
    │       ├── If NEW → create DataSubmission with:
    │       │     - form_data = raw JSON (entire payload)
    │       │     - validation_status = 'pending_pa_review'
    │       │     - is_processed = False
    │       │     - database_committed = False
    │       │
    │       └── If EXISTS → update form_data if changed
    │
    ├── 3. Auto-assign submissions to ScheduledSurveys
    │       (matches by time range and transect route)
    │
    └── 4. Create DataRetrievalLog entry
          (submissions_found, new, updated, skipped, status)
```

### What Gets Stored

The entire KoboToolbox JSON payload is stored in `DataSubmission.form_data` as a `JSONField`. This raw data is **never modified** — it's the source of truth. All derived data comes from processing this JSON later.

### Duplicate Prevention

The `submission_uuid` field (unique) prevents duplicates. If a submission with the same UUID already exists, it's updated rather than duplicated.

---

## 5.4 Stage 3: PA Staff Review

After syncing, submissions start with `validation_status = 'pending_pa_review'`.

### What PA Staff See

PA Staff log in and visit `/dashboard/my-submissions/`. They see their own submissions with status badges:

| Status | Badge Color | Meaning |
|--------|------------|---------|
| `pending_pa_review` | Yellow | Needs their review before validators see it |
| `---` | Blue | Reviewed, waiting for validator |
| `approved` | Green | Validator approved |
| `not_approved` | Red | Validator rejected |
| `on_hold` | Orange | Needs more information |
| `flagged` | Purple | Special attention needed |

### What PA Staff Can Do

1. **Review** — Check that the data looks correct (right species, GPS makes sense, photos attached)
2. **Mark as reviewed** — Sets `reviewed_by_submitter = True` and status changes to `---` (pending validation)
3. **Edit** — If the submission is on hold, they can correct data and resubmit

### How the Status Changes

```python
# In the view or API:
submission.reviewed_by_submitter = True
submission.submitter_reviewed_at = timezone.now()
submission.validation_status = '---'  # Now visible to validators
submission.save()
```

---

## 5.5 Stage 4: Validator Review

Submissions with status `---` appear in the Validator's queue.

### The Validation Queue

Validators visit `/dashboard/queue/` and see a filterable table of all pending submissions. They can filter by:
- Date range
- PA Staff (who submitted)
- Record type (wildlife, disturbance, etc.)
- Protected area

### The Validation Workspace

Clicking a submission opens `/dashboard/workspace/<pk>/` — a detailed review page showing:

1. **Submission data** — All form fields in a readable format
2. **Map** — GPS location plotted on a Leaflet map
3. **Photos** — Field photos from KoboToolbox
4. **Species identification** — If it's a wildlife observation, the system shows the matched species from the checklist
5. **Quality score** — Automated data quality indicators
6. **History** — Previous validation actions and notes

### Validation Actions

The validator can take one of four actions:

| Action | New Status | What Happens Next |
|--------|-----------|-------------------|
| **Approve** | `approved` | Triggers auto-processing (Stage 5) |
| **Reject** | `not_approved` | Submission is archived, not processed |
| **Hold** | `on_hold` | PA Staff is notified to correct issues |
| **Flag** | `flagged` | Marked for special review |

### What Happens Under the Hood

```python
# api/submission_services.py - DataSubmissionService.update_validation_status()

# 1. Update local database
submission.validation_status = 'approved'
submission.validated_by = validator_user
submission.validated_at = timezone.now()
submission.validation_remarks = remarks
submission.save()  # ← This triggers the signal!

# 2. Sync to KoboToolbox (update status there too)
# PUT /api/v2/assets/{uid}/data/{id}/validation_status/
```

---

## 5.6 Stage 5: Auto-Processing (Signals)

This is the most automated and critical stage. When `submission.save()` is called with `validation_status = 'approved'`, Django's signal system takes over.

### The Signal Chain

```python
# api/signals.py

# 1. PRE-SAVE: Track the old status
@receiver(pre_save, sender=DataSubmission)
def track_validation_status_change(sender, instance, **kwargs):
    if instance.pk:
        old = DataSubmission.objects.get(pk=instance.pk)
        instance._old_validation_status = old.validation_status

# 2. POST-SAVE: Auto-process if approved
@receiver(post_save, sender=DataSubmission)
def auto_process_validated_submission(sender, instance, **kwargs):
    if (instance.validation_status == 'approved'
        and not instance.is_processed
        and hasattr(instance, '_old_validation_status')
        and instance._old_validation_status != 'approved'):
        
        # Process the raw JSON into structured data
        processor = BMSDataProcessor()
        success, message, processed_data = processor.process_validated_submission(instance)
        
        if success:
            # Save to normalized database tables
            commit_service = BMSDataCommitService()
            commit_service._commit_to_database(instance, processed_data)
            
            instance.is_processed = True
            instance.database_committed = True
            instance.save(update_fields=['is_processed', 'database_committed'])

# 3. POST-SAVE: Log the status change
@receiver(post_save, sender=DataSubmission)
def log_validation_status_change(sender, instance, **kwargs):
    # Creates ValidationHistory record for audit trail
```

### What BMSDataProcessor Does

The processor reads the raw JSON and extracts structured data:

```
form_data (JSON) ──► _flatten_form_data() ──► _extract_common_data()
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    │                 │                 │
                              _process_          _process_        _process_
                              wildlife_data()    disturbance_     resource_use_
                                    │            data()           data()
                                    │                 │                 │
                                    └────────┬────────┘                 │
                                             │                         │
                                    processed_data (dict)──────────────┘
```

1. **Flatten** — Removes group prefixes (`group_wildlife/species_name` → `species_name`)
2. **Extract common** — GPS, dates, personnel, protected area, transect segment
3. **Extract specific** — Species name/count/type (wildlife), disturbance type/action (disturbance), etc.
4. **Link species** — Tries to match the species name text to a `FaunaChecklist` or `FloraChecklist` entry

### What BMSDataCommitService Does

Takes the processed dictionary and creates the appropriate Django model instance:

```python
# Simplified version of what _commit_to_database does:

record_type = processed_data['record_type']

if record_type == 'wildlife_observation':
    record = WildlifeObservation.objects.create(
        submission=submission,
        species=processed_data['species'],
        species_count=processed_data['species_count'],
        latitude=processed_data['latitude'],
        longitude=processed_data['longitude'],
        fauna_species=processed_data.get('linked_fauna_species'),
        # ... all other fields
    )
elif record_type == 'resource_use_incident':
    record = ResourceUseIncident.objects.create(...)
elif record_type == 'disturbance_record':
    record = DisturbanceRecord.objects.create(...)
elif record_type == 'landscape_monitoring':
    record = LandscapeMonitoring.objects.create(...)
```

### After Commit

Additional signals fire after the processed record is created:
1. **Auto-assign transect segment** — Uses PostGIS to find the nearest segment within 50 meters
2. **Sync status to KoboToolbox** — Updates the validation status on the KoboToolbox server
3. **Auto-link species** — If it's a wildlife observation, tries to match species and link media/articles

---

## 5.7 Stage 6: Structured Records → Dashboard

Once processed records exist in the database, they're available throughout the system:

### Dashboard Views

| Feature | What It Shows | Data Source |
|---------|-------------|-------------|
| **Observation Lists** | `/dashboard/observations/` | `WildlifeObservation.objects.all()` |
| **Observation Detail** | `/dashboard/observations/<pk>/` | Single record with map, photos, species info |
| **Species Observations** | `/dashboard/species/<type>/<pk>/observations/` | All sightings of a specific species |
| **Interactive Map** | `/dashboard/map/` | GeoJSON API → Leaflet markers for each observation |
| **PA Staff Home** | `/dashboard/pa-staff-home/` | User's own observations, recent submissions |
| **Analytics** | Various widgets | Aggregated counts, trends, charts |

### Reports

The `BaseReportGenerator` queries processed records with date range and geographic filters:

```python
# Simplified from dashboard/report_services.py
wildlife_data = WildlifeObservation.objects.filter(
    submission_time__range=(start_date, end_date),
    protected_area=protected_area,
)
species_richness = wildlife_data.values('species').distinct().count()
```

Report types and their outputs:
- **GIS Report** → GeoJSON + Excel with coordinates
- **Semestral Report** → Excel with statistics + Markdown summary
- **Digital Backup** → Full data export (Excel + CSV + GeoJSON)
- **PDF Reports** → HTML → PDF via WeasyPrint

### GeoJSON API

The interactive map loads observation data via AJAX calls to GeoJSON endpoints:

```javascript
// In map.js or map-enhanced.js:
fetch('/dashboard/api/geo/observations/')
  .then(response => response.json())
  .then(geojson => {
    L.geoJSON(geojson).addTo(map);
  });
```

The GeoJSON endpoint in `dashboard/api_geo.py` queries the processed records and converts them to RFC 7946 GeoJSON:

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {"type": "Point", "coordinates": [125.073, 6.357]},
      "properties": {
        "species": "Philippine Eagle",
        "count": 1,
        "date": "2026-01-15",
        "observer": "Juan Cruz"
      }
    }
  ]
}
```

---

## 5.8 Choice Sync Pipeline

Separate from the main data flow, there's a bidirectional sync between Django choice lists and KoboToolbox form options:

### Django → KoboToolbox (Updating Forms)

When species or monitoring locations are added/removed in Django, the form needs to be updated so field workers see the new options:

```
1. Admin adds new species to FaunaChecklist
2. Admin triggers "Update Form Choices" from /dashboard/admin/choices/
3. process_form_with_choice_updates(asset_uid):
     a. Download current XLS form from KoboToolbox
     b. Parse XLS to find choice lists
     c. Update choice options from Django ChoiceList/Choice records
     d. Upload modified XLS form back to KoboToolbox
     e. Redeploy the form so mobile devices get the update
4. FormUpdateLog created for audit trail
```

### KoboToolbox → Django (Syncing Choices)

When a form is modified directly in KoboToolbox, sync the choices back:

```
1. Admin triggers "Sync Choices" from /dashboard/admin/choices/
2. sync_form_choices_to_database(asset_uid):
     a. Download XLS form from KoboToolbox
     b. Parse all select_one/select_multiple questions
     c. For each mapped question (via FormChoiceMapping):
          - Compare KoboToolbox choices with Django Choice records
          - Create/update/deactivate choices as needed
3. Django Choice records now match the KoboToolbox form
```

---

## 5.9 Image Download Pipeline

Field photos go through a separate pipeline:

```
1. PA Staff takes photo during field survey
2. Photo uploaded to KoboToolbox with the submission
3. Photo URL stored in DataSubmission.form_data JSON
4. During processing (Stage 5), URL is extracted and stored in WildlifeObservation.image
5. When the observation is viewed, the photo is either:
     a. Proxied: Browser → /api/kobo-image/?url=... → KoboToolbox → Browser
     b. Downloaded: Celery task → stores in media/field_media/ → served locally

Download pipeline:
     FieldMedia.schedule_download()
         │
         ▼
     Celery task: download_field_media_image(media_id)
         │
         ├── Download from KoboToolbox URL
         ├── Calculate content_hash (SHA256)
         ├── Save to media/field_media/<file>
         ├── Generate thumbnail
         ├── Update FieldMedia.download_status = 'downloaded'
         │
         └── On failure:
              ├── Increment download_attempts
              ├── Set download_status = 'failed'
              └── Store last_download_error
              └── Celery auto-retries (max 3 times)
```

---

## 5.10 Error Handling & Edge Cases

### What if the sync fails?

- The Celery task has built-in retry logic
- `DataRetrievalLog` records the failure with error message
- The next 10-minute sync will pick up whatever was missed

### What if processing fails?

- The signal handler catches exceptions and logs them
- `DataSubmission.is_processed` stays `False`
- Admins can manually trigger reprocessing via `/api/processing/reprocess/`

### What if species can't be linked?

- The observation is still created, but `fauna_species`/`flora_species` remains `null`
- The `species` field (text) is preserved from the original submission
- Validators can later manually link species through the dashboard

### What about editing after approval?

- Once a `DataSubmission` is approved and processed, editing is restricted
- The `can_edit_by_submitter()` method on `DataSubmission` returns `False` for approved/processed submissions
- If corrections are needed, a new submission should be made

### What about KoboToolbox downtime?

- Sync tasks fail gracefully — logged but don't crash the system
- Previously synced data is unaffected (stored in local database)
- The `health_check()` function in `api/form_services.py` can detect connectivity issues

---

**Next:** [Chapter 6 — Frontend Architecture →](06-frontend-architecture.md)

**Previous:** [← Chapter 4: The `species` App](04-species-app.md)
