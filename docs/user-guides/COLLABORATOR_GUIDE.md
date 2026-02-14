# Collaborator Access Documentation

## 📋 Overview

This document provides comprehensive documentation for Collaborator/Researcher users in the PRISM-Matutum system. Collaborators are external researchers, academic partners, or conservation organizations granted read-only access to biodiversity monitoring data for research and collaboration purposes.

**Role Code:** `collaborator`  
**Primary Function:** Read-only data access for research and analysis

---

## 🎯 Role Definition & Capabilities

### What Collaborators Can Do:
- ✅ View validated biodiversity data and observations
- ✅ Access species checklists and taxonomic information
- ✅ View and download completed reports
- ✅ Export data for research purposes
- ✅ Access interactive maps with observation locations
- ✅ View data visualizations and analytics
- ✅ Download species information and media

### What Collaborators Cannot Do:
- ❌ Submit field data (data collection restricted to PA Staff)
- ❌ Validate or approve submissions
- ❌ Edit any data or system configuration
- ❌ Manage users or organizations
- ❌ Generate certain report types (limited to viewing)
- ❌ Access pending/unvalidated data
- ❌ Access PA Staff personal submission details

**Key Principle:** Collaborators have **read-only** access to **validated data only** for **research and conservation purposes**.

---

## 🏠 1. Collaborator Dashboard

**URL:** `/dashboard/collaborator/`  
**View:** `CollaboratorDashboardView` ([views_dashboard.py](dashboard/views_dashboard.py#L850-L894))  
**Template:** [collaborator_dashboard.html](dashboard/templates/dashboard/collaborator_dashboard.html)

### Dashboard Purpose

Provides researchers and external partners with an overview of available biodiversity data and quick access to research resources.

### Dashboard Components

#### Overview Statistics

| Stat Card | Information | Research Value |
|-----------|-------------|----------------|
| **Species Documented** | Total validated species records | Dataset richness |
| **Observations Available** | Total validated observations | Sample size |
| **Reports Available** | Public reports accessible | Published findings |
| **Study Period** | Date range of available data | Temporal scope |

#### Quick Access Links

**Research Resources:**
- **Data Browser** - Explore observations with filters
- **Species Explorer** - Browse taxonomic database
- **Report Archive** - View published reports
- **Data Export** - Download datasets for analysis
- **Map Viewer** - Geographic visualization

#### Recent Updates Feed

Shows latest additions to the database:
- Newly validated observations
- Updated species information
- New reports published
- System announcements

---

## 📊 2. Data Access & Browsing

### 2.1 Observation Browser

**URL:** `/dashboard/observations/`  
**Permission:** `CollaboratorAccessMixin` ([permissions.py](dashboard/permissions.py#L160))

**Available Observation Types:**
- Wildlife Observations
- Resource Use Incidents (for threat analysis)
- Disturbance Records (for habitat assessment)
- Landscape Monitoring

**Filter Options:**
- **Date Range** - Temporal scope
- **Species** - By scientific or common name
- **Location** - Protected area, Region, Province
- **Conservation Status** - Endangered, Threatened, etc.
- **Observation Method** - Seen, Heard, Traces, etc.

**Displayed Fields:**
- Species (scientific name)
- Common name(s)
- Observation date
- Location (coordinates, PA name)
- Count/Abundance
- Observer (anonymized as "PA Staff")
- Validation status (Approved only)

**Data Quality Indicators:**
- GPS accuracy
- Photo availability
- Data quality score

### 2.2 Species Explorer

**URL:** `/dashboard/species/`

**Features:**

**A. Searchable Species List**
- Search by scientific name, common name, family
- Filter by:
  - Conservation status (IUCN, National)
  - Endemicity (Endemic, Native, Introduced)
  - Taxonomic rank
  - Observation availability

**B. Species Detail Pages**

For each species:
- **Taxonomy** - Complete classification
- **Conservation Status** - IUCN Red List, CITES, National status
- **Physical Description** - Morphology, size, distinguishing features
- **Habitat & Ecology** - Preferred habitats, elevation range, diet
- **Distribution** - Geographic range, endemic areas
- **Observation Summary** - Count, frequency, locations (validated only)
- **Media Gallery** - Field photos, reference images
- **References** - Scientific literature, field guides

**C. Observation Records for Species**

View all validated observations for a species:
- Date, location, count
- GPS coordinates
- Habitat notes
- Field photos (if available)

**Data Restrictions:**
- Only validated observations visible
- PA Staff names anonymized
- Exact GPS coordinates rounded to nearest 100m (for sensitive species)

### 2.3 Interactive Map

**URL:** `/dashboard/map/`

**Map Layers:**
- **Protected Area Boundaries** - PA polygons
- **Observation Points** - Validated wildlife sightings
- **Heat Maps** - Observation density
- **Species Distribution** - Range maps for selected species

**Interactions:**
- Click marker → View observation details
- Filter by species, date, location
- Draw selection → Export observations in area
- Toggle layers on/off

**Privacy Protection:**
- For Critically Endangered species: Locations obscured (general area only)
- For Endangered species: Coordinates rounded to 1km
- For other species: Precise locations shown

### 2.4 Data Export for Research

**URL:** `/dashboard/data-export/`  
**Permission:** Limited export access

**Available Exports:**

| Export Type | Format | Contents | Usage |
|-------------|--------|----------|-------|
| **Observation Dataset** | CSV, Excel | Observations with filters applied | Statistical analysis |
| **Species Checklist** | CSV, PDF | Complete species list | Reference, publications |
| **Occurrence Records** | CSV (Darwin Core) | Biodiversity data standard format | Global biodiversity databases |
| **Summary Statistics** | Excel | Aggregated metrics | Reports, presentations |

**Export Limitations:**
- Maximum 10,000 records per export
- Rate limit: 5 exports per day
- Personal identifiers removed (PA Staff anonymized)
- Sensitive species: Coordinates obscured

**Export Process:**
1. Select export type
2. Apply filters (date, species, location)
3. Choose format
4. Agree to data use terms
5. Download file

**Data Use Agreement:**

Collaborators must agree to:
- Cite PRISM-Matutum as data source
- Use data for research/conservation only
- Not share raw data publicly
- Acknowledge PA Staff contributors (collectively)
- Share findings/publications with PRISM

---

## 📈 3. Analytics & Visualizations

**URL:** `/dashboard/analytics/`

### Available Analytics

**Species Richness Analysis:**
- Species accumulation curves
- Diversity indices (Shannon, Simpson)
- Species-area relationships

**Temporal Trends:**
- Observation frequency over time
- Seasonal patterns
- Population trend estimates

**Spatial Patterns:**
- Species distribution maps
- Observation heat maps
- Protected area coverage

**Threat Assessment:**
- Resource use incident frequencies
- Disturbance types and locations
- Threat severity mapping

### Visualization Tools

**Interactive Charts:**
- **Bar Charts** - Species counts, comparisons
- **Line Charts** - Time series, trends
- **Pie Charts** - Proportions (observation types, conservation status)
- **Heat Maps** - Geographic distribution
- **Scatter Plots** - Correlations

**Features:**
- Hover for details
- Click to filter
- Export chart as PNG or SVG
- Download underlying data as CSV
- Embed code for presentations

---

## 📄 4. Report Access

**URL:** `/dashboard/reports/archive/`

### Available Reports

Collaborators can view public reports:
- **Standard BMS Reports** - Regular monitoring summaries
- **Semestral Reports** - Bi-annual comprehensive reports
- **Species-Specific Reports** - Focused on particular taxa
- **Conservation Status Reports** - Threat assessments

**Report Formats:**
- PDF (view/download)
- Summary statistics (web view)

**Report Metadata:**
- Title, date range, geographic scope
- Authors/Contributors
- Summary/Abstract
- Download count

### Report Restrictions

Collaborators **cannot** access:
- Draft/unpublished reports
- Internal operational reports
- PA Staff performance reports
- Reports with sensitive information (e.g., poaching locations)

**Requesting Access:**

For restricted reports:
1. Submit request via system
2. Provide justification (research purpose)
3. Await approval from System Admin
4. Receive notification when granted

---

## 🦜 5. Species Information

### Species Detail Access

**Full Access to:**
- Taxonomy and classification
- Conservation status
- Physical descriptions
- Habitat information
- Literature references
- Media gallery (photos, audio)

**Limited Access to:**
- Observation locations (obscured for sensitive species)
- Observer identities (anonymized)
- Unvalidated observation data (not visible)

### Contributing Information

Collaborators can suggest updates:
- Species description improvements
- Additional literature references
- Conservation status updates
- Distribution information

**Process:**
1. Navigate to species detail page
2. Click "Suggest Update"
3. Fill form with proposed changes
4. Submit for admin review
5. Receive notification on decision

---

## 🔒 6. Data Privacy & Ethics

### Data Use Guidelines

**Permitted Uses:**
- Academic research and publications
- Conservation planning
- Species assessments
- Educational purposes
- Policy advocacy

**Prohibited Uses:**
- Commercial exploitation
- Public disclosure of sensitive locations
- Identification of individual PA Staff
- Data resale or redistribution

### Attribution Requirements

**When using PRISM data:**

**Citation Format:**
```
PRISM-Matutum Biodiversity Monitoring System. [Date Range]. 
Protected Area Biodiversity Data. DENR Region XI. 
Accessed: [Access Date]. https://prism-matutum.org
```

**In Publications:**
- Acknowledge PRISM-Matutum system
- Mention PA Staff contributors collectively
- Include data access date and scope
- Share publication with PRISM administrators

### Privacy Protection

**Protected Information:**
- PA Staff personal details (names, contact)
- Exact locations of Critically Endangered species
- Poaching/illegal activity specific locations
- Validator identities and remarks
- Internal system communications

**Public Information:**
- Validated observations (with privacy filters)
- Species checklists
- Published reports
- Aggregated statistics

---

## 🧪 7. Testing Checklist for Collaborators

### 7.1 Dashboard & Access

- [ ] **Test:** Collaborator dashboard loads
- [ ] **Test:** Overview statistics display
- [ ] **Test:** Quick access links work
- [ ] **Test:** Cannot access admin functions (403)
- [ ] **Test:** Cannot access validation queue (403)
- [ ] **Test:** Cannot access PA Staff personal pages (403)

### 7.2 Data Browsing

- [ ] **Test:** Can view observation list
- [ ] **Test:** Only validated observations visible
- [ ] **Test:** Can filter by species, date, location
- [ ] **Test:** Can view observation details
- [ ] **Test:** PA Staff names are anonymized
- [ ] **Test:** Cannot edit any observations

### 7.3 Species Explorer

- [ ] **Test:** Can search species by name
- [ ] **Test:** Can filter by conservation status
- [ ] **Test:** Species detail pages load
- [ ] **Test:** Taxonomy information displays
- [ ] **Test:** Observation list shows validated only
- [ ] **Test:** Media gallery displays images
- [ ] **Test:** Cannot edit species information directly

### 7.4 Map Access

- [ ] **Test:** Interactive map loads
- [ ] **Test:** Observation markers display
- [ ] **Test:** Can filter map observations
- [ ] **Test:** Sensitive species locations obscured
- [ ] **Test:** Can export visible observations
- [ ] **Test:** Protected area boundaries visible

### 7.5 Data Export

- [ ] **Test:** Can access data export center
- [ ] **Test:** Can export observations to CSV
- [ ] **Test:** Export contains only validated data
- [ ] **Test:** PA Staff names anonymized in export
- [ ] **Test:** Data use agreement must be accepted
- [ ] **Test:** Rate limit enforced (max 5 exports/day)
- [ ] **Test:** Cannot export > 10,000 records at once

### 7.6 Reports

- [ ] **Test:** Can view report archive
- [ ] **Test:** Can download public reports
- [ ] **Test:** Cannot access restricted/draft reports
- [ ] **Test:** Can view report metadata
- [ ] **Test:** Cannot delete reports

### 7.7 Analytics

- [ ] **Test:** Can access analytics dashboard
- [ ] **Test:** Charts render correctly
- [ ] **Test:** Can customize date ranges
- [ ] **Test:** Can export chart data
- [ ] **Test:** Interactive features work

---

## 🛠️ 8. Troubleshooting

### Common Issues

#### Issue: "Access denied" errors

**Cause:** Trying to access restricted areas

**Solution:** Collaborators have read-only access. Validation queue, admin panels, and PA Staff personal pages are restricted.

#### Issue: Cannot find certain observations

**Causes:**
1. Observations not yet validated
2. Observations filtered out by privacy rules
3. Date/location filters too restrictive

**Solution:** Adjust filters, check date range, verify species name spelling

#### Issue: Export failing or incomplete

**Causes:**
1. Export too large (>10,000 records)
2. Rate limit reached (5 exports/day)
3. Network timeout

**Solution:**
- Apply more restrictive filters to reduce record count
- Wait for daily limit reset
- Try exporting in smaller batches

#### Issue: Map markers not showing

**Causes:**
1. No validated observations in view
2. JavaScript error
3. Privacy filter hiding sensitive species

**Solution:**
- Zoom to different area
- Check browser console for errors
- Verify filters aren't excluding all data

---

## 📚 9. Best Practices for Research Collaborators

### 9.1 Data Collection for Research

**Before Starting Analysis:**
1. Define research questions clearly
2. Determine required data scope (species, dates, locations)
3. Check data availability and quality
4. Document data access date and filters used

**Data Quality Considerations:**
- Review GPS accuracy for spatial analysis
- Check observation effort (not all areas equally surveyed)
- Consider seasonal biases
- Account for detection probability differences between species

### 9.2 Data Management

**Organizing Exports:**
```
research_project/
├── data/
│   ├── raw/
│   │   ├── observations_2026-01-06.csv
│   │   └── species_checklist_2026-01-06.csv
│   ├── processed/
│   │   └── cleaned_observations.csv
│   └── metadata/
│       └── export_parameters.txt
├── analysis/
└── outputs/
```

**Document Metadata:**
- Export date
- Filters applied
- Number of records
- Date range of data
- Geographic scope

### 9.3 Collaboration & Communication

**Sharing Findings:**
- Submit manuscript preprints to PRISM admins
- Present findings at DENR workshops
- Provide feedback on data quality
- Suggest improvements to system

**Requesting Additional Data:**
- Use formal data request form
- Specify research purpose and justification
- Provide ethics approval (if human subjects)
- Sign data sharing agreement

### 9.4 Ethical Research Conduct

**Guidelines:**
- Respect indigenous knowledge and local communities
- Do not disclose sensitive locations publicly
- Acknowledge local expertise (PA Staff)
- Share benefits of research with local stakeholders
- Follow ethical guidelines for biodiversity research

---

## 🔗 10. Related Documentation

- **Data Export:** [DATA_EXPORT_GUIDE.md](../features/DATA_EXPORT.md)
- **Species Management:** [SPECIES_MANAGEMENT.md](../features/SPECIES_MANAGEMENT.md)
- **GIS Features:** [GIS_FEATURES.md](../features/GIS_FEATURES.md)
- **Access Control:** [RBAC_DOCUMENTATION.md](../architecture/RBAC_DOCUMENTATION.md)

---

## 📞 Support

**For Collaborators:**
- Data access requests: prism-data@denr.gov.ph
- Technical support: prism-support@denr.gov.ph
- Report issues or suggestions via feedback form
- Request training materials for data analysis

**For Developers:**
- Reference: [views_dashboard.py](dashboard/views_dashboard.py) (CollaboratorDashboardView)
- Permissions: [permissions.py](dashboard/permissions.py) (CollaboratorAccessMixin)
- Data export: [views_api.py](dashboard/views_api.py)

---

## 📋 Appendix: Data Use Agreement Template

**PRISM-Matutum Data Use Agreement**

I, [Name], affiliated with [Organization], agree to the following terms for accessing biodiversity data from the PRISM-Matutum system:

**Permitted Uses:**
- Academic research and publication
- Conservation planning and assessment
- Educational purposes
- Policy development

**Restrictions:**
- Data will not be redistributed publicly
- Sensitive species locations will not be disclosed
- Data will not be used for commercial purposes
- Individual PA Staff will not be identified

**Attribution:**
- I will cite PRISM-Matutum as the data source
- I will acknowledge PA Staff contributors collectively
- I will share resulting publications with PRISM administrators

**Data Scope:**
- Date Range: [Range]
- Geographic Area: [Area]
- Species/Taxa: [Taxa]
- Number of Records: [Count]

Agreed: [Signature] [Date]

---

*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Capstone Documentation Project*
